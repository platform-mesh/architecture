# ADR 006: GraphQL Type Naming Convention with Group-Version Prefix and Input Suffix Separator

| Status  | Proposed   |
|---------|------------|
| Date    | 2026-05-08 |

## Context and Problem Statement

The kubernetes-graphql-gateway generates GraphQL type names from Kubernetes resource kinds by concatenating parent type names with capitalized field paths. This naming convention can produce collisions. For example, a CRD `Component` with a field `dataInput` generates `ComponentDataInput`, which is indistinguishable from the input type for a `ComponentData` resource. When a collision occurs, the gateway fails at startup with a schema uniqueness error and cannot serve any requests.

These collisions are triggered by valid Kubernetes resource definitions that conform to standard API conventions. Consumers with affected CRDs are entirely blocked from using the gateway, not because their resources are misconfigured, but because the gateway's internal naming convention is incompatible with what Kubernetes allows. As the number of registered CRDs grows, the flat PascalCase namespace makes further collisions increasingly likely.

## Decision

Two naming changes to the generated GraphQL schema:

1. **Type names** are prefixed with API group and version: `ConfigMap` → `V1ConfigMap`, `Deployment` → `AppsV1Deployment`.
2. **Input type suffix** changes from `Input` to `_Input`: `ConfigMapInput` → `V1ConfigMap_Input`. The underscore separator prevents collisions when a CRD field path produces a name indistinguishable from an input type. For example, a CRD `Component` with a field `dataInput` would generate `ComponentDataInput`, which is ambiguous with the input type for a `ComponentData` type. With the separator, the input type becomes `V1ComponentData_Input`, making the distinction explicit.

### Prefix Construction

The prefix is derived from the Kubernetes API group and version. Core API resources (group `""`) use only the version. CRDs use the full group converted to PascalCase (dots and hyphens are word boundaries) followed by the version.

| Kind | Group | Version | GraphQL Type Name |
|------|-------|---------|-------------------|
| ConfigMap | *(core)* | v1 | `V1ConfigMap` |
| Deployment | apps | v1 | `AppsV1Deployment` |
| Certificate | cert-manager.io | v1 | `CertManagerIoV1Certificate` |
| Order | acme.cert-manager.io | v1 | `AcmeCertManagerIoV1Order` |
| IstioOperator | install.istio.io | v1alpha1 | `InstallIstioIoV1alpha1IstioOperator` |
| Component | acme.example.com | v1beta2 | `AcmeExampleComV1beta2Component` |

Group-to-PascalCase conversion rules:

1. Split on `.` and `-` characters.
2. Capitalize the first letter of each segment.
3. Concatenate without separator.

For example, `acme.cert-manager.io` → split into `[acme, cert, manager, io]` → `AcmeCertManagerIo`.

The schema's field structure (query/mutation names, arguments, response shapes) is unchanged. The type names that back those fields have been renamed.

This is a **breaking change**.

### Downstream Impact

Any consumer that references GraphQL type names explicitly is affected. This includes:

* Mutation variables that declare input types (e.g., `mutation($input: ConfigMapInput!)` → `mutation($input: V1ConfigMap_Input!)`)
* Named fragments and inline fragment type conditions (`... on TypeName`)
* Code-generated type bindings from schema introspection
* Dynamic type name construction at runtime

Consumers that only use query/mutation field names without referencing type names directly are unaffected.

Known high-impact consumers:

| Consumer | Required Change |
|----------|-----------------|
| platform-mesh/portal-ui-lib | `resource.service.ts` constructs input types as `` `${entity}Input!` `` and needs the versioned name + `_Input` suffix |
| platform-mesh/generic-resource-ui | `buildVersionedTypeName()` and mutation variable construction need full versioned prefix + `_Input` |
| platform-mesh/marketplace-ui | Schema regeneration + one input type reference |

## Considered Options

### Option A: Unconditional Group-Version Prefix (chosen)

Every type is always prefixed with the full API group and version, regardless of whether a collision exists.

* Good, because type names are deterministic — derivable from the Kubernetes resource alone without knowledge of other registered CRDs.
* Good, because the schema is stable over time — adding or removing CRDs never renames existing types.
* Good, because all gateway instances produce identical type names for the same resource, independent of which CRDs are registered.
* Good, because tooling (code generators, documentation, IDE completions) can rely on a single formula.
* Bad, because type names are longer and more verbose, even when no collision exists (e.g., `V1ConfigMap` instead of `ConfigMap`).
* Bad, because it is a breaking change for all consumers that reference type names explicitly.

### Option B: Conditional Prefixing (rejected)

Apply prefixes only when the gateway detects a naming collision at startup. Non-colliding types keep their short names (e.g., `ConfigMap` stays `ConfigMap`).

* Good, because type names remain short and readable when no collision exists.
* Good, because existing consumers are unaffected until an actual collision occurs.
* Bad, because consumers must understand the gateway's internal collision-detection logic to predict whether a given type will be prefixed — there is no way to derive the type name from the Kubernetes resource alone.
* Bad, because the schema is unstable over time — adding a new CRD that collides with an existing short name forces the existing type to gain a prefix, breaking consumers that reference it.
* Bad, because two gateway instances serving different CRD sets produce different type names for the same resource, making shared tooling and documentation unreliable.
* Bad, because code generators and documentation tools must handle conditional branches rather than a single deterministic formula.

In a more static Kubernetes environment with a fixed, rarely-changing set of CRDs, conditional prefixing could reduce verbosity. For the platform-mesh gateway — where tenants register CRDs dynamically and the set of types grows continuously — deterministic unconditional prefixing is the safer choice.

### Consequences

* Good, because type collisions are structurally impossible since group+version is unique per resource.
* Good, because the schema can safely grow with arbitrary CRDs without risk of name clashes.
* Good, because query/mutation field names are stable so most consumers using only field-level access are unaffected.
* Bad, because consumers using named fragments, inline fragment type conditions, or dynamically constructed type names must migrate.
* Bad, because type names become longer and slightly less readable in introspection output.

Type names are coupled to Kubernetes API versioning. When a resource graduates between API versions (e.g., `v1beta1` → `v1`), its GraphQL type names change accordingly.

## References

* [platform-mesh/kubernetes-graphql-gateway#227](https://github.com/platform-mesh/kubernetes-graphql-gateway/pull/227)
* [platform-mesh/kubernetes-graphql-gateway#222](https://github.com/platform-mesh/kubernetes-graphql-gateway/issues/222)
