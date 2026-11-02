# GEP-0080: Introduce `GardenState` resource for `Garden` restoration

## Summary

Disaster recovery for Gardener landscapes currently involves many manual steps, distributed separate backup contents, and little automation assistance from the platform itself.
This proposal introduces a change to the Gardener codebase by adding a new `GardenState` resource that bundles all secrets, extension state, and the `Garden` resource required for a from-scratch `Garden` restoration.
This change reduces a landscape operator's manual burden, makes the separate backup contents explicit and machine-readable, and embeds the recovery logic in tested, version-controlled code rather than prose documentation.

## Motivation

Recovering a Gardener landscape from a `Garden` runtime cluster loss currently requires an operator to follow a guide while manually identifying, and restoring `Secret`s and resources under time pressure.
The Gardener components only provide little automation support for those tasks.
As Gardener evolves rapidly, written guides are at risk of becoming outdated.
Embedding the recovery logic in code in the `gardener-operator` that can be continuously tested mitigates this risk for all landscape operators.

### Goals

- Reduce the number of manual steps required for a `Garden` runtime cluster restoration to applying the `GardenState` resource and starting the `gardener-operator`.
- Make all data required for a from-scratch `Garden` restoration available in a single, well-known resource (`GardenState`) that can be backed up in one step.
- Provide an automated end-to-end test that validates the `GardenState` restoration path on every relevant change, creating ongoing confidence in the procedure.

### Non-Goals

- Automating the separate backup itself (scheduling, off-cluster storage of the `GardenState` `Secret`). This remains the responsibility of the landscape operator.
- Covering extension state for non-trivial infrastructure providers beyond what is already captured via the `persist=true` label mechanism.

## Proposal

A `GardenState` `Secret` resource is added to the Gardener codebase.
It is written and maintained by the `gardener-operator` after each successful `Garden` reconciliation.
It contains the `Garden` resource including its `status` subresource, all `Secret` labeled with `persist=true` (encryption keys, CA, configuration), and extension state.
On `gardener-operator` startup, a bootstrapper checks for the presence of a `GardenState` resource without a corresponding `Garden`.
If found, it triggers a restoration procedure that applies the contained data and hands off to the regular reconciler with a `Restore` operation type.

### Notes/Constraints/Caveats

- The `GardenState` resource must be applied to the new runtime cluster before `gardener-operator` is deployed. The bootstrap check only runs on `gardener-operator` startup.
- The `GardenState` is implemented as a `Secret` rather than a `CRD` to allow application to an empty cluster before any `CRD` or `gardener-operator` are installed.
This has the advantage that it can be applied to an empty cluster without installing `CRD`s or `gardener-operator` first.
- A manual separate backup step for copying the `GardenState` `Secret` out of the runtime cluster to external storage is still required to be handled by a landscape operator and is not automated by this proposal.
- The `GardenState` is updated after each successful reconciliation, without atomicity guarantees.
Changes applied between the last successful `Garden` reconciliation and the disaster are not reflected and constitute an implicit Recovery Point Objective (RPO) risk for the `GardenState` contents.

### Risks and Mitigations

- If the separate backup was not taken after the most recent successful reconciliation, the restored `GardenState` may be slightly behind the last-known landscape state.
To minimize this window, the `gardener-operator` writes a new `GardenState` on every successful `Garden` reconciliation.
The documentation should warn operators to take a fresh backup before any planned maintenance.
- If a secret required for restoration is not labeled `persist=true`, it will be silently absent from the `GardenState`.
The label coverage needs to be verified and corrected if necessary.
Any unlabeled required `Secret`s are a known implementation risk to be confirmed in code review or an automated validation, e.g., by an automated end-to-end test.
- The `GardenState` captures extension state via the `persist=true` mechanism, but non-trivial providers may require additional state not yet labeled.
This is documented as a known limitation and provider-specific coverage is to be tracked as follow-up work per provider extension.
- Restoring a highly available etcd from a single-replica bootstrap avoids data inconsistency between replicas but introduces a window where only one replica holds the restored data.
To address this, etcd is scaled to one for restore, then scaled back up. Additional replicas synchronize before being admitted to the cluster.

