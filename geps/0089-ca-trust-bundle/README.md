# GEP-0089: Configurable CA Trust Bundle for Shoot Nodes and Workloads

## Summary

This GEP proposes a single, landscape-level mechanism that allows operators to define the CA
trust bundle used by shoot nodes and offer it to workloads running on shoot clusters. It names a
curated bundle and a mode: *append* (the existing behaviour, extended) or *replace* (new).

Two delivery mechanisms consume it:

- **Nodes** — the existing `rootcertificates` component, which already appends the cluster CA and
  `CloudProfile.spec.caBundle` to the node system trust store, is extended to optionally *replace*
  the trust anchors pre-installed in the machine image. The bundle is fetched during node
  bootstrap, before the first image pull, so that replace mode does not depend on trust anchors
  it is itself replacing.
- **Workloads** — gardenlet publishes the bundle as a well-known `ClusterTrustBundle` in every shoot
  running Kubernetes ≥ 1.37. Pods consume it via the `clusterTrustBundle` projected volume source.

## Motivation

Operators running Gardener behind a private PKI, or shipping into sovereignty-regulated
environments, need full control over the CA trust configuration of the artefacts their clusters
are built from.

This affects two kinds of artefacts, each with a different current state:

- **Nodes.** The node system trust store is already extended by `rootcertificates` (cluster CA
  plus `CloudProfile.spec.caBundle`), so host processes such as kubelet and containerd already
  trust those CAs. It is, however, not possible to *replace* the trust anchors pre-installed in
  machine images.
- **Workloads (pods).** Not covered at all: container images ship their own trust bundles, and
  operators have no Gardener-native means of offering a curated bundle to them. Existing
  workarounds (per-namespace `ConfigMap` replication, mutating webhooks, rebuilt images) are
  fragile or couple trust policy to image build pipelines.

For both, full control over the trust configuration is needed for two distinct reasons:

1. **Regulatory requirements.** Rules may mandate an explicitly curated set of trust anchors,
   independent of whatever the underlying OS or base image ships.
2. **Rapid revocation.** With trust anchors pre-installed in images, the only current recourse to
   a compromised or distrusted root CA (e.g. DigiNotar, TrustCor, or the recent eMudhra case) is
   to rebuild and roll out every affected machine and container image, and wait for distribution
   updates. This is too slow for an incident response.

### Goals

- Provide one landscape-level mechanism naming a curated CA bundle, plus an append/replace mode.
- Nodes: in *append* mode, add the bundle to the node system trust store (superset of the current
  behaviour); in *replace* mode, use it *instead of* the trust anchors shipped with the machine
  image.
- Nodes: work across all OS extensions that are supported by `rootcertificates` today, without
  requiring OS-specific changes for the default path.
- Workloads: make the bundle available to pods in a Kubernetes-native, opt-in manner.
- Changes to the bundle propagate without rebuilding any image (nodes roll, `ClusterTrustBundle`
  objects update in place).

### Non-Goals

- Forcing the bundle onto all pods (e.g. via mutating admission). Consumption is opt-in.
- Supporting workload delivery on shoots running Kubernetes < 1.37. A second code path
  (e.g. operator-curated projected `ConfigMap`s) could be added as a follow-up if demand
  materialises.
- Build-time customisation of machine images. Each OS image pipeline exposes its own mechanism
  for this (e.g. GardenLinux features, Ignition/Butane for Flatcar); this GEP covers the runtime
  path.
