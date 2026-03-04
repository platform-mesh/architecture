---
status: proposed
date: 2026-03-03
decision-makers: Platform Mesh team
---

# Use Email as User Identifier Across the Platform

## Context and Problem Statement

Platform Mesh authenticates users via Keycloak-issued JWT tokens (potentially through a federated IdP). The token passes through multiple systems — each with its own constraints on which claims are available:

1. **Keycloak** signs the JWT containing claims like `sub`, `email`, `preferred_username`, etc.
2. **KCP / Kubernetes API Server** validates the JWT via OIDC integration and extracts a username based on the `username.claim` configured in a `WorkspaceAuthenticationConfiguration` (KCP) / `StructuredAuthenticationConfiguration` (Kubernetes) resource.
3. **rebac-authorization-webhook** receives a `SubjectAccessReview` from the API server for authorization decisions. At this point, only the extracted `user`, `groups`, `uid`, and `extra` fields are available — the original JWT claims are gone.
4. **OpenFGA** stores relationship tuples that reference users by identifier. The authorization webhook must be able to correlate the user from the `SubjectAccessReview` with tuples stored in OpenFGA.

The central question is: **which JWT claim should serve as the canonical user identifier across all these systems?**

A typical JWT issued by Keycloak looks like this:

```json
{
  "sub": "dec76aa7-7bd0-40f8-93e4-65ac473dbc5e",
  "email": "tom.example@example.com",
  "preferred_username": "tom.example@example.com",
  "name": "Tom Example",
  "iss": "https://portal.localhost:8443/keycloak/realms/welcome",
  ...
}
```

## Decision Drivers

* **Kubernetes OIDC integration ergonomics** — the identifier must work naturally with Kubernetes RBAC and tooling (e.g. `kubectl`)
* **Authorization webhook compatibility** — the identifier must be derivable from the `SubjectAccessReview` request, which does not carry raw JWT claims
* **Consistency across systems** — a single identifier must correlate users in Kubernetes, OpenFGA, and internal services without additional lookups
* **Data privacy / GDPR compliance** — Art. 17 GDPR (right to erasure) requires that personal data can be deleted on request; using PII as a primary identifier complicates this
* **Resilience to identity changes** — users may change their email address; the identifier should ideally be stable over time

## Considered Options

* **Option 1: Email (`email` claim)**
* **Option 2: Subject (`sub` claim / opaque UID)**

## Decision Outcome

Chosen option: **"Email (`email` claim)"**, because it is the only option that provides a seamless end-to-end correlation from Kubernetes OIDC through the authorization webhook to OpenFGA.

This is a deliberate trade-off: we accept that email addresses are visible to platform administrators with direct access to OpenFGA or Kubernetes audit logs, and that email changes require tuple migration, in exchange for a significantly simpler integration with the Kubernetes ecosystem.

### Consequences

