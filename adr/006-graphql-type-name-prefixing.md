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
