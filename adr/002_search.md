---
# These are optional metadata elements. Feel free to remove any of them.
status: "proposed"
date: 2025-11-11
decision-makers: {aaronschweig, FWuermse, pteich, tjbutz},
consulted: {nexus49, tobias-oetzel}
informed: {tjbutz}
---

# Support for a Generic Search Engine within Platform Mesh

**See also**: [RFC-001: Search Architecture](../rfc/rfc-001-search-architecture.md) for detailed architectural design.

## Context and Problem Statement

Currently Platform Mesh does not enable advanced searching such as partial word searches, fuzzy search, or semantic search. Additionally, using the K8s/KCP APIs directly is not a valid approach at scale, as the K8s/KCP gateway is too slow to query thousands of entries and lacks proper pagination. That means that every user has to come up with it's own search architecture and permission management within the search and search index.

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

Chosen option: "per-organization-search", because it provides the best balance between:
- Resource isolation per organization (security)
- Manageable FGA request volume (filtering at organization boundary)
- Implementation feasibility (proven in hackathon)
- Scalability (indices can be distributed across OpenSearch clusters)

The per-organization approach aligns with Platform Mesh's existing tenancy model where organizations are the primary isolation boundary.

### Consequences

* Good, because each organization has isolated search indices preventing information leakage
* Good, because OpenFGA authorization is scoped to organization reducing request volume
* Good, because indices can be distributed across OpenSearch clusters for scalability
* Good, because follows Platform Mesh's existing per-organization tenancy model
* Bad, because requires managing multiple OpenSearch indices (operational overhead)
* Bad, because eventual consistency between KCP resources and search indices
* Bad, because additional infrastructure component increases complexity

### Confirmation

TBD

## Pros and Cons of the Options

### per-organization-search

Each organization has it's own index and narrowing search down the hierarchy is configured via filters (e.g. one dedicated field holding the kcp subpath).

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