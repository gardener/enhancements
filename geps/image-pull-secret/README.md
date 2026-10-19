# Image Pull Secret Propagation

## Summary

This proposal introduces a mechanism for propagating `kubernetes.io/dockerconfigjson`
secrets from the virtual garden cluster all the way to shoot cluster `kube-system` namespaces and
to every running workload pod, enabling fully automated private-registry support across the
Gardener stack.

---

## Motivation

Gardener operators increasingly run deployments in environments where all container images are
served from private registries that require authentication — internal mirrors, air-gapped setups,
or enterprise artifact registries with access control. Today, Gardener provides no mechanism to
configure pull credentials for those registries.

The `imageVectorOverwrite` and `componentImageVectorOverwrites` values allow operators to redirect
image pulls to a custom registry. However, they accept no credentials. The implicit requirement is
that any registry pointed to by the image vector must be publicly accessible — which defeats the
purpose of a private registry and is unacceptable in many enterprise and regulated environments.

The problem spans every layer of the Gardener stack:

- **Gardener components on the seed** (etcd-druid, GRM, kube-apiserver, etc.) run as pods on the
  seed cluster and must pull their images from the configured registry.
- **System components in shoot `kube-system`** (coredns, kube-proxy, node-exporter, etc.) are
  deployed into the shoot cluster and also pull from that registry.
- **Extension-deployed components** (CSI node drivers, cloud-controller-manager, etc.) may pull
  from a different registry altogether, with different credentials.

Four specific challenges make this non-trivial:

1. **Credential propagation depth** — credentials must reach not just the seed but also shoot
   clusters. A simple Kubernetes
   `imagePullSecret` reference on a single namespace is insufficient; the credential must be
   available at every level of the hierarchy.

2. **Per-image vs. global scoping** — a single global credential is too broad. In multi-registry
   setups, different images may require different secrets (e.g. Gardener images from one registry,
   Kubernetes components from another). The solution must support both a global fallback credential
   and per-image overrides.

3. **Security boundary concerns** — propagating the same credential from the Gardener control
   plane all the way into shoot clusters crosses tenant security boundaries. An end user's shoot
   cluster would receive the operator's registry credentials, which may grant access beyond what
   the user is entitled to pull. In multi-tenant seeds this is unacceptable. The design must allow
   operators to control how far credentials propagate and must not force broad registry access to
   be shared with shoot tenants.

4. **Extension image vectors are opaque to Gardener** — each extension (e.g.
   `gardener-extension-provider-aws`) ships its own image vector, entirely independent of
   Gardener's. Extensions deploy seed-side components (cloud-controller-manager, CSI controller)
   and shoot-side components (CSI node DaemonSet, cloud-node-manager applied via ManagedResources
   into shoot `kube-system`). When those images come from a private registry the extension needs
   its own pull credentials delivered to the right places. Gardener has no visibility into the
   extension's image vector and cannot inject pull secrets for images it does not know about.
   Coordination between Gardener's propagation chain and extension-managed credentials requires an
   explicit extension point.

---

## Goals

- Allow operators to register pull secrets centrally (once, in the virtual garden `garden` namespace)
  and have them automatically distributed to all target locations.
- Support scoping a pull secret to a specific subset of seeds via a lightweight annotation.
- Support wildcard (`*`) to copy a secret to all registered seeds.
- Deliver pull secrets to:
  - the seed cluster's `garden` namespace (for Gardener-managed seed components)
  - every shoot control-plane namespace on the seed (for shoot-level components)
  - the shoot cluster's `kube-system` namespace (for in-cluster system components)
  - every extension namespace on the seed (for extensions deployed there)
- Inject `imagePullSecrets` references into pod specs at object-construction time and at admission
  time so that pods never need manual surgery to pull from a private registry.
- Support secret rotation: a change to the source secret propagates through the full chain
  automatically.

---

## API Changes

### New annotation: `seed.gardener.cloud/names`

A `kubernetes.io/dockerconfigjson` Secret in the virtual garden's `garden` namespace may carry:

```yaml
metadata:
  annotations:
    seed.gardener.cloud/names: "seed-a,seed-b"   # or "*" for all seeds
  labels:
    gardener.cloud/role: image-pull-secret
```

