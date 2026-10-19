# GEP-0084: Observability Extension for Gardener

## Summary

This GEP proposes a new extension `gardener-extension-observability-config` of type `observability` in the Gardener GitHub organization. It deploys a mutating admission webhook which injects additional configuration into the `OpenTelemetryCollector` resources Gardener deploys. The injected configuration adds pipelines that reuse the receivers Gardener already configures and reference custom processors and exporters, without touching the pipelines Gardener configures. No additional collectors are deployed.

The extension is configured via the `providerConfig` in the `Garden` and `Seed` resources. The `seed` class applies to all shoot control plane collectors on a seed without per-shoot configuration. Shoot owners keep using the existing [`gardener-extension-otelcol`](https://github.com/gardener/gardener-extension-otelcol) to export the signals of their control plane. For now this targets only logs, but the mechanism is signal-agnostic and covers metrics once Gardener routes them through the collectors.

## Motivation

With the `OpenTelemetryCollector` feature gate, Gardener deploys an `OpenTelemetryCollector` per shoot control plane namespace and one in the `garden` namespace of seeds and of the garden runtime cluster. Per [GEP-34](../0034-observability2.0-opentelemetry/README.md), these collectors are the abstraction layer for observability signals, which makes them a stable entry point for operators.

What is missing is a supported way for operators to hook into the collector configuration. It is fully owned by Gardener, so an operator who wants to route the collected signals to a custom backend or use custom processors has no extension point.

Operators may have requirements for the signals collected in their setup that go beyond what Gardener provides out of the box. A common one may be shipping logs to a custom backend, for example a central log store outside of Gardener. Another one is processing logs before they leave the cluster, for example attaching landscape-specific attributes (e.g., the stage).

Operators can cover this with a custom fluent-bit `ClusterOutput` that ships logs directly to a custom backend and `ClusterFilter`s to drop certain logs. Here fluent-bit is directly coupled to the log storage backend and processing is limited to what fluent-bit filters can do. Alternatively, operators can deploy their own OpenTelemetry Collector next to Gardener's and feed it via an additional fluent-bit output. This duplicates infrastructure that already receives the signals and adds another component to maintain. Neither option allows injecting configuration into the collectors Gardener deploys. These are the better entry point as they receive all signals of a seed or of the garden runtime cluster and they stay in place even if the components around them change.

The existing `gardener-extension-otelcol` deploys an additional collector per shoot control plane and gives shoot owners a restricted, shoot-spec-driven way to export signals of their control plane. This fits shoot owners, but not operators. It requires per-shoot configuration and does not cover the `garden` collectors. It also does not allow changing the collectors Gardener deploys and does not include the data plane logs from the shoot `kube-system` namespace that arrive at Gardener's collectors. Operators need a unified way to apply configuration at the seed level across all shoot control planes. This was discussed in [gardener/gardener#15700](https://github.com/gardener/gardener/issues/15700), where the consensus was to create a new, dedicated extension.

### Goals

- Introduce `gardener-extension-observability-config` as a new extension of type `observability` in the Gardener GitHub organization.
- Allow operators to add custom configuration to the collectors Gardener deploys, reusing the receivers Gardener already configures.
- Apply configuration at the `Garden` and `Seed` level, covering the `garden` namespace collectors and all shoot control plane collectors on a seed without per-shoot configuration.
- Keep the API signal-agnostic so that the same mechanism covers metrics and traces once Gardener routes them through the collectors.

### Non-Goals

- Configuration by shoot owners. Shoot owners use `gardener-extension-otelcol` instead (see [Alternatives](#alternatives)).
- Changing how Gardener collects or stores signals (fluent-bit, the `gardener` plugin).
- Modelling the OpenTelemetry Collector configuration in a Gardener-specific API. The extension passes configuration fragments through and validates them.
- Replacing `gardener-extension-otelcol`.
- Deploying additional collectors or changing the collector pipelines Gardener configures.
- Metrics and traces in the first iteration.

## Proposal

The `OpenTelemetryCollector` resources Gardener deploys are the extension point. Their configuration is extended instead of deploying anything next to them.

Since `gardener-resource-manager` would revert changes to the collectors, a mutating admission webhook injects the configuration on every apply.

The webhook only adds processors, exporters, connectors, pipelines and collector extensions. Receivers cannot be added. The injected pipelines reference the receivers Gardener already configures.

The collectors Gardener deploys only carry network policy labels for the in-cluster log backends, DNS and, if required, the `kube-apiserver`. To allow egress to custom backends, the webhook adds the label `networking.gardener.cloud/to-public-networks: allowed` to the collector if `networking.allowPublicNetworks` is set and `networking.gardener.cloud/to-private-networks: allowed` if `networking.allowPrivateNetworks` is set.

The `garden` class is configured in `Garden.spec.extensions[]` and covers the collector in the `garden` namespace of the garden runtime cluster, which `gardener-operator` deploys. The `seed` class is configured in `Seed.spec.extensions[]` and covers the collector in the `garden` namespace of the seed and all shoot control plane collectors on the seed. Both classes accept every component of Gardener's collector distribution and are validated the same way (see [Validation](#validation)). If the runtime cluster is also a seed, gardenlet skips the collector in its `garden` namespace and the collector deployed by `gardener-operator` receives both the `garden` and the `seed` configuration.

### Notes/Constraints/Caveats

- Only processors, exporters, connectors and collector extensions contained in the [Gardener OpenTelemetry Collector distribution](https://github.com/gardener/opentelemetry-collector/blob/main/manifest.yml) can be configured, since the collector image only contains these components. Missing components are contributed to the distribution. Since the extension only acts on `Seed` and `Garden` level, operators can still replace the collector image with one that contains further components, e.g., the full `opentelemetry-collector-contrib` distribution. Gardener's own distribution exists to avoid deploying all contrib components by default.
- Injected pipelines reference Gardener's receivers and exporters by name. Gardener should document these names as a stable interface for extensions (see [Required Changes in gardener/gardener](#required-changes-in-gardenergardener)).
- The webhook only runs when the collector resource is written. When the `providerConfig` or a referenced secret changes, the extension controller first updates the secret copies and then sends an empty patch to the affected collectors. The patch triggers the webhook, which recomputes the hash in the pod annotations (see [Webhook](#webhook)). The same happens once the webhook becomes ready on a cluster, so collectors applied before the webhook existed are mutated as well.
- All signals arriving at a collector are included. For a shoot control plane collector, these are the control plane logs, the logs of the Gardener-managed components in the shoot `kube-system` namespace and the logs of the shoot node systemd units.

### Risks and Mitigations

- An invalid configuration prevents the collector from starting, which stops all pipelines of the affected collector, including Gardener's. The extension validates the structure of the fragment before it is accepted and again on mutation (see [Validation](#validation)). Errors inside a component's configuration only show up when the collector starts.
- A `Seed`-level change affects all shoot control plane collectors on the seed at once. A `Garden`-level change only affects the collector in the runtime cluster.
- Injected pipelines share the receiver with Gardener's pipelines. Additional processors and exporters increase the resource consumption of the collector and can degrade its performance. A custom backend that is unavailable or misbehaves can degrade all pipelines of the collector, including Gardener's.
- The webhook uses `failurePolicy: Fail`. With `Ignore`, an outage would drop the injected pipelines on the next apply. Signals would then stop reaching the custom backend without any error in the `Seed` or `Garden` status. With `Fail`, an outage blocks the apply of the collector and thereby the reconciliation of the `Garden`, the `Seed` and the shoots on that seed until the webhook is back. It also makes the health checks of the affected `ManagedResource`s fail, since `gardener-resource-manager` cannot apply them.

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
      clusterCompatibility:
        - garden
        - seed
```

The extension controller serves the mutating webhook that changes the collectors (see [Webhook](#webhook)). For the `seed` class, it copies the secrets listed in `collector.secrets` into every shoot control plane namespace on the seed. This includes namespaces that are created later, e.g., for new shoots or after a control plane migration. It also triggers the webhook again when the `providerConfig` or a referenced secret changes (see [Notes/Constraints/Caveats](#notesconstraintscaveats)).

The admission component validates the `providerConfig` of `Seed`s and `Garden`s (see [Validation](#validation)).

### API

The `providerConfig` is of kind `ObservabilityConfig` in the API group `observability.extensions.gardener.cloud`. `collector.config` is plain OpenTelemetry Collector configuration.

```go
// ObservabilityConfig is the providerConfig of the observability extension.
type ObservabilityConfig struct {
  metav1.TypeMeta

  // Collector configures what the webhook injects into the collectors.
  Collector *CollectorSpec
}

// CollectorSpec describes what is added to the collectors.
type CollectorSpec struct {
  // Config is a piece of OpenTelemetry Collector configuration that is added
  // to .spec.config of the collectors. Allowed top-level keys are processors,
  // exporters, extensions, connectors and service. Under service, only
  // pipelines and extensions are allowed. Receivers cannot be configured.
  Config runtime.RawExtension
  // Secrets are names of entries in .spec.resources of the Garden or Seed.
  // Each listed secret is mounted into the collectors at
  // /etc/observability/<name>/. Config reads the files, e.g., via
  // ${file:/etc/observability/<name>/<key>}.
  Secrets []string
  // Networking configures the egress of the collectors.
  Networking *NetworkingSpec
}

// NetworkingSpec describes which networks the collectors may reach.
type NetworkingSpec struct {
  // AllowPublicNetworks allows the collectors to reach public networks.
  AllowPublicNetworks bool
  // AllowPrivateNetworks allows the collectors to reach private networks.
  AllowPrivateNetworks bool
}
```

The webhook adds the content of `collector.config` to `.spec.config` of each collector.

- Processors, exporters, extensions, connectors and pipelines are added next to Gardener's. If a name already exists in the collector, the injected component overrides Gardener's. Overriding Gardener's components is at the operator's own risk.
- Extensions listed under `service.extensions` are added to Gardener's list.
- Pipelines can use Gardener's receivers and exporters in addition to the components from `collector.config`.

Credentials come from [referenced resources](https://github.com/gardener/gardener/blob/master/docs/extensions/referenced-resources.md) and are mounted as files.

1. The operator lists a `Secret` in `.spec.resources` of the `Seed` or `Garden`.
2. `collector.secrets` names the entry, e.g., `central-loki`.
3. `collector.config` reads the files, e.g., `${file:/etc/observability/central-loki/password}` or `ca_file: /etc/observability/central-loki/ca.crt`.
4. The webhook mounts the secret into the collector pod.

Operators only deal with entry names, so the `Garden` and `Seed` can use the same `providerConfig`.

#### Example

The following `Seed` sends all logs of the seed to a central Loki via OTLP. A `resource` processor adds the attribute `stage` to every log. The Loki credentials come from the secret `central-loki-credentials`, which is referenced as `central-loki` in `.spec.resources`. `networking.allowPublicNetworks` lets the collectors reach the Loki endpoint. A `Garden` uses the same `providerConfig` for the collector in the runtime cluster.

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
        collector:
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

### Validation

The admission component validates the `providerConfig` when a `Garden` or `Seed` is saved. References to Gardener's components depend on the actual collector, so the webhook checks them when it mutates a collector.

- Every component type must be part of Gardener's collector distribution, e.g., `otlphttp`. The configuration inside a component is not validated.
- `receivers` cannot be configured. Under `service`, only `pipelines` and `extensions` are allowed.
- Every pipeline must start at a receiver of Gardener or a connector from `collector.config`.
- Every entry in `collector.secrets` must name a `Secret` in `.spec.resources` of the `Garden` or `Seed`. Only listed secrets are copied and mounted. Secrets of other extensions in `.spec.resources` stay out of the collectors.

### Webhook

The webhook mutates `OpenTelemetryCollector` resources on `CREATE` and `UPDATE`. It is restricted to the collectors Gardener deploys.

- A `namespaceSelector` limits it to the `garden` namespace and to shoot control plane namespaces (label `gardener.cloud/role: shoot`).
- An `objectSelector` limits it to resources with the label `observability.gardener.cloud/app: opentelemetry-collector`.
- The webhook itself checks that the resource is named `opentelemetry-collector`. This keeps it away from collectors of other extensions in the same namespaces, e.g., `gardener-extension-otelcol`.

If there is no matching `Extension` resource, the collector stays unchanged.

The webhook mounts the secrets listed in `collector.secrets` read-only at `/etc/observability/<name>/` (see [API](#api)). It also adds the network policy labels if `networking.allowPublicNetworks` or `networking.allowPrivateNetworks` is set (see [Proposal](#proposal)). It adds a hash of the added configuration and the secret data to the pod annotations of the collector, so that secret changes roll out new pods.

### Required Changes in gardener/gardener

- Document the name `opentelemetry-collector` of the `OpenTelemetryCollector` resource, its label `observability.gardener.cloud/app: opentelemetry-collector` and its receiver and exporter names as a stable interface for extensions.

## Drawbacks

- The extension mutates resources that Gardener owns. Renames of the collector resource, its labels, its receivers or its exporters in `gardener/gardener` break the injected pipelines, which is why this GEP asks to document them as a stable interface.
- A mutating webhook with `failurePolicy: Fail` is an additional dependency for `Garden`, `Seed` and `Shoot` reconciliations on every seed that lists the extension and on the runtime cluster if the `Garden` lists the extension.
- A misconfigured processor or exporter in the `seed` class affects all shoot control plane collectors on the seed.

## Alternatives

### Extend gardener-extension-otelcol

`gardener-extension-otelcol` already configures exporters via `CollectorConfig.spec.targets`, but it is shoot-spec-driven and deploys an additional collector. It does not allow modifying the collectors Gardener deploys. Letting operators reconfigure the Gardener collectors through it would change its purpose and the maintainers prefer a separate extension (see gardener/gardener#15700).

For shoot owners, `gardener-extension-otelcol` stays the way to export signals of their control plane. The additional collector isolates the owner configuration, so a misconfiguration or high signal volumes do not affect the collector Gardener deploys. This is why this GEP does not cover the `shoot` scope.

### Deploy a dedicated operator-owned collector

An additional collector per seed or per shoot control plane would need either an additional fluent-bit output or an additional exporter in the Gardener collectors pointing to it, which brings back mutating their configuration. An additional fluent-bit output would also not include the data plane logs from the shoot `kube-system` namespace, since they arrive at Gardener's collectors directly. It duplicates infrastructure that already receives the signals and adds another component to maintain.

### A controller that patches the collectors

Since the collectors are applied via `ManagedResource`s, `gardener-resource-manager` reverts the patch on the next apply and the two components fight over the resource. Making `gardener-resource-manager` ignore parts of the spec would require changes in Gardener core for this one use case.

### Model the extension point in Gardener core

A field in the `Seed` and `Garden` APIs carrying additional collector pipelines, rendered by `gardenlet` and `gardener-operator`. This puts landscape-specific configuration and the OpenTelemetry Collector configuration format into Gardener's core APIs and makes core responsible for validating it.
