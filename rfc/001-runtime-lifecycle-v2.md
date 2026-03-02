# RFC 001: Runtime Lifecycle v2

| Status  | Draft      |
|---------|------------|
| Date    | 2026-03-02 |

## Motivation

The `golang-commons/controller/lifecycle` package has served us well, but has accumulated design debt:

1. **Dual-runtime maintenance burden** — Parallel implementations for `controller-runtime` and `multicluster-runtime` with almost identical logic and separate builder paths.
2. **Coarse flow control** — Subroutines can only succeed or fail-with-retry/fail-without-retry. No way to express "I succeeded but further subroutines should not run," "I'm not ready yet but let others continue," or "I failed but it's non-critical, keep going."
3. **OperatorError indirection** — A custom error type with `Retry()` and `Sentry()` booleans that's harder to reason about than an explicit result type.
4. **Implicit behavior** — Finalizer management, condition setting, spread scheduling, and status patching are all woven into a single 300+ line `Reconcile` function with complex branching.
5. **Limited observability** — No per-subroutine timing, no structured outcome logging, and condition reasons are too generic ("Processing", "Error").

The `runtime` package is a clean-room rewrite. It targets **multicluster-runtime only** and makes no attempt to be backward-compatible with the existing lifecycle package.

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

1. **One path, not two** — One subroutine interface, one result type, one processing loop.
2. **Explicit over implicit** — The result type tells the lifecycle *exactly* what to do. No boolean flags on error types.
3. **Composable, not configurable** — Prefer small, composable pieces over a builder with many flags.
4. **Results carry intent, errors are unexpected** — `Result` communicates the subroutine's intent: continue, stop, fail, requeue timing. The Go `error` return signals unexpected failures (API timeouts, network issues). These are two separate concerns: `Result` drives the business outcome, `error` signals something went wrong. After all subroutines run and status is updated, collected errors are returned to controller-runtime so its rate limiter handles backoff.
5. **Standard `error`, no custom error interface** — The current lifecycle uses `OperatorError` (a custom interface with `Err() error`, `Retry() bool`, `Sentry() bool`) to bundle flow control and reporting hints into the error itself. This is unnecessary when `Result` carries flow control and `ErrorReporter` handles reporting. Runtime v2 uses plain `error` everywhere. Standard Go errors, `fmt.Errorf` wrapping, `errors.Is`/`errors.As` — nothing custom.
6. **Errors don't stop the chain** — Unexpected errors (Go `error` return) do not halt processing. The lifecycle continues to run all subroutines so that conditions reflect the full state. Only `StopWithRequeue` and `Fail` (explicit `Result` values) halt the chain — these are intentional decisions, not side effects of failure.
7. **Subroutines don't persist directly** — Subroutines mutate the in-memory object (e.g., setting status fields, adding labels), but never call `client.Status().Update()` or `client.Update()` on the reconciled object. The lifecycle detects diffs and patches once after all subroutines have run — status and metadata always, spec only when opted in via `WithSpecPatch()`. This prevents partial writes, conflicting updates, and ensures patches reflect the complete reconciliation outcome. The cluster-scoped client is available via context (`runtime.ClientFromContext(ctx)`) for looking up other resources.

---

## Core API

### Result Type

Every subroutine returns `(Result, error)`. `Result` carries the subroutine's intent (continue, stop, fail, requeue timing). The `error` return signals unexpected failures — the standard Go convention.

```go
type Result struct {
    action  action         // what the lifecycle should do next (zero value = continue)
    requeue time.Duration  // when to reconcile again (zero = don't requeue from this subroutine)
    reason  string         // human-readable explanation (for conditions/logging)
}
```

The zero value `Result{}` means "success, continue to next subroutine" — the `action` iota is ordered so that the zero value (`continue`) is the default.

**`Result` is always error-free.** Unexpected errors are returned as the second return value from the subroutine method, following the standard Go `(value, error)` pattern — identical to how controller-runtime's `Reconcile` returns `(ctrl.Result, error)`:

```go
Process(ctx context.Context, instance client.Object) (Result, error)    // Subroutine
Finalize(ctx context.Context, instance client.Object) (Result, error)   // Finalizer (opt-in)
Initialize(ctx context.Context, instance client.Object) (Result, error) // Initializer (opt-in)
Terminate(ctx context.Context, instance client.Object) (Result, error)  // Terminator (opt-in)
```

