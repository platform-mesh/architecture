# RFC 003: Runtime Lifecycle v2

| Status  | Draft      |
|---------|------------|
| Date    | 2026-03-02 |

## Motivation

The `golang-commons/controller/lifecycle` package has served us well, but has accumulated design debt:

1. **Dual-runtime maintenance burden** — Parallel implementations for `controller-runtime` and `multicluster-runtime` with almost identical logic and separate builder paths.
2. **Coarse flow control** — Subroutines can only succeed or fail-with-retry/fail-without-retry. No way to express "I succeeded but further subroutines should not run" or "I'm not ready yet but let others continue."
3. **OperatorError indirection** — A custom error type with `Retry()` and `Sentry()` booleans that's harder to reason about than an explicit result type.
4. **Implicit behavior** — Finalizer management, condition setting, spread scheduling, and status patching are all woven into a single 300+ line `Reconcile` function with complex branching.
5. **Limited observability** — No per-subroutine timing, no structured outcome logging, and condition reasons are too generic ("Processing", "Error").

The `github.com/platform-mesh/subroutines` module is a clean-room rewrite. It targets **multicluster-runtime only** and makes no attempt to be backward-compatible with the existing lifecycle package.

## Scope

### In Scope
- Subroutine-based reconciliation orchestration
- Explicit, typed flow control for subroutine outcomes
- Automatic finalizer management
- KCP initializer/terminator support
- Status condition management (opt-in)
- Status patching via subresource
- Reconciliation spreading/scheduling (opt-in)
- Builder API for ergonomic configuration
- Multicluster-runtime integration (first-class)

### Out of Scope (explicitly dropped)
- `controller-runtime` single-cluster support — use multicluster-runtime's single-cluster mode instead

## Design Principles

1. **One path, not two** — One base subroutine interface, opt-in phase interfaces, one result type, one processing loop.
2. **Explicit over implicit** — The result type tells the lifecycle *exactly* what to do. No boolean flags on error types.
3. **Composable, not configurable** — Prefer small, composable pieces over a builder with many flags.
4. **Results carry intent, errors are unexpected** — `Result` communicates the subroutine's intent: continue, stop, requeue timing. The Go `error` return signals unexpected failures (API timeouts, network issues). These are two separate concerns: `Result` drives the business outcome, `error` signals something went wrong. When a subroutine returns an error, the chain stops — the error is returned to controller-runtime so its rate limiter handles backoff.
5. **Standard `error`, no custom error interface** — The current lifecycle uses `OperatorError` (a custom interface with `Err() error`, `Retry() bool`, `Sentry() bool`) to bundle flow control and reporting hints into the error itself. This is unnecessary when `Result` carries flow control and `ErrorReporter` handles reporting. Runtime v2 uses plain `error` everywhere. Standard Go errors, `fmt.Errorf` wrapping, `errors.Is`/`errors.As` — nothing custom.
6. **Errors stop the chain** — Unexpected errors (Go `error` return) halt processing, matching Go convention and controller-runtime behavior. The lifecycle updates the failing subroutine's condition, calls the `ErrorReporter`, and returns the error to controller-runtime for rate-limited backoff. Subroutines that are "not ready" but want to let others continue use `Pending()` — a known state, not an unexpected failure.
7. **Subroutines don't persist directly** — Subroutines mutate the in-memory object (e.g., setting status fields, adding labels), but never call `client.Status().Update()` or `client.Update()` on the reconciled object. The lifecycle detects diffs and patches once after all subroutines have run — status and metadata always, spec only when opted in via `WithSpecPatch()`. This prevents partial writes, conflicting updates, and ensures patches reflect the complete reconciliation outcome. The cluster-scoped client is available via context (`subroutines.ClientFromContext(ctx)`) for looking up other resources.

---

## Core API

### Result Type

Every subroutine returns `(Result, error)`. `Result` carries the subroutine's intent (continue, stop, requeue timing). The `error` return signals unexpected failures — the standard Go convention.

```go
type Result struct {
    action  action         // what the lifecycle should do next (zero value = continue)
    requeue time.Duration  // when to reconcile again (zero = don't requeue from this subroutine)
    message string         // human-readable explanation (for conditions/logging)
}
```

The zero value `Result{}` means "success, continue to next subroutine" — the `action` iota is ordered so that the zero value (`continue`) is the default.

**`Result` is always error-free.** Unexpected errors are returned as the second return value from the subroutine method, following the standard Go `(value, error)` pattern — identical to how controller-runtime's `Reconcile` returns `(ctrl.Result, error)`:

```go
Process(ctx context.Context, instance client.Object) (Result, error)    // Processor (opt-in)
Finalize(ctx context.Context, instance client.Object) (Result, error)   // Finalizer (opt-in)
Initialize(ctx context.Context, instance client.Object) (Result, error) // Initializer (opt-in)
Terminate(ctx context.Context, instance client.Object) (Result, error)  // Terminator (opt-in)
```