- Changing per-registry containerd CA configuration (`RegistryCABundle`), which stays orthogonal.
- Trust bundles for seed-system and garden components. These are handled by separate,
  API-only changes (gardener/gardener#16210, gardener/gardener#16211) which this GEP complements.

## Proposal

### Configuration

Operators reference a `Secret` holding the concatenated PEM bundle (key `ca-certificates.crt`) and
choose a mode. The candidate location is the `Seed` resource, as proposed in
gardener/gardener#16211:

```yaml
spec:
  settings:
    trustBundle:
      secretRef:
        name: custom-ca-bundle
      mode: Append # or Replace
```

When unset, behaviour is unchanged.

### Nodes

`rootcertificates` is extended as follows:

- **Append** — the bundle is concatenated with the cluster CA and `CloudProfile.spec.caBundle`
  into the existing local CA file under `/var/lib/ca-certificates-local`, and
  `update-local-ca-certificates.sh` runs as today.
- **Replace** — the system bundle is generated from the curated bundle (plus the cluster CA, which
  is always required) only; the distribution's default anchors are not included. On
  Debian-based systems this maps to `update-ca-certificates --fresh` with the distribution set
  disabled; other OS families get an equivalent branch in the existing script.

The files are persisted under `/var`, so they survive OS in-place updates and rebuilds of `/etc`.

**Bootstrap.** In *replace* mode, the trust store must be in place before the node pulls its first
image (`gardener-node-agent`), because the pull may already depend on the curated anchors. The
node-init script therefore fetches the bundle from a well-known `ConfigMap` in the shoot's
`kube-public` namespace, readable with the bootstrap token, and installs it before
`ctr images pull`. This follows the existing pattern for bootstrap-time data.

**Rollout.** The bundle content is part of the `OperatingSystemConfig` hash, so changes trigger a
rolling update of worker pools, consistent with today's CA handling.

### Workloads

For every shoot running Kubernetes ≥ 1.37 (where `ClusterTrustBundle` and the
`clusterTrustBundle` projected volume source are GA), gardenlet reconciles a signer-unlinked
`ClusterTrustBundle` with a well-known name holding the curated bundle. Workloads opt in:

```yaml
volumes:
- name: trust-bundle
  projected:
    sources:
    - clusterTrustBundle:
        name: gardener-trust-bundle
        path: ca-certificates.crt
```

The kubelet keeps the projected file up to date, so rotations propagate without pod restarts.
Shoots below 1.37 are skipped without error; the object becomes available
upon upgrade.

Mode semantics for workloads are determined by how the workload consumes the projected volume:
mounting over the image's default bundle path replaces it, whereas mounting elsewhere and
pointing e.g. `SSL_CERT_FILE` to it is additive. Both patterns are valid; this document does not
mandate either.

### Notes/Constraints/Caveats

- *Replace* mode makes the curated bundle authoritative. Operators are responsible for including
  all anchors required by the infrastructure (cloud provider endpoints, registries, OIDC issuers).
- The cluster CA is always retained on nodes, regardless of mode.

### Risks and Mitigations

- *Incomplete bundle in replace mode breaks node bootstrap.* Mitigation: replace mode is opt-in;
  validation rejects empty or unparseable bundles; worker pools roll with the standard
  `maxUnavailable` semantics, so a broken bundle halts the rollout early.
- *Bootstrap-token access to the bundle.* The bundle contains public certificates only; exposing it
  to `system:bootstrappers` does not leak secrets.
- *Unauthorized creation of the well-known `ClusterTrustBundle`.* gardenlet owns the object and
  reconciles drift; `ClusterTrustBundle` creation is a privileged operation in shoots.
- *Privileged configuration.* Configuring this mechanism requires write access to `Seed`
  resources, which carries the same risk profile as other seed-level configuration.

## Design Details

- API: add `mode` (`Append` | `Replace`, default `Append`) to the `SeedSettingTrustBundle` type
  proposed in gardener/gardener#16211.
- Data flow: `Seed.spec.settings.trustBundle.secretRef` → `Secret` in the seed's `garden`
  namespace → (a) `OperatingSystemConfig` via `rootcertificates`, (b) `ConfigMap` in shoot
  `kube-public` for bootstrap, (c) `ClusterTrustBundle` in the shoot.
- `rootcertificates`: extend `components.Context` with the custom bundle and mode; branch in
  `update-local-ca-certificates.tpl.sh` per OS family for replace mode.
- `nodeinit`: when the bundle is configured in replace mode, download and install it before the
  first image pull.
- Shoot reconciliation: create/update/delete the `ConfigMap` and `ClusterTrustBundle`; skip the
  latter for Kubernetes < 1.37.

## Drawbacks

- Replace mode shifts responsibility for a complete trust store from the OS vendor to the
  operator.
- One additional object per shoot is reconciled (negligible cost).
- Workload delivery is unavailable on Kubernetes < 1.37 without a follow-up.

## Alternatives

- **Rebuild machine and container images with curated anchors.** Couples trust policy to image
  pipelines, multiplies artefacts per landscape, and is too slow for incident response.
- **Expose generic node files via `OperatingSystemConfig`.** Lower-level and harder to validate;
  every operator would re-implement OS-specific trust-store handling.
- **Mutating webhook injecting the bundle into all pods.** Automatic but fragile, opaque to users,
  and with a wide blast radius when failing closed.
- **Per-namespace `ConfigMap` replication.** Requires a replication controller and still leaves
  wiring to each workload author; strictly inferior to the `clusterTrustBundle` projection on
  Kubernetes ≥ 1.37.
