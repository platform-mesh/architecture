# ADR 012: Separate Local Development from Production Delivery

| Status          | Proposed                                                                                   |
|-----------------|--------------------------------------------------------------------------------------------|
| Date            | 2026-09-28                                                                                 |
| Decision-makers | Platform Mesh TSC; maintainers of `helm-charts`, the `platform-mesh` monorepo and `ocm`   |
| Builds on       | [RFC 009](../rfc/009_repo-consolidation.md) (chart co-location), [ADR 010](010-consolidate-helm-charts-and-ocm.md) |
| Related         | [RFC 007](../rfc/007_platform-simplification.md), [RFC 008](../rfc/008-platform-mesh-modularization.md), [ADR 001](001-sbom-generation-and-ocm-component-restructuring.md) |
| Spikes          | [Helm-only local setup](https://github.com/xrstf/pm-helm-charts/tree/local-setup2/local-setup2), [Helm charts as the production lowest common denominator](https://github.com/xrstf/platform-mesh-from-scratch/) |

## Context and Problem Statement

Platform Mesh's deployment architecture rests on an unstated requirement: **the
local development setup must mirror production**. `task local-setup` therefore
installs charts from the developer's working tree through the full production
delivery chain: OCM component build, in-cluster OCI registry, transfer pod,
ocm-k8s-toolkit, kro, Flux, and the platform-mesh-operator's deployment
subroutine. A second split compounds it: the charts live in `helm-charts`, the
code in the monorepo and the UI repos, so a value, a CRD field and the Go that
reads them can never change in one commit or be tried together in one cluster.

Both requirements are right for production and wrong for local development, and
one pipeline cannot serve both.

### Two environments, two goals

| | Local development | Production |
|---|---|---|
| **What matters** | Speed of the edit→run→observe loop | Consistency, provenance, auditability |
| **Source of truth** | The working tree | A signed, versioned artifact in a registry |
| **Who reconciles** | The developer, on demand | A controller, continuously |
| **Trust model** | None needed; you built it | Signatures, pinned digests, mirrored registries |
| **Failure to avoid** | "I waited 20 minutes and it didn't pick up my change" | "Something ran that we didn't release" |
| **Acceptable to skip** | Signing, verification, drift detection, ordering | Nothing |
| **Composition** | Some components in-cluster, some as `go run` | Everything in-cluster, as released |
| **Flavours** | One: "works on my kind cluster" | Many: air-gapped, split-cluster, external IdP/DB |

> **Every row in the right-hand column is a feature in production and a cost in
> local development. When one pipeline must satisfy both, production wins, because
> its requirements are non-negotiable and developer time is invisible.**

Nobody decides to slow developers down. A production requirement arrives
(signing, a second delivery engine, a split-cluster topology) and is added to the
one pipeline that exists. Each addition is justified; none is removed, because
that would make local "not like production". The cost lands on every contributor
as minutes per iteration, tools to learn before a first chart change, and a
reconciler to fight when running a component from source, and appears on no
dashboard. The only structural fix is to stop having one pipeline.

### What the delivery tools contribute locally

> Skip if you are familiar with the architecture.

| Tool | Production value | Local value | Local cost |
|---|---|---|---|
| **OCM** | Signed, digest-pinned release graph; air-gap transfer | None: the local build renames the aggregate to `prerelease`, stamps everything `1.0.0`, and the kro graph sets `skipVerify: true` | In-cluster registry, transfer pod, CA plumbing, a component build, `ocmcrds/` |
| **Flux** | Continuous reconciliation, ordering, drift correction | None: the source of truth is the working tree and the reconciler is the developer | 15m timeouts, Flux 2.17 pin, one more layer per failure, undoes `go run` by design |
| **kro** | Composes the operator's OCM `Resource` → `OCIRepository` → `HelmRelease` | None: without OCM there is nothing to compose; the operator is a chart | One more controller, CRD and readiness wait |
| **Operator deploy subroutine** | One `PlatformMesh` CR renders the whole platform per target | None: the developer already has the values files; templating `portal.localhost` at reconcile time is a slower way of writing it in a file | Profile ConfigMap indirection, `syncWave`, `imageResources`, two-phase templating |

None of these tools is wrong. All of them are delivery tools, and local
development is not delivery. Both spikes run the operator with the deployment
subroutine off and keep only its kcp bootstrap role.

### Configuration is split from code

A component release crosses four repositories: a monorepo tag builds the image,
`platform-mesh/.github` dispatches into `helm-charts`, one bot edits `Chart.yaml`
and a second approves it, the chart pipeline publishes, and `ocm` rebuilds the
service component and aggregate. Roughly two in five commits and over half of
merged PRs in `helm-charts` are that ping-pong (RFC 009's term). Beyond churn:

- **Generated artifacts drift.** `apigen` writes `APIResourceSchema`s to
  `operators/<name>/config/resources/`; the same files ship from
  `charts/<name>-crds/templates/`, copied by hand. Today the monorepo generates
  `v260109-82344be.accounts.core.platform-mesh.io` while the chart installs
  `v250915-1185c0b`, four months older, with no CI check that would notice.
- **No atomic change.** A flag and its value, a CRD field, a port: two PRs, an
  ordering agreement, and a window where `main` of one repo breaks `main` of the
  other.
- **No YAML-with-code loop.** The monorepo's own Tilt environment pulls pinned
  charts from the registry into a cache and offers `HELM_CHARTS_DIR` "for chart
  development", which is the split felt from the other side.
- **Reviews lose context.** A chart reviewer cannot see the Go change; a Go
  reviewer cannot see whether the flag is wired into the chart.

RFC 009 already proposes moving charts next to their code (Wave 3). This ADR
makes that a prerequisite: a Helm-only local loop that still fetches charts from
another repository fixes half the problem.

### Prior art

- **Inner vs. outer loop.** Tilt's tagline is "Kubernetes for Prod, Tilt for
  Dev". Microsoft's GitOps guidance: render manifests locally from the developer's
  own values for the inner loop; CI produces what the GitOps repo reconciles.
  Cluster API and Prow develop with kind + Tilt; users install with `clusterctl`
  or manifests, and the Cluster API docs say the Tilt path is not the management
  path.
- **Helm as the lowest common denominator.** Knative calls its plain-manifest
  path "the lowest common denominator approach" for Flux or Argo, operator
  optional. cilium-cli made "helm mode" the default in v0.15: the CLI wraps
  `helm install`/`upgrade` and the values file is the canonical configuration.
  Helm is the most used Kubernetes package manager (75%, CNCF survey); Flux and
  Argo CD consume charts from OCI natively.
- **Install operators are being removed.** Istio deprecated its in-cluster
  operator (1.23, Aug 2024): "a high-privilege controller running inside your
  cluster", "a level of indirection, where you have to have options in your
  custom resource to configure everything", three install methods that "caused
  confusion"; Helm is the recommendation. GitLab's distribution team describes
  an operator rendering a chart from another repo as "leaky abstractions" and "a
  more convoluted release pattern".
- **OCM is going deployer-agnostic.** Its open epic states "the deployer
  technology leaks into the component bundle" and a component "should not need
  to know about" Flux vs. Argo.
- **Configuration lives with code.** Kubebuilder keeps `config/` with the
  controller and its Helm plugin regenerates the chart from `config/` on every
  run so it cannot drift from the CRDs. cert-manager, Cilium, Linkerd, Kyverno,
  External Secrets and Loki keep their charts in-repo. Argo CD's "separate config
  vs. source repos" advice concerns rendered, environment-specific manifests (CI
  loops, access), not the chart template; under this ADR the template stays with
  the code and environment values live with the environment. The
  separate-repo counterexample is argo-helm: "community maintained", latest
  version only, with users asking for a chart-to-app version mapping table.
- **Twelve-factor parity is about backing services, not delivery tooling.** The
  "tools gap" is Postgres vs. SQLite, not Flux vs. `helm install`. The local
  setup here keeps kcp, etcd, Keycloak, OpenFGA and CNPG identical and removes
  only the delivery chain. Humble & Farley's "build once, deploy the same
  artifact everywhere" is satisfied by chart + image being the artifact.

## Decision Drivers

1. **Complexity**: tools, controllers, hops and repositories between a chart
   change and a pod.
2. **Development speed**: edit-to-result time; running any component from source.
3. **A clear production split**: delivery stays complete and opinionated, in its
   own environment, never as a tax on local development.
4. **Configuration changes with code**: value, CRD and Go in one commit, one
   review, one cluster.

## Considered Options

**A. Status quo.** One pipeline; charts in `helm-charts`, synced by bots.
Good: nothing moves; the deploy subroutine gets incidental coverage. Bad: nine
hops and five controllers per iteration; production's shape without its
guarantees; reconcilers fight `go run`; two checkouts to change a chart with its
code; silent schema drift.

**B. Helm as the common denominator; split environments; charts next to code
(chosen).** Good: one-hop iteration; components can be skipped and run from
source; chart, CRDs and code change and test together; charts must stand alone,
which adopters also need; both spikes prove it on current charts. Bad: two setups
to keep green (smaller together than today's `local-setup/` + `production-setup/`
+ `--remote`); deploy-subroutine changes are tested in the box or CI; chart CI
becomes a reusable workflow; a few chart gaps to close first.

**C. Keep OCM/Flux locally, install only CI-built prerelease components.** Good:
smaller change, genuinely verified locally. Bad: every chart change needs a CI
round trip; Flux still fights `go run`; this is what the box already is.

**D. Helm everywhere; drop OCM, Flux and kro.** One path, but it discards signing,
air-gap transfer and the release graph for the adopters who need them. Rejected.

**Chart location (within B).** Keep in `helm-charts` with bot sync: rejected
above. Add a cross-repo drift check: detects, does not prevent; a stopgap only.
Co-locate per RFC 009: chosen, as a prerequisite.

## Proposed Outcome

**Option B.** It is the only option that removes the mechanism in the context: as
long as one pipeline serves both environments, every future production
requirement lands on developers again. A and C keep the single pipeline; D
removes production's requirements instead of separating them.

**Lowest common denominator.** Every component installs with
`helm upgrade --install <name> oci://ghcr.io/platform-mesh/helm-charts/<name> --version <v> -f values.yaml`,
or from a chart directory in a checkout. Nothing else is a hard requirement.
Anything a chart cannot express itself is a gap in the chart, not a reason for an
orchestrator.

**Prerequisite: charts live with their code.**

- Component charts move to the repo that builds the image (monorepo; UI repos).
  Layout (`charts/<name>` per RFC 009 or `operators/<name>/chart/`) is an
  implementation detail.
- `<name>-crds` folds into the chart as `crds/`, generated by the same
  `task generate` as `config/resources/`; CI fails on difference.
- Platform charts without a component (`common`, `infra`, `observability`,
  `gateway-api-crds`, `keycloak-operator`, `kcp-*-vw`) stay in `helm-charts`,
  which per ADR 010 also holds the aggregate, the box and production docs.
- A `<component>/vX.Y.Z` tag builds image, chart (same OCI path as today, so
  consumers and `componentReference`s are unchanged) and service OCM component in
  one run. The `.github` → `helm-charts` → `ocm` dispatch chain is deleted.
- Chart `version` equals `appVersion`; chart-only fixes are a patch tag.

**Three environments.**

1. **Local development**: kind + `helm upgrade --install` from the checkout for
   in-repo charts and from OCI for platform and third-party charts. No OCM, Flux,
   kro, registry or transfer pod. Operator with deploy subroutine off (kcp
   bootstrap only). `--skip <name>` leaves a component out so it can be run from
   source.
2. **Platform Mesh in a box**: production lookalike for demos, verification and
   upgrade tests. Installs a *published* OCM version via OCM + Flux/Argo + kro +
   operator as `production-setup/` describes, optionally two-cluster. Never
   builds from the working tree; PR prereleases are built in CI.
3. **Production**: adopter's choice of plain Helm, Flux/Argo HelmReleases at
   pinned versions, or the full OCM aggregate. OCM stays recommended, not
   required.

Releases publish every component as both chart and OCM component. The alignment
we want is not "local looks like production" but "production installs what local
tested".

**Chart gaps to close first.** Hand-assembled `kcp-webhook-secret`; operator
hard-requiring OCM CRDs; script-generated bootstrap secrets (charts need
`existingSecret` plus a non-production `generate` mode); mandatory empty
`platform-mesh-profile` ConfigMap; remaining `extraArgs` that should be values;
cross-chart ordering expressed as dependencies or documented once, not as
`syncWave` numbers.

**Sequencing.** Co-location (RFC 009 Wave 3, pulled forward) → Helm-only local
setup landed next to the current one and flipped when green → box
(`production-setup/` on kind, working-tree mode removed). Each step ships and
reverts on its own.

## Consequences

* Good: chart iteration is one `helm upgrade --install`; no build, transfer or
  reconcile wait.
* Good: run one component from source by uninstalling its release, pulling its
  kubeconfig secret, `go run`; nothing reverts it.
* Good: flag, value, CRD and test change in one PR; generated CRDs cannot drift.
* Good: a release is image + chart + service component in one run; no bot PRs, no
  approver app.
* Good: contributors need kind, helm, kubectl, mkcert; adding a service is a
  chart directory next to the code plus one aggregate line.
* Good: adopters with their own GitOps or supply chain install without OCM.
* Bad: deploy-subroutine, profile-templating and OCM image-resolution changes are
  tested in the box, not on a laptop by default.
* Bad: third-party chart versions must agree across local, box and aggregate; one
  shared version list is needed.
* Bad: chart CI becomes a reusable workflow; `common` is a pinned cross-repo
  dependency.
* Bad: two documented production paths, two sets of upgrade notes.

To revisit: whether kcp bootstrap becomes a chart hook or `init` binary so local
needs no operator; whether the from-scratch `installer` (resolve versions from the
OCM descriptor, merge values, emit HelmReleases) becomes a supported tool; whether
`platform-mesh-deployer`'s Module model (RFC 008) supersedes the aggregate; the
box's name and location.

## Relationship to Other Records

- **RFC 009**: proposes co-location as Wave 3 and deletes the ping-pong; this ADR
  adds the local-dev rationale and pulls it forward as a prerequisite. Its Goal 7
  (operator applies resources directly, not chart-in-chart) is a production
  delivery choice and compatible: the chart is the artifact.
- **ADR 010**: the merged `helm-charts`+`ocm` repo is where platform charts, the
  aggregate and the box live after component charts move out.
- **ADR 001**: preserved; the service component is built by the component's own
  release.
- **RFC 008**: orthogonal; local `--skip` is a convenience, not runtime
  composition.
- **RFC 007**: names deployment complexity and the contributor funnel as adoption
  blockers; this is one concrete step against both.

## Appendix: the current paths

Chart in the working tree → pod in kind:

```text
 working tree charts → ocm-build-local-charts.sh (helm package)
   → ocm-build-component.sh (aggregate renamed github.com/platform-mesh/prerelease:1.0.0)
   → ocm-transfer-pod (OCM CLI; registry CA copied into its trust store)
   → in-cluster OCI registry (mkcert TLS, NodePort 30500)
   → ocm-k8s-toolkit (patched to trust the registry, restarted)
   → OCM Component / Resource CRs
   → kro ResourceGraphDefinition (Resource → OCIRepository → HelmRelease)
   → Flux installs platform-mesh-operator
   → operator renders ~20 HelmReleases from the profile ConfigMap
   → Flux installs everything else (timeout 15m, retries -1)
```

Nine hops, five controllers, two of which exist only to install a third whose job
is to generate input for a fourth. The Helm-only spike reaches the same
`https://portal.localhost:8443` in one hop. The `--iterate` fast path still runs
the full build and transfer; it skips only cluster creation, certs, Flux and the
registry.

Component tag → released chart:

```text
 monorepo tag <component>/vX.Y.Z (image, SBOM, release)
   → platform-mesh/.github job-chart-version-update (dispatch)
   → helm-charts update-chart-parameters (bot edits Chart.yaml, bot approves, auto-merge)
   → helm-charts pipeline-chart (package, push, chart OCM component)
   → platform-mesh/ocm service component → aggregator (bump, sign, publish)
```

## References

Inner loop vs. outer loop

- Tilt: <https://docs.tilt.dev/>
- Microsoft Learn, inner loop for teams adopting GitOps:
  <https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/conceptual-inner-loop-gitops>
- Cluster API, developing with Tilt: <https://cluster-api.sigs.k8s.io/developer/core/tilt>
- Prow, local development with Tilt: <https://docs.prow.k8s.io/docs/local-dev-tilt/>

Helm as the common denominator

- Knative, installing Knative: <https://knative.dev/docs/install/>
- cilium-cli v0.15.0 release notes: <https://github.com/cilium/cilium-cli/releases/tag/v0.15.0>
- Cilium, installation using Helm: <https://docs.cilium.io/en/stable/installation/k8s-install-helm/>
- CNCF Annual Cloud Native Survey 2024: <https://www.cncf.io/reports/cncf-annual-survey-2024/>
- Argo CD Helm and OCI sources: <https://argo-cd.readthedocs.io/en/latest/user-guide/helm/>,
  <https://argo-cd.readthedocs.io/en/latest/user-guide/oci/>
- Flux HelmRelease: <https://fluxcd.io/flux/components/helm/helmreleases/>

Install operators and indirection

- Istio in-cluster operator deprecation (Aug 2024):
  <https://istio.io/latest/blog/2024/in-cluster-operator-deprecation-announcement/>
- Istio, introducing the Sail Operator: <https://istio.io/latest/blog/2024/introducing-sail-operator/>
- John Howard, Istio installation: <https://blog.howardjohn.info/posts/istio-install/>
- GitLab Distribution, operator dependency on the Helm charts:
  <https://gitlab.com/gitlab-org/distribution/team-tasks/-/issues/990>
- The New Stack, when to avoid the operator pattern:
  <https://thenewstack.io/kubernetes-when-to-use-and-when-to-avoid-the-operator-pattern/>

OCM positioning

- OCM epic, deployer-agnostic deployment: <https://github.com/open-component-model/ocm-project/issues/1339>
- OCM, Helm and OCM tutorial: <https://ocm.software/docs/tutorials/build-deploy-applications-with-helm-and-ocm/>
- OCM spec, `helmChart` type: <https://github.com/open-component-model/ocm-spec/blob/main/doc/04-extensions/01-artifact-types/helmchart.md>

Configuration with code

- Kubebuilder project layout: <https://book.kubebuilder.io/cronjob-tutorial/basic-project>
- Kubebuilder `helm/v2-alpha` plugin: <https://book.kubebuilder.io/plugins/available/helm-v2-alpha>
- Argo CD best practices, config vs. source repos: <https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/>
- In-repo charts: [cert-manager](https://github.com/cert-manager/cert-manager/tree/master/deploy/charts/cert-manager),
  [Cilium](https://github.com/cilium/cilium/tree/main/install/kubernetes/cilium),
  [Linkerd](https://github.com/linkerd/linkerd2/tree/main/charts),
  [Kyverno](https://github.com/kyverno/kyverno/tree/main/charts),
  [External Secrets](https://github.com/external-secrets/external-secrets/tree/main/deploy/charts/external-secrets),
  [Loki](https://github.com/grafana/loki/tree/main/production/helm/loki)
- argo-helm: <https://github.com/argoproj/argo-helm>, issue #2093:
  <https://github.com/argoproj/argo-helm/issues/2093>
- Potvin & Levenberg, Google's monorepo, CACM 2016:
  <https://cacm.acm.org/research/why-google-stores-billions-of-lines-of-code-in-a-single-repository/>

Delivery principles

- Twelve-Factor, dev/prod parity: <https://12factor.net/dev-prod-parity>
- Humble & Farley, *Continuous Delivery*: <https://martinfowler.com/books/continuousDelivery.html>
- DORA, working in small batches: <https://dora.dev/capabilities/working-in-small-batches/>
