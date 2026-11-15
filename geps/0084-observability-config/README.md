# GEP-0084: Observability Extension for Gardener

## Summary

This GEP proposes a new extension `gardener-extension-observability-config` of type `observability-config` in the Gardener organization. Its mutating admission webhooks inject operator configuration into the `OpenTelemetryCollector` resources Gardener deploys, adding to or overriding Gardener's configuration. The extension deploys no additional collectors.

Operators configure the extension in the `Garden` and `Seed` resources. The `Seed` carries one configuration for its own collector and one for the shoot control plane collectors. Each change creates a new immutable version. A shoot control plane is pinned to a version during the reconciliation of its shoot, so a change reaches the shoot control planes one shoot reconciliation at a time. The extension does not appear in the `Shoot` resource, so shoot owners can neither configure nor disable it.

## Motivation

With the `OpenTelemetryCollector` feature gate, Gardener deploys an `OpenTelemetryCollector` per shoot control plane namespace and one in the `garden` namespace of seeds and of the garden runtime cluster. Per [GEP-34](../0034-observability2.0-opentelemetry/README.md), these collectors are the abstraction layer for observability signals, which makes them a stable entry point for operators.

Operators may need more than Gardener provides out of the box. Examples are shipping logs to a central log store outside of Gardener and attaching landscape-specific attributes, such as the stage, before logs leave the cluster. Gardener fully owns the collector configuration and offers no extension point for this. In [gardener/gardener#15700](https://github.com/gardener/gardener/issues/15700), the consensus was to create a new extension that offers this entry point.

### Goals

- Let operators add configuration to the collectors Gardener deploys, reusing the receivers Gardener already configures.
- Configure the `garden` namespace collectors and all shoot control plane collectors of a seed at `Garden` and `Seed` level, without per-shoot configuration and without appearing in the `Shoot` resource.
- Roll out `Seed` changes to a shoot control plane collector only during the reconciliation of its shoot.
- Keep the API signal-agnostic, so that it covers metrics and traces once Gardener routes them through the collectors.

### Non-Goals

- Configuration by shoot owners. They use [`gardener-extension-otelcol`](https://github.com/gardener/gardener-extension-otelcol), which this extension does not replace (see [Alternatives](#alternatives)).
- Changing how Gardener collects or stores signals.
- Modelling the OpenTelemetry Collector configuration in a Gardener API or validating it beyond its structure.
- Metrics and traces in the first iteration.

## Proposal

The extension builds on the `OpenTelemetryCollector` resources Gardener deploys and extends their configuration without deploying additional collectors. Gardener deploys the collectors via `ManagedResource`s, and `gardener-resource-manager` replaces their `.spec` with Gardener's desired state on every write, so it would revert any patch. Instead, a mutating admission webhook injects the configuration whenever a collector is written.

Operators can inject `processors`, `exporters`, `connectors`, `extensions`, `service.pipelines` and `service.extensions` into the configuration of Gardener's collectors. Receivers cannot be added: Gardener's collectors are meant for the signals Gardener emits, and other signals belong in a separate collector.

The extension has a `garden` and a `seed` class, but no `shoot` class. All collectors receive their configuration through immutable versions in the `garden` namespace. A version consists of one `ConfigMap` with the configuration and one snapshot of each referenced `Secret` or `ConfigMap`. An annotation on the namespace of a collector pins it to a version. The collector webhook injects the pinned version, and the extension controller triggers a write of the collector when the pin changes.

Gardener's network policies only allow the collectors to reach the in-cluster log backends, DNS and the `kube-apiserver`. To reach custom backends, the webhook sets the labels `networking.gardener.cloud/to-public-networks: allowed` and `networking.gardener.cloud/to-private-networks: allowed` as configured in `networking` and removes them otherwise, since `gardener-resource-manager` keeps labels it does not manage.

### Garden Runtime Cluster

Operators configure the collector in the `garden` namespace of the garden runtime cluster via `collector` in the `providerConfig` in `Garden.spec.extensions[]`. During the `Garden` reconciliation, `gardener-operator` deploys the `Extension` resource `garden-observability-config`, and the extension controller creates the version, pins the `garden` namespace to it and triggers the collector. A change, including one of a referenced resource, thus reaches the collector with the next `Garden` reconciliation. `gardener-operator` deploys the collector independently of the extension, so the collector may run without the configuration until the extension is reconciled for the first time.

### Seed

Operators configure the collector in the `garden` namespace of a seed via `collector` in the `providerConfig` in `Seed.spec.extensions[]`. During the `Seed` reconciliation, gardenlet deploys the `Extension` resource `observability-config` and waits until it is ready before it deploys the collector. The extension controller proceeds as for the `Garden`.

### Shoot Control Planes

Only the `Seed` configures the shoot control plane collectors, via `shootCollectors` in the same `providerConfig`. There is no per-shoot configuration, and the `Garden` has no `shootCollectors`.

A broken configuration must not break the collectors of all shoots on a seed at once. Each shoot control plane therefore picks up a change only during the reconciliation of its shoot:

1. During the `Seed` reconciliation, the extension controller creates the version of `shootCollectors` and copies its snapshots into all shoot control plane namespaces on the seed. Then it records the version as current in the status of its `Extension` resource.
2. At the start of every shoot reconciliation, gardenlet patches the shoot control plane namespace, even if nothing changed. Mutating admission webhooks are called for every patch, including an empty one, so the namespace webhook pins the namespace to the current version.
3. The extension controller notices the changed pin and triggers the collector.

The namespace patch is the first step of the shoot reconciliation, so the pin is set before gardenlet deploys the collector, and no new lifecycle constraint for extensions is needed.

Between two reconciliations of its shoot, a shoot control plane keeps its version, including the snapshots of the referenced resources, so a rotated credential also arrives only with the next reconciliation. Several versions can thus be in use on a seed at the same time. At its next reconciliation, a shoot moves straight to the current version and skips any intermediate ones.

A shoot control plane namespace created after the last `Seed` reconciliation, e.g., for a new shoot or after a control plane migration, is pinned to the current version on creation. The extension controller then copies the snapshots into it, and the collector pods start once their snapshots exist.

### Garden Runtime Cluster that is also a Seed

On such a cluster, the extension runs twice, deployed by `gardener-operator` for the `Garden` and by gardenlet for the `Seed`, but there is only one collector in the `garden` namespace. Merging both configurations for this collector would be confusing, so the `Garden` takes precedence. The collector in the `garden` namespace receives `collector` of the `Garden`, while the shoot control plane collectors receive `shootCollectors` of the `Seed`. gardenlet passes `gardener.clusterTypes.gardenRuntimeCluster: true` to the seed deployment, so the `seed` class ignores `collector` of the `Seed` and reports this in a condition on its `Extension` resource. The documentation of the extension describes this setup.

### Notes/Constraints/Caveats

- Operators can only use components contained in the collector image, by default the [Gardener OpenTelemetry Collector distribution](https://github.com/gardener/opentelemetry-collector/blob/main/manifest.yml). They can replace the image, e.g., with the full `opentelemetry-collector-contrib` distribution.
- `Seed` changes do not trigger shoot reconciliations. Depending on the gardenlet configuration and the maintenance settings of a shoot, it is reconciled periodically or only in its maintenance time window. Failed and ignored shoots keep their pin.

### Risks and Mitigations

- An invalid configuration or a broken override of a Gardener component keeps the collector from starting, which stops Gardener's own pipelines too. Pinning limits the impact of a `shootCollectors` change to the shoots reconciled since the change. Operators have to test configuration changes themselves.
- Injected pipelines share the receiver and the resources of the collector with Gardener's pipelines. Additional processing or an unavailable or misbehaving custom backend can degrade all pipelines of the collector, including Gardener's.
- The collector webhook uses `failurePolicy: Fail`, so a collector is never written without its pinned configuration. While the webhook is unavailable, `gardener-resource-manager` cannot write the collectors: running collectors keep their configuration, but Gardener's changes wait and the collectors of new shoots are not created. The failed writes show up in the `ObservabilityComponentsHealthy` condition of the shoot.
- The namespace webhook uses `failurePolicy: Ignore`, since a failing namespace patch would fail every shoot reconciliation on the seed. While it is unavailable, namespaces keep their pin, and new namespaces stay unpinned, so their collectors run without the custom configuration until the next reconciliation of the shoot.
- gardenlet waits for the `Extension` resource of class `seed` before it deploys the seed system components. A shoot control plane namespace that the snapshots cannot be copied into blocks the `Seed` reconciliation. To avoid this for shoots that are being deleted, the extension controller skips terminating namespaces.

## Design Details

### Extension Registration

```yaml
apiVersion: operator.gardener.cloud/v1alpha1
kind: Extension
metadata:
  name: observability-config
spec:
  resources:
    - kind: Extension
      type: observability-config
      primary: true
      clusterCompatibility:
        - garden
        - seed
```

The extension controller reconciles the `Extension` resources of both classes and serves the webhooks. The admission component validates `Garden`s and `Seed`s. Since `clusterCompatibility` does not contain `shoot`, Gardener rejects `Shoot`s that list `observability-config` and installs the extension only on seeds that list it in `Seed.spec.extensions[]`.

### API

The `providerConfig` is of kind `ObservabilityConfig` in the API group `observability.extensions.gardener.cloud`.

```go
// ObservabilityConfig is the providerConfig of the observability-config extension.
type ObservabilityConfig struct {
  metav1.TypeMeta

  // Collector configures what the webhook injects into the collector in the
  // garden namespace.
  Collector *CollectorSpec
  // ShootCollectors configures what the webhook injects into the shoot
  // control plane collectors on the seed. If unset, nothing is injected.
  ShootCollectors *CollectorSpec
}

// CollectorSpec describes what is added to a collector.
type CollectorSpec struct {
  // Config is plain OpenTelemetry Collector configuration that is merged into
  // .spec.config of the collector.
  Config runtime.RawExtension
  // Resources names entries in .spec.resources of the Garden or Seed that are
  // mounted into the collector.
  Resources []string
  // Networking configures the egress of the collector.
  Networking *NetworkingSpec
}

// NetworkingSpec describes which networks the collector may reach.
type NetworkingSpec struct {
  // AllowPublicNetworks allows the collector to reach public networks.
  AllowPublicNetworks *bool
  // AllowPrivateNetworks allows the collector to reach private networks.
  AllowPrivateNetworks *bool
}

// ObservabilityStatus is the providerStatus of the Extension resources of both
// classes.
type ObservabilityStatus struct {
  metav1.TypeMeta

  // CollectorVersion is the version of Collector the collector in the garden
  // namespace is pinned to.
  CollectorVersion *string
  // ShootCollectorsVersion is the current version of ShootCollectors. The
  // namespace webhook pins the shoot control plane namespaces to it. Only set
  // by the seed class.
  ShootCollectorsVersion *string
}
```

Injected components and pipelines are added next to Gardener's, and `service.extensions` is appended to Gardener's list. An injected component or pipeline with the name of an existing one replaces Gardener's entirely, at the operator's own risk. Injected pipelines may reference Gardener's receivers and exporters by name.

Credentials come from [referenced resources](https://github.com/gardener/gardener/blob/master/docs/extensions/referenced-resources.md). The operator lists a `Secret` or `ConfigMap` in `.spec.resources` of the `Garden` or `Seed` and names the entry in `resources`, e.g., `central-loki`. The webhook mounts it at `/etc/observability/central-loki/`, where `config` reads it, e.g., via `${file:/etc/observability/central-loki/password}`. Since operators only deal with entry names, the same `CollectorSpec` works in `collector` and `shootCollectors` and in the `Garden` and the `Seed`. gardenlet copies the referenced objects of a `Seed` into the `garden` namespace as `ref-<object name>`, while `gardener-operator` uses those of a `Garden` in place.

#### Example

The following `Seed` sends the logs of all its shoot control planes to a central Loki via OTLP and adds the attribute `stage` to every log. The Loki credentials come from the referenced secret `central-loki-credentials`.

```yaml
apiVersion: core.gardener.cloud/v1beta1
kind: Seed
metadata:
  name: seed-eu01
spec:
  extensions:
    - type: observability-config
      providerConfig:
        apiVersion: observability.extensions.gardener.cloud/v1alpha1
        kind: ObservabilityConfig
        shootCollectors:
          resources:
            - central-loki
          networking:
            allowPublicNetworks: true
          config:
            extensions:
              basicauth/central:
                client_auth:
                  username: ${file:/etc/observability/central-loki/username}
                  password: ${file:/etc/observability/central-loki/password}
            processors:
              resource/central:
                attributes:
                  - key: stage
                    value: prod-eu01
                    action: insert
              batch/central: {}
            exporters:
              otlphttp/central:
                logs_endpoint: https://loki.example.com/otlp/v1/logs
                auth:
                  authenticator: basicauth/central
            service:
              extensions: [basicauth/central]
              pipelines:
                logs/central:
                  receivers: [otlp]
                  processors: [resource/central, batch/central]
                  exporters: [otlphttp/central]
  resources:
    - name: central-loki
      resourceRef:
        apiVersion: v1
        kind: Secret
        name: central-loki-credentials
```

### Versions and Rollout

#### Versions

The name of a version is a checksum over the class, the `CollectorSpec` and the data of its referenced resources, so every change, including a rotated credential, creates a new version, and the two classes never share one. In the `garden` namespace, a version consists of the immutable `ConfigMap` `observability-config-<version>`, which contains the `CollectorSpec`, and one immutable snapshot `observability-config-<version>-<name>` per referenced resource, of the same kind as the original. All objects of a version are labelled with its class. The annotation `observability.extensions.gardener.cloud/version: <version>` pins a namespace to a version.

In a shoot control plane namespace, the snapshots are the content of the `ManagedResource` `observability-config`, labelled `care.gardener.cloud/condition-type: ObservabilityComponentsHealthy`. `gardener-resource-manager` thus deletes snapshots that are removed from it and reports failures in the `ObservabilityComponentsHealthy` condition of the shoot. The controller records a version as current only after its snapshots are applied, so the namespace webhook pins existing namespaces only to versions whose snapshots they already contain.

#### Triggering the Collector

The collector webhook only acts when a collector is written, but a shoot reconciliation that does not change Gardener's collector configuration does not change its `ManagedResource`. To trigger a collector, the extension controller annotates its `ManagedResource` `opentelemetry-collector` with `gardener.cloud/operation: reconcile`. `gardener-resource-manager` then writes the collector if it differs from Gardener's desired state, which is always the case once it carries injected configuration, and the webhook records the injected version in the annotation `observability.extensions.gardener.cloud/injected-version`. A collector that never received an injection may match Gardener's desired state and not be written. If it does not carry the pinned version afterwards, the controller sends it an empty patch, which passes the webhook as well.

#### Cleanup

A version is in use while it is current, a namespace is pinned to it or a collector pod mounts one of its snapshots. The extension controller deletes unused versions of its own class, including their snapshots in the shoot control plane namespaces, during every reconciliation of its `Extension` resources and whenever a pin or a collector changes.

#### Deletion

When an operator removes the extension from `Seed.spec.extensions[]`, gardenlet deletes the `Extension` resource, and Gardener uninstalls the extension afterwards. Waiting for the reconciliation of every shoot would block this, so the removal applies to all collectors on the seed at once. The namespace webhook stops pinning, and the extension controller removes its pins and triggers the collectors, which `gardener-resource-manager` then writes without the custom configuration. Once no collector pod mounts a snapshot anymore, the controller deletes the snapshots and versions and removes its finalizer. The same applies to the `Garden`. The webhook configurations are removed together with the extension, so the collector webhook with `failurePolicy: Fail` does not outlive it.

Removing only `collector`, `shootCollectors` or an entry in `resources` creates a new version that rolls out like any other change.

### Validation

The admission component rejects a `Garden` or `Seed` if

- `config` contains keys other than those listed in the [Proposal](#proposal) or exceeds 64 KiB,
- a `Garden` sets `shootCollectors`, or
- an entry in `resources` does not name a `Secret` or `ConfigMap` in `.spec.resources`.

It does not validate the components or the pipeline wiring, since operators may run collector images with other components. The collector reports such errors when it starts.

### Webhooks

The `seed` class serves both webhooks and selects shoot control plane namespaces by the label `gardener.cloud/role: shoot`, since gardenlet sets `extensions.gardener.cloud/<type>` only for extensions enabled for shoots. The `garden` class only serves the collector webhook for the `garden` namespace. The extension uses dedicated webhooks instead of the generic `controlplane` webhook of the extensions library, which is meant for extensions of kind `ControlPlane` and filters by the provider type of a shoot.

#### Namespace Webhook

The namespace webhook mutates shoot control plane `Namespace`s on `CREATE` and `UPDATE` and pins them to `shootCollectorsVersion` of the `Extension` resource of class `seed`. If that resource does not exist, is being deleted or has no `shootCollectorsVersion`, the webhook removes the pin. For `Namespace`s, its `namespaceSelector` matches the labels of the object itself, also on `CREATE`. It uses `failurePolicy: Ignore` (see [Risks and Mitigations](#risks-and-mitigations)).

#### Collector Webhook

The collector webhook mutates `OpenTelemetryCollector` resources with the label `observability.gardener.cloud/app: opentelemetry-collector` in the `garden` namespace and in shoot control plane namespaces on `CREATE` and `UPDATE`. On a garden runtime cluster that is also a seed, the webhook of the seed deployment only serves the shoot control plane namespaces. It uses `failurePolicy: Fail` (see [Risks and Mitigations](#risks-and-mitigations)).

The webhook reads the pin of the namespace and the `ConfigMap` of the pinned version and merges `config` into `.spec.config` (see [API](#api)). It mounts the snapshots at `/etc/observability/<name>/`, sets the network policy labels and records the injected version. Without a pin, it injects nothing and removes the labels and the annotation. Since `gardener-resource-manager` always writes Gardener's unmodified configuration, removed and overridden components revert with the next write, without bookkeeping in the webhook. A new version mounts snapshots with new names, which changes the pod template, so the collector rolls out new pods.

### Required Changes in gardener/gardener

- Document the label `observability.gardener.cloud/app: opentelemetry-collector` of the `OpenTelemetryCollector` resource, its receiver and exporter names and the name of its `ManagedResource` `opentelemetry-collector` as a stable interface for extensions.

## Drawbacks

- The extension mutates resources that Gardener owns. Renaming the collector label, its receivers, its exporters or its `ManagedResource` in `gardener/gardener` breaks the extension, which is why this GEP asks to document them as a stable interface.
- The rollout relies on gardenlet patching the shoot control plane namespace during every shoot reconciliation, which is not documented so far.
- Urgent changes, e.g., replacing a leaked credential, require the reconciliation of all shoots on the seed.
- Every new version of `shootCollectors` copies the referenced resources into all shoot control plane namespaces on the seed.

## Alternatives

### Extend gardener-extension-otelcol

`gardener-extension-otelcol` is configured per shoot in the `Shoot` spec, does not cover the `garden` collectors and deploys an additional collector, which does not receive the data plane logs from the shoot `kube-system` namespace. Reconfiguring Gardener's collectors through it would change its purpose, which is why [gardener/gardener#15700](https://github.com/gardener/gardener/issues/15700) settled on a separate extension.

For shoot owners, it stays the way to export the signals of their control plane. Its additional collector isolates their configuration, so a misconfiguration or high signal volumes do not affect the collector Gardener deploys. Shoot owner configuration in Gardener's collectors would also need sanitization and guardrails, since configuration providers such as `${file:...}` or `${env:...}` can read anything the collector pod sees.

### Roll out Seed changes to all shoots immediately

The collector webhook could read `shootCollectors` directly from the `Extension` resource of class `seed`. A `Seed` change would then reach all shoots at once, and also whenever `gardener-resource-manager` happens to write a collector. A broken configuration would break the collectors of all shoots on the seed at the same time.

### Shoot class

An `Extension` resource of class `shoot` for every shoot, enabled via `autoEnable: [shoot]`, could carry the configuration into each shoot control plane during the shoot reconciliation. This was the previous version of this GEP. The extension would then be visible in every `Shoot`, where shoot owners could try to disable or configure it, which is confusing. It would also install the extension on every seed that hosts shoots and add an `Extension` step to every shoot reconciliation, even on seeds that do not use it.

### Custom fluent-bit outputs and filters

A custom fluent-bit `ClusterOutput` could ship logs to a custom backend, with `ClusterFilter`s to drop or modify them. This couples fluent-bit to the backend, limits processing to what fluent-bit filters can do and bypasses the collectors, which Gardener introduced as the abstraction layer for all signals.

### Deploy a dedicated operator-owned collector

An additional collector per seed or per shoot control plane would need either an additional fluent-bit output or an exporter in Gardener's collectors. A fluent-bit output misses the data plane logs from the shoot `kube-system` namespace, and an exporter again requires mutating the configuration of Gardener's collectors. It also duplicates infrastructure that already receives the signals.

### A controller that patches the collectors

`gardener-resource-manager` would revert the patch with its next write of the collector, and the two components would overwrite each other's changes. Making `gardener-resource-manager` ignore parts of the spec would require changes in Gardener core for this one use case.

### Model the extension point in Gardener core

gardenlet and `gardener-operator` could render additional collector pipelines from a field in the `Seed` and `Garden` APIs. This puts landscape-specific configuration and the OpenTelemetry Collector configuration format into Gardener's core APIs and makes core responsible for validating it.

### Configure the extension via its controller deployment

The configuration could live in the `ControllerDeployment` of the extension, with one `ControllerDeployment` and seed selector per configuration. A `ControllerDeployment` can reference resources only to template them into the Helm values, while referenced resources of the `Seed` and `Garden` are available to the extension, and those of a `Seed` already support `WorkloadIdentity`s. Configuring the shoot control plane collectors there and the seed collector in the `Seed` would also split the configuration across two places.

### Reference the configuration instead of inlining it

The `providerConfig` could reference a `ConfigMap` instead of containing the configuration. The `Seed` spec is copied into the `Cluster` resource of every shoot on the seed, so an inlined configuration grows every `Cluster` resource. The configuration is expected to stay small, since operators add a few processors, exporters and pipelines. Anything larger belongs in a separate collector, and the admission component limits it to 64 KiB. Inlining keeps the configuration visible in the `Seed` and validated when the `Seed` is saved. The size of inlined configuration concerns all extensions and is better solved for all of them.
