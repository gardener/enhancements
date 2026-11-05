# GEP-0084: Observability Extension for Gardener

## Summary

This GEP proposes a new extension `gardener-extension-observability-config` of type `observability` in the Gardener organization. It deploys a mutating admission webhook which injects additional configuration into the `OpenTelemetryCollector` resources Gardener deploys. The injected configuration can add to and override Gardener's collector configuration. The extension deploys no additional collectors.

Operators configure the extension via the `providerConfig` in the `Garden` and `Seed` resources. Shoot owners cannot configure it. The `Seed` carries one configuration for its own collector and one for the shoot control plane collectors, which reaches them during the reconciliation of each shoot. Shoot owners who want custom configuration for the signals of their control plane use the existing [`gardener-extension-otelcol`](https://github.com/gardener/gardener-extension-otelcol), which deploys an additional collector for them. For now this targets only logs, but the mechanism is signal-agnostic and covers metrics once Gardener routes them through the collectors.

## Motivation

With the `OpenTelemetryCollector` feature gate, Gardener deploys an `OpenTelemetryCollector` per shoot control plane namespace and one in the `garden` namespace of seeds and of the garden runtime cluster. Per [GEP-34](../0034-observability2.0-opentelemetry/README.md), these collectors are the abstraction layer for observability signals, which makes them a stable entry point for operators.

Operators may have requirements for the signals collected in their setup that go beyond what Gardener provides out of the box. A common one may be shipping logs to a custom backend, for example a central log store outside of Gardener. Another one is processing logs before they leave the cluster, for example attaching landscape-specific attributes (e.g., the stage). Gardener fully owns the collector configuration, so there is no extension point for this. This was discussed in [gardener/gardener#15700](https://github.com/gardener/gardener/issues/15700), where the consensus was to create a new extension that offers an entry point into configuring the Gardener-owned collectors.

### Goals

- Introduce `gardener-extension-observability-config` as a new extension of type `observability` in the Gardener organization.
- Allow operators to add custom configuration to the collectors Gardener deploys, reusing the receivers Gardener already configures.
- Apply configuration at the `Garden` and `Seed` level, covering the `garden` namespace collectors and all shoot control plane collectors on a seed without per-shoot configuration.
- Roll out `Seed`-level changes to the shoot control plane collectors during shoot reconciliation.
- Keep the API signal-agnostic so that the same mechanism covers metrics and traces once Gardener routes them through the collectors.

### Non-Goals

- Configuration by shoot owners. Shoot owners use `gardener-extension-otelcol` instead (see [Alternatives](#alternatives)).
- Changing how Gardener collects or stores signals (fluent-bit, the `gardener` plugin).
- Modelling the OpenTelemetry Collector configuration in a Gardener-specific API. The extension passes configuration fragments through.
- Validating the OpenTelemetry Collector configuration beyond its structure.
- Replacing `gardener-extension-otelcol`.
- Deploying additional collectors.
- Metrics and traces in the first iteration.

## Proposal

The `OpenTelemetryCollector` resources Gardener deploys are the extension point. The extension extends their configuration instead of deploying anything next to them.

Since `gardener-resource-manager` would revert changes to the collectors, a mutating admission webhook injects the configuration on every apply.

The webhook adds processors, exporters, connectors, pipelines and collector extensions. Receivers cannot be added. Gardener's collectors are meant for the signals Gardener emits. Other signals belong in a separate collector. Injected components may override Gardener's components of the same name (see [API](#api)).

The extension has three classes, which share the webhook and differ in where the configuration comes from:

- Operators configure the `garden` class in `Garden.spec.extensions[]`. It covers the collector in the `garden` namespace of the garden runtime cluster.
- Operators configure the `seed` class in `Seed.spec.extensions[]`. It covers the collector in the `garden` namespace of the seed (`collector`). It also carries the configuration for the shoot control plane collectors on the seed (`shootCollectors`).
- The `shoot` class has no configuration of its own. During shoot reconciliation, it fetches `shootCollectors` from the `seed` class for the shoot control plane collector (see [Rollout](#rollout)). Operators enable it for all shoots with `autoEnable: [shoot]` in the extension registration (see [Extension Registration](#extension-registration)). Shoot owners can neither enable nor disable it.

Every class writes the configuration for its collector into the status of its `Extension` resource. The webhook only reads the status of the `Extension` resource in the namespace of the collector it mutates (see [Webhook](#webhook)). Without such an `Extension` resource or with an empty status, nothing is injected.

If the runtime cluster is also a seed, the extension runs twice there, once deployed by `gardener-operator` for the `Garden` and once by gardenlet for the `Seed`. There is one collector in the `garden` namespace, deployed by `gardener-operator`, and it receives the `collector` configuration of the `Garden` only. Gardenlet passes `gardener.clusterTypes.gardenRuntimeCluster: true` to the Helm chart of the seed deployment. With this value, the `seed` class ignores `collector`, writes no `collector` status, copies no secrets for it and sets a condition on its `Extension` resource that says so. Its webhook only serves the shoot control plane namespaces. `shootCollectors` of that `Seed` still applies to the shoot control plane collectors.

Gardener's network policies allow the collectors to reach only the in-cluster log backends, DNS and the `kube-apiserver`. To reach custom backends, the webhook adds the label `networking.gardener.cloud/to-public-networks: allowed` when `networking.allowPublicNetworks` is set and `networking.gardener.cloud/to-private-networks: allowed` when `networking.allowPrivateNetworks` is set. `gardener-resource-manager` keeps labels it does not manage, so the webhook removes the labels when the fields are not set.

### Notes/Constraints/Caveats

- Operators can only use processors, exporters, connectors and collector extensions contained in the collector image. By default this is the [Gardener OpenTelemetry Collector distribution](https://github.com/gardener/opentelemetry-collector/blob/main/manifest.yml). Since the extension only acts on `Seed` and `Garden` level, operators can replace the collector image with one that contains further components, e.g., the full `opentelemetry-collector-contrib` distribution.
- Injected pipelines reference Gardener's receivers and exporters by name. Gardener should document these names as a stable interface for extensions (see [Required Changes in gardener/gardener](#required-changes-in-gardenergardener)).
- A `Seed`-level change, including a change of a referenced secret, reaches a shoot control plane collector during the next reconciliation of that shoot. Changes to the `Seed` do not trigger shoot reconciliations.
- `autoEnable: [shoot]` cannot be restricted to the shoots of selected seeds. On seeds that do not list the extension, the webhook changes nothing.
- All signals arriving at a collector are included. For a shoot control plane collector, these are the control plane logs, the logs of the Gardener-managed components in the shoot `kube-system` namespace and the logs of the shoot node systemd units.

### Risks and Mitigations

- An invalid configuration prevents the collector from starting, so Gardener's own pipelines stop too. The extension only validates the structure of the fragment and the referenced secrets (see [Validation](#validation)). Errors inside the configuration only show up when the collector starts.
- Injected pipelines share the receiver with Gardener's pipelines. Additional processors and exporters increase the resource consumption of the collector and can degrade its performance. A custom backend that is unavailable or misbehaves can degrade all pipelines of the collector, including Gardener's.
- Overriding Gardener's components replaces their configuration entirely. A broken override breaks Gardener's own pipelines.
- The webhook uses `failurePolicy: Ignore`, so an outage of the webhook does not block the `Garden`, `Seed` and shoot reconciliations. During the outage `gardener-resource-manager` applies the collectors without the custom configuration. The controller detects this, triggers the webhook again and fails the reconciliation of the `Extension` resource until the configuration is back (see [Rollout](#rollout)).

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
      type: observability
      primary: true
      # Set by the operator to configure all shoot control plane collectors.
      autoEnable:
        - shoot
      clusterCompatibility:
        - garden
        - seed
        - shoot
      # Workerless shoots also have a control plane collector.
      workerlessSupported: true
```

The extension controller reconciles the `Extension` resources of all three classes and serves the mutating webhook (see [Webhook](#webhook)). The admission component validates `Seed`s, `Garden`s and `Shoot`s (see [Validation](#validation)).

### API

The `providerConfig` is of kind `ObservabilityConfig` in the API group `observability.extensions.gardener.cloud`. `config` is plain OpenTelemetry Collector configuration.

```go
// ObservabilityConfig is the providerConfig of the observability extension.
type ObservabilityConfig struct {
  metav1.TypeMeta

  // Collector configures what the webhook injects into the collector in the
  // garden namespace.
  Collector *CollectorSpec
  // ShootCollectors configures what the webhook injects into the shoot
  // control plane collectors on the seed. Only allowed in a Seed. If unset,
  // the shoot control plane collectors are left unchanged.
  ShootCollectors *CollectorSpec
}

// CollectorSpec describes what is added to a collector.
type CollectorSpec struct {
  // Config is a piece of OpenTelemetry Collector configuration that is added
  // to .spec.config of the collector. Allowed top-level keys are processors,
  // exporters, extensions, connectors and service. Under service, only
  // pipelines and extensions are allowed. Receivers cannot be configured.
  Config runtime.RawExtension
  // Secrets are names of entries in .spec.resources of the Garden or Seed.
  // Each listed secret is mounted into the collector at
  // /etc/observability/<name>/. Config reads the files, e.g., via
  // ${file:/etc/observability/<name>/<key>}.
  Secrets []string
  // Networking configures the egress of the collector.
  Networking *NetworkingSpec
}

// NetworkingSpec describes which networks the collector may reach.
type NetworkingSpec struct {
  // AllowPublicNetworks allows the collector to reach public networks.
  AllowPublicNetworks bool
  // AllowPrivateNetworks allows the collector to reach private networks.
  AllowPrivateNetworks bool
}

// ObservabilityStatus is the providerStatus of the Extension resources of all
// classes.
type ObservabilityStatus struct {
  metav1.TypeMeta

  // Collector is the configuration the webhook injects into the collector in
  // the namespace of the Extension resource.
  Collector *CollectorSpec
  // ShootCollectors is the configuration the shoot class picks up. Only set
  // by the seed class.
  ShootCollectors *CollectorSpec
}
```

The webhook adds `config` to `.spec.config` of the collector.

- The webhook adds processors, exporters, extensions, connectors and pipelines next to Gardener's and appends `service.extensions` to Gardener's list. Injected pipelines may reference Gardener's receivers and exporters by name.
- If a name already exists in the collector, the injected component replaces Gardener's component entirely. Overriding Gardener's components is at the operator's own risk.

Credentials come from [referenced resources](https://github.com/gardener/gardener/blob/master/docs/extensions/referenced-resources.md), which the webhook mounts as files.

1. The operator lists a `Secret` in `.spec.resources` of the `Seed` or `Garden`.
2. `secrets` names the entry, e.g., `central-loki`.
3. `config` reads the files, e.g., `${file:/etc/observability/central-loki/password}` or `ca_file: /etc/observability/central-loki/ca.crt`.
4. For a `Seed`, gardenlet copies the referenced `Secret` into the `garden` namespace as `ref-<name>`. For a `Garden`, the referenced `Secret` already lives in the `garden` namespace. The extension controller copies it from there into the namespace of the collector (see [Rollout](#rollout)) and the webhook mounts the copy.

Operators only deal with entry names, so the same `CollectorSpec` works in `collector` and `shootCollectors` and in the `Garden` and the `Seed`.

#### Example

The following `Seed` sends the logs of all shoot control planes on the seed to a central Loki via OTLP. A `resource` processor adds the attribute `stage` to every log. The Loki credentials come from the secret `central-loki-credentials`, which is referenced as `central-loki` in `.spec.resources`. `networking.allowPublicNetworks` lets the collectors reach the Loki endpoint. The same configuration under `collector` would apply to the collector in the `garden` namespace.

```yaml
apiVersion: core.gardener.cloud/v1beta1
kind: Seed
metadata:
  name: seed-eu01
spec:
  extensions:
    - type: observability
      providerConfig:
        apiVersion: observability.extensions.gardener.cloud/v1alpha1
        kind: ObservabilityConfig
        shootCollectors:
          secrets:
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

### Rollout

The extension controller reconciles the `Extension` resources of all three classes with the same steps. The `garden` and `seed` classes take the configuration from their `providerConfig`, the `shoot` class from `shootCollectors` in the status of the `Extension` resource of class `seed`. Secrets come from the referenced resources in the `garden` namespace.

For every reconciliation, the controller

1. copies each secret in `secrets` into the namespace of the `Extension` resource as `observability-<name>`,
2. writes the configuration to `collector` in the status of the `Extension` resource (the `seed` class also writes `shootCollectors`),
3. sends an empty patch to the collector in the namespace, which triggers the webhook (see [Webhook](#webhook)),
4. deletes secret copies that are no longer listed once the collector no longer mounts them.

This order means the webhook never mounts a secret that does not exist yet. If no `Extension` resource of class `seed` exists or its status has no `shootCollectors`, the `shoot` class writes an empty status and the webhook removes everything it injected before.

Gardenlet reconciles the `Extension` resources of class `shoot` as part of the shoot reconciliation, so a `Seed`-level change reaches the shoot control plane collectors one shoot at a time. The `seed` and `garden` classes are reconciled during the `Seed` and `Garden` reconciliation.

The controller also watches the collector in the namespace of each `Extension` resource and enqueues the `Extension` resource when the collector changes. During the reconciliation, it compares the hash the webhook writes to the pod annotations (see [Webhook](#webhook)) with the hash of the status and the secret copies. If they differ, e.g., because `gardener-resource-manager` applied the collector while the webhook was unavailable, the controller sends the empty patch again. If the collector still lacks the configuration afterwards, the reconciliation fails, so the `Garden`, `Seed` or shoot reconciliation reports the error and retries.

#### Deletion

When an operator removes the extension from `Seed.spec.extensions[]`, gardenlet deletes the `Extension` resource of class `seed`. The `shoot` class then runs the steps above with an empty configuration. The controller clears the status, the empty patch makes the webhook remove the injected configuration, the volumes and the network policy labels, and the controller deletes the secret copies once the collector no longer mounts them. The controller does the same for the `garden` namespace collector when gardenlet or `gardener-operator` deletes the `Extension` resource of class `seed` or `garden`. It handles a dropped entry in `secrets` or a dropped `shootCollectors` section the same way.

### Validation

The admission component validates the `providerConfig` when a `Garden` or `Seed` is saved and rejects shoots that enable or disable the extension.

- Only the top-level keys `processors`, `exporters`, `extensions`, `connectors` and `service` are allowed. Under `service`, only `pipelines` and `extensions` are allowed. `receivers` cannot be configured.
- `shootCollectors` is only allowed in a `Seed`.
- Every entry in `secrets` must name a `Secret` in `.spec.resources` of the `Garden` or `Seed`. The extension copies and mounts only listed secrets.
- `Shoot.spec.extensions[]` must not contain an entry of type `observability`. The `shoot` class exists only through `autoEnable`.

The extension does not validate the content of the components, the component types or the pipeline wiring. Operators may run collector images with a different set of components, so the extension cannot know which component types exist. The collector reports errors when it starts.

### Webhook

The webhook mutates `OpenTelemetryCollector` resources on `CREATE` and `UPDATE`. It only touches the collectors Gardener deploys.

- A `namespaceSelector` limits it to the `garden` namespace and to shoot control plane namespaces (label `gardener.cloud/role: shoot`). On a runtime cluster that is also a seed, the seed deployment's webhook serves only the shoot control plane namespaces (see [Proposal](#proposal)).
- An `objectSelector` limits it to resources with the label `observability.gardener.cloud/app: opentelemetry-collector`.

The webhook reads the `ObservabilityStatus` of the `Extension` resource in the namespace of the collector. In a shoot control plane namespace this is the `Extension` resource of class `shoot`. In the `garden` namespace it is the `Extension` resource of class `garden` (`garden-observability`) on the runtime cluster and the one of class `seed` (`observability`) on a seed. If there is no matching `Extension` resource or its status is empty, the webhook injects nothing and removes what it injected before.

Mutation is idempotent. The webhook records the names of the injected components, pipelines, extensions and volumes in an annotation on the collector and removes them before injecting the current configuration. This way, components removed from the `providerConfig` disappear with the next empty patch.

The webhook mounts the secret copies at `/etc/observability/<name>/` (see [API](#api)), sets the network policy labels depending on `networking` (see [Proposal](#proposal)) and adds a hash of the injected configuration and the secret data to the pod annotations, so that changes roll out new pods.

The webhook uses `failurePolicy: Ignore` (see [Risks and Mitigations](#risks-and-mitigations)).

### Required Changes in gardener/gardener

- Document the label `observability.gardener.cloud/app: opentelemetry-collector` of the `OpenTelemetryCollector` resource and its receiver and exporter names as a stable interface for extensions.

## Drawbacks

- The extension mutates resources that Gardener owns. Renames of the collector label, its receivers or its exporters in `gardener/gardener` break the injected pipelines, which is why this GEP asks to document them as a stable interface.
- With `autoEnable: [shoot]`, the extension is installed on every seed and adds an `Extension` step to the reconciliation of every shoot, even on seeds that do not use it.
- A misconfigured processor or exporter in `shootCollectors` affects all shoot control plane collectors on the seed, one shoot reconciliation at a time.

## Alternatives

### Extend gardener-extension-otelcol

`gardener-extension-otelcol` already configures exporters via `CollectorConfig.spec.targets`, but it is shoot-spec-driven, requires per-shoot configuration and does not cover the `garden` collectors. It deploys an additional collector, which does not receive the data plane logs from the shoot `kube-system` namespace, and does not modify the collectors Gardener deploys. Letting operators reconfigure the Gardener collectors through it would change its purpose, which is why gardener/gardener#15700 settled on a separate extension.

For shoot owners, `gardener-extension-otelcol` stays the way to export signals of their control plane. The additional collector isolates the owner configuration, so a misconfiguration or high signal volumes do not affect the collector Gardener deploys. This is why this GEP does not allow configuration in the `Shoot` spec. Configuration providers such as `${file:...}` or `${env:...}` can read anything the collector pod sees, so shoot owner configuration in Gardener's collectors would need sanitization and guardrails.

### Seed-level controller without a shoot class

The `seed` class alone could copy secrets into every shoot control plane namespace and trigger the webhook for all collectors on the seed at once. This rolls out a `Seed` change to all shoots immediately, outside of the shoot reconciliation. It also has to handle shoot namespaces that appear later, e.g., after a control plane migration, and races with gardenlet creating the collector before the secrets are copied. The `shoot` class ties all of this to the shoot reconciliation.

### Custom fluent-bit outputs and filters

A custom fluent-bit `ClusterOutput` ships logs directly to a custom backend and `ClusterFilter`s drop or modify logs before that. This couples fluent-bit to the backend, limits processing to what fluent-bit filters can do and bypasses the collectors, which Gardener introduced as the abstraction layer for all signals.

### Deploy a dedicated operator-owned collector

An additional collector per seed or per shoot control plane would need either an additional fluent-bit output or an additional exporter in the Gardener collectors pointing to it. The latter again requires mutating their configuration. An additional fluent-bit output would also not include the data plane logs from the shoot `kube-system` namespace, since they arrive at Gardener's collectors directly. It duplicates infrastructure that already receives the signals and adds another component to maintain.

### A controller that patches the collectors

Since `gardener-resource-manager` applies the collectors via `ManagedResource`s, it reverts the patch on the next apply and the two components overwrite each other's changes. Making `gardener-resource-manager` ignore parts of the spec would require changes in Gardener core for this one use case.

### Model the extension point in Gardener core

`gardenlet` and `gardener-operator` could render additional collector pipelines from a field in the `Seed` and `Garden` APIs. This puts landscape-specific configuration and the OpenTelemetry Collector configuration format into Gardener's core APIs and makes core responsible for validating it.

### Use the generic controlplane mutator

The `controlplane` webhook of the extensions library is meant for extensions of kind `ControlPlane` and filters by the provider type of a shoot. Reusing it here would complicate it for a use case outside of its scope. The extension uses a dedicated webhook.
