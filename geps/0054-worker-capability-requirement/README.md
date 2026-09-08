# GEP-0054: Worker Capability Requirements for Machine Image Selection


## Summary

Some Gardener features, like [in-place node updates](../0031-inplace-node-updates/README.md), and (planned) secure boot, only work when the **machine image, the machine type, and Gardener** all support them. The image- and machine-type side fits [GEP-33](../0033-machine-image-capabilities/README.md)'s capability mechanism, but the *names* of these capabilities form a contract owned by Gardener.

This GEP reserves the prefix `gardener-` inside GEP-33 capabilities for this family of features and offers an option for existing or future typed worker pool fields (e.g. `updateStrategy`) into the GEP-33 selection algorithm. **Users keep configuring features via typed worker fields**; Gardener internally derives the matching capability requirements.

## Motivation

Today, in-place node update is the first example of a Gardener feature that depends on a three-way compatibility contract: the worker pool must request the feature, the selected machine image version (flavor) must support it, and the selected machine type must support it as well. More features with the same shape are expected, for example secure boot.

Today this contract is only partially expressed. In the case of in-place updates:

1. the [worker pool configuration](https://github.com/gardener/gardener/blob/159fbd0df8a3a272bfdf054638ce1ec1f45ccc9e/pkg/apis/core/types_shoot.go#L1372) is `updateStrategy`
1. the [machine image version](https://github.com/gardener/gardener/blob/159fbd0df8a3a272bfdf054638ce1ec1f45ccc9e/pkg/apis/core/types_cloudprofile.go#L375) metadata is `InPlaceUpdates.Supported`
1. **machine type** metadata is not defined
1. compatibility is enforced by several separate **filter implementations**: e.g. [maintenance operations](https://github.com/gardener/gardener/blob/b8a336572b2305befa05d358480ff22f3e11d0a9/pkg/controllermanager/controller/shoot/maintenance/helper/helper.go#L206), [shoot admission](https://github.com/gardener/gardener/blob/159fbd0df8a3a272bfdf054638ce1ec1f45ccc9e/plugin/pkg/shoot/validator/admission.go#L1594-L1610), [default image selection](https://github.com/gardener/gardener/blob/159fbd0df8a3a272bfdf054638ce1ec1f45ccc9e/plugin/pkg/shoot/mutator/admission.go#L632-L634), ...
1. **image version flavor selection** is missing; if more than one flavor is grouped under a machine image version, selecting a compatible flavor is not guaranteed

As a result, Gardener can accept or derive combinations whose full compatibility is not expressed in one place, and operators may need to split images into separate versions just to encode flavor-specific support and allow automatic image maintenance to work.

#### Concrete configuration example

Because feature support cannot be expressed per flavor today, the only way to model a feature like in-place updates is to split it into a dedicated machine image version entry:

```yaml
machineImages:
  - name: gardenlinux
    versions:
      - version: "2150.4.0"
        capabilityFlavors:
          - architecture: [amd64]

      - version: "2150.4.0-inplace"
        inPlaceUpdates:
          supported: true
        capabilityFlavors:
          - architecture: [amd64]
```

Already here the same semver appears twice and the feature leaks into the version string. As more orthogonal features may be added (e.g. FIPS, secure boot, GPU support), the cloud profile explodes into a cartesian product of versions, each carrying its own feature flags:

```yaml
machineImages:
  - name: gardenlinux
    versions:
      - version: 2150.4.0
        fipsSupported: false
        capabilityFlavors:
          - architecture: [amd64]
          - architecture: [arm64]
      - version: 2150.4.0-inplace
        inPlaceUpdates: { supported: true }
        fipsSupported: false
        capabilityFlavors:
          - architecture: [amd64]
          - architecture: [arm64]
      - version: 2150.4.0-fips
        fipsSupported: true
        capabilityFlavors:
          - architecture: [amd64]
          - architecture: [arm64]
      - version: 2150.4.0-fips-inplace
        inPlaceUpdates: { supported: true }
        fipsSupported: true
        capabilityFlavors:
          - architecture: [amd64]
          - architecture: [arm64]
      # ... plus secureboot variants
```

The consequences are:

- The same semver is duplicated up to `2^F` times for `F` orthogonal features.
- Every new feature requires a new top-level API field on the machine image version (`inPlaceUpdates`, `fipsSupported`, `secureBootSupported`, ...).
- Every such field requires its own filter implementation in shoot admission, the maintenance controller, and default image selection (see the locations listed above).
- Operators are forced to encode feature support in version naming conventions (`-inplace`, `-fips`, `-fips-inplace`) so that automatic image maintenance picks a compatible version.

The underlying problem: these features need a shared compatibility check across worker pool configuration, machine image flavors, and machine types. This GEP therefore builds on the existing GEP-33 capability mechanism by reserving a Gardener-owned capability namespace and by deriving capability requirements from typed worker pool fields, so that admission and image selection can consistently reject incompatible combinations. With it the matrix above collapses back to a single version carrying a set of `capabilityFlavors`, expressing feature support declaratively in one place and reusing the existing capability matching logic instead of adding new filters. Once implemented, only minimal changes are required to the worker pool API to support new features.

### Goals

- Reserve the `gardener-` prefix inside `spec.machineCapabilities` for capability names whose keys and values are owned by Gardener.
- Extend GEP-33's selection and validation algorithm to also satisfy capability requirements derived from typed worker pool fields.
- Reject incompatible combinations of machine image, machine type, and worker pool at admission time.

### Non-Goals

- Defining the full list of reserved capabilities up front — it grows with new features.
- Changing the GEP-33 mechanism. Reserved capabilities behave like any other — only their **names and values** are owned by Gardener.
- Implementing the features themselves (e.g. wiring secure boot end-to-end is out of scope).

## Proposal

GEP-33 is unchanged. This GEP adds two ingredients on top:

1. **A reserved namespace.** Capability names starting with `gardener-` are owned by Gardener. Operators must register them in `spec.machineCapabilities` using the keys and values Gardener defines; CloudProfile admission validates this. Anything *outside* `gardener-` is free for operators, exactly as today.
2. **A derivation step.** When a user configures a worker pool, Gardener internally translates relevant typed fields into capability requirements (e.g. `updateStrategy: AutoInPlaceUpdate` → `gardener-update-type: in-place`). These requirements feed the GEP-33 matching algorithm alongside machine type and machine image capabilities.

The mechanism is **additive**: CloudProfiles and worker pools that don't use any reserved capability behave exactly as today.

**Key terms:**

- **Reserved capability** — capability whose name starts with `gardener-`; key and values defined by Gardener.
- **Capability requirement** — `(name, value)` pair derived internally from a worker pool's typed fields. Never written by users.

### Risks and Mitigations

| Risk | Mitigation |
|---|---|
| An operator accidentally registers a `gardener-*` capability with values that don't match Gardener's authoritative definition | CloudProfile admission rejects the registration |
| Reserved-capability definitions drift between a Gardener release and a CloudProfile maintained by an operator | CloudProfile admission validates registered values against the Gardener version in use; mismatches surface immediately rather than at runtime |
| Existing CloudProfiles or worker pools break | The mechanism is additive — CloudProfiles and worker pools that don't use any reserved capability behave exactly as today |

## Design Details

### Reserved Capability Names

Reserved names and values are defined as constants in Gardener core. The two driving examples:

```go
const (
    // gardener-update-type — drives in-place node updates (GEP-0031).
    GardenerCapabilityUpdateType        = "gardener-update-type"
    GardenerCapabilityUpdateTypeRolling = "rolling"
    GardenerCapabilityUpdateTypeInPlace = "in-place"

    // gardener-boot-type — drives secure boot (planned).
    GardenerCapabilityBootType         = "gardener-boot-type"
    GardenerCapabilityBootTypeStandard = "standard"
    GardenerCapabilityBootTypeSecure   = "secure"
)
```

CloudProfile admission validates that any `gardener-*` capability registered in `spec.machineCapabilities` matches Gardener's authoritative definition.

Reserved capabilities are part of Gardener's public API surface. Adding, renaming, changing, deprecating, or removing a reserved capability name or value follows the same API conventions as a dedicated field on the CloudProfile API (deprecation periods, compatibility guarantees, release-note requirements). This is what makes the prefix approach a viable substitute for first-class typed fields.

### Deriving Capability Requirements

The mapping is part of the image selection process in shoot admission, the maintenance controller and provider extension:

```go
func deriveWorkerCapabilityRequirements(worker core.Worker) map[string][]string {
    requirements := map[string][]string{}

    // updateStrategy is defaulted earlier (default: rolling).
    switch ptr.Deref(worker.UpdateStrategy, core.UpdateStrategyRollingUpdate) {
    case core.UpdateStrategyAutoInPlace, core.UpdateStrategyManualInPlace:
        requirements[GardenerCapabilityUpdateType] = []string{GardenerCapabilityUpdateTypeInPlace}
    default:
        requirements[GardenerCapabilityUpdateType] = []string{GardenerCapabilityUpdateTypeRolling}
    }

    // Planned: worker.Machine.SecureBoot.
    if worker.Machine.SecureBoot != nil && *worker.Machine.SecureBoot {
        requirements[GardenerCapabilityBootType] = []string{GardenerCapabilityBootTypeSecure}
    } else {
        requirements[GardenerCapabilityBootType] = []string{GardenerCapabilityBootTypeStandard}
    }

    return requirements
}
```

Only typed worker pool fields drive requirements — users of the shoot resource never write capability names.

### Image Selection Algorithm

GEP-33 introduced a selection algorithm based on capability compatibility.

The parameters are the capabilities of one machine image flavor and a machine type, plus the capability definitions (`spec.machineCapabilities`) from the CloudProfile:

```go
AreCapabilitiesCompatible(imageFlavor, machineType, capabilityDefinitions){
  defaultedCapabilities1 := GetCapabilitiesWithAppliedDefaults(imageFlavor, capabilityDefinitions)
  defaultedCapabilities2 := GetCapabilitiesWithAppliedDefaults(machineType, capabilityDefinitions)

  commonCapabilities := GetCapabilitiesIntersection(defaultedCapabilities1, defaultedCapabilities2)
  // If the intersection has at least one value for each capability, the capabilities are compatible.
  for _, values := range commonCapabilities {
    if len(values) == 0 {
      return false
    }
  }
  return true
}
```

This GEP adds a third capability input for the worker pool's derived requirements:

```go
AreCapabilitiesCompatible(imageFlavor, machineType, workerRequirements, capabilityDefinitions){
  defaultedCapabilities1 := GetCapabilitiesWithAppliedDefaults(imageFlavor, capabilityDefinitions)
  defaultedCapabilities2 := GetCapabilitiesWithAppliedDefaults(machineType, capabilityDefinitions)
  defaultedCapabilities3 := GetCapabilitiesWithAppliedDefaults(workerRequirements, capabilityDefinitions)

  commonCapabilities := GetCapabilitiesIntersection(defaultedCapabilities1, defaultedCapabilities2, defaultedCapabilities3)
  for _, values := range commonCapabilities {
    if len(values) == 0 {
      return false
    }
  }
  return true
}
```

The algorithm succeeds if, for every capability defined in `spec.machineCapabilities`, the value sets from image flavor, machine type, and worker requirements have a non-empty intersection (with image- and machine-type-side defaulting per GEP-33).

The same algorithm is invoked from four existing call sites:

1. [Shoot validator admission](https://github.com/gardener/gardener/blob/e6263d6a575e4181f0289345803ccb59117605f6/plugin/pkg/shoot/validator/admission.go#L1000) — rejects user-selected machine image / machine type pairs that are incompatible with the worker pool's capability requirements.
2. [Shoot mutator admission](https://github.com/gardener/gardener/blob/d9897865ab9181c307efdfa93f14268fcd09fe88/plugin/pkg/shoot/mutator/admission.go#L559) — picks a default machine image compatible with the chosen machine type and worker pool when the user does not specify one.
3. [Maintenance controller](https://github.com/gardener/gardener/blob/d9897865ab9181c307efdfa93f14268fcd09fe88/pkg/controllermanager/controller/shoot/maintenance/helper/helper.go#L27) — chooses a machine image compatible with the worker pool's machine type and capability requirements during automatic updates.
4. Worker controller in provider extensions.

### Validation

| Concern | Validated by |
|---|---|
| `gardener-*` capability registrations match Gardener's authoritative list | gardener-apiserver CloudProfile admission |
| Worker pool's capability requirements satisfied by the selected machine image and machine type | gardener-apiserver shoot admission |
| Maintenance image selection respects all requirements | maintenance controller |

## Drawbacks

- **Internal complexity grows.** Selection gains a third input and a derivation step. The user-side validation together with a framework for future feature development is the explicit trade.
- **Operator setup per CloudProfile.** Operators who want a reserved feature must register the capability in `spec.machineCapabilities` and declare per-image-flavor support. CloudProfiles that expose no reserved features are unaffected.
- **Coupling between Gardener releases and CloudProfile content.** Reserved definitions are owned by Gardener; operators must keep CloudProfiles in sync with the version they run to use new features.

## Alternatives

- **Generic `capabilities` map on the worker pool API.** Most consistent with GEP-33, but rejected because: 
  1. it forces users to learn the CloudProfile's capability vocabulary to configure worker features.
  2. most capabilities are infrastructure-level concerns that are irrelevant to shoot users (e.g. hypervisor type, or which bare-metal machine type works with which image). Exposing them on the worker pool API would surface implementation detail with no user benefit
  3. it leaks a CloudProfile internal contract into user-facing API. 
  4. changing/removing capabilities today can be done isolated within a CloudProfile. This would add a dependency to the workers using that CloudProfile.

- **Implicit reserved capabilities (no `spec.machineCapabilities` entry).** Inconsistent with GEP-33, which declares every capability there. Rejected — as this would contradict the idea that ´spec.machineCapabilities´ is the authoritative source for capability definitions.

- **Provider-extension-owned capabilities in this GEP.** Earlier drafts let provider extensions own a `gardener-<provider>-` sub-namespace and derive capability requirements from typed `WorkerConfig` fields (e.g. OpenStack trusted launch). This was deferred because `WorkerConfig` is unknown to gardener-apiserver, the maintenance controller, and the Dashboard (only the extension can decode it) which makes the mapping mechanism a substantial design problem on its own. Solving it is independent of the core mechanism this GEP introduces.

- **Dedicated first-class API fields for Gardener-owned capabilities.** Instead of reserving a `gardener-` prefix inside the generic `spec.machineCapabilities` list, the CloudProfile API could expose Gardener-relevant capabilities as typed fields, e.g.:

  ```yaml
  spec:
    machineCapabilities:
      gardenerOwned:
        nodeUpdateTypes: ["rolling", "in-place"]
        bootTypes: ["secure", "legacy"]
      custom:
        - name: architecture
          values: ["amd64", "arm64"]
  ```

  Internally these would still be encoded as capabilities so the matching algorithm is unaffected. Rejected because it is a breaking change to `spec.machineCapabilities`, and the prefix approach gives the same orchestration guarantees once the same API conventions are applied to reserved `gardener-*` names. A future API version can still promote individual reserved capabilities to first-class fields.
