# GEP-0054: Worker Capabilities for Machine Image Selection

## Table of Contents

- [GEP-0054: Worker Capabilities for Machine Image Selection](#gep-0054-worker-capabilities-for-machine-image-selection)
  - [Table of Contents](#table-of-contents)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
  - [Proposal](#proposal)
    - [Notes/Constraints/Caveats](#notesconstraintscaveats)
    - [Risks and Mitigations](#risks-and-mitigations)
  - [Design Details](#design-details)
    - [Reserved Capability Names](#reserved-capability-names)
    - [Worker Properties to Capabilities Mapping](#worker-properties-to-capabilities-mapping)
    - [Updated Image Selection Algorithm](#updated-image-selection-algorithm)
    - [Provider-Specific Worker Properties](#provider-specific-worker-properties)
    - [Component Changes](#component-changes)
  - [Drawbacks](#drawbacks)


## Summary

This GEP extends [GEP-0033 (Machine Image Capabilities)](../0033-machine-image-capabilities/README.md) to support selecting machine images based on worker-defined properties in addition to machine type characteristics. While GEP-0033 enables image selection based on hardware capabilities of machine types (CPU architecture, hypervisor type, etc.), this proposal introduces worker capabilities that describe software/feature requirements requested by the worker configuration. Both machine type capabilities and worker capabilities must be satisfied for image selection to succeed.

## Motivation

Currently, machine image capabilities as defined in GEP-0033 only support selecting images based on hardware properties of the machine type (CPU architecture, hypervisor type, bare-metal vs virtualized). However, users increasingly need the ability to select images based on software or feature requirements that are specified in the worker configuration, such as:

- **Secure Boot**: Select images that support UEFI secure boot
- **In-Place Updates**: Select images compatible with in-place update mechanisms (as defined in [GEP-0031](../0031-inplace-node-updates/README.md))
- **GPU Support**: Select images with specific GPU driver support (nvidia, amd, intel)
- ...

Without this feature, users must manually ensure that their worker configuration is compatible with the selected machine image, which is error-prone and limits the automation benefits provided by Gardener's maintenance operations.

### Goals

- Extend the capabilities matching algorithm from GEP-0033 to include worker-defined capabilities alongside machine type capabilities.
- Define a set of reserved capability names prefixed with `gardener-` to avoid conflicts with user-defined or provider-specific capabilities.
- Provide a mechanism for mapping worker properties to capabilities that can be used in image selection.
- Enable Gardener admission to validate that selected images support the requested worker features.
- Allow provider extensions to define additional provider-specific worker properties that map to capabilities.

### Non-Goals

- Defining the complete list of all possible worker capabilities (this will evolve over time).
- Implementing the actual features controlled by these capabilities (e.g., secure boot enablement is separate from capability-based image selection).
- Changing the Shoot API to add new worker fields (this GEP focuses on the capability system; API changes are separate).
- Replacing or deprecating machine type capabilities from GEP-0033.

## Proposal

Extend the capability matching system from GEP-0033 to include a third dimension: worker capabilities. The image selection algorithm currently performs a capability intersection between machine image capabilities and machine type capabilities. This proposal adds worker capabilities to the intersection, ensuring selected images satisfy all three capability sets.

The key distinction between capability types:
- **Machine type capabilities**: Describe what the hardware supports (e.g., ARM64 architecture, virtualized hypervisor)
- **Worker capabilities**: Describe what features the worker configuration requests (e.g., secure boot enabled, in-place updates required)

Worker capabilities are derived from worker specification properties. To prevent naming conflicts between Gardener-reserved capabilities and custom or provider-defined capabilities, reserved capability names use a `gardener-` prefix.

### Notes/Constraints/Caveats

- Worker capabilities are **additive** to the existing capability system. Existing CloudProfiles and image selection continue to work without modification.
- The `gardener-` prefix is reserved for Gardener core capabilities. Provider extensions should use their own prefixes (e.g., `azure-`, `aws-`) for provider-specific capabilities.
- Images must explicitly declare support for worker capabilities in their `capabilityFlavors`. If an image does not declare a capability, it is assumed to not support that feature.
- Most worker capabilities will have default values (e.g., `gardener-bootType: standard`) to maintain backwards compatibility.

### Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Capability name conflicts with existing custom capabilities | Use `gardener-` prefix for all Gardener-reserved capability names |
| Breaking existing CloudProfiles | Worker capabilities are opt-in; existing profiles continue to work |
| Increased CloudProfile complexity | Provide sensible defaults; only images requiring special features need explicit capability declarations |
| Provider extensions not updated | Validation of the shoot by the extension is only required if the provider supports custom worker capabilities |

## Design Details

### Reserved Capability Names

The following example capability names are reserved for Gardener core use. All reserved capabilities use the `gardener-` prefix:

```go
const (
    // Boot type capability
    CapabilityNameBootType           = "gardener-bootType"
    CapabilityValueBootTypeSecure    = "secure"
    CapabilityValueBootTypeStandard  = "standard"

    // OS update type capability
    CapabilityNameUpdateType           = "gardener-updateType"
    CapabilityValueUpdateTypeInPlace   = "inPlace"
    CapabilityValueUpdateTypeRolling   = "rolling"

    // GPU support capability
    CapabilityNameGpuSupport         = "gardener-gpuSupport"
    CapabilityValueGpuSupportNvidia  = "nvidia"
    CapabilityValueGpuSupportAmd     = "amd"
    CapabilityValueGpuSupportIntel   = "intel"
    CapabilityValueGpuSupportNone    = "none"
)
```

These reserved names must be validated during CloudProfile admission to prevent users from defining custom capabilities with these names.

### Worker Properties to Capabilities Mapping

Worker properties from the Shoot specification are translated to capabilities internally. This mapping is performed during admission and image selection:

```go
func mapWorkerPropertiesToCapabilities(worker core.Worker) map[string][]string {
    capabilities := make(map[string][]string)

    // Boot type mapping
    if worker.Machine.SecureBoot != nil && *worker.Machine.SecureBoot {
        capabilities[CapabilityNameBootType] = []string{CapabilityValueBootTypeSecure}
    } else {
        capabilities[CapabilityNameBootType] = []string{CapabilityValueBootTypeStandard}
    }

    // Update type mapping
    if worker.UpdateStrategy != nil && *worker.UpdateStrategy == core.UpdateStrategyInPlace {
        capabilities[CapabilityNameUpdateType] = []string{CapabilityValueUpdateTypeInPlace}
    } else {
        capabilities[CapabilityNameUpdateType] = []string{CapabilityValueUpdateTypeRolling}
    }

    // GPU support mapping (if applicable)
    if worker.Machine.GPU != nil {
        switch worker.Machine.GPU.Type {
        case "nvidia":
            capabilities[CapabilityNameGpuSupport] = []string{CapabilityValueGpuSupportNvidia}
        case "amd":
            capabilities[CapabilityNameGpuSupport] = []string{CapabilityValueGpuSupportAmd}
        case "intel":
            capabilities[CapabilityNameGpuSupport] = []string{CapabilityValueGpuSupportIntel}
        default:
            capabilities[CapabilityNameGpuSupport] = []string{CapabilityValueGpuSupportNone}
        }
    }

    return capabilities
}
```

### Updated Image Selection Algorithm

The current image selection algorithm from GEP-0033 checks compatibility between image capabilities and machine type capabilities:

```go
// Current (GEP-0033)
if AreCapabilitiesCompatible(imageFlavorCapabilities, machineTypeCapabilities, capabilityDefinitions) {
    // allowed - overlap between image and machine type
}
```

This is extended to include worker capabilities:

```go
// Proposed (GEP-0040)
if AreCapabilitiesCompatible(imageFlavorCapabilities, machineTypeCapabilities, workerCapabilities, capabilityDefinitions) {
    // allowed - overlap between image, machine type, AND worker capabilities
}
```

The matching algorithm performs a three-way intersection. An image is compatible if the intersection is non-empty across all three sets of capabilities.

**Example Locations:**
1. **Shoot Validator Admission** ([worker admission](https://github.com/gardener/gardener/blob/e6263d6a575e4181f0289345803ccb59117605f6/plugin/pkg/shoot/validator/admission.go#L1000))

2. **Shoot Mutator Admission** ([default image selection](https://github.com/gardener/gardener/blob/d9897865ab9181c307efdfa93f14268fcd09fe88/plugin/pkg/shoot/mutator/admission.go#L559))

3. **Maintenance Controller** ([image selection helper](https://github.com/gardener/gardener/blob/d9897865ab9181c307efdfa93f14268fcd09fe88/pkg/controllermanager/controller/shoot/maintenance/helper/helper.go#L27C1-L27C149))

### Provider-Specific Worker Properties

Provider extensions can define additional worker properties that map to capabilities. This is achieved through the existing `WorkerConfig` extension mechanism:

```yaml
# Example: openstack-specific worker config that requires trusted launch
apiVersion: openstack.provider.extensions.gardener.cloud/v1alpha1
kind: WorkerConfig
trustedLaunch:
  enabled: true
```

The provider extension translates this to capabilities:

```go
// In provider-openstack extension
func mapProviderWorkerConfigToCapabilities(config *openstack.WorkerConfig, workerCapabilities Capabilities) map[string][]string {
    if config.TrustedLaunch != nil && config.TrustedLaunch.Enabled {
        workerCapabilities["openstack-trustedLaunch"] = []string{"enabled"}
    }

    return workerCapabilities
}
```

Provider extensions must implement a validating admission webhook to ensure worker configurations are compatible with selected images.

### Component Changes

The following components require updates:

**Gardener Core:**
- `pkg/api/core/validation/cloudprofile.go`: Add validation for reserved capability name prefixes
- `plugin/pkg/shoot/validator/admission.go`: Include worker capabilities in validation
- `plugin/pkg/shoot/mutator/admission.go`: Include worker capabilities in defaulting
- `pkg/controllermanager/controller/shoot/maintenance/`: Include worker capabilities in maintenance image selection
- `pkg/utils/gardener/`: Add utility functions for capability mapping

**Provider Extensions:**
- Each provider extension may implement additional provider-specific worker-to-capability mappings
- Provider shoot admission webhooks validate compatibility between worker config and image capabilities

**Gardener Dashboard:**
- Filter available images based on worker configuration settings

## Drawbacks

- **Increased Complexity**: Adds a third dimension to capability matching, which increases the complexity of the image selection algorithm.
- **CloudProfile Size**: Adding worker capabilities to image flavors will increase CloudProfile size, though the impact should be minimal if defaults are used effectively as planned already in GEP-0033.
- **Migration Effort**: Existing CloudProfiles will need updates to declare worker capability support on images.