When `error` is non-nil, the lifecycle logs the error, sets the subroutine's condition to False, calls the `ErrorReporter`, and **continues to the next subroutine**. Unexpected errors do not stop the chain — the lifecycle runs all subroutines and requeues at the end if any errors occurred. This ensures conditions reflect the full state of all subroutines, not just the first failure.

To explicitly stop the chain, a subroutine returns `StopWithRequeue()` or `Fail()` as the `Result` — these are intentional decisions, not unexpected failures. The only field from `Result` the lifecycle uses when `error` is non-nil is `requeue` — allowing the subroutine to control retry timing (e.g., via a rate limiter).

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
Pending(d, reason)                // not yet ready, continue chain, requeue after duration

// Halt chain with requeue — return (Result, nil) — resource is NOT ready
StopWithRequeue(d, reason)        // halt chain, requeue after duration (transient blocker)

// Halt chain permanently — return (Result, nil) — resource will NOT become ready
Fail(reason)                      // halt chain, do NOT requeue (user intervention needed)

// Unexpected errors — return (Result, error)
// Just return the error. Optionally set requeue timing for custom retry delay.
RequeueAfter(d)                   // helper: Result with only requeue set (use with error return)
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

// Unexpected error — chain continues, requeue at end
// → Subroutine condition: False/Error ("context deadline exceeded")
// → Ready condition: False
// → ErrorReporter: called
func (s *MySubroutine) Process(ctx context.Context, instance client.Object) (Result, error) {
    if err := s.callAPI(ctx); err != nil {
        return Result{}, err
    }
    return Result{}, nil
}