`gardener-controller-manager` watches secrets with this label and copies them into the
`seed-<name>` namespace for each listed seed. The wildcard value `"*"` copies to every registered
seed.

### New constant: `GardenRoleImagePullSecret = "image-pull-secret"`

Secrets carrying `gardener.cloud/role=image-pull-secret`
are recognised by the propagation controllers and by the GRM webhook.

### Image vector: `pullCredentials` field

`ImageSource` and `Image` in `pkg/utils/imagevector/types.go` gain a `PullCredentials` field:

```go
type PullCredentials struct {
    Type        PullCredentialsType `json:"type"`
    SecretName string            `json:"secretName,omitempty"`
}
```

`PullCredentialsTypeStaticSecret` is the only type defined. The field may appear at the top level
of an image vector file (global credential) or nested under an individual image entry
(per-image credential that overrides the global one).

Example image vector override:

```yaml
# Global credential — applies to all images with no per-image override.
pullCredentials:
  type: StaticSecret
  secretName: "global-pull-secret"

images:
  - name: coredns
    repository: my-registry.example.com/coredns
    pullCredentials:           # per-image override
      type: StaticSecret
      secretName: "coredns-pull-secret"
```

---

## Implementation Details

### Propagation chain

```
virtual garden / garden ns
  [Secret with gardener.cloud/role=image-pull-secret, seed.gardener.cloud/names annotation]
      │
      │  gardener-controller-manager  (new seed-image-pull-secret controller)
      ▼
virtual garden / seed-<name> ns
      │
      │  gardenlet  (new seed-image-pull-secret controller)
      ▼
seed cluster / garden ns
      │
      ├──────────────────────────────────────────────────────┐
      │  gardenlet (same controller)                         │
      ▼                                                      ▼
seed cluster / shoot--<proj>--<shoot> ns              seed cluster / extension-<foo> ns
      │
      │  gardenlet  (ManagedResource "image-pull-secret", class=nil)
      ▼
shoot cluster / kube-system
```

### gardener-controller-manager: seed-image-pull-secret controller

- **Watches**: Secrets in `garden` namespace with label `gardener.cloud/role=image-pull-secret`.
- **Reconcile logic**:
  1. Parse `seed.gardener.cloud/names` annotation into a set of seed names (supports `*` wildcard).
  2. List all `seed-<name>` namespaces (identified by `gardener.cloud/role=seed` label).
  3. For each seed whose name is in the desired set, create-or-update a copy of the secret in
     that seed's `seed-<name>` namespace.
  4. For seeds no longer in the desired set (or when the source secret is deleted), delete the
     stale copy.

### gardenlet: seed-image-pull-secret controller

- **Watches**: Secrets in `seed-<seedName>` namespace on the virtual garden cluster (the same
  objects GCM produced in step above).
- **Reconcile logic**:
  1. Fetch the source secret from the virtual garden's `seed-<seedName>` namespace.
  2. Create-or-update a copy in the seed cluster's `garden` namespace.
  3. Propagate copies to every namespace labelled `gardener.cloud/role=extension` (extension
     namespaces) and `gardener.cloud/role=shoot` (shoot control-plane namespaces).
  4. For each shoot control plane namespace, call `updateShootManagedResource` which builds a
     `ManagedResource` named `image-pull-secret` (no `spec.class`) carrying the pull secrets
     with `namespace: kube-system`. GRM then applies them to the shoot cluster.
  5. On deletion, remove all copies and delete the ManagedResource (GRM cleans up kube-system).

`updateShootManagedResource` iterates `imagevector.AllContainerImagePullCredentials()` to determine
which secrets should land in kube-system, reads the matching secrets from the shoot CP namespace,
serialises them, and creates/updates the ManagedResource via `managedresources.CreateForShoot`.

### GRM mutating webhook: image pull secret injection

- **Activation**: Only registered when `imagevector.ContainerImagePullCredential()` is non-nil
  (i.e. `IMAGEVECTOR_OVERWRITE` carries a global or per-image `pullCredentials` block).
- **Handler**: On every Pod admission, calls `InjectImagePullSecrets` which walks each container's
  image string, performs a repository-prefix match against the image vector, and appends the
  matching secret names to `spec.imagePullSecrets`.
- This covers runtime pod mutation for any pod that was not pre-injected at object-construction
  time (user workloads, dynamically scheduled pods, controller installations).
