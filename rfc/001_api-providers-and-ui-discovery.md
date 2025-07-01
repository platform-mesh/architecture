# API Provider data and Provider UI Discovery in Platform Mesh 

## Context and Problem Statement

Platform Mesh adopts KCP to offer a central KRM based control plane for its users. KCP introduces a set of resources that allow service providers to register their API's and to bind such API's to a user workspace. Key resources here are `APIExport`, `APIBinding`, `APIResourceSchema`, and `Workspace`.

With these preconditions in place there are certain concerns that are not possible / addressed by these core KCP resources:

- Where to provide additional metadata about the API Provider that could be used for discovery purposes. Examples of such metadata could be the Name, Description, Icon, Website, Support Channels, Documentation lings, etc.
- In a platform-mesh installation it may not always be desirable that introduced API's can be bound by any workspace. For example adopters can create may `organizations` and there will be API's that are limited to a specific organization. It needs to be possible to constraint where a given APIExport can be bound to.
- Next to the available API's it is also desirable that providers can register a UI to be integrated into the platform mesh portal. This UI should only be integrated when rendering the UI for a particular Workspace where the given UI is bound to. The portal application needs to be able to discover such registered UI's on the fly.


This RFC will describe a proposal on how to close the above outlined gaps to seek feedback. Some aspects could be introduced as part of the KCP core, while others may remain platform-mesh specific.


## API Provider Metadata
- It is desirable to keep provider metadata co-located to the other resources that a provider would need to provide. Therefore, this information should be in the same workspace as the `APIExport`.
- The Provider Metadata should be an optional aspect as internal API's may not have a provider and there can be installation scenarios where the provider metadata may not be relevant. 
- A provider can be responsible for multiple APIExports, therefore the Provider Metadata should be a separate resource that can then be related to the APIExport.

### Proposal
Introduce a new resource `APIProvider` that can be used to store the provider metadata. It should be possible to infer the `APIProvider` from a given APIExport. This could be done by using a optional field on the `APIExport` or by using a label/annotation on the `APIExport`.

A possible spec could look like this: 
```yaml
apiVersion: api.platform-mesh.io/v1alpha1
kind: APIProvider
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
Above data is meant for demonstration purposes only and can be extended or changed as needed.

## UI Discovery and Provider UI Binding

- It is desirable to configure a provider's UI next to the other resources that a provider would need to provide.  Therefore this information should be in the same workspace as the `APIExport`.
- OpenMFP makes use of a ContentConfiguration resources to configure a UI. As a consequence it would be good to have the content configration next to the other resources.
- It should be avoided that the ContentConfiguration is copied to every workspace, therefore the proposal is to make use of a binding resource similar to the `APIBinding` resource.
- This way only the `ContentConfiguration` resource will be validated and updated once and the UI's in all workspaces with a binding will update automatically.

Example of a ContentConfiguration resource:
```yaml
apiVersion: ui.platform-mesh.io/v1alpha1
kind: ContentConfiguration
metadata:
  name: acme.provider.io
spec:
  # Either inline or remote configuration can be used.
  inlineConfiguration:
    content: |-
      {
        <!-- This is a JSON object that configure the to be integrated UI -->
      }
    contentType: json
  remoteConfiguration:
    url: "https://my-ui.com/config.json"
    contentType: json
```

The ContentConfigurationBinding resource would look like this:
```yaml
apiVersion: ui.platform-mesh.io/v1alpha1
kind: ContentConfigurationBinding
metadata:
  name: acme.provider.io
spec:
  reference:
    export:
      name: acme.provider.io
      path: root:providers:acme-provider:acme.provider.io
```

-- TODO add a diagram to illustrate how these resources would be used