// Unexpected error with custom retry timing (e.g., rate limiter)
// → Subroutine condition: False/Error ("rate limited")
// → Ready condition: False
// → ErrorReporter: called
func (s *MySubroutine) Process(ctx context.Context, instance client.Object) (Result, error) {
    if err := s.callAPI(ctx); err != nil {
        delay := s.rateLimiter.When(instance.GetName())
        return RequeueAfter(delay), err
    }
    s.rateLimiter.Forget(instance.GetName())
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

// Known permanent failure — stop chain, no requeue
// → Subroutine condition: False/Failed ("spec.endpoint is required")
// → Ready condition: False
// → Remaining subroutines: skipped
// → No requeue (user must fix the spec)
func (s *MySubroutine) Process(ctx context.Context, instance client.Object) (Result, error) {
    if instance.Spec.Endpoint == "" {
        return Fail("spec.endpoint is required"), nil
    }
    return Result{}, nil
}
```

Only `StopWithRequeue` and `Fail` halt the chain — these are intentional decisions by the subroutine. Unexpected errors (`error` return) do not stop the chain; the lifecycle continues to give all subroutines a chance to run.

After all subroutines have run, conditions are updated, ErrorReporters called, and status patched, the lifecycle **returns collected subroutine errors to controller-runtime**. This lets controller-runtime's built-in rate limiter handle exponential backoff and ensures errors are visible in controller-runtime's own metrics and logging. If multiple subroutines errored, errors are joined via `errors.Join`.

### Flow Control Summary

| Return | Continue? | Requeue? | ErrorReporter? | Condition |
|---|---|---|---|---|
| `Result{}, nil` / `OK(), nil` | yes | no | no | True |
| `OKWithRequeue(d), nil` | yes | after `d` | no | True |
| `Pending(d, reason), nil` | yes | after `d` | no | **Unknown** |
| `StopWithRequeue(d, reason), nil` | **no** | after `d` | no | **False** |
| `Fail(reason), nil` | **no** | **no** | no | **False** |
| `Result{}, err` | **yes** | error returned to CR | **yes** | **False** |
| `RequeueAfter(d), err` | **yes** | after `d` | **yes** | **False** |

**Key distinction:**
- `StopWithRequeue` / `Fail` = intentional chain halt (the subroutine knows further processing is pointless)
- `error` = unexpected failure, but the lifecycle continues — other subroutines may be independent and should still run

This means conditions always reflect the **full state** of all subroutines, not just the first failure. Operators see which subroutines are healthy and which are failing, in a single reconciliation.

`ErrorReporter` is only called for non-nil `error` returns. Known business conditions (`StopWithRequeue`, `Fail`) are not reported.

After all subroutines run, conditions are updated, and status is patched, `Lifecycle.Reconcile` returns the collected subroutine errors (via `errors.Join` if multiple) to controller-runtime. This lets controller-runtime apply its rate limiter for exponential backoff. Infrastructure failures (status patch, finalizer patch) are also returned as errors.

### Subroutine Interface

```go
// Subroutine is the core interface. Every subroutine implements this.
type Subroutine interface {
    GetName() string
    Process(ctx context.Context, instance client.Object) (Result, error)
}
```

The core interface is minimal: a name and the forward-processing method. Everything else — finalization, initialization, termination — is opt-in via additional interfaces. The lifecycle detects them via type-switch, the same pattern for all.

**Changes from current:**

- `Process` returns `(Result, error)` instead of `(ctrl.Result, OperatorError)`.
- `Finalize`, `Finalizers`, `Initialize`, `Terminate` are no longer on the core interface — they move to opt-in interfaces (see below). A subroutine that only reconciles implements `Subroutine` and nothing else.
- Uses `client.Object` directly instead of today's custom `RuntimeObject` type.

#### Optional Subroutine Interfaces

All lifecycle phases beyond `Process()` are opt-in. The lifecycle detects them via type-switch — same mechanism for all:

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
// When the lifecycle is configured with a terminator name, subroutines implementing this
// interface have Terminate() called instead of Finalize() during the termination phase
// (deletion without finalizer).
type Terminator interface {
    Terminate(ctx context.Context, instance client.Object) (Result, error)
}
```

A subroutine composes the interfaces it needs:

```go
// Reconcile only — no cleanup on deletion
type ConfigMapSubroutine struct{}
var _ runtime.Subroutine = &ConfigMapSubroutine{}

// Reconcile + cleanup on deletion
type StoreSubroutine struct{}
var _ runtime.Subroutine = &StoreSubroutine{}
var _ runtime.Finalizer = &StoreSubroutine{}

// Reconcile + cleanup + KCP initialization
type WorkspaceSubroutine struct{}
var _ runtime.Subroutine = &WorkspaceSubroutine{}
var _ runtime.Finalizer = &WorkspaceSubroutine{}
var _ runtime.Initializer = &WorkspaceSubroutine{}
```

The lifecycle is configured with the initializer/terminator name when KCP support is needed:

```go
lifecycle := runtime.NewLifecycle(mgr, sub1, sub2, sub3).
    WithInitializer("my-initializer").
    WithTerminator("my-terminator")
```

**How it works:**

- **Finalizer**: On first reconcile, the lifecycle adds finalizers from all subroutines that implement `Finalizer`. On deletion, `Finalize()` is called in reverse order; finalizers are removed after successful finalization. Subroutines that don't implement `Finalizer` are skipped during deletion.
- **Initializer**: When configured and the object is not being deleted, subroutines implementing `Initializer` have `Initialize()` called instead of `Process()`. On successful reconciliation (no requeue), the lifecycle removes the initializer name from the object's status.
- **Terminator**: When configured and the object is being deleted, subroutines implementing `Terminator` have `Terminate()` called instead of `Finalize()`. This handles the KCP pattern where cleanup happens without finalizers. On successful reconciliation, the lifecycle removes the terminator from the object's status.

**Changes from current:**

- `Finalize`/`Finalizers` move from core interface to opt-in `Finalizer` interface — consistent with `Initializer`/`Terminator`
- All optional interfaces use the same type-switch detection pattern
- All return `(Result, error)` instead of `(ctrl.Result, OperatorError)`
- Initializer/Terminator configured via `WithInitializer`/`WithTerminator` on the lifecycle (same as today's builder)

### Lifecycle Reconciler

The lifecycle reconciler is the core orchestration engine. It is **not** a full `reconcile.Reconciler` — it's a function you call from your reconciler:

```go
type Lifecycle struct {
    subroutines []Subroutine
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
lifecycle := runtime.NewLifecycle(mgr, sub1, sub2, sub3).
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
│     │     - Otherwise → Process()              │           │
│     │   - Update condition from result/error   │           │
│     │   - If error: call ErrorReporter,        │           │
│     │     track for requeue, continue           │           │
│     │   - Track min requeue duration           │           │
│     │   - On StopWithRequeue/Fail: break loop   │           │
│     │     (intentional halt only)              │           │
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
│      - Collected subroutine errors via errors.Join          │
│      - Controller-runtime rate limiter handles backoff      │
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
| `Pending(d, reason), nil` | Unknown | `Pending` | False | `reason` string |
| `StopWithRequeue(d, reason), nil` | False | `Stopped` | False | `reason` string |
| `Fail(reason), nil` | False | `Failed` | False | `reason` string |
| `_, err` | False | `Error` | False | `err.Error()` |

The aggregate `Ready` condition is only True when **all** subroutines completed with `OK()` (including `OKWithRequeue`) and no subroutine returned `Pending`, `StopWithRequeue`, `Fail`, or `error`.

The `reason` string from `Pending`/`StopWithRequeue`/`Fail` and `err.Error()` from error returns flow into the subroutine condition message, giving operators clear visibility into exactly which subroutine is blocking readiness and why.

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
lifecycle := runtime.NewLifecycle(mgr,
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

In runtime v2, **subroutines know nothing about error reporting**. The `Result` type has no Sentry field, no reporting flags. Instead, the package defines an `ErrorReporter` interface that adopters implement to integrate their error reporting backend of choice (Sentry, OpenTelemetry, plain logging, etc.):

```go
lifecycle := runtime.NewLifecycle(mgr, sub1, sub2, sub3).
    WithErrorReporter(reporter)
```

The `ErrorReporter` interface:

```go
// ErrorReporter receives unexpected errors that occurred during reconciliation.
// It is called only when a subroutine returns a non-nil error — NOT for Fail(),
// which represents a known business condition, not an unexpected error.
//
// The runtime package provides this interface only — no implementations are shipped.
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

func (r *SentryReporter) Report(ctx context.Context, err error, info runtime.ErrorInfo) {
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
lifecycle := runtime.NewLifecycle(mgr, sub1, sub2, sub3).
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
lifecycle := runtime.NewLifecycle(mgr, sub1, sub2, sub3).
    WithRateLimiter(workqueue.DefaultTypedItemBasedRateLimiter[reconcile.Request]())
```

This is the same concept as today's `WithStaticThenExponentialRateLimiter()`, but accepts any `workqueue.TypedRateLimiter` — not locked to a single strategy.

**Per-subroutine** — Fine-grained backoff within a reconciliation. Each subroutine can manage its own rate limiter and communicate retry timing via `RequeueAfter`:

```go
func (s *MySubroutine) Process(ctx context.Context, instance client.Object) (Result, error) {
    if err := s.doWork(ctx, instance); err != nil {
        // The subroutine decides its own backoff
        delay := s.rateLimiter.When(instance.GetName())
        return RequeueAfter(delay), err
    }
    s.rateLimiter.Forget(instance.GetName())
    return Result{}, nil
}
```

A subroutine can use any rate limiter implementation — `workqueue.DefaultTypedItemBasedRateLimiter`, a custom exponential backoff, a static delay, or no rate limiter at all (just return a fixed duration).

**How they interact:**

- The lifecycle tracks the minimum `RequeueAfter` across all subroutine results and returns it as `ctrl.Result{RequeueAfter: minDuration}`
- The controller-level rate limiter governs the work queue independently — it applies regardless of subroutine-level timing
- Both levels are optional and independent

**What changes from current:**

| Aspect | golang-commons | runtime v2 |
|---|---|---|
| Controller-level limiter | `WithStaticThenExponentialRateLimiter()` (fixed strategy) | `WithRateLimiter()` (any `workqueue.TypedRateLimiter`) |
| Per-subroutine limiter | Possible but not part of the lifecycle API | First-class via `RequeueAfter` in `Result` |
| Backoff strategy | Fixed (static-then-exponential) | Any — at both levels |

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
| `subroutine.outcome` | `"ok"`, `"pending"`, `"stop"`, `"fail"`, `"error"` |
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

Outcome labels: `ok`, `pending`, `stop`, `fail`, `error`.

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
- Result outcomes (stop, fail, pending) at Info level — these are state transitions
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

2. **Aggregate Ready condition** — A top-level `Ready` condition that summarizes the overall state. Only `True` when all subroutines completed with `OK()` or `OKWithRequeue()` — no `Pending`, `StopWithRequeue`, `Fail`, or errors.

3. **Generation-aware** — Conditions are updated with the observed generation, so tools can distinguish "ready at gen 3" from "ready at gen 5."

4. **Reason/message from results** — The `reason` string from `Pending(d, reason)`, `StopWithRequeue(d, reason)`, and `Fail(reason)` flows into the condition message, and `err.Error()` from error returns does the same, giving operators clear visibility (e.g., `"spec.endpoint is required"` instead of just `"Error"`).

5. **Opt-in** — Controllers that don't need conditions (e.g., simple cleanup controllers) pay no cost.

**What changes:**

- Condition reasons are richer, mapping directly to result types (`Complete`, `Pending`, `Stopped`, `Failed`, `Error`)
- The `ConditionAccessor` interface replaces `RuntimeObjectConditions` (same shape, shorter name)

### Automatic Finalizer Management (preserved)

The lifecycle handles the full finalizer lifecycle for subroutines that implement the `Finalizer` interface:

1. **On first reconcile** — Adds finalizers from all subroutines that implement `Finalizer` and declare them via `Finalizers(instance client.Object) []string`
2. **On deletion** — Calls `Finalize()` on each `Finalizer` subroutine in reverse order; removes the subroutine's finalizer after successful finalization. Subroutines that don't implement `Finalizer` are skipped.
3. **Requeue-aware** — If `Finalize()` returns a requeue result, the finalizer is kept (the subroutine needs more time)
4. **Read-only safe** — In read-only mode, no finalizer mutations happen

**What changes:**

- `Finalize`/`Finalizers` move from core `Subroutine` interface to opt-in `Finalizer` interface — consistent with `Initializer`/`Terminator`
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
| **Flow control** | error = stop, no error = continue | `Result` for intent (OK/Pending/StopWithRequeue/Fail), `error` for unexpected failures |
| **Error semantics** | `error` return stops chain + requeues | `error` continues chain; returned to CR after all subs run |
| **Error type** | `OperatorError` custom interface | Standard Go `error`; flow control via `Result`, not error type |
| **Error reporting** | Sentry baked into reconcile loop | `ErrorReporter` interface only; no implementation shipped |
| **Object fetching** | Lifecycle fetches via `client.Get` | Same — lifecycle fetches internally |
| **Client resolution** | Lifecycle resolves from manager | Same — lifecycle resolves from manager |
| **Condition reasons** | Generic (Processing/Complete/Error) | Specific (maps to result type) |
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
| **Subroutine types** | `Subroutine` + `ChainSubroutine` + `BaseSubroutine` | Single `Subroutine` |
| **Lifecycle types** | `Lifecycle` + `ChainLifecycle` + `LifecycleCore` | Single `Lifecycle` |
| **Backward compat** | Maintains, via type-switch | None (clean break) |
| **Outcomes** | 6 (Continue, StopChain, Skipped, ErrorRetry, ErrorContinue, ErrorStop) | 4 Result actions (OK, Pending, StopWithRequeue, Fail) + Go `error` for unexpected failures |
| **Error propagation** | First error stops chain + returns immediately | All subs run; errors collected and returned after status patch |
| **Error behavior** | ErrorContinue/ErrorStop split | Errors always continue; only `StopWithRequeue`/`Fail` halt the chain |
| **Skipped** | Explicit outcome | Not needed — return `OK()` |
| **Condition manager** | `ConditionManager` | Single `ConditionManager` |
| **Result construction** | Exported fields, multiple constructors | Unexported fields, constrained constructors |

The fundamental issue with PR #160 is that it creates a parallel API surface to maintain backward compatibility. Since we're in a new repo, we don't need that. One interface, one result type, one processing loop.

**Additional differences in Result design:**

- No `Skipped` outcome. A subroutine that has nothing to do returns `Result{}, nil` — skip is not a meaningful state for the lifecycle to act on differently from success.
- No `ErrorContinue` / `ErrorStop` distinction. Unexpected errors (Go `error` return) always continue the chain — the lifecycle runs all subroutines. To explicitly stop, use `StopWithRequeue()` or `Fail()`. This is simpler and gives better observability (all conditions updated in one pass).
- No `ErrorRetry` / `ErrorStop` in `Result`. Unexpected errors use Go's standard `error` return. `Fail(reason)` is a known business condition (string, not error). Clean separation.
- Fields are unexported to prevent construction of invalid states.
- Subroutine signature is `(Result, error)` — immediately familiar to any Go developer and controller-runtime user.

---

## Package Structure

```
runtime/
├── lifecycle.go           // Lifecycle struct, Reconcile, builder
├── result.go              // Result type and constructors
├── subroutine.go          // Subroutine, Finalizer, Initializer, Terminator interfaces
├── context.go             // ClientFromContext, logger setup
├── errors.go              // ErrorReporter interface, ErrorInfo
├── conditions/
│   ├── manager.go         // ConditionManager implementation
│   └── accessor.go        // ConditionAccessor interface
├── spread/
│   ├── manager.go         // SpreadManager implementation
│   └── status.go          // SpreadReconcileStatus interface
└── metrics/
    └── metrics.go         // Prometheus metric registration
```

The core package (`runtime/`) contains the lifecycle, result type, and subroutine interfaces — the API surface every user touches. Opt-in features with their own interfaces and implementation logic get sub-packages. Users configure everything through the builder on `Lifecycle`; sub-package imports are only needed when implementing feature interfaces (e.g., `conditions.ConditionAccessor` on a CRD type).