When `error` is non-nil, the lifecycle logs the error, sets the subroutine's condition to False, calls the `ErrorReporter`, and **stops the chain** — matching standard Go and controller-runtime convention. The error is returned to controller-runtime so its rate limiter handles backoff. When `error` is non-nil, the `Result` is ignored.

Both `error` returns and explicit `StopWithRequeue()`/`Stop()` results halt the chain. The difference: `error` signals an unexpected failure (and triggers the `ErrorReporter`), while `StopWithRequeue`/`Stop` are intentional decisions. For subroutines that are "not ready" but don't need to block others, use `Pending()` — this continues the chain because it represents a known state, not a failure.

Fields are unexported. Construction happens through factory functions, which makes illegal states unrepresentable. For the common case, the **zero value is meaningful**:

```go
// The simplest subroutine — do work, return
func (s *MySubroutine) Process(ctx context.Context, instance client.Object) (Result, error) {
    s.doWork(ctx, instance)
    return Result{}, nil  // zero value = success, continue
}
```

`Result{}` is equivalent to `OK()` — succeeded, continue to next subroutine. This keeps the simple case simple: a subroutine that just does its work doesn't need to know about the result API at all.

For subroutines that need flow control, factory functions make intent explicit:

```go
// Success — return (Result, nil)
OK()                              // succeeded, continue (same as Result{})
OKWithRequeue(30 * time.Second)   // succeeded, but requeue after duration (e.g., polling)

// Not ready, but let others run — return (Result, nil)
Pending(d, message)               // not yet ready, continue chain, requeue after duration

// Halt chain with requeue — return (Result, nil) — resource is NOT ready
StopWithRequeue(d, message)       // halt chain, requeue after duration (transient blocker)

// Halt chain, no explicit requeue — return (Result, nil)
Stop(message)                     // halt chain, rely on default resync for next reconcile
```

Examples:

```go
// Simple subroutine — just do work
// → Subroutine condition: True/Complete
// → Ready condition: True (if all other subroutines also OK)
func (s *MySubroutine) Process(ctx context.Context, instance client.Object) (Result, error) {
    s.doWork(ctx, instance)
    return Result{}, nil
}

// Unexpected error — chain stops, requeue via CR rate limiter
// → Subroutine condition: False/Error ("context deadline exceeded")
// → Ready condition: False
// → ErrorReporter: called
// → Remaining subroutines: skipped
func (s *MySubroutine) Process(ctx context.Context, instance client.Object) (Result, error) {
    if err := s.callAPI(ctx); err != nil {
        return Result{}, err
    }
    return Result{}, nil
}

// Dependency not ready — stop chain, poll again in 30s
// → Subroutine condition: False/Stopped ("waiting for Store CRD to be ready")
// → Ready condition: False
// → Remaining subroutines: skipped
func (s *MySubroutine) Process(ctx context.Context, instance client.Object) (Result, error) {
    if !s.isStoreReady(ctx, instance) {
        return StopWithRequeue(30*time.Second, "waiting for Store CRD to be ready"), nil
    }
    return Result{}, nil
}

// Not ready yet, but other subroutines are independent — let them run
// → Subroutine condition: Unknown/Pending ("waiting for certificate to be issued")
// → Ready condition: False (not all subs complete)
// → Remaining subroutines: continue running
func (s *CertSubroutine) Process(ctx context.Context, instance client.Object) (Result, error) {
    if !s.isCertReady(ctx) {
        return Pending(30*time.Second, "waiting for certificate to be issued"), nil
    }
    return Result{}, nil
}

// Known failure — stop chain, no explicit requeue (default resync will re-trigger)
// → Subroutine condition: False/Stopped ("spec.endpoint is required")
// → Ready condition: False
// → Remaining subroutines: skipped
// → No explicit requeue (default resync will re-trigger)
func (s *MySubroutine) Process(ctx context.Context, instance client.Object) (Result, error) {
    if instance.Spec.Endpoint == "" {
        return Stop("spec.endpoint is required"), nil
    }
    return Result{}, nil
}
```

`StopWithRequeue`, `Stop`, and `error` returns all halt the chain. For subroutines that need to express "not ready, but let others continue," use `Pending()` — this is a known state (not a failure) and the chain continues.

After the subroutine chain completes (either all subroutines ran or the chain was halted), conditions are updated and status is patched. If a subroutine returned an error, the lifecycle **returns it to controller-runtime**. This lets controller-runtime's built-in rate limiter handle exponential backoff and ensures errors are visible in controller-runtime's own metrics and logging.

### Flow Control Summary

| Return | Continue? | Requeue? | ErrorReporter? | Condition |
|---|---|---|---|---|
| `Result{}, nil` / `OK(), nil` | yes | no | no | True |
| `OKWithRequeue(d), nil` | yes | after `d` | no | True |
| `Pending(d, message), nil` | yes | after `d` | no | **Unknown** |
| `StopWithRequeue(d, message), nil` | **no** | after `d` | no | **False** |
| `Stop(message), nil` | **no** | default resync | no | **False** |
| `Result{}, err` | **no** | CR rate limiter | **yes** | **False** |