## Design Details

The `GardenState` resource (named `garden-state` and annotated with `operator.gardener.cloud/purpose=garden-state`) is stored as a Kubernetes `Secret` in the runtime cluster under a well-known name and namespace, readable by the `gardener-operator`.
Its data fields mirror the set of `Secret`s labeled `persist=true` in the virtual Garden cluster, plus the serialized `Garden` resource with its `status` subresource.
The `gardener-operator` bootstrapper is a new startup hook that runs before the main reconciler loop.
It reads the `GardenState` and `Garden` resources and, depending on their presence, proceeds either with the normal reconciliation or the restoration path.

Suggested `GardenState` struct:
```golang
// GardenState describes a persisted state of the garden.
type GardenState struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object metadata.
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// Gardener holds the data required to restore resources deployed by the garden operator.
	// +optional
	Gardener []gardencorev1beta1.GardenerResourceData `json:"gardener,omitempty"`
	// Garden holds the garden resource state needed for restore.
	Garden GardenStateGarden `json:"garden,omitempty"`
	// Extensions holds the state of custom resources reconciled by extension controllers in the runtime cluster.
	// +optional
	Extensions []gardencorev1beta1.ExtensionResourceState `json:"extensions,omitempty"`
	// Resources holds the data of resources referred to by extension controller states.
	// +optional
	Resources []gardencorev1beta1.ResourceData `json:"resources,omitempty"`
}

// GardenStateGarden contains the Garden resource data persisted in a GardenState.
// It uses explicit metadata fields instead of embedding ObjectMeta, because nested ObjectMeta
// gets pruned by CRD structural schema validation.
type GardenStateGarden struct {
    // Name is the name of the Garden resource.
    Name string `json:"name"`
    // Labels are the labels of the Garden resource.
    // +optional
    Labels map[string]string `json:"labels,omitempty"`
    // Annotations are the annotations of the Garden resource.
    // +optional
    Annotations map[string]string `json:"annotations,omitempty"`
    // UID is the UID of the Garden resource. It is needed during restore to preserve the etcd
    // backup bucket name which defaults to "garden-<UID>".
    // +optional
    UID string `json:"uid,omitempty"`
    // Spec is the spec of the Garden resource.
    Spec GardenSpec `json:"spec"`
    // Status is the status of the Garden resource.
    // +optional
    Status GardenStatus `json:"status,omitempty"`
}
```

## Drawbacks

- Using a `Secret` as a state store is an unusual choice, as `Secret`s are typically used for credential data rather than operational state.
For large landscapes with many extensions, the size of the `GardenState` `Secret` may become a concern, though this was not observed in the tested landscape.
- The bootstrapper adds complexity to the `gardener-operator` startup path.
A regression in the bootstrap logic could prevent normal operator startup even in non-restore scenarios.
- The `GardenState` write-on-reconcile approach couples the freshness of the disaster recovery resource to the reconciliation frequency, but could create an increased RPO if changes are not synchronized into the `GardenState` resource in a timely manner.

## Alternatives

- Using a dedicated `CRD` for `GardenState` would be more conventional for Kubernetes.
However, the `CRD` must be installed in the cluster first.
Since `gardener-operator` normally installs the `CRD`, depending on it would create a circular dependency.
The `gardener-operator` must be running to install the `CRD`, but the `GardenState` `Secret` must be present before the operator starts.
Another option would be to install the `CRD` manually, which would create an additional manual step.
A `Secret` can be applied to any empty cluster without prerequisites.

- Improving the disaster recovery guide without code changes would reduce implementation effort but does not address the root cause of manual steps that are error-prone under disaster pressure, and prose documentation drifts as the codebase evolves.
Code-based automation can be continuously validated by tests while the prose cannot be as easily validated.

- The `gardener-operator` could place its separate backup for the runtime cluster besides the virtual Garden etcd backup on an object store.
This was ruled out because it requires provider-specific object store client tooling, which would either mean deploying Gardener provider extensions first or including provider-specific client code in the `gardener-operator`.
Resources would still need to be encrypted and the decryption key provided prior to restore, making this approach more complex without a clear benefit over the `Secret`-based solution.
