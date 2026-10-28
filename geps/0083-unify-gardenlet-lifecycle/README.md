# GEP-0083: Unify `gardenlet` Lifecycle

## Summary

This GEP proposes to extend the existing `Gardenlet` resource (`seedmanagement.gardener.cloud`) with a `shootRef` field so it can also represent managed seeds, making it the single API for deploying and upgrading all `gardenlet`s.
The `ManagedSeed{Set}` APIs are deprecated and eventually removed.

In the same effort, `gardenlet`s stop registering their own `Seed` resource at startup.
A central controller part of `gardener-controller-manager` takes over authorship of `Seed` based on the `seedConfig` embedded in the `Gardenlet` resource, and the `gardenlet`'s authorization is narrowed so it may no longer write `Seed.spec`.

The two changes are complementary: the first unifies the lifecycle model; the second removes a class of privilege escalation in which a compromised `gardenlet` rewrites its own `Seed` spec.

## Motivation

### Two divergent lifecycle models

Today there are two separate paths to run a `gardenlet`:

- **Unmanaged seeds** where `gardenlet` is deployed directly via its Helm chart (by a person or by some automation/operator/controller) have a `Gardenlet` resource *only when self-upgrade is enabled* — it is opt-in today: the [chart's `selfUpgrade`](https://github.com/gardener/gardener/blob/8d067240f5284279481cf5e78977f83e1f1fded8/charts/gardener/gardenlet/values.yaml#L236-L242) values render a `gardenlet-selfupgrade-config` `ConfigMap`, which the `gardenlet` turns into a `Gardenlet` resource at startup ([`createSelfUpgradeConfig`](https://github.com/gardener/gardener/blob/8d067240f5284279481cf5e78977f83e1f1fded8/cmd/gardenlet/app/app.go#L671-L697)); without it, no `Gardenlet` resource exists.
- **Managed seeds** use the `ManagedSeed` API, reconciled by the *parent* `gardenlet` through the shared `gardenletdeployer` actuator, which deploys the child `gardenlet`'s Helm chart into the shoot cluster. The parent keeps reconciling it for the resource's lifetime — no self-upgrades.

The result is two code paths and two sets of API machinery for the same component.
Notably, a `Gardenlet` resource for a managed seed is explicitly *rejected* by admission today — the two worlds are kept apart by design.

> [!NOTE]
> **Self-hosted shoots** always have a `Gardenlet` resource with self-upgrade enabled ([GEP-28](../0028-self-hosted-shoot-clusters/README.md)).

### `Seed` self-registration as a privilege escalation vector

At startup, the `gardenlet` creates or patches its own `Seed` from the `seedConfig` in its `GardenletConfiguration`:

```go
// cmd/gardenlet/app/app.go — registerSeed()
controllerutils.GetAndCreateOrMergePatch(ctx, gardenClient, seed, func() error {
    // ...
    seed.Spec = g.config.SeedConfig.Spec // entire Seed spec written verbatim
    // ...
})
```

This happens on first startup and on every restart.
A compromised `gardenlet` can therefore set an arbitrary `Seed.spec` — e.g. point reference fields at credentials it should not read, or alter scheduling-relevant configuration.

[PR #15802](https://github.com/gardener/gardener/pull/15802) added the `AllowlistSeedReferences` feature gate. It is a *referenced-resource* allowlist: the `gardener-apiserver` requires every resource referenced from a `Seed` spec (`.spec.backup.credentialsRef`, the `.spec.dns.*.credentialsRef` fields, `.spec.resources[].resourceRef`) to carry a `seed.gardener.cloud/names` annotation naming the seed.
This narrows *reference abuse* but does not remove the `gardenlet`'s write access to its own `Seed.spec`, and the allowlist itself needs ongoing maintenance.
The structural fix is to take `Seed.spec` authorship away from the `gardenlet` entirely.

### Goals

- Make self-upgrade (and thus the `Gardenlet` resource) mandatory for unmanaged seeds too — it already is for self-hosted shoots — rather than the opt-in it is today. This makes the `Gardenlet` resource the single source of truth for a `gardenlet`'s deployment: the self-upgrade controller continuously reconciles the running deployment to match it, so lifecycle management flows through the resource rather than through ad-hoc, out-of-band `helm` operations. The single lifecycle API only works if the resource universally exists.
- Represent managed seeds via the `Gardenlet` resource with a new `shootRef` field.
- Deprecate and eventually remove the `ManagedSeed` API (and the `ManagedSeedSet` API as well — already planned for removal, see [hackathon#52](https://github.com/gardener/hackathon/issues/52)).
- Remove the `gardenlet`'s ability to author its own `Seed` desired state; forbid it from creating `Seed`s or changing `Seed.spec`, while keeping metadata and status writable.
- Move authorship of the desired `SeedSpec` to a central, higher-trust controller.
- Drop the seed template (`seedConfig`) from `GardenletConfiguration` — the `gardenlet`'s component config no longer carries a `Seed` template; the desired state lives in a dedicated `Gardenlet.spec.seedTemplate` field instead.
- Unify the "initial deployment by a parent, self-upgrade thereafter" model across all seed kinds, including jittered self-upgrades.

### Non-Goals

- Changing how the `gardenlet` reports `Seed.status` (health, capacity, conditions) — that stays in the `gardenlet`.
- Changing the TLS bootstrapping mechanism.

## Proposal

### 1. Extend the `Gardenlet` resource with `shootRef`

A new optional field references the `Shoot` to be registered as a seed:

```yaml
apiVersion: seedmanagement.gardener.cloud/v1alpha1
kind: Gardenlet
metadata:
  name: my-seed      # equals the Seed name (existing convention)
  namespace: garden
spec:
  shootRef:              # new, Shoot the gardenlet runs in; set for managed seeds AND self-hosted shoots
    name: my-shoot
  seedTemplate: { ... }  # new, Seed metadata + spec (desired state, see §3) — top-level, owned centrally
  deployment: { ... }
  config:
    apiVersion: gardenlet.config.gardener.cloud/v1alpha1
    kind: GardenletConfiguration
    # no seedConfig anymore (see §3)
```

`shootRef` is the explicit link from a `Gardenlet` to the `Shoot` it runs in.
It is set in two cases, replacing today's two implicit conventions:

- **Managed seed** → `shootRef` set *and* a `Seed` is registered. The *parent* `gardenlet` (responsible for the shoot's seed) performs the **initial** deployment of the child `gardenlet` into the shoot cluster via the existing `gardenletdeployer` actuator, then stops — mirroring what `gardener-operator` already does for unmanaged seeds. Subsequent upgrades are self-upgrades. (Today: the separate `ManagedSeed` API.)
- **Self-hosted shoot** → `shootRef` set and *no* `Seed` is registered (`seedTemplate` is `nil`). (Today: encoded in a `self-hosted-shoot-` prefix on the `Gardenlet` name, matched via `IsResponsibleForSelfHostedShoot`.) `shootRef` replaces that name-prefix hack with an explicit reference.
- **Unmanaged seed** → `shootRef` absent; a `Seed` is registered. Unchanged.

The admission restriction that currently rejects `Gardenlet` resources for managed seeds is lifted for the `shootRef` case.

The `bootstrap` and `mergeWithParent` fields of `ManagedSeed.spec.gardenlet` are **not** carried over.
`bootstrap` is obsolete now that a virtual garden always exists and bootstrap tokens can be used uniformly (`gardener-operator` is THE deployment model for Gardener).
`mergeWithParent` is effectively always `true` in practice. Dropping both keeps the `Gardenlet` API lean.

> [!NOTE]
> Whether a `Seed` is registered is determined by the presence of `seedTemplate`, **not** by whether the referenced `Shoot` is self-hosted: a `Gardenlet` with `seedTemplate` registers a `Seed` (it is a *seed* `gardenlet`); one without does not (it is a *shoot* `gardenlet`).
> The distinction matters because a self-hosted shoot can itself be promoted to a seed — see below.

> [!NOTE]
> **Promoting a self-hosted shoot to a seed.**
> The cluster's own shoot `gardenlet` keeps its `Gardenlet` resource unchanged (no `seedTemplate`, no `Seed`).
> Promotion adds a *second* `Gardenlet` resource — the seed `gardenlet` — referencing the same `Shoot` and carrying a `seedTemplate`.
> The shoot `gardenlet` performs the initial deployment of the seed `gardenlet`; afterwards the seed `gardenlet` self-upgrades. (Today this is done via a `ManagedSeed`.)

### 2. Mandatory self-upgrade, unified with jitter

Self-upgrade becomes mandatory for unmanaged seeds as well — self-hosted shoots already have it — so that every `gardenlet` always has a `Gardenlet` resource.
The opt-in `selfUpgrade` chart values and the `gardenlet-selfupgrade-config` `ConfigMap` indirection are removed; the resource is created unconditionally.
This is a precondition for §3 — the central controller can only own `Seed.spec` from `seedConfig` if the `Gardenlet` resource reliably exists.

All `gardenlet`s converge on: *parent deploys once, `gardenlet` self-upgrades thereafter.*
For unmanaged seeds the "parent" is `gardener-operator`; for managed seeds it is the `gardenlet` hosting the shoots control plane; for self-hosted shoots the "parent" is the `gardenadm connect` call.

One special case persists for unmanaged seeds, as today: if the seed cluster is not network-reachable from `gardener-operator`, the `gardenlet` Helm chart is deployed into the seed manually, once (by a person or by some external automation).
From then on, the mandatory self-upgrade mechanism takes over — the `Gardenlet` resource is created and the `gardenlet`'s lifecycle is managed centrally from the garden cluster via that resource.
The initial deployer differs (`gardener-operator` vs. a manual/out-of-band deployment); the steady state does not.

> [!IMPORTANT]
> Besides deploying the `gardenlet` Helm chart into the seed cluster, this manual deployment must also create the `Gardenlet` resource in the garden cluster.
> The `gardenlet` will no longer be able to self-create it via today's `gardenlet-selfupgrade-config` `ConfigMap` indirection.

The self-upgrade controller re-applies the Helm chart idempotently each `SyncPeriod`.
Today it has **no jitter** — only the `ManagedSeed` controller staggers work: the parent `gardenlet` delays each `ManagedSeed` reconciliation by a random offset within a configurable window (default `SyncJitterPeriod` `5m`), so that managed seed upgrades do *not* all hit `gardener-apiserver` at the same instant.
In the new model, there is no parent doing this centrally; instead each `gardenlet` jitters its *own* self-upgrade — it independently draws a random delay within the configurable window before (re-)applying its chart.
The staggering is thus decentralized across the fleet rather than orchestrated by a parent, but the net effect is the same as today: upgrades spread evenly over the window instead of thundering in together.

### 3. Central ownership of the `Seed` desired state

A new controller in `gardener-controller-manager` watches `Gardenlet` resources and reconciles the corresponding `Seed` desired state from `Gardenlet.spec.seedTemplate`.
The `gardenlet` no longer writes `Seed.spec` at startup (its startup step is reduced to the [update handshake](#the-update-handshake) below).

- It creates the `Seed` if absent and reconciles its desired state — `.spec` *and* the metadata (labels, annotations) carried in `seedTemplate` — when they change. It never touches `Seed.status`.
- The `gardenlet` loses the ability to **create** `Seed`s and to modify `Seed.spec`; it may still update `Seed` metadata and `status` (it sets operational labels/annotations and finalizers at runtime). This is enforced structurally via a new `metadata` subresource — see [Restricting `gardenlet` to metadata and status](#restricting-gardenlet-to-metadata-and-status) — not by diffing spec on admission.
- The seed template (`seedConfig`) is removed from `GardenletConfiguration`. The `gardenlet` no longer carries a `Seed` template in its own config; where it reads `SeedConfig.Spec` today (runtime decisions), it reads the live `Seed` object it already watches. Its seed identity (today `SeedConfig.Name`, used ~24×) must come from a minimal replacement — a dedicated `seedName` config value or the `Gardenlet` resource name — settled during design.
- With self-registration gone, the `AllowlistSeedReferences` mitigation becomes unnecessary and can be retired.

Because self-upgrades are jittered (§2), an old `gardenlet` can keep running for a while after a new `seedTemplate` is committed.
To prevent it from reconciling a spec written for a newer version, spec application is gated by an **update handshake** between the gardenlet and the controller — see [Design Details](#design-details).

> [!IMPORTANT]
> There is **no precedent** for a central controller authoring the `Seed` desired state — today every path (operator-deployed, self-upgrade, `ManagedSeed`) ends with the `gardenlet` self-registering its own `Seed`.
> This is the genuinely new architectural concern in this GEP.

## Design Details

### API change

```go
// GardenletSpec (pkg/apis/seedmanagement/types_gardenlet.go)
type GardenletSpec struct {
    Deployment          GardenletSelfDeployment
    Config              runtime.Object
    KubeconfigSecretRef *corev1.LocalObjectReference

    // ShootRef references the Shoot the gardenlet runs in. It is set for managed
    // seeds and for self-hosted shoots, replacing the former name-prefix convention.
    // +optional
    ShootRef *corev1.LocalObjectReference

    // SeedTemplate is the desired Seed metadata and spec, owned by the central
    // controller. Moved here from GardenletConfiguration.seedConfig.
    // +optional
    SeedTemplate *gardencorev1beta1.SeedTemplate
}
```

`SeedConfig` is removed from `GardenletConfiguration`.
The `bootstrap` and `mergeWithParent` fields of `ManagedSeed.spec.gardenlet` are not ported (see §1).

### `Seed` desired-state controller

- **Watches:** `Gardenlet`.
- **Writes:** the `Seed` desired state — `.spec` and the metadata — from `Gardenlet.spec.seedTemplate` (the core `SeedTemplate`, i.e. metadata + spec). Never `Seed.status`.
- **Placement:** `gardener-controller-manager` is the natural host. `Seed` is a garden-API resource whose *ongoing* reconciliation already lives there — the existing `seed` controller handles its secrets, references, and lifecycle, and both `Gardenlet` and `Seed` are garden-cluster resources it reconciles directly. Owning the `Seed` desired state is the same kind of work. `gardener-operator`, by contrast, manages the *garden cluster itself* (the `Garden` resource and its control plane) and touches `gardenlet`s only for the **initial** deployment — it deliberately steps back afterwards. Making it the continuous, per-seed `Seed.spec` owner would pull it into steady-state reconciliation outside its scope and split `Seed` ownership across two components. `gardener-controller-manager` is also present in every landscape, whereas `gardener-operator` is specific to the operator-based installation model.

### Restricting `gardenlet` to metadata and status

Once the central controller owns `Seed.spec`, the `gardenlet` must be prevented from writing it.
The first attempt at a general spec-immutability guard — [PR #15764](https://github.com/gardener/gardener/pull/15764), which rejected `.spec` modifications on UPDATE in the `seed-restriction` and `shoot-restriction` admission handlers — had to be reverted for `Shoot`s ([PR #15974](https://github.com/gardener/gardener/pull/15974)): mutating webhooks and API defaulting change `.spec` fields *before* the object reaches the handler, so the deep-equal comparison produced false positives and broke reconciliations ([issue #15992](https://github.com/gardener/gardener/issues/15992)).
Detecting illegitimate spec writes after the fact is fragile.

Instead, we make the restriction **structural** by introducing a new `metadata` subresource (e.g. `seeds/metadata`, `shoots/metadata`), analogous to the existing `status` subresource.
Carving write access to parts of an object into subresources is a well-established Kubernetes pattern — besides `status`, core Kubernetes exposes `scale` (on `Deployment`/`StatefulSet`), `finalize` (on `Namespace`), and `approval` (on `CertificateSigningRequest`), each scoped to a specific slice of the object. A `metadata` subresource is a natural addition to that family, not an esoteric construct.
Just as `status` can only write `.status`, the `metadata` subresource can only write object metadata (labels, annotations, finalizers).
The `gardenlet` is switched to perform all its metadata mutations through this subresource for every resource it reconciles, and the `Seed`/`Shoot` authorizers grant it only the `status` and `metadata` subresources — never `update`/`patch`/`create` on the resource itself.
With no permission to write the main resource, the `gardenlet` *cannot* change `.spec` at all — there is nothing to diff and no false positives.

This generalizes beyond `Seed`: it is the mechanism tracked by [issue #15992](https://github.com/gardener/gardener/issues/15992) to re-establish spec-immutability for `gardenlet`-initiated `Shoot` updates as well, and it is a prerequisite for this GEP's security goal.

### The update handshake

Today, the `gardenlet` writes its own `Seed.spec` and then reconciles it, so it always reconciles a spec compatible with its own code.
Moving spec ownership to the central controller breaks that coupling: an operator may bump the `gardenlet` image and a `seedTemplate` field together, the controller would apply the new spec immediately, but the old `gardenlet` keeps running until its (jittered, possibly hours-long) self-upgrade — and would reconcile a spec written for its successor.
Today's invariant to preserve: **a `gardenlet` only ever reconciles a `Seed.spec` applied for its own running version.**

A handshake via annotations on the `Seed` enforces this.
The `gardenlet` knows its own image — the image reference (repository plus tag/digest) of its own running deployment, injected as an env var at deployment time via its Helm chart.
On startup, instead of running `registerSeed`, it sets `seed.gardener.cloud/ready-for-update: "<current-image>"` annotation on its `Seed` (where `<current-image>` is that running image reference), and **waits — it does not start its controllers until that annotation is removed.**

Two annotations are involved: `seed.gardener.cloud/ready-for-update` (transient, set by the `gardenlet` on startup, carrying its running image) and `seed.gardener.cloud/applied-image` (persistent, maintained by the controller, recording the image the current `Seed.spec` was applied for).

The central controller reacts to `ready-for-update` by comparing the `gardenlet`'s desired image (`Gardenlet.spec.deployment`) against the annotation's value (the running image):

- **Match** → the running `gardenlet` is on the desired version: apply the pending `seedTemplate` to `Seed.spec` (if any), set `applied-image` to that image, and remove `ready-for-update`. The `gardenlet` then starts its controllers.
- **Mismatch** → the `gardenlet` is still on the old version: just remove the `ready-for-update` annotation to allow `gardenlet` to start, but don't change `Seed.spec`. The `gardenlet` self-upgrades eventually, restarts on the new image, and re-runs the handshake.

Because every image change restarts the `gardenlet` and thus triggers a fresh handshake, `applied-image` always tracks the running `gardenlet`'s image once a handshake has completed.

**Spec-only changes** (no image bump) skip the handshake: the `gardenlet` is already running and does not restart, so it never sets `ready-for-update`.
The controller instead compares the desired image (`Gardenlet.spec.deployment`) against `applied-image`:
- Equal → the running `gardenlet` is already on the desired version, so the new `seedTemplate` is applied to `Seed.spec` directly; the running `gardenlet` picks it up from the `Seed` it watches.
- Not equal → an image bump is still pending (the old `gardenlet` has not self-upgraded yet); the controller holds the spec change until the next handshake applies image and spec together.

The protocol is idempotent: the annotation value is a function of the current state, so crashes on either side, repeated reconciles, and rapid successive updates all converge — the latest spec is applied once the gardenlet reports the desired image.

### Jitter

Each `gardenlet` jitters its own self-upgrade: on every sync it computes a random delay within a configurable window before (re-)applying its chart.
This replaces today's parent-driven staggering with a decentralized equivalent — no central component coordinates the fleet, yet upgrades still spread evenly over the window.
Newly created resources reconcile immediately (no jitter on first deployment), and spec-driven changes are not delayed; only the periodic self-upgrade re-applies are jittered.

### Migration

The concrete migration steps are determined during implementation.
The goal is to minimize user impact — e.g. auto-creating `Gardenlet` resources for existing `ManagedSeed`s — but users will still need to adapt their workflows to the `Gardenlet` API before the `ManagedSeed` API is removed.

## Drawbacks

- Migrating off `ManagedSeed` is a sizable, multi-release effort (see [Caveats](#notesconstraintscaveats) for the surface).

## Notes/Constraints/Caveats

- **`ManagedSeed` machinery is large.** Full removal touches three admission plugins, validation, registry/strategy, defaulting, and the `managedseed` controller. The shared `gardenletdeployer` actuator is reused by the `Gardenlet` path, so it stays.
- **`gardenlet` runtime reads of the seed template.** With `seedConfig` gone from `GardenletConfiguration`, the `gardenlet` must obtain its seed identity (today `SeedConfig.Name`) from a minimal replacement, and read runtime spec details from the live `Seed` object rather than its own config. The initial-bootstrap flow is reworked so the `gardenlet` no longer authors the `Seed`.

## Risks and Mitigations

- **Central controller as a new failure domain.** It gates both `Seed` creation and every subsequent update — a `gardenlet` blocks on the [update handshake](#the-update-handshake) until the controller gives its "go", so an unavailable controller stalls startups and spec rollouts. Mitigate by keeping its logic minimal (spec copy from `seedTemplate`) and idempotent, surfacing failures on the `Gardenlet` status, and ensuring the handshake fails safe (the `gardenlet` keeps running its last-known-good spec rather than being blocked indefinitely).
- **Migration races** between the converter, the parent `gardenlet`, and the new `Seed` desired-state owner. Mitigate with a phased, feature-gated rollout, converting resources before disabling the `gardenlet`'s self-registration of `Seed.spec`.

## Alternatives

- **Keep `ManagedSeed`, fix only the security issue.** Retaining `AllowlistSeedReferences` mitigates reference abuse but leaves two lifecycle models, ongoing allowlist maintenance, and the `gardenlet`'s write access to `Seed.spec` intact.
- **`gardener-operator` as the `Seed.spec` owner** instead of `gardener-controller-manager`. Viable (operator already owns the initial `Gardenlet` deployment), but it manages the garden cluster itself and intentionally only bootstraps `gardenlet`s; continuous per-seed `Seed.spec` reconciliation belongs with `gardener-controller-manager`, where `Seed` lifecycle already lives.