**Key distinction:**
- `error` = unexpected failure → chain stops, ErrorReporter called, CR rate limiter handles backoff
- `StopWithRequeue` / `Stop` = intentional halt → chain stops, no ErrorReporter (this is a known business condition)
- `Pending` = known not-ready state → chain **continues**, subroutine signals it needs more time but doesn't block others

This matches Go convention (`if err != nil { return }`) and controller-runtime behavior where a non-nil error stops the reconciliation. Subroutines that are independent of each other use `Pending()` to express "not ready yet" without halting the chain.

**Error reporting** — `ErrorReporter` is called when a subroutine returns a non-nil `error`. Known business conditions (`StopWithRequeue`, `Stop`, `Pending`) are not reported — they are intentional decisions, not unexpected failures.

**Error propagation** — When a subroutine returns an error, the lifecycle updates conditions, patches status, and returns the error to controller-runtime. This lets controller-runtime's rate limiter handle exponential backoff. Infrastructure failures (status patch, finalizer patch) are joined into the same return error via `errors.Join`.

### Subroutine Interface

```go
// Subroutine is the base interface. Every subroutine implements this.
type Subroutine interface {
    GetName() string
}

// Processor is implemented by subroutines that perform forward-processing (reconciliation).
// The lifecycle calls Process() for non-deleting reconciliations when no Initializer applies.
type Processor interface {
    Process(ctx context.Context, instance client.Object) (Result, error)
}
```

The base interface carries only the name — the identity of the subroutine. All lifecycle phases — processing, finalization, initialization, termination — are opt-in via additional interfaces. The lifecycle detects them via type-switch, the same pattern for all.

A subroutine that only needs cleanup on deletion implements `Subroutine` + `Finalizer` — no need for a no-op `Process()`. A subroutine that only reconciles implements `Subroutine` + `Processor`. Most subroutines will implement both.

**Changes from current:**

- `Process` moves from core `Subroutine` interface to opt-in `Processor` interface — consistent with `Finalizer`, `Initializer`, `Terminator`.
- `Process` returns `(Result, error)` instead of `(ctrl.Result, OperatorError)`.
- `Finalize`, `Finalizers`, `Initialize`, `Terminate` are no longer on the core interface — they move to opt-in interfaces (see below).
- Uses `client.Object` directly instead of today's custom `RuntimeObject` type.

#### Optional Subroutine Interfaces

All lifecycle phases are opt-in. The lifecycle detects them via type-switch — same mechanism for all:

```go
// Finalizer is implemented by subroutines that need cleanup on deletion.
// Subroutines implementing this interface have Finalize() called during deletion
// (in reverse order), and their declared finalizers are automatically managed.
type Finalizer interface {
    Finalize(ctx context.Context, instance client.Object) (Result, error)
    Finalizers(instance client.Object) []string
}

// Initializer is implemented by subroutines that participate in KCP workspace initialization.
// When the lifecycle is configured with an initializer name, subroutines implementing this
// interface have Initialize() called instead of Process() during the initialization phase.
type Initializer interface {
    Initialize(ctx context.Context, instance client.Object) (Result, error)
}

// Terminator is implemented by subroutines that participate in KCP workspace termination.
// When the lifecycle is configured with a terminator name and the object is being deleted,
// subroutines implementing this interface have Terminate() called. This covers KCP workspace
// deletion where objects are removed without honoring finalizers.
type Terminator interface {
    Terminate(ctx context.Context, instance client.Object) (Result, error)
}
```

A subroutine composes the interfaces it needs:

```go
// Reconcile only — no cleanup on deletion
type ConfigMapSubroutine struct{}
var _ subroutines.Subroutine = &ConfigMapSubroutine{}
var _ subroutines.Processor  = &ConfigMapSubroutine{}

// Reconcile + cleanup on deletion
type StoreSubroutine struct{}
var _ subroutines.Subroutine = &StoreSubroutine{}
var _ subroutines.Processor  = &StoreSubroutine{}
var _ subroutines.Finalizer  = &StoreSubroutine{}

// Reconcile + cleanup + KCP initialization
type WorkspaceSubroutine struct{}
var _ subroutines.Subroutine   = &WorkspaceSubroutine{}
var _ subroutines.Processor    = &WorkspaceSubroutine{}
var _ subroutines.Finalizer    = &WorkspaceSubroutine{}
var _ subroutines.Initializer  = &WorkspaceSubroutine{}

// Cleanup only — no forward-processing needed
type CleanupOnlySubroutine struct{}
var _ subroutines.Subroutine = &CleanupOnlySubroutine{}
var _ subroutines.Finalizer  = &CleanupOnlySubroutine{}
```

The lifecycle is configured with the initializer/terminator name when KCP support is needed:

```go
lc := lifecycle.New(mgr, sub1, sub2, sub3).
    WithInitializer("my-initializer").
    WithTerminator("my-terminator")
```

**How it works:**