* Good, because Kubernetes RBAC bindings work naturally with human-readable identifiers (e.g. `--user=tom@example.com`)
* Good, because the authorization webhook can directly use the `user` field from `SubjectAccessReview` to query OpenFGA without any claim translation
* Good, because operators and administrators can easily reason about permissions using familiar email addresses
* Bad, because if a user's email changes, all stored references (OpenFGA tuples, Kubernetes RBAC bindings, audit trails) must be updated or become stale
* Bad, because email addresses embedded in OpenFGA tuples directly expose user identity to anyone with access to the tuple store, conflicting with [OpenFGA best practices on user identifiers](https://openfga.dev/docs/getting-started/tuples-api-best-practices)

### Confirmation

* Verify that `username.claim` is set to `email` in the `WorkspaceAuthenticationConfiguration` resources on all KCP workspaces.
* Verify that OpenFGA tuples reference users by email.
* Document a runbook for handling email changes and GDPR erasure requests covering OpenFGA tuples, Kubernetes RBAC bindings, and audit logs.

## Pros and Cons of the Options

### Option 1: Email (`email` claim)

Configure `username.claim: email` in the `WorkspaceAuthenticationConfiguration` / `StructuredAuthenticationConfiguration` resource. The API server extracts the email from the JWT and passes it as the `user` field in `SubjectAccessReview`. OpenFGA tuples store `user:<email>`.

* Good, because when `username.claim` is set to `email` in the authentication configuration, RBAC bindings can be created directly against the email address (e.g. `kubectl create clusterrolebinding ... --user=user@example.com`) without the `<issuer>#<claim>` prefix that other claims require ([Kubernetes OIDC docs](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens))
* Good, because the authorization webhook receives the email as the `user` field in the `SubjectAccessReview` and can use it directly for OpenFGA checks — no mapping or lookup required ([Webhook Authorization docs](https://kubernetes.io/docs/reference/access-authn-authz/webhook/))
* Good, because email addresses are human-readable, making debugging, auditing, and manual RBAC management straightforward
* Neutral, because email is generally unique per user within an IdP realm, but uniqueness guarantees depend on Keycloak configuration
* Bad, because email addresses are directly identifiable — anyone with access to OpenFGA tuples, Kubernetes audit logs, or RBAC bindings can see which user has which permissions, increasing the exposure surface compared to a pseudonymous identifier
* Bad, because email addresses can change (e.g. name change, company domain migration) — when this happens, all OpenFGA tuples, RBAC bindings, and references tied to the old email become invalid and must be migrated
* Bad, because it contradicts [OpenFGA best practices](https://openfga.dev/docs/getting-started/tuples-api-best-practices) which recommend against storing directly identifiable information in tuples

### Option 2: Subject (`sub` claim / opaque UID)

Configure `username.claim: sub` (or another opaque UID claim) in the `WorkspaceAuthenticationConfiguration` / `StructuredAuthenticationConfiguration` resource. OpenFGA tuples store `user:<uuid>`.

* Good, because the `sub` claim is a stable identifier that never changes — it survives email changes, name changes, and domain migrations
* Good, because it is pseudonymous — while still personal data under GDPR (it can be correlated back to a natural person via Keycloak), it does not directly reveal the user's identity to anyone inspecting the tuple store or audit logs
* Good, because it partially aligns with OpenFGA best practices ("do not store PII in tuples") — though strictly speaking, a `sub` that maps to a Keycloak user is still personal data, it is not *directly identifiable* PII like an email address
* Good, because it decouples identity from any mutable user attribute
* Bad, because Kubernetes requires the `<issuer-url>#<claim-value>` prefix format for non-email/non-username claims — the user field in RBAC bindings becomes something like `https://portal.localhost:8443/keycloak/realms/welcome#dec76aa7-7bd0-40f8-93e4-65ac473dbc5e`, which is unwieldy for manual operations
* Bad, because the authorization webhook receives this prefixed string as the `user` in `SubjectAccessReview` — the webhook and OpenFGA must either store tuples using the full prefixed form or strip the prefix, both of which add complexity
* Bad, because UUIDs are opaque — debugging authorization issues requires a lookup from UUID to human-readable identity, increasing operational overhead
* Bad, because any tooling or UI that displays permissions must resolve UUIDs back to user-readable names, requiring an additional user info service or Keycloak lookup

**Note on PII:** Under GDPR, the `sub` claim is still considered personal data because it can be used to identify a natural person when combined with other data (e.g. Keycloak's user database). The distinction is between *directly identifiable* PII (email) and *pseudonymous* PII (sub). Both require GDPR compliance, but pseudonymous data reduces the risk of casual exposure and may simplify some compliance scenarios.

## More Information

### Mitigations for Known Risks

The following mitigations should be implemented to address the drawbacks of using email:

1. **GDPR erasure**: Implement a user deletion workflow that removes or anonymizes user data across all systems — OpenFGA tuples, Kubernetes RBAC bindings, audit logs, and any internal databases. This is required regardless of which identifier is chosen (both email and `sub` are personal data under [Art. 4(1) GDPR](https://gdpr-info.eu/art-4-gdpr/)), but using email means the identifier itself must be scrubbed rather than just the mapping.

2. **Email change handling**: Implement an email migration workflow that updates all references when a user's email changes in Keycloak. This includes OpenFGA tuples, Kubernetes RBAC bindings, and any cached identity information.

3. **Tuple store access control**: Restrict access to the OpenFGA tuple store to minimize exposure of email addresses embedded in tuples.

### References

* [Kubernetes OIDC Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)
* [Kubernetes Webhook Authorization](https://kubernetes.io/docs/reference/access-authn-authz/webhook/)
* [OpenFGA Tuples API Best Practices](https://openfga.dev/docs/getting-started/tuples-api-best-practices)
* [Art. 4 GDPR — Definitions](https://gdpr-info.eu/art-4-gdpr/) (defines "personal data" and "pseudonymisation" in points 1 and 5)
* [Art. 17 GDPR — Right to Erasure](https://gdpr-info.eu/art-17-gdpr/)
