# GEP-68: Gateway API Extension for Gardener Shoot Clusters

## Table of Contents

- [GEP-68: Gateway API Extension for Gardener Shoot Clusters](#gep-68-gateway-api-extension-for-gardener-shoot-clusters)
  - [Table of Contents](#table-of-contents)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
  - [Proposal](#proposal)
      - [Deployment Topology: Seed vs Shoot](#deployment-topology-seed-vs-shoot)
    - [Selected Implementation: Envoy Gateway](#selected-implementation-envoy-gateway)
    - [Notes/Constraints/Caveats](#notesconstraintscaveats)
    - [Risks and Mitigations](#risks-and-mitigations)
  - [Design Details](#design-details)
    - [Extension Registration](#extension-registration)
    - [API](#api)
    - [Provider Modes](#provider-modes)
    - [Admission Webhook](#admission-webhook)
    - [Lifecycle Management](#lifecycle-management)
    - [Coexistence with `shoot-traefik`](#coexistence-with-shoot-traefik)
    - [Scope Restriction: Evaluation Shoots Only](#scope-restriction-evaluation-shoots-only)
  - [Evaluation of Gateway API Implementations](#evaluation-of-gateway-api-implementations)
    - [Evaluation Criteria](#evaluation-criteria)
    - [Candidates Considered](#candidates-considered)
      - [1. Envoy Gateway](#1-envoy-gateway)
      - [2. Traefik Gateway API](#2-traefik-gateway-api)
      - [3. Istio Ingress Gateway (with Gateway API)](#3-istio-ingress-gateway-with-gateway-api)
      - [4. Kgateway (formerly Gloo Gateway, k8sgateway)](#4-kgateway-formerly-gloo-gateway-k8sgateway)
      - [5. Cilium Gateway API](#5-cilium-gateway-api)
      - [6. Kong Gateway](#6-kong-gateway)
      - [7. NGINX Gateway Fabric](#7-nginx-gateway-fabric)
    - [Final Decision Matrix](#final-decision-matrix)
    - [Why Envoy Gateway Over Traefik Gateway API](#why-envoy-gateway-over-traefik-gateway-api)
  - [Future Enhancements](#future-enhancements)
  - [Drawbacks](#drawbacks)
  - [Alternatives](#alternatives)
    - [1. Do Nothing — Rely on `shoot-traefik` Plus User-Installed Gateway API](#1-do-nothing--rely-on-shoot-traefik-plus-user-installed-gateway-api)
    - [2. Pick Traefik Gateway API for Operational Reuse](#2-pick-traefik-gateway-api-for-operational-reuse)
    - [3. Pick Istio for Best Conformance](#3-pick-istio-for-best-conformance)
    - [4. Pick Kgateway for Best Performance](#4-pick-kgateway-for-best-performance)
    - [5. Cilium Gateway API for CNI Co-location](#5-cilium-gateway-api-for-cni-co-location)
    - [6. NGINX Gateway Fabric for "Familiar" NGINX Branding](#6-nginx-gateway-fabric-for-familiar-nginx-branding)
    - [7. Kong Gateway](#7-kong-gateway)
    - [8. Use Gateway API CRDs Without a Bundled Implementation](#8-use-gateway-api-crds-without-a-bundled-implementation)

## Summary

The Kubernetes ecosystem is converging on the [Gateway API](https://gateway-api.sigs.k8s.io/)
as the long-term successor to the `Ingress` resource. Gateway API graduated to GA
with v1.0 in October 2023 and has since received broad implementation support
across the CNCF landscape. With [GEP-57](../0057-replace-nginx-ingress-shoot-addon-with-traefik-extension/README.md)
already establishing a Traefik-based replacement for the retired Ingress NGINX
shoot addon, Gardener users now need a dedicated, first-class option for
Gateway API workloads in their shoot clusters.

After a detailed evaluation of all major open-source Gateway API
implementations — Envoy Gateway, Traefik, Istio, Kgateway, Cilium, Kong, and
NGINX Gateway Fabric — informed by the upstream conformance benchmark
[gateway-api-bench](https://github.com/howardjohn/gateway-api-bench), this GEP
recommends **[Envoy Gateway](https://gateway.envoyproxy.io/)** as the
implementation shipped to Gardener shoot clusters, with **Traefik Gateway API**
documented as the runner-up.

The extension follows the standard Gardener extension contract (controller
registration, `ManagedResource`-based deployment, admission webhooks) and is
deliberately scoped narrowly: it manages (installs and updates) the Gateway
API CRDs in the shoot, deploys the Envoy Gateway control plane and the Envoy
data-plane proxies, and installs a `GatewayClass` named `envoy-gateway` that
shoot owners can reference from their `Gateway` and `*Route` objects. (Envoy
Gateway's helm chart does not ship a `GatewayClass`; the extension creates
one, using the controller name
`gateway.envoyproxy.io/gatewayclass-controller`.)


## Motivation

Gardener's current ingress story is anchored on the legacy `Ingress` resource
and — going forward — on the Traefik-based replacement defined in GEP-57.
Both options serve the same purpose: a single resource type for HTTP host/path
routing. The shortcomings of `Ingress` are well known:

* The Kubernetes `Ingress` API has been frozen since 2020. New routing features
  (header rewrites, traffic splitting, mirroring, multi-protocol routing,
  cross-namespace references) cannot be expressed without controller-specific
  annotations, which destroy portability.
* No native L4 (TCP/TLS-passthrough) or gRPC routing.
* No clear separation of concerns between platform admins (who provision
  load balancers) and application teams (who attach routes).

Gateway API was designed by SIG-Network specifically to fix these issues. It
introduces a role-oriented set of resources:

* `GatewayClass` — declared by the infrastructure provider (analogous to
  `StorageClass`).
* `Gateway` — provisioned by the platform/cluster admin; describes the L4/L7
  listener (port, protocol, TLS, allowed namespaces).
* `HTTPRoute`, `GRPCRoute`, `TLSRoute`, `TCPRoute`, `UDPRoute` — owned by
  application teams; attach to a `Gateway` and describe routing rules.
* `ReferenceGrant` — explicit cross-namespace authorisation for backend
  references.

Key problems this GEP addresses:

* **Ecosystem convergence**: Most major service-mesh and ingress vendors
  (Istio, Cilium, Envoy Gateway, Traefik, Kong, NGINX, Kgateway, HAProxy)
  now implement Gateway API. Gardener users who do not have a supported
  Gateway API option in their shoots either roll their own (creating a
  fragmented, unsupported landscape) or stay on `Ingress` and accept its
  limitations.

* **Forward-compatible migration**: Workloads built today on the new Traefik
  ingress extension or on legacy NGINX `Ingress` resources will eventually
  need a path to Gateway API. Providing a first-class extension now means
  this migration can happen incrementally and per-shoot, instead of as a
  big-bang break.

* **Multi-tenancy and role separation**: Gardener shoots are commonly used
  by multiple teams. Gateway API's persona-based resource split aligns much
  better with how Gardener users organise platform vs. application
  ownership inside a shoot than annotation-driven `Ingress` does.

* **Vendor neutrality and conformance**: Gateway API has a published
  conformance suite. Selecting a conformant implementation gives Gardener
  users portable routing semantics — workloads written against `HTTPRoute`
  in a Gardener shoot will behave the same way in any other conformant
  cluster.

* **L4 + L7 unified**: Several Gardener users today bolt MetalLB or
  cloud-provider LBs in front of `Ingress` controllers to handle TCP/UDP.
  Gateway API's `TCPRoute`/`TLSRoute`/`UDPRoute` collapse this into a single
  programming model.

### Goals

1. Introduce `gardener-extension-shoot-envoy-gateway` as an extension in the
   [Gardener GitHub organisation](https://github.com/gardener).
2. Ship **Envoy Gateway** as the bundled Gateway API implementation. The
   extension installs a `GatewayClass` named `envoy-gateway` bound to the
   Envoy Gateway controller (`gateway.envoyproxy.io/gatewayclass-controller`);
   Envoy Gateway's helm chart does not ship a `GatewayClass` itself, so the
   extension is responsible for creating one. Shoot users reference it from
   their `Gateway` resources via `spec.gatewayClassName: envoy-gateway`.
3. Install (or reconcile) the Gateway API standard channel CRDs
   (`gateway.networking.k8s.io/v1`) in the shoot cluster.
4. Integrate with Gardener's standard resource-management, observability,
   and lifecycle mechanisms (`ManagedResource`, heartbeat, metrics, VPA/HPA).
5. Provide an opinionated baseline configuration that works out of the box
   on every cloud provider Gardener supports, while allowing users to
   override behaviour through the extension's `providerConfig`.
6. Enable safe, incremental rollout: restrict the extension initially to
   shoots with `purpose: evaluation`, mirroring the rollout strategy of
   GEP-57.
7. Document a clear migration path from `Ingress` (both legacy NGINX and
   the GEP-57 Traefik extension) to `HTTPRoute`.

### Non-Goals

1. This GEP does **not** propose migrating Gardener core
   (`gardener/gardener`) to Gateway API. Gardener core is currently in the
   process of replacing its built-in NGINX-based ingress with another
   `Ingress`-based solution; an internal switch from `Ingress` to Gateway
   API is not on the roadmap and is not what this GEP is about. This GEP
   strictly concerns user-facing ingress *inside* the shoot cluster.
2. This GEP does **not** deprecate or remove the GEP-57
   `gardener-extension-shoot-traefik` extension. Both extensions are
   designed to coexist; an operator can offer either, both, or neither.
3. This GEP does **not** prescribe a service mesh. Gateway API has a separate
   GAMMA initiative for east-west mesh routing; this is out of scope here.
4. This GEP does **not** introduce multiple competing `GatewayClass`
   objects. The extension installs a single `GatewayClass` named
   `envoy-gateway` bound to the Envoy Gateway controller. Users may install
   additional `GatewayClass` objects pointing at other implementations
   independently of this extension.
5. This GEP does **not** cover network policy, mTLS automation, or
   advanced traffic policies (rate limiting, JWT auth, WAF). These can be
   layered on top via the chosen implementation's policy CRDs but are not
   exercised by the extension itself in the initial release.


## Proposal

Introduce `gardener-extension-shoot-envoy-gateway` as a new extension in the
Gardener GitHub organisation. The extension follows the well-established
[Gardener Extension Concept](https://gardener.cloud/docs/gardener/extensions/)
and implements the `Extension` reconciler contract.

When enabled on a Shoot, the extension:

1. Reconciles `ManagedResource` objects in the shoot's control-plane namespace
   on the seed. The `gardener-resource-manager` then applies the contained
   manifests into the shoot cluster (CRDs, RBAC, the Envoy Gateway control-plane
   Deployment, the `envoy-gateway` `GatewayClass`, etc.).
2. Registers an admission webhook that validates `Shoot` objects enabling the
   extension to enforce the evaluation-purpose scope constraint and to
   validate the `EnvoyGatewayConfig` provider config.
3. Exposes a `/metrics` endpoint on the extension controller for Gardener's
   monitoring stack to scrape.
4. Participates in the Gardener heartbeat protocol to report extension health.

The extension type identifier is **`shoot-envoy-gateway`** (referenced in
`spec.extensions[].type` of the Shoot manifest).

#### Deployment Topology: Seed vs Shoot

To avoid ambiguity about *what runs where*, the table below lists every
component the extension is responsible for and the cluster it ends up in.

| Component | Cluster | Notes |
|-----------|---------|-------|
| `gardener-extension-shoot-envoy-gateway` controller | Seed (per seed) | Watches `Extension` objects of type `shoot-envoy-gateway` and reconciles `ManagedResource` objects. This is the Gardener-side "operator". |
| Admission webhook | Garden runtime / virtual garden | Validates `Shoot` resources at admission time. Standard Gardener admission deployment pattern. |
| Gateway API CRDs | Shoot | Standard channel `gateway.networking.k8s.io/v1`; optionally experimental channel. Delivered via `ManagedResource`. |
| Envoy Gateway CRDs (`EnvoyProxy`, `BackendTrafficPolicy`, `ClientTrafficPolicy`, `SecurityPolicy`, …) | Shoot | Required for the Envoy Gateway control plane to function. |
| Envoy Gateway **control plane** (Deployment, Service, RBAC, PDB, optional VPA/HPA) | Shoot | Runs as Pods inside the shoot. Translates `Gateway`/`*Route` resources into Envoy xDS configuration. |
| `GatewayClass` (`envoy-gateway`) | Shoot | Created by this extension via `ManagedResource`. Bound to controller `gateway.envoyproxy.io/gatewayclass-controller`. Envoy Gateway's helm chart does not ship a `GatewayClass`, so the extension provides one. |
| Envoy **data plane** (proxy Pods) | Shoot | Spawned by the Envoy Gateway control plane in response to user-created `Gateway` objects. Each `Gateway` gets its own Envoy Deployment + Service inside the shoot. |
| LoadBalancer `Service` per `Gateway` | Shoot | Provisioned by the cloud-provider load-balancer controller running inside the shoot, exactly like an `Ingress`-mode LB today. |

There are deliberately **no components running in the seed control plane on
behalf of the data path**. The seed only hosts the Gardener extension
controller. All ingress traffic, all xDS reconciliation, and all CRD storage
is shoot-local.

### Selected Implementation: Envoy Gateway

Following the evaluation in [Evaluation of Gateway API Implementations](#evaluation-of-gateway-api-implementations),
the extension ships **Envoy Gateway** as the implementation behind the
`envoy-gateway` `GatewayClass`. Envoy Gateway was selected over Traefik
Gateway API and the other candidates because of:

* **Performance**: ~325k qps in the upstream
  [gateway-api-bench](https://github.com/howardjohn/gateway-api-bench) at 512
  connections, ~1.5x Traefik's throughput on the same hardware.
* **Architectural cleanliness**: Proper control-plane / data-plane separation,
  per-namespace Gateway isolation that respects the Gateway API spec.
* **Conformance**: Full support for the Gateway API standard channel; no
  reported correctness issues in the upstream benchmark beyond a known
  memory leak under churn (see [Risks and Mitigations](#risks-and-mitigations)).
* **Foundation**: Built on Envoy, the same data plane used by Istio,
  Kgateway, and most large-scale production gateway deployments — meaning
  the data path itself is the most battle-tested codebase in the field.
* **CNCF stewardship**: Hosted under the Envoy organisation in CNCF, with
  multi-vendor maintainers — no single-vendor lock-in.

Traefik Gateway API was a strong runner-up but rejected for two concrete
reasons documented in the upstream benchmark: (1) it consolidates Gateways
across namespaces into a shared data-plane process, which violates the
Gateway API spec's namespace isolation expectations and creates a noisy-
neighbour risk in multi-tenant shoots; and (2) its status reporting is slow
(~180 seconds for status updates on large route sets) and it fails to apply
large route volumes — both deal-breakers for shoots with many application
teams.

### Notes/Constraints/Caveats

* **Gateway API CRDs are installed cluster-wide.** The extension installs the
  Gateway API standard-channel CRDs (`gateway.networking.k8s.io/v1`:
  `GatewayClass`, `Gateway`, `HTTPRoute`, `GRPCRoute`, `ReferenceGrant`).
  These CRDs are non-namespaced and will exist in any shoot where the
  extension is enabled, even if no `Gateway` resource is created.

* **Experimental channel is opt-in.** The `gateway.networking.k8s.io/v1alpha2`
  experimental CRDs (`TCPRoute`, `TLSRoute`, `UDPRoute`, `BackendTLSPolicy`)
  are not installed by default. They can be enabled via
  `spec.experimentalFeatures: true` in `EnvoyGatewayConfig`. Operators should
  be aware that experimental APIs may change in backwards-incompatible ways.

* **Envoy Gateway CRDs (e.g. `EnvoyProxy`, `BackendTrafficPolicy`,
  `ClientTrafficPolicy`, `SecurityPolicy`) are also installed.** These are
  namespaced and required for advanced Envoy-specific configuration. Users
  who only consume the standard Gateway API surface can ignore them.

* **`GatewayClass` is `envoy-gateway`.** The extension installs a single
  `GatewayClass` named `envoy-gateway` bound to controller
  `gateway.envoyproxy.io/gatewayclass-controller`. Envoy Gateway's helm
  chart does not create a `GatewayClass` on its own, so the extension is
  responsible for it. Users reference it from their `Gateway` objects via
  `spec.gatewayClassName: envoy-gateway`.

* **Coexistence with shoot-traefik.** Both the GEP-57 Traefik ingress
  extension and this extension can be installed in the same shoot. They
  do not conflict at the IngressClass / GatewayClass level. See
  [Coexistence with `shoot-traefik`](#coexistence-with-shoot-traefik).

* **Scope is currently restricted to `purpose: evaluation` shoots.** This
  constraint is enforced by the admission webhook and exists to allow the
  extension to mature in low-risk environments before being opened up to
  development and production shoots.

### Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Envoy Gateway memory leak under route/config churn (reported in [gateway-api-bench](https://github.com/howardjohn/gateway-api-bench)) | Medium | Medium | The control-plane pod is configured with a memory limit and a VPA `Recreate` updatePolicy so leaked memory is reclaimed by periodic restart. The issue is actively tracked upstream; the extension pins to a release where mitigations have landed. |
| Initial-request errors during bootstrap (also reported upstream) | Medium | Low | Readiness gates on the data-plane Envoy pods; documentation advises users to use a startup-probe-based health check from their LB. |
| Gateway API CRD conflicts if user pre-installed them | Low | High | CRDs are part of the shoot `ManagedResource` and applied via server-side apply; existing CRDs are updated idempotently without field-ownership conflicts. The extension's CRD reconciliation is opt-out via `spec.manageCRDs: false` for users who manage CRDs themselves. |
| Limited annotation/feature parity with NGINX or Traefik Ingress | High | Medium | Documented migration guide: most NGINX `Ingress` annotations have a direct `HTTPRoute` filter or Envoy `BackendTrafficPolicy` equivalent. Users requiring features not yet expressible in standard Gateway API are advised to remain on the Traefik extension until the experimental channel covers their case. |
| Operators end up with three concurrent ingress paths in one shoot (legacy NGINX, Traefik, Gateway API) | Medium | Medium | Documentation strongly recommends a single ingress path per shoot in production. The evaluation-purpose scope of both extensions limits the blast radius during the rollout phase. |
| Ecosystem churn: Gateway API adds new GA features in v1.2 / v1.3 | High | Low | The extension declares conformance against a specific Gateway API release in its release notes and bumps deliberately, not automatically. |


## Design Details

### Extension Registration

The extension is installed as Gardener resources, either as `Extension`:

```yaml
# Extension
apiVersion: operator.gardener.cloud/v1alpha1
kind: Extension
metadata:
  name: gardener-extension-shoot-envoy-gateway
spec:
  deployment:
    admission:
      runtimeCluster:
        helm:
          ociRepository:
            ref: europe-docker.pkg.dev/gardener-project/releases/charts/gardener/extensions/admission-shoot-envoy-gateway-runtime:latest
        values:
          image:
            repository: europe-docker.pkg.dev/gardener-project/releases/gardener/extensions/gardener-extension-shoot-envoy-gateway
            tag: latest
      virtualCluster:
        helm:
          ociRepository:
            ref: europe-docker.pkg.dev/gardener-project/releases/charts/gardener/extensions/admission-shoot-envoy-gateway-application:latest
    extension:
      helm:
        ociRepository:
          ref: europe-docker.pkg.dev/gardener-project/releases/charts/gardener/extensions/gardener-extension-shoot-envoy-gateway:latest
      values:
        image:
          repository: europe-docker.pkg.dev/gardener-project/releases/gardener/extensions/gardener-extension-shoot-envoy-gateway
          tag: latest
        replicaCount: 1
        resources:
          requests:
            cpu: 50m
            memory: 192Mi
        vpa:
          enabled: true
          resourcePolicy:
            minAllowed:
              memory: 128Mi
          updatePolicy:
            updateMode: Recreate
  resources:
  - clusterCompatibility:
    - shoot
    kind: Extension
    lifecycle:
      delete: BeforeKubeAPIServer
      migrate: AfterKubeAPIServer
      reconcile: AfterKubeAPIServer
    type: shoot-envoy-gateway
    workerlessSupported: false
```

or as `ControllerDeployment` and `ControllerRegistration`:

```yaml
# ControllerDeployment
apiVersion: core.gardener.cloud/v1beta1
kind: ControllerDeployment
metadata:
  name: gardener-extension-shoot-envoy-gateway
helm:
  rawChart: <base64-encoded Helm chart>
```

```yaml
# ControllerRegistration
apiVersion: core.gardener.cloud/v1beta1
kind: ControllerRegistration
metadata:
  name: shoot-envoy-gateway
spec:
  resources:
    - kind: Extension
      type: shoot-envoy-gateway
      globallyEnabled: false   # opt-in per shoot
      lifecycle:
        reconcile: AfterKubeAPIServer
        delete:   BeforeKubeAPIServer
  deployment:
    deploymentRefs:
      - name: gardener-extension-shoot-envoy-gateway
```

The extension controller is deployed per seed and watches `Extension` objects
of type `shoot-envoy-gateway`.

### API

The extension introduces a new API group `envoy-gateway.extensions.gardener.cloud`
with a single versioned kind `EnvoyGatewayConfig`:

```yaml
apiVersion: envoy-gateway.extensions.gardener.cloud/v1alpha1
kind: EnvoyGatewayConfig
spec:
  # Number of Envoy Gateway control-plane replicas (default: 2)
  controlPlaneReplicas: 2

  # Number of Envoy data-plane (proxy) replicas per Gateway (default: 2)
  dataPlaneReplicas: 2

  # Log level for both control-plane and data-plane: debug | info | warn | error (default: info)
  logLevel: info

  # Manage (install and update) the Gateway API CRDs. Set to false if CRDs
  # are owned externally. (default: true)
  manageCRDs: true

  # Install the experimental-channel Gateway API CRDs (TCPRoute, TLSRoute, UDPRoute,
  # BackendTLSPolicy). (default: false)
  experimentalFeatures: false

  # Optional pinned EnvoyProxy template applied to every Gateway via the
  # `gateway.envoyproxy.io/v1alpha1.EnvoyProxy` reference.
  envoyProxyDefaults:
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
    accessLogging: true
```

This object is embedded as `providerConfig` in the Shoot's
`spec.extensions[].providerConfig` field. Internal type conversion and
defaulting are handled by the API machinery registered under
`envoy-gateway.extensions.gardener.cloud/v1alpha1`.

### Provider Modes

The extension does not expose multiple back-end providers. Unlike the
`shoot-traefik` extension, where `KubernetesIngress` vs `KubernetesIngressNGINX`
modes serve a clear migration purpose, Gateway API is a single, well-defined
spec — there is no analogous compatibility shim to model. The implementation
behind the `envoy-gateway` `GatewayClass` is always Envoy Gateway in this release.

A user wishing to run a different implementation (for example Istio for mesh
integration) can do so independently of this extension by installing their
own `GatewayClass`. Such side-by-side installations are out of scope for the
extension's own validation but are not blocked.

### Admission Webhook

A `ValidatingWebhookConfiguration` is registered at path
`/webhooks/validate-shoot-envoy-gateway`. It validates:

1. **Purpose restriction**: The extension may only be enabled (`disabled:
   false`) on Shoots whose `spec.purpose` is `evaluation`. Adding the
   extension to a non-evaluation shoot is rejected with a descriptive error.
2. **Provider config schema**: If a non-nil `providerConfig` is supplied, it
   must decode successfully as an `EnvoyGatewayConfig` object. Unknown fields
   cause a validation error (strict decoding).
3. **Field constraints**: `spec.controlPlaneReplicas` and
   `spec.dataPlaneReplicas` must be ≥ 1; `spec.logLevel` must be one of the
   accepted enum values.

### Lifecycle Management

The extension interacts with Gardener's lifecycle protocol as follows:

| Phase | Behaviour |
|-------|-----------|
| Reconcile | Creates/updates a `ManagedResource` in the shoot namespace on the seed with all shoot-cluster resources (Gateway API CRDs, Envoy Gateway CRDs, control-plane Deployment, Service, RBAC, PDB, optional HPA/VPA, and the `envoy-gateway` `GatewayClass`). Waits for the `ManagedResource` to become healthy before marking the `Extension` as reconciled. |
| Delete (extension disabled, shoot kept) | Refuses to remove the extension while user-owned `Gateway` objects still exist in the shoot (to prevent silent traffic loss). The admission webhook surfaces this as a validation error on the `Shoot` update. Once the user has cleaned up their `Gateway`/`*Route` objects, the `ManagedResource` is deleted and the extension waits up to 5 minutes for managed objects to disappear. |
| Delete (shoot deletion) | Shoot deletion bypasses the "Gateways still exist" guard — the entire shoot is going away anyway, so blocking would only leak the shoot. The extension's `Extension` resource is reconciled with `lifecycle.delete: BeforeKubeAPIServer`, so the `ManagedResource` (Envoy Gateway control plane, CRDs, the `envoy-gateway` `GatewayClass`, EnvoyProxy/HTTPRoute/Gateway instances) is torn down before the shoot's API server is removed. The cloud-provider `LoadBalancer` Services that were created for each `Gateway` are deleted as part of the shoot's normal `Service` cleanup, freeing the underlying load balancers. No manual cleanup of `Gateway` objects is required from the user. |
| Heartbeat | Extension controller participates in the Gardener heartbeat protocol and reports liveness. |
| Metrics | Prometheus metrics are exposed on port 8080 under `/metrics`; Gardener's monitoring stack can scrape them via `ServiceMonitor`. The Envoy data-plane and Envoy Gateway control-plane also expose Prometheus metrics that are scraped via separate `ServiceMonitor` objects. |
| VPA/HPA | Optional VPA and HPA manifests for the control-plane and data-plane pods are provided in the Helm chart (VPA enabled by default, HPA opt-in). |

### Coexistence with `shoot-traefik`

A shoot may have both `shoot-traefik` and `shoot-envoy-gateway` enabled. The
two extensions reconcile disjoint resources:

| Resource | Owner |
|----------|-------|
| `IngressClass: nginx` / `IngressClass: traefik` | `shoot-traefik` |
| `GatewayClass: envoy-gateway` | `shoot-envoy-gateway` (registered by Envoy Gateway itself) |
| Traefik Deployment, IngressRoute CRDs | `shoot-traefik` |
| Envoy Gateway Deployment, Gateway API CRDs, EnvoyProxy CRDs | `shoot-envoy-gateway` |

There is no IP/port conflict at the `Service` level because each extension
provisions its own `LoadBalancer` Service. Nonetheless, running both in
parallel doubles the LB cost and the cognitive load; documentation will
recommend choosing one path per shoot in production.

### Scope Restriction: Evaluation Shoots Only

The current restriction to `purpose: evaluation` shoots is a deliberate
safety measure adopted during the initial incubation phase of the extension.
This allows:

* Early adopters to validate the extension's behaviour in evaluation
  environments.
* The maintainer team to gather operational feedback (in particular the
  Envoy Gateway memory-leak behaviour under realistic route churn).

The restriction is enforced exclusively in the admission webhook and can be
removed or made configurable by operators without any API change.


## Evaluation of Gateway API Implementations

This section documents the comparative evaluation that led to the selection
of Envoy Gateway as the implementation shipped by the extension.

The evaluation draws on the upstream
[gateway-api-bench](https://github.com/howardjohn/gateway-api-bench)
benchmark by John Howard (Istio maintainer), which evaluates seven
implementations on conformance, scale, and performance.

### Evaluation Criteria

| # | Criterion | Why it matters for Gardener |
|---|-----------|------------------------------|
| C1 | **Conformance** to the Gateway API standard channel | Portability for shoot users; no surprise behaviour |
| C2 | **Performance** under realistic load (qps, p99 latency, CPU/memory) | Shoots are multi-tenant; data-plane efficiency directly affects cost |
| C3 | **Architectural correctness**: control- vs data-plane separation, per-namespace Gateway isolation | Multi-team shoots require strong tenant isolation |
| C4 | **Status reporting timeliness** | Slow status updates break GitOps workflows |
| C5 | **Scale**: number of Gateways, Routes, backends supported on a single control plane | A shoot can host hundreds of routes |
| C6 | **Project governance**: vendor neutrality, CNCF status, multi-vendor maintainers | Avoid vendor lock-in |
| C7 | **Operational footprint**: control-plane components, dependency on a service mesh, CRD count | Operability inside a Gardener shoot |
| C8 | **NGINX/Ingress migration story**: support for users coming from `Ingress` | Backwards compatibility with GEP-57 Traefik ingress users |
| C9 | **L4 routing** (`TCPRoute`, `TLSRoute`, `UDPRoute`) | Gardener users running databases, gRPC, custom protocols |
| C10 | **Already in use within Gardener** | Reuse operational knowledge |

### Candidates Considered

The candidate list includes every implementation that ships a usable
Gateway API standard-channel controller as of late 2025. Implementations
shipping only the Ingress API are excluded.

#### 1. Envoy Gateway

Selected. See [Why Envoy Gateway Over Traefik Gateway API](#why-envoy-gateway-over-traefik-gateway-api).

* **Throughput** (gateway-api-bench, 512 conns): ~325k qps
* **Architecture**: Clean control-plane (Go) / data-plane (Envoy) split.
  Per-namespace Gateway isolation respected.
* **Conformance**: Full standard channel, partial experimental channel.
  No correctness issues reported in the upstream benchmark beyond a memory
  leak under churn.
* **Governance**: CNCF, hosted under the Envoy organisation. Multi-vendor
  maintainers (Tetrate, VMware, AIS, Bloomberg, others).
* **L4**: `TCPRoute`, `TLSRoute`, `UDPRoute` all supported.
* **Dependencies**: None beyond Kubernetes and the Gateway API CRDs.

#### 2. Traefik Gateway API

Runner-up. Rejected primarily on architectural correctness in multi-tenant
scenarios, on slow status reconciliation, and on benchmarked failures with
large route volumes.

* **Throughput** (gateway-api-bench): ~217k qps — solid mid-pack.
* **Architecture concern**: The upstream benchmark notes that Traefik
  *"consolidates all Gateways across namespaces unsafely"*, meaning a
  single Traefik process serves Gateways from multiple namespaces. In a
  multi-tenant Gardener shoot, this is a tenant-isolation violation.
* **Status reporting**: ~180 seconds latency to reflect status on large route
  sets — incompatible with GitOps tooling that polls for `Ready` conditions.
* **Scale failure**: Reported failure to apply large route volumes. The
  extension would need to publish hard upper bounds on routes per shoot.
* **Migration story**: Best-in-class for users coming from NGINX `Ingress`,
  thanks to its `KubernetesIngressNGINX` provider — but this is the *Ingress*
  surface, not the Gateway API surface, so it does not benefit Gateway API
  workloads. For Ingress migration, GEP-57 already covers this.
* **Already in Gardener**: Yes, via GEP-57. This is a real plus for operational
  reuse, but not enough to outweigh the architectural and scale issues for
  the Gateway API path.
* **Governance**: Single-vendor (Traefik Labs). Open source, but no
  CNCF status.

#### 3. Istio Ingress Gateway (with Gateway API)

Strong technical fit but rejected on operational footprint.

* **Throughput** (gateway-api-bench): ~325k qps — tied with Envoy Gateway.
* **Architecture**: The upstream benchmark concludes *"no issues were found"*
  with Istio's Gateway API implementation — the most stable result of the
  field.
* **Operational ergonomics**: Istio auto-creates the `istio` `GatewayClass`
  on its own once installed; no manual `GatewayClass` provisioning is needed.
  (Envoy Gateway behaves the same way with `envoy-gateway`.)
* **Why rejected**: Istio bundles a full service mesh (sidecars, mTLS, mesh
  control plane). Installing it merely to get a Gateway API ingress imposes
  a significant operational footprint on every shoot. Users who *want* a
  mesh in their shoot can install Istio independently; its `istio`
  `GatewayClass` will be registered automatically and they can use it from
  their `Gateway` resources. This extension targets the lighter, mesh-free
  use case.

#### 4. Kgateway (formerly Gloo Gateway, k8sgateway)

Best raw throughput in the benchmark but ruled out on maturity and
governance.

* **Throughput** (gateway-api-bench): ~400k qps — the highest of the field.
* **Architecture**: Clean separation, Envoy-based data plane, no
  correctness issues reported.
* **Why rejected**: The project rebranded from Gloo Gateway to Kgateway in
  2024 and entered the CNCF as a sandbox project shortly after. The CNCF
  sandbox stage is too early for a default Gardener shipping decision.
  Re-evaluation is appropriate once the project graduates to incubation.

#### 5. Cilium Gateway API

Rejected on performance and configuration robustness.

* **Throughput** (gateway-api-bench): ~22k qps — by far the lowest of the
  field, ~15x slower than Envoy Gateway.
* **Architecture concern**: The upstream benchmark notes *"configuration
  updates fail silently beyond 1.5mb"*, meaning large route sets
  effectively stop reconciling without surfacing an error.
* **Why considered at all**: Tight Cilium integration is attractive for
  shoots that already use Cilium as their CNI. The extension does not
  preclude users from installing the Cilium `GatewayClass` independently;
  it just doesn't use it as the default.

#### 6. Kong Gateway

Rejected on correctness and namespace isolation.

* **Throughput** (gateway-api-bench): ~104k qps — below Traefik.
* **Correctness concerns**: The upstream benchmark notes that Kong
  *"incorrectly reports route counts"* and *"consolidates Gateways
  unsafely"* across namespaces — the same multi-tenant isolation violation
  flagged for Traefik, with the additional issue of incorrect status
  reporting.
* **Governance**: Kong Inc. with a CNCF-adjacent ecosystem. Not blocking,
  but combined with the correctness issues this candidate did not advance.

#### 7. NGINX Gateway Fabric

Rejected on performance and stability.

* **Throughput** (gateway-api-bench): ~91k qps — roughly 3.5–4.5x lower
  than the Envoy-based front-runners (Envoy Gateway ~325k qps, Kgateway
  ~400k qps), and behind Traefik (~217k qps) and Kong (~104k qps) as well.
  In other words: NGINX Gateway Fabric is the second-slowest implementation
  in the field, only ahead of Cilium.
* **Stability concern**: The upstream benchmark reports that NGINX Gateway
  Fabric *"crashes during route changes"* — disqualifying for a
  multi-tenant shoot ingress.
* **Migration story**: Despite the NGINX brand, NGINX Gateway Fabric is a
  separate codebase from Ingress NGINX and does not provide annotation
  compatibility with the retired `ingress-nginx` controller. So the apparent
  migration benefit does not materialise.

### Final Decision Matrix

Score: **+** good, **o** acceptable, **−** problematic.

| Criterion | Envoy GW | Traefik | Istio | Kgateway | Cilium | Kong | NGINX GF |
|-----------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| C1 Conformance | + | + | + | + | o | o | o |
| C2 Performance | + | o | + | + | − | o | − |
| C3 Architecture / tenant isolation | + | − | + | + | o | − | o |
| C4 Status reporting | + | − | + | + | − | − | o |
| C5 Scale | + | − | + | + | − | o | − |
| C6 Governance / CNCF | + | o | + | o | + | o | + |
| C7 Operational footprint | + | + | − | + | + | o | + |
| C8 Ingress migration | o | + | o | o | o | o | − |
| C9 L4 routing | + | + | + | + | + | + | o |
| C10 Already in Gardener | − | + | o | − | o | − | − |
| **Result** | **Selected** | **Runner-up** | Rejected (footprint) | Rejected (maturity) | Rejected (perf) | Rejected (correctness) | Rejected (perf/stability) |

### Why Envoy Gateway Over Traefik Gateway API

The decision between Envoy Gateway and Traefik Gateway API is the core
trade-off this GEP resolves. Both are credible options. The argument for
Traefik is operational reuse — Gardener already ships a Traefik-based
ingress extension via GEP-57, and a Traefik Gateway API extension would let
operators consolidate on a single binary. The argument for Envoy Gateway is
that it is the technically better Gateway API implementation in every axis
that matters for a multi-tenant managed Kubernetes service. Specifically:

1. **Tenant isolation.** Gardener shoots are routinely shared by multiple
   teams, each with their own namespaces. Gateway API was explicitly designed
   so that a `Gateway` in namespace A and a `Gateway` in namespace B are
   isolated control surfaces. Envoy Gateway implements this faithfully:
   each `Gateway` gets its own data-plane Envoy instance. Traefik
   consolidates them into a shared process — a noisy-neighbour and
   blast-radius problem that contradicts Gateway API's design intent. For a
   product that markets itself as a multi-tenant Kubernetes platform,
   tenant isolation must be non-negotiable.

2. **Performance under load.** ~325k qps vs ~217k qps in the same benchmark
   is a 1.5x advantage. In a managed-service context, where data-plane
   resource cost is borne by every shoot, that margin compounds.

3. **Status reconciliation.** The 180-second status reconciliation latency
   reported for Traefik on large route sets is incompatible with GitOps
   workflows (Argo CD / Flux) that wait for `Ready` conditions before
   declaring a sync successful. Envoy Gateway reconciles in seconds.

4. **Scale failures.** Traefik failed to apply *all* routes in the upstream
   benchmark's largest scale test. Envoy Gateway did not. We must not ship
   a default that breaks at the upper end of expected production load.

5. **Forward path.** Envoy is the data plane behind Istio, Kgateway, Google
   Cloud Service Mesh, AWS App Mesh, and most major service-mesh products.
   Operational knowledge built on Envoy Gateway transfers to those
   neighbouring products. Traefik knowledge does not.

The remaining argument in Traefik's favour — that GEP-57 already runs
Traefik in shoots — does not carry over to the Gateway API surface. The
GEP-57 extension exists to provide a smooth migration off the retired
NGINX Ingress controller, and its value lies in NGINX annotation
translation, which is wholly an Ingress-API feature. Gateway API users do
not benefit from that translation. Therefore the operational-reuse argument
applies only to the binary, not to the user-facing API — and even there,
the two extensions can coexist (see
[Coexistence with `shoot-traefik`](#coexistence-with-shoot-traefik)).

For these reasons the extension ships Envoy Gateway. Operators who prefer
Traefik for the Gateway API surface can install it independently in their
shoots and register a separate `GatewayClass`; this GEP does not block
them.


## Future Enhancements

* **Promote out of `purpose: evaluation`.** Once the extension has soaked in
  evaluation shoots and the upstream Envoy Gateway memory-leak issue is
  resolved, the admission-webhook restriction will be relaxed to allow
  `development` and `production` shoots.

* **Feature gates** for opt-in experimental Gateway API features
  (`TCPRoute`, `BackendTLSPolicy`, mesh GAMMA bindings) without requiring a
  new extension release.

* **Implementation-version decoupling.** In the initial release, the Envoy
  Gateway version is pinned in the extension binary. A future iteration
  will allow operators to manage a catalog of Envoy Gateway versions with
  lifecycle classifications (aligned with
  [GEP-32](../0032-version-classification-lifecycle/README.md)) and let
  shoot owners pin a specific version.

* **GatewayClass parameters.** Expose a curated set of `EnvoyProxy`
  template fields (resources, access logs, tracing, rate-limit backend)
  through `EnvoyGatewayConfig` so shoot owners can tune common knobs without
  hand-writing `EnvoyProxy` objects.

* **Migration tooling.** A helper command (`gardenctl ingress-to-gateway`)
  that converts NGINX-style `Ingress` and Traefik `IngressRoute` objects
  into equivalent `Gateway` + `HTTPRoute` resources for users migrating off
  GEP-57.

* **Re-evaluation of Kgateway.** Once Kgateway graduates from CNCF sandbox,
  re-run the evaluation. Its raw throughput (~400k qps) is the highest of
  the field, and it is built on the same Envoy data plane. Note that any
  potential adoption would *not* be a drop-in switch: although both
  implementations sit on Envoy, their CRDs (`EnvoyProxy`,
  `BackendTrafficPolicy`, `SecurityPolicy`, …) are not API-compatible, and
  shoot users who rely on Envoy-Gateway-specific extensions would need a
  documented migration path. A switch is therefore conditional on (a)
  Kgateway graduating to CNCF incubation and (b) a clear plan for migrating
  existing user-facing configuration. If those preconditions are not met, the
  extension stays on Envoy Gateway.


## Drawbacks

* **Two ingress extensions to maintain.** Gardener will end up with both
  `shoot-traefik` (Ingress API) and `shoot-envoy-gateway` (Gateway API).
  This doubles the maintenance surface and creates a "which one do I
  use?" question for new users. Documentation will need to address this
  clearly, and there is a real risk of operator confusion in the
  transition window.

* **Envoy Gateway is younger than Traefik.** Envoy Gateway 1.0 shipped in
  2024. Although Envoy itself is mature, the Gateway-API-specific control
  plane has fewer production-years behind it than Traefik. The known
  memory-leak issue is one symptom of this.

* **CRD proliferation.** The extension installs Gateway API CRDs (5
  standard-channel + up to 4 experimental) and Envoy Gateway CRDs (5+).
  Even a user who only deploys a single `HTTPRoute` will see >10 CRDs
  appear in their shoot. This is unavoidable for any Gateway API
  implementation and is shared with all Gateway API extensions.

* **Coexistence cost.** Running both `shoot-traefik` and
  `shoot-envoy-gateway` in the same shoot doubles the load-balancer cost.
  Operators need to communicate this to shoot owners.

* **No annotation-compat shim for Gateway API.** Unlike `shoot-traefik`'s
  `KubernetesIngressNGINX` mode, this extension does not translate `Ingress`
  resources or NGINX annotations into `HTTPRoute`. Users migrating from
  `Ingress` will need to rewrite their routing manifests. A future
  migration tool is mentioned in [Future Enhancements](#future-enhancements).


## Alternatives

### 1. Do Nothing — Rely on `shoot-traefik` Plus User-Installed Gateway API

Users who want Gateway API in their shoot today install one of the
implementations themselves. This works but produces a fragmented landscape:
every team picks a different implementation, none of them benefit from the
extension's lifecycle integration (`ManagedResource`, heartbeat,
observability hooks), and there is no curated, supported default. **Rejected.**

### 2. Pick Traefik Gateway API for Operational Reuse

The argument: GEP-57 already ships Traefik; ship the same binary in Gateway
API mode and consolidate operational knowledge. **Rejected** for the
reasons spelled out in
[Why Envoy Gateway Over Traefik Gateway API](#why-envoy-gateway-over-traefik-gateway-api):
tenant-isolation violations, slow status reconciliation, and scale failures.

### 3. Pick Istio for Best Conformance

The argument: Istio's Gateway API is the most mature and stable in the
upstream benchmark. **Rejected** because it requires installing a full
service mesh (sidecars, mesh control plane, mTLS infrastructure) into
every shoot that wants Gateway API ingress. The operational footprint is
disproportionate. Operators who already run Istio for mesh purposes can
add an `istio` `GatewayClass` independently.

### 4. Pick Kgateway for Best Performance

The argument: ~400k qps, clean architecture, no reported correctness
issues. **Rejected** because Kgateway is a recently-renamed CNCF sandbox
project. Sandbox is too early for a default Gardener shipping decision.
Reconsidered as a future enhancement once it graduates to incubation.

### 5. Cilium Gateway API for CNI Co-location

The argument: shoots using Cilium as CNI could collapse CNI and Gateway
into a single component. **Rejected** because of the upstream benchmark's
performance numbers (~22k qps) and the silent-failure behaviour beyond
1.5mb of configuration. These are blocking for production shoots.

### 6. NGINX Gateway Fabric for "Familiar" NGINX Branding

The argument: continuity for users coming from Ingress NGINX. **Rejected.**
NGINX Gateway Fabric is a separate codebase from the retired
`ingress-nginx`, so there is no continuity in practice — and the upstream
benchmark numbers put it 3.5–4.5x below the Envoy-based front-runners and
report that it crashes during route changes.

### 7. Kong Gateway

**Rejected** on the namespace consolidation issue (same multi-tenant
isolation problem as Traefik) plus incorrect route-count reporting,
both flagged in the upstream benchmark.

### 8. Use Gateway API CRDs Without a Bundled Implementation

The argument: install only the Gateway API CRDs and a `GatewayClass`
registry; let users pick their implementation. **Rejected** because it
provides no working ingress out of the box, contradicting the goal of a
turnkey extension. This is functionally equivalent to "do nothing" for
most users.
