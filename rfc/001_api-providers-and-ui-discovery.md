# RFC: API Provider Data and Provider UI Discovery in Platform Mesh

| Status  | Proposed                                                     |
|---------|--------------------------------------------------------------|
| Author  | @nexus49                                                     |
| Created | 2025-07-01                                                   |
| RFC PR  | [link](https://github.com/platform-mesh/architecture/pull/1) |

## Summary

This RFC proposes a mechanism for managing API provider metadata and UI discovery within Platform Mesh. It introduces new resources to address gaps in KCP's core resources, enabling richer provider metadata, controlled API binding, and dynamic UI integration for workspaces.

## Context and Problem Statement

Platform Mesh leverages KCP to provide a central KRM-based control plane. KCP introduces resources such as `APIExport`, `APIBinding`, `APIResourceSchema`, and `Workspace` for API registration and binding.

However, several concerns are not addressed by these core resources:

- **Provider Metadata**: There is no standard way to attach metadata (name, description, icon, website, support, docs, etc.) to API providers for discovery purposes.
- **Binding Constraints**: Not all APIs should be bindable by any workspace; some APIs may be organization-specific and require binding constraints.
- **Provider UI Registration**: Providers may want to register UIs for integration into the Platform Mesh portal, visible only in relevant workspaces.

This RFC proposes solutions to address these gaps, some of which may be candidates for KCP core, while others remain platform-mesh specific.

## Motivation

- Enhance discoverability and management of API providers.
- Enable fine-grained control over which workspaces can bind to specific APIs.
- Support dynamic UI integration for providers within the portal.

## Proposal

### API Provider Metadata

- Provider metadata should be co-located with other provider resources, ideally in the same workspace as the `APIExport`.
- Metadata should be optional, as not all APIs require a provider or relevant metadata.
- A provider may own multiple `APIExport` resources; thus, metadata should be a separate resource, referenced from `APIExport`.

#### New Resource: `ProviderMetadata`

Introduce an `ProviderMetadata` resource to store provider metadata. It should be possible to associate an `ProviderMetadata` with an `APIExport` via an optional field, label, or annotation.

**Example:**
```yaml
apiVersion: api.platform-mesh.io/v1alpha1
kind: ProviderMetadata
metadata:
  name: acme-example-provider
spec:
  tags:
    - infra
    - eu
  description: This is the acme example provider corp.
  displayName: ACME Example Provider corp
  icon:
    dark:
      url: "https://myimage.com/image.png"
    light:
      url: "https://myimage.com/image.png"
  contacts:
    - displayName: John Doe
      email: jd@acme.corp
      roles:
        - support
        - sales
      contactLink: "https://acme.corp/contact/jd"
  documentation:
    - name: API Documentation
      url: "https://acme.corp/docs"
    - name: End User Documentation
      url: "https://acme.corp/end-user-docs"
  links:
    - name: Website
      url: "https://acme.corp"
      default: true
    - name: Wiki
      url: "https://acme.corp/wiki"
  preferredSupportChannels:
    - name: Support
      url: "https://acme.corp/support"
  helpCenterData:
    - name: Issue Tracker
      url: "https://acme.corp/issues"
    - name: Feedback Tracker
      url: "https://acme.corp/feedback"
```
*Note: The schema is illustrative and can be extended as needed.*

### UI Discovery and Provider UI Binding

- Provider UI configuration should be co-located with other provider resources in the same workspace as the `APIExport`.
- OpenMFP uses `ContentConfiguration` resources for UI configuration; these should not be duplicated across workspaces.
- Make use of the already existing binding relation (`APIBinding`) between the api export in the workspace. This is beneficial over adding a binding resource for the content configuration.

**Example: ContentConfiguration**
```yaml
apiVersion: ui.platform-mesh.io/v1alpha1
kind: ContentConfiguration
metadata:
  name: acme.provider.io
  labels:
    ui.platform-mesh.io/content-for: "acme.example.corp"
spec:
  # Either inline or remote configuration can be used.
  inlineConfiguration:
    content: |-
      {
        <!-- This is a JSON object that configures the UI to be integrated -->
      }
    contentType: json
  remoteConfiguration:
    url: "https://my-ui.com/config.json"
    contentType: json
```
**Overview**

![diagram](./assets/entities.png)


## Operators and Services and their use of the new Resources

The diagram below illustrates on a high level what other components will use the new resources:
![diagram](./assets/container-diagram.png)

## Alternatives Considered

- Embedding provider metadata directly in `APIExport` (less flexible, harder to reuse).
- Duplicating UI configuration in each workspace (leads to drift and maintenance overhead).
- Using separate binding resources for UI configuration (unnecessary complexity).


## References

- [KCP Documentation](https://github.com/kcp-dev/kcp)
- [OpenMFP Documentation](https://github.com/openmfp/openmfp)
- [Kubernetes API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