- **Finalizer**: On first reconcile, the lifecycle adds finalizers from all subroutines that implement `Finalizer`. On deletion, `Finalize()` is called in reverse order; finalizers are removed after successful finalization. Subroutines that don't implement `Finalizer` are skipped during deletion.
- **Initializer**: When configured and the object is not being deleted, subroutines implementing `Initializer` have `Initialize()` called instead of `Process()`. On successful reconciliation (no requeue), the lifecycle removes the initializer name from the object's status.
- **Terminator**: When configured and the object is being deleted, subroutines implementing `Terminator` have `Terminate()` called. In KCP, workspace deletion removes objects without honoring finalizers — `Terminate()` gives subroutines a chance to clean up in that scenario. On successful reconciliation, the lifecycle removes the terminator from the object's status.

**Changes from current:**

- `Finalize`/`Finalizers` move from core interface to opt-in `Finalizer` interface — consistent with `Initializer`/`Terminator`
- All optional interfaces use the same type-switch detection pattern
- All return `(Result, error)` instead of `(ctrl.Result, OperatorError)`
- Initializer/Terminator configured via `WithInitializer`/`WithTerminator` on the lifecycle (same as today's builder)

### Lifecycle Reconciler

The lifecycle reconciler is the core orchestration engine. It is **not** a full `reconcile.Reconciler` — it's a function you call from your reconciler:

```go
package lifecycle

type Lifecycle struct {
    subroutines []subroutines.Subroutine
    options     Options
}

type Options struct {
    // Condition management (nil = disabled)
    Conditions ConditionManager
    // Spread scheduling (nil = disabled)
    Spread SpreadManager
    // ReadOnly mode — don't patch status or finalizers
    ReadOnly bool
    // Hook called before subroutines run, for context enrichment
    PrepareContext func(ctx context.Context, instance client.Object) (context.Context, error)
}

// Reconcile fetches the object, runs the subroutine chain, and patches status.
// Standard controller-runtime reconciler signature — the lifecycle handles everything.
func (l *Lifecycle) Reconcile(ctx context.Context, req reconcile.Request) (ctrl.Result, error)
```

The lifecycle is constructed with the manager, so it can resolve the cluster-scoped client internally:

```go
lc := lifecycle.New(mgr, sub1, sub2, sub3).
    WithConditions().
    WithSpread()
```

**Key design decisions:**

1. **Object fetching is internal** — The lifecycle calls `client.Get()` with the request's NamespacedName, handles NotFound (returns `ctrl.Result{}, nil`), and passes the object to subroutines. Same as today — no boilerplate for callers.

2. **Client resolution is internal** — The lifecycle holds the manager and resolves the cluster-scoped client from the context automatically.

3. **Status patching is internal** — The lifecycle handles `client.Status().Patch()` calls internally when conditions or spread state change. The caller doesn't need to worry about it.

4. **Metadata patching is automatic, spec patching is opt-in** — The lifecycle diffs the object against the initial snapshot and patches metadata (labels, annotations) if changed — this is standard operator behavior. Spec changes are only patched when enabled via `WithSpecPatch()`, since controllers normally shouldn't mutate the user's desired state. No diff, no API call.

### Reconciliation Flow

```
┌─ Caller (your reconciler) ─────────────────────┐
│ 1. Call lifecycle.Reconcile(ctx, req)           │
└─────────────────────────────────────────────────┘
        │
        ▼
┌─ Lifecycle.Reconcile ──────────────────────────────────────┐
│                                                            │
│  1. Resolve cluster-scoped client from context             │
│                                                            │
│  2. Fetch object via client.Get(req.NamespacedName)        │
│     └─ If NotFound: return (no-op)                         │
│                                                            │
│  3. Snapshot original object (for status diff)             │
│                                                            │
│  4. If spread enabled: check if reconcile is needed        │
│     └─ If not: return cached requeue time                  │
│                                                            │
│  5. If not deleting: add finalizers                        │
│                                                            │
│  6. If PrepareContext set: enrich context                   │
│                                                            │
│  7. If conditions enabled: init unknown conditions         │
│                                                            │
│  8. Run subroutines (reversed order when deleting):        │
│     ┌──────────────────────────────────────────┐           │
│     │ For each subroutine:                     │           │
│     │   - Set condition to Unknown             │           │
│     │   - Determine method to call:            │           │
│     │     - Deleting + Terminator impl?        │           │
│     │       → Terminate()                      │           │
│     │     - Deleting + Finalizer impl?          │           │
│     │       → Finalize(), then remove finalizer│           │
│     │     - Deleting + no Finalizer impl?       │           │
│     │       → skip (nothing to clean up)       │           │
│     │     - Not deleting + Initializer impl?   │           │
│     │       → Initialize()                     │           │
│     │     - Not deleting + Processor impl?     │           │
│     │       → Process()                        │           │
│     │     - Otherwise → skip (no applicable    │           │
│     │       phase interface)                   │           │
│     │   - Update condition from result/error   │           │
│     │   - If error: call ErrorReporter,        │           │
│     │     update condition, break loop          │           │
│     │   - Track min requeue duration           │           │
│     │   - On StopWithRequeue/Stop: break loop  │           │
│     └──────────────────────────────────────────┘           │
│                                                            │
│  9. Set Ready condition based on aggregate outcome         │
│                                                            │
│ 10. If successful + terminator configured + deleting:      │
│     remove terminator from status                          │
│ 11. If successful + initializer configured + not deleting: │
│     remove initializer from status                         │
│                                                            │
│ 12. If spread enabled: update next reconcile time          │
│                                                            │
│ 13. Diff against snapshot:                                 │
│     - Patch metadata if labels/annotations changed         │
│     - Patch spec if WithSpecPatch enabled and changed      │
│                                                            │
│ 14. Patch status if changed (and not read-only)            │
│                                                            │
│ 15. Return (ctrl.Result, error)                             │
│      - Subroutine error (if any) for CR rate limiter        │
│      - Infrastructure errors joined via errors.Join         │
└────────────────────────────────────────────────────────────┘
```

### Condition Management

Conditions are opt-in. When enabled, the lifecycle manages:

- **Per-subroutine condition**: `{SubroutineName}_Ready` (or `{SubroutineName}_Finalize` during deletion)
- **Aggregate condition**: `Ready`

Condition reasons map directly to the result:

| Return | Subroutine Condition | Reason | Ready Condition | Message source |
|---|---|---|---|---|
| `OK(), nil` | True | `Complete` | True (if all subs complete) | static |
| `OKWithRequeue(d), nil` | True | `Complete` | True (if all subs complete) | static |
| `Pending(d, message), nil` | Unknown | `Pending` | False | `message` string |
| `StopWithRequeue(d, message), nil` | False | `Stopped` | False | `message` string |
| `Stop(message), nil` | False | `Stopped` | False | `message` string |
| `_, err` | False | `Error` | False | `err.Error()` |

The aggregate `Ready` condition is only True when **all** subroutines completed with `OK()` (including `OKWithRequeue`) and no subroutine returned `Pending`, `StopWithRequeue`, `Stop`, or `error`.

The `message` string from `Pending`/`StopWithRequeue`/`Stop` and `err.Error()` from error returns flow into the condition's `.message` field, giving operators clear visibility into exactly which subroutine is blocking readiness and why. The condition's `.reason` field (machine-readable, CamelCase) is set automatically from the result type (`Complete`, `Pending`, `Stopped`, `Error`).

The instance must implement:

```go
type ConditionAccessor interface {
    GetConditions() []metav1.Condition
    SetConditions([]metav1.Condition)
}
```

### Object Interface

No custom type. Subroutines operate on `client.Object` directly — the standard controller-runtime interface that combines `runtime.Object` and `metav1.Object`. No alias, no wrapper.

### Builder API

```go
lc := lifecycle.New(mgr,
    subroutine1,
    subroutine2,
    subroutine3,
).
    WithConditions().
    WithSpread().
    WithPrepareContext(myContextFunc)
```

The builder is on the `Lifecycle` struct itself — no separate `Builder` type.

### Error Reporting

In `golang-commons`, Sentry is deeply embedded: every `OperatorError` carries a `Sentry() bool` flag, the reconciliation loop checks it inline, and subroutines must decide at return time whether their error "deserves" Sentry. PR #160 propagates this via a `Sentry bool` field on `Result`. Both approaches couple subroutine business logic to a specific error reporting backend.

In runtime v2, **subroutines know nothing about error reporting**. The `Result` type has no Sentry field, no reporting flags. Instead, the `lifecycle` package defines an `ErrorReporter` interface that adopters implement to integrate their error reporting backend of choice (Sentry, OpenTelemetry, plain logging, etc.):

```go
lc := lifecycle.New(mgr, sub1, sub2, sub3).
    WithErrorReporter(reporter)
```

The `ErrorReporter` interface:

```go
// ErrorReporter receives unexpected errors that occurred during reconciliation.
// It is called only when a subroutine returns a non-nil error — NOT for Stop(),
// which represents a known business condition, not an unexpected error.
//
// The lifecycle package provides this interface only — no implementations are shipped.
// Adopters register their own implementation (e.g., Sentry, OTel, structured logging).
type ErrorReporter interface {
    Report(ctx context.Context, err error, info ErrorInfo)
}

type ErrorInfo struct {
    Subroutine string        // which subroutine returned the error
    Object     client.Object // the object being reconciled
    Action     string        // "process", "finalize", "initialize", or "terminate"
}
```

**What the package provides:** the interface and the call site (invoked when a subroutine returns a non-nil `error`). Nothing more.

**What adopters provide:** the implementation, including any filtering logic.

Example — a Sentry reporter that skips transient Kubernetes errors:

```go
type SentryReporter struct {
    hub *sentry.Hub
}

func (r *SentryReporter) Report(ctx context.Context, err error, info lifecycle.ErrorInfo) {
    // Skip transient errors that resolve on retry
    if apierrors.IsNotFound(err) || apierrors.IsConflict(err) {
        return
    }

    r.hub.WithScope(func(scope *sentry.Scope) {
        scope.SetTag("subroutine", info.Subroutine)
        scope.SetTag("action", info.Action)
        scope.SetTag("object", fmt.Sprintf("%s/%s",
            info.Object.GetNamespace(), info.Object.GetName()))
        r.hub.CaptureException(err)
    })
}

// Register one or more reporters — each WithErrorReporter call appends
lc := lifecycle.New(mgr, sub1, sub2, sub3).
    WithErrorReporter(&SentryReporter{hub: sentry.CurrentHub()}).
    WithErrorReporter(&SlackAlerter{webhook: slackURL})
```

Common filtering patterns:

- Skip specific error types (e.g., `apierrors.IsNotFound`, `apierrors.IsConflict`)
- Skip errors from specific subroutines
- Only report on generation change (object info is available via `ErrorInfo`)
- Rate-limit duplicate errors
- Chain multiple reporters

If no `ErrorReporter` is registered, the lifecycle simply logs errors — no silent swallowing.

### Rate Limiting

Rate limiting operates at two complementary levels:

**Controller-level** — A coarse safety net on the work queue. The lifecycle supports configuring a rate limiter that controls how fast items re-enter the queue:

```go
lc := lifecycle.New(mgr, sub1, sub2, sub3).
    WithRateLimiter(workqueue.DefaultTypedItemBasedRateLimiter[reconcile.Request]())
```

This is the same concept as today's `WithStaticThenExponentialRateLimiter()`, but accepts any `workqueue.TypedRateLimiter` — not locked to a single strategy.

**How they interact:**

- For successful results (`nil` error), the lifecycle tracks the minimum requeue duration across all subroutine results and returns it as `ctrl.Result{RequeueAfter: minDuration}`
- For error results, controller-runtime [ignores the `Result`](https://github.com/kubernetes-sigs/controller-runtime/blob/v0.23.1/pkg/reconcile/reconcile.go#L117) and uses its rate limiter for backoff
- The controller-level rate limiter governs the work queue independently — it applies regardless of subroutine-level timing

**What changes from current:**

| Aspect | golang-commons | runtime v2 |
|---|---|---|
| Controller-level limiter | `WithStaticThenExponentialRateLimiter()` only — no way to pass an arbitrary limiter | `WithRateLimiter()` accepts any `workqueue.TypedRateLimiter` |

### Observability: Tracing

The current lifecycle creates a single span per `Reconcile` call and one child span per subroutine. This is preserved and extended:

```
Reconcile (span)
├── PrepareContext (span, if configured)
├── AddFinalizers (span, if mutation needed)
├── Subroutine: StoreSubroutine (span)
│   ├── attributes: outcome=ok, requeue=0s
│   └── duration: 12ms
├── Subroutine: AuthModelSubroutine (span)
│   ├── attributes: outcome=error, error="context deadline exceeded"
│   └── duration: 2004ms
└── PatchStatus (span, if mutation needed)
```

Every subroutine span carries structured attributes:

| Attribute | Example |
|---|---|
| `subroutine.name` | `"StoreSubroutine"` |
| `subroutine.action` | `"process"` or `"finalize"` |
| `subroutine.outcome` | `"ok"`, `"pending"`, `"stop"`, `"error"` |
| `subroutine.requeue` | `"30s"` or `""` |
| `subroutine.error` | error message (if any) |

The tracer name is derived from the operator/controller config, same as today.

### Observability: Metrics

The current lifecycle has **no metrics**. Runtime v2 registers metrics on controller-runtime's Prometheus registry (`metrics.Registry`) — the same registry every operator already exposes via `/metrics`. Metrics are always on, no opt-in needed.

Controller-runtime already provides reconcile-level metrics (`controller_runtime_reconcile_total`, `controller_runtime_reconcile_time_seconds`). The lifecycle adds subroutine-level metrics that controller-runtime doesn't know about:

| Metric | Type | Labels | Description |
|---|---|---|---|
| `lifecycle_subroutine_duration_seconds` | Histogram | `controller`, `subroutine`, `action`, `outcome` | Per-subroutine duration |
| `lifecycle_subroutine_errors_total` | Counter | `controller`, `subroutine`, `outcome` | Subroutine error count by type |
| `lifecycle_subroutine_requeue_seconds` | Histogram | `controller`, `subroutine`, `outcome` | Requeue delay requested by subroutine |

Outcome labels: `ok`, `pending`, `stop`, `error`.

This gives operators dashboards for:
- Which subroutines are slow
- Which subroutines fail most often (and how — transient vs permanent)
- Overall reconciliation health per controller

### Observability: Logging

The lifecycle manages structured logging automatically. Before each subroutine call, the lifecycle creates a child logger enriched with contextual attributes and sets it on the context. Subroutines retrieve it via `log.FromContext(ctx)` — no setup needed.

**Automatic attributes per subroutine:**

| Attribute | Example |
|---|---|
| `reconcileID` | `"a1b2c3d4"` |
| `cluster` | `"2aqwnMHPbx2oAsssJgiVJYFkkUr"` |
| `controller` | `"StoreController"` |
| `subroutine` | `"AuthModelSubroutine"` |
| `action` | `"process"`, `"finalize"`, `"initialize"`, `"terminate"` |
| `object` | `"default/my-store"` (namespace/name) |
| `generation` | `3` |

**What the lifecycle logs:**

- Subroutine start/completion at Debug level
- Result outcomes (stop, pending) at Info level — these are state transitions
- Errors at Error level, with the full error message
- Status patch at Debug level (only when changed)

**What subroutines get for free:**

Every `log.FromContext(ctx)` call inside a subroutine already carries the base attributes. A subroutine that logs `log.Info("created configmap")` produces:

```
INFO  created configmap  controller=StoreController subroutine=ConfigMapSubroutine action=process object=default/my-store generation=3
```

No manual attribute setup in subroutines — the lifecycle handles it.

### Condition Management (preserved)

Condition management is one of the strongest features of the current lifecycle and is fully preserved. The lifecycle automatically manages Kubernetes status conditions so that operators and external tools can observe reconciliation progress without reading logs.

**What it provides:**

1. **Per-subroutine conditions** — Each subroutine gets a `{Name}_Ready` (or `{Name}_Finalize`) condition that reflects its last outcome. This makes it trivial to see which subroutine is blocking readiness.

2. **Aggregate Ready condition** — A top-level `Ready` condition that summarizes the overall state. Only `True` when all subroutines completed with `OK()` or `OKWithRequeue()` — no `Pending`, `StopWithRequeue`, `Stop`, or errors.

3. **Generation-aware** — Conditions are updated with the observed generation, so tools can distinguish "ready at gen 3" from "ready at gen 5."

4. **Reason/message from results** — The `message` string from `Pending(d, message)`, `StopWithRequeue(d, message)`, and `Stop(message)` flows into the condition's `.message` field, and `err.Error()` from error returns does the same. The condition's `.reason` (machine-readable) is set automatically from the result type. This gives operators clear visibility (e.g., message `"spec.endpoint is required"` with reason `Stopped` instead of a generic `"Error"`).

5. **Opt-in** — Controllers that don't need conditions (e.g., simple cleanup controllers) pay no cost.

**What changes:**

- Condition reasons are richer, mapping directly to result types (`Complete`, `Pending`, `Stopped`, `Error`)
- The `ConditionAccessor` interface replaces `RuntimeObjectConditions` (same shape, shorter name)

### Automatic Finalizer Management (preserved)

The lifecycle handles the full finalizer lifecycle for subroutines that implement the `Finalizer` interface:

1. **On first reconcile** — Adds finalizers from all subroutines that implement `Finalizer` and declare them via `Finalizers(instance client.Object) []string`
2. **On deletion** — Calls `Finalize()` on each `Finalizer` subroutine in reverse order; removes the subroutine's finalizer after successful finalization. Subroutines that don't implement `Finalizer` are skipped.
3. **Requeue-aware** — If `Finalize()` returns a requeue result, the finalizer is kept (the subroutine needs more time)
4. **Read-only safe** — In read-only mode, no finalizer mutations happen

**What changes:**

- `Finalize`/`Finalizers` move from core `Subroutine` interface to opt-in `Finalizer` interface — consistent with `Processor`, `Initializer`, `Terminator`
- Finalizer patch uses `client.MergeFrom` (same as today)

### Reconciliation Spreading (preserved, opt-in)

Spreading distributes reconciliations over time to prevent thundering herd effects on large clusters where many resources exist but change infrequently.

**How it works:**

1. On successful reconcile, the lifecycle sets a `nextReconcileTime` in the object's status (randomized within a configurable window)
2. On next trigger (e.g., from a periodic re-list), the lifecycle checks: has the generation changed? Is the refresh label set? Is it past `nextReconcileTime`?
3. If none of these: skip reconciliation and return a requeue for the remaining time

**What changes:**

- Same core logic, same `RuntimeObjectSpreadReconcileStatus` interface requirement
- Cleaner integration point (checked before subroutine loop, same as today)

---

## Key Changes from golang-commons Lifecycle

| Aspect | golang-commons | runtime v2 |
|---|---|---|
| **Runtime support** | controller-runtime + multicluster-runtime | multicluster-runtime only |
| **Subroutine result** | `(ctrl.Result, OperatorError)` | `(Result, error)` — mirrors Go/controller-runtime convention |
| **Flow control** | error = stop, no error = continue | `Result` for intent (OK/Pending/StopWithRequeue/Stop), `error` for unexpected failures — both stop the chain |
| **Error semantics** | `error` return stops chain + requeues | `error` stops chain + requeues (same); richer Result vocabulary for non-error flow control |
| **Error type** | `OperatorError` custom interface | Standard Go `error`; flow control via `Result`, not error type |
| **Error reporting** | Sentry baked into reconcile loop | `ErrorReporter` interface only; no implementation shipped |
| **Object fetching** | Lifecycle fetches via `client.Get` | Same — lifecycle fetches internally |
| **Client resolution** | Lifecycle resolves from manager | Same — lifecycle resolves from manager |
| **Condition reasons** | Generic (Processing/Complete/Error) | Specific (maps to result type) |
| **Processing** | `Process` on core interface | Opt-in `Processor` interface |
| **Finalization** | `Finalize`/`Finalizers` on core interface | Opt-in `Finalizer` interface |
| **KCP init/terminate** | Built-in optional interfaces | Same pattern, returns `(Result, error)` |
| **Sentry** | Built into reconcile loop, per-error `Sentry()` flag | No Sentry dependency; adopters bring their own `ErrorReporter` |
| **Rate limiter** | Lifecycle-level, single strategy | Both controller-level and per-subroutine, any strategy |
| **Spread scheduling** | Built-in, checked first | Built-in, checked first (same) |

---

## Key Changes vs PR #160 (ChainSubroutine)

PR #160 adds `ChainSubroutine` as a parallel interface alongside the existing `Subroutine`. The runtime v2 takes a different approach:

| Aspect | PR #160 | runtime v2 |
|---|---|---|
| **Subroutine types** | `Subroutine` + `ChainSubroutine` + `BaseSubroutine` | `Subroutine` (base) + opt-in `Processor`, `Finalizer`, `Initializer`, `Terminator` |
| **Lifecycle types** | `Lifecycle` + `ChainLifecycle` + `LifecycleCore` | Single `Lifecycle` |
| **Backward compat** | Maintains, via type-switch | None (clean break) |
| **Outcomes** | 6 (Continue, StopChain, Skipped, ErrorRetry, ErrorContinue, ErrorStop) | 4 Result actions (OK, Pending, StopWithRequeue, Stop) + Go `error` for unexpected failures |
| **Error propagation** | First error stops chain + returns immediately | Error stops chain; status patched before returning error to CR |
| **Error behavior** | ErrorContinue/ErrorStop split | Errors stop the chain; `Pending()` handles the "continue despite not-ready" case |
| **Skipped** | Explicit outcome | Not needed — return `OK()` |
| **Condition manager** | `ConditionManager` | Single `ConditionManager` |
| **Result construction** | Exported fields, multiple constructors | Unexported fields, constrained constructors |

The fundamental issue with PR #160 is that it creates a parallel API surface to maintain backward compatibility. Since we're in a new repo, we don't need that. One base interface, opt-in phase interfaces, one result type, one processing loop.

**Additional differences in Result design:**

- No `Skipped` outcome. A subroutine that has nothing to do returns `Result{}, nil` — skip is not a meaningful state for the lifecycle to act on differently from success.
- No `ErrorContinue` / `ErrorStop` distinction. Unexpected errors (Go `error` return) stop the chain — matching Go convention and controller-runtime behavior. Subroutines that need "continue despite not-ready" use `Pending()`, which is a known state, not an error.
- No `ErrorRetry` / `ErrorStop` in `Result`. Unexpected errors use Go's standard `error` return. `Stop(message)` is a known business condition (string, not error). Clean separation.
- Fields are unexported to prevent construction of invalid states.
- Subroutine signature is `(Result, error)` — immediately familiar to any Go developer and controller-runtime user.

---

## Module and Package Structure

**Module:** `github.com/platform-mesh/subroutines`

```
subroutines/                         # package subroutines — what subroutine authors import
├── subroutine.go                    //   Subroutine, Processor, Finalizer, Initializer, Terminator interfaces
├── result.go                        //   Result type and factory functions (OK, Pending, StopWithRequeue, ...)
├── context.go                       //   ClientFromContext, logger enrichment utilities
├── lifecycle/                       # package lifecycle — what controller setup code imports
│   ├── lifecycle.go                 //   Lifecycle struct, Reconcile method, builder API (New, With*)
│   ├── errors.go                    //   ErrorReporter interface, ErrorInfo
│   └── options.go                   //   Options, PrepareContext, ReadOnly, WithSpecPatch
├── conditions/                      # package conditions — opt-in condition management
│   ├── manager.go                   //   ConditionManager implementation
│   └── accessor.go                  //   ConditionAccessor interface
├── spread/                          # package spread — opt-in reconciliation spreading
│   ├── manager.go                   //   SpreadManager implementation
│   └── status.go                    //   SpreadReconcileStatus interface
└── metrics/                         # package metrics — Prometheus metric registration
    └── metrics.go
```

**Import ergonomics:**

```go
import (
    "github.com/platform-mesh/subroutines"            // subroutines.Processor, subroutines.OK(), subroutines.Result
    "github.com/platform-mesh/subroutines/lifecycle"   // lifecycle.New(mgr, sub1, sub2)
    "github.com/platform-mesh/subroutines/conditions"  // conditions.ConditionAccessor (on CRD types)
)
```

The root package (`subroutines`) contains the interfaces and types that subroutine authors use daily — this is the primary API surface. The `lifecycle` subpackage contains the orchestration engine that controller setup code imports once. Opt-in features (`conditions`, `spread`, `metrics`) are sub-packages imported only when needed.
