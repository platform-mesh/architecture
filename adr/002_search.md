---
# These are optional metadata elements. Feel free to remove any of them.
status: "proposed"
date: 2025-11-11
decision-makers: {aaronschweig, FWuermse, pteich, tjbutz}, 
consulted: {nexus49, tobias-oetzel}
informed: {tjbutz}
---

# Support for a Generic Search Engine within Platform Mesh

## Context and Problem Statement

Currently Platform Mesh does not enable advanced searching such as partial word searches, fuzzy search, or semantic search. That means that every user has to come up with it's own search architecture and permission management within the search and search index.

![Search Architecture](images/002_search_architecture.png)

### Prerequisites

What needs to be in place before implementation can start:
* Choice for a reference search engine (e.g. Solr, Elasticsearch, Open Search)
* Identify if there are current efforts around search in the Apeiro Space

## Decision Drivers

* K8s should be leveraged for data-storage and APIs
* FGA should be leveraged for authorization
* The Apeiro Showroom should benefit from the search 

## Considered Options

* A configurable search engine per account (per-account-search)
* One global search engine with delegated permission management (central-search)
* One global logical search engine per organization in Platform Mesh

## Decision Outcome

Chosen option: "{title of option 1}", because {justification. e.g., only option, which meets k.o. criterion decision driver | which resolves force {force} | … | comes out best (see below)}.

### Consequences

* Good, because {positive consequence, e.g., improvement of one or more desired qualities, …}
* Bad, because {negative consequence, e.g., compromising one or more desired qualities, …}

### Confirmation

TBD

## Pros and Cons of the Options

### per-organization-search

Each organization has it's own index and narrowing search down the hierarchy is configured via filters (e.g. one dedicated field holding the kcp subpath).

#### Roadmap

##### Hackathon Stage
* SearchIndex CRD + Search Operator + OpenSearch integration
* Added to `core.platform-mesh.io` APIExport
* Apply a APIResourceSchema `searchindex` as part in a custom setup script in helm-charts ([see example](local-setup/scripts/load-custom-images.sh.example))
* **Repositories**:
  - `platform-mesh-hackathon-0126/helm-charts`
  - `platform-mesh/search-operator`
* **Key files**:
  - `local-setup/hsp/root/platform-mesh-system/apiresourceschema-searchindices.core.platform-mesh.io.yaml`
  - `local-setup/hsp/root/platform-mesh-system/apiexport-core.platform-mesh.io.yaml`
  - `charts/search-operator/`
  - `local-setup/hsp/deployments/opensearch/`

##### POC Next Steps
1. Workspacetype `search` extended by the [org](https://github.com/platform-mesh/platform-mesh-operator/blob/main/manifests/kcp/workspace-type-org.yaml) type
    - [`platform-mesh/platform-mesh-operator`](https://github.com/platform-mesh/platform-mesh-operator/pull/335/changes)
2. Initializer for auto-creating SearchIndex resources
    - creates a resource of APIResourceSchema `searchindex` in `root:platform-mesh-system`
    - deletes initializer string
    - `charts/search-operator/templates/initializer-deployment.yaml`
3. Operator reconciliation logic for indexing
    - `search-operator` repository (unstructured operator configurable via yaml config)
    - Discover Resources via reading export (either core-platform-mesh for now or dedicated search export)

##### Platform Mesh integration

1. Move existing changes to `github.com/platform-mesh`
2. Discover Indexable Resources and index per organization (fga in mind)
3. Search endpoint

#### validation steps
in `:root:platform-mesh-system`:
`k get apiresourceschemas` // returns 
`k get apibindings --server='https://kcp.api.portal.dev.local:8443/services/apiexport/1cklpfb2n05i2klh/core.platform-mesh.io/clusters/*/' -A` // should list bindings of all orgs
`k get searchindices --server='https://kcp.api.portal.dev.local:8443/services/apiexport/1cklpfb2n05i2klh/core.platform-mesh.io/clusters/*/' -A` // should not fail (e.g. `No resources found`)
`k api-resources --server='https://kcp.api.portal.dev.local:8443/services/apiexport/1cklpfb2n05i2klh/core.platform-mesh.io/clusters/*/'` // 

In `:root`:
`k get workspacetypes` // includes searchindices
`k get workspacetypes search -o yaml` // returns spec
`k get workspacetypes org -o yaml` // extends security and search

### per-account-search

* Good, because fewer openFGA requests needed
* Good, because users can configure the granularity of instances depending on their hierarchies
* Neutral, because configuration has to be provided
* Bad, because implementation effort is greater

### central-search

* Good, because easy to set up
* Neutral, because no configuration needed
* Bad, because implementation details of authentication are complicated
* Bad, because information can potentially leak
* Bad, because too many FGA requests will be needed

## More Information

This ADR is the basis for a Spike where we explore what's possible and develop a proof of concept. Thus, the following features are currently not in scope and may only be relevant once the outcome of the Spike has been explored:

* Search engine is exchangeable
* AI-features (depends on the capabilities of the search engine chosen)
* APIs for non-k8s searchable resources (external DB etc.)