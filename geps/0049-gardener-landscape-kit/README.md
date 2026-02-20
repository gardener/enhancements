# GEP-0049: Gardener Landscape Kit

## Table of Contents

- [GEP-0049: Gardener Landscape Kit](#gep-0049-gardener-landscape-kit)
  - [Table of Contents](#table-of-contents)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
  - [Proposal](#proposal)
    - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
    - [Risks and Mitigations](#risks-and-mitigations)
  - [Design Details](#design-details)
    - [Repositories](#repositories)
    - [Components](#components)
    - [Versions and OCI References](#versions-and-oci-references)
    - [Configuration API](#configuration-api)
  - [Drawbacks](#drawbacks)
  - [Alternatives](#alternatives)
  - [Appendix](#appendix)

## Summary

The Gardener Landscape Kit (GLK, or `gardener-landscape-kit`) aims to deliver tools and best practices for managing Gardener, from small-size to large-scale landscapes.
It introduces a modular framework that **generates** resources for Gardener and well known extensions, building on Kubernetes as the underlying deployment system.
Leveraging GitOps principles, GLK simplifies operations, reduces integration overhead, and enhances developer productivity.
With [Kubernetes](https://kubernetes.io/), [Flux](https://fluxcd.io/), [Kustozmiation](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) and co., a wide range of widely accepted tools are utilized, enabling an efficient and extensible configuration management for operators.

## Motivation

While Gardener brings great abstraction and extensibility for managed Kubernetes clusters, it currently lacks a project addressing the setup routines and maintenance of such landscapes.
Community members are required to implement configuration and landscape management themselves, leading not only to additional complexity for new-starters, but also to significant investments in enterprises.
It is especially challenging for the community to maintain the version vector with Gardener and extensions.

### Goals

- Generate Kubernetes manifests for Gardener, well known extensions and utility controllers.
- Apply GitOps principles for configuration management.
- Allow generic modifications to the deployment process.
- Support update and migration scenarios for included components.
- Simplify component implementations and increase developer productivity.
- Support manifest template processing through [OCM](https://ocm.software/), as the standard transport tool in NeoNephos.
- Maintaining a default, static vector of compatible component versions.

### Non-Goals

- Serving a unified API (abstraction) for landscape management.
- Definition of a compatibility matrix for Gardener and extensions.
- Provide holistic default configuration.
- Building OCM component descriptors.

## Proposal

The Gardener Landscape Kit will be an executable toolkit that bootstraps landscapes and updates Gardener and dependent components. It exists of the following building blocks:

**Manifest Generation**

GLK generates Kubernetes manifests for Gardener and its extensions using a modular, component-based approach. Each component (e.g., `gardener-operator`, `garden`, `provider-xyz`) is responsible for producing one or more manifests and may define dependencies on other components. Operators can review and modify generated manifests, with manual changes preserved across subsequent generations. These manifests are intended to be managed in a Git repository, enabling GitOps workflows.
Running against such repositories allows GLK update versions to migrate existing resources. If migration is required during runtime, a component would typically generate a special `Job` manifest accomplishing this job in the cluster.

**OCI Image Resolution**

GLK automates the resolution and substitution of OCI image references (e.g., in `Helm` charts or `Extension` resources). It consults either an [OCM](https://ocm.software/) component descriptor or a default image vector maintained by GLK.
This ensures:
- **Consistent Deployments:** Operators can deploy Gardener and extensions with validated versions out of the box, without manual release management.
- **Image Transportation:** Validated deployments and images can be promoted across environments (e.g., `Dev` → `Canary` → `Live`).
- **Sovereign Cloud Support:** Required images can be mirrored to private registries for isolated or air-gapped environments.

**Deployment System**

GLK standardizes on [Flux](https://fluxcd.io/) as the primary deployment system, leveraging its strong GitOps support and Kubernetes-native design. GLK generates Flux based configuration to establish a seamless and automated deployment flow.
Further key benefits include:
- Native Kubernetes integration
- First-class GitOps workflows
- Built-in encryption support via [SOPS](https://fluxcd.io/flux/guides/mozilla-sops/) or [Sealed Secrets](https://fluxcd.io/flux/guides/sealed-secrets/)
- CNCF graduation and broad community adoption ([adopters](https://fluxcd.io/adopters/))
- Extensive documentation and support

![Gardener Landscape Kit diagram](glk-basics.svg)

### Notes/Constraints/Caveats (Optional)

The Gardener Landscape Kit (GLK) is a standalone toolkit, not a Kubernetes controller, meaning it does not directly watch or reconcile cluster resources.
Its primary use cases are:

- **Local Execution**: For initial setup, such as onboarding a new landscape by generating manifests and initializing repositories.
- **Automated Execution**: For ongoing operations, GLK is designed to run in automated environments like CI/CD pipelines or Kubernetes `Job`s. This is the intended way to apply updates, adopt new GLK releases, and integrate new component versions.

To support these scenarios, GLK will provide reference workflows and best-practice configurations.

### Risks and Mitigations

GLK is planned as a new product, on top of existing Gardener projects and components.
At this point, there hasn't been identified any major risks.

## Design Details

The Gardener Landscape Kit is a standalone executable. Its primary subcommand, `generate`, creates the Kubernetes manifest files that operators commit to Git. These manifests are then automatically deployed to the target cluster(s) by Flux as part of a GitOps workflow.

### Repositories

Git repositories are an essential part for landscape and configuration management. In essence, the landscape kit differentiates between two types of repositories, to foster reusability across landscapes:

- **Base Repository**: This repository contains the core landscape configurations, modules, and shared resources that are common across multiple landscapes.
- **Landscape Repository**: These repositories are specific to individual landscapes and typically contain overlay configurations that are merged with the ones found in the base repository.

Also see the Kustomize [base and overlay documentation](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#bases-and-overlays) for more information.

Technically, the Gardener Landscape Kit doesn't use Git directly, but prepares and operates on the directory structure of the filesystem.

### Components

**Definition**

In GLK, a component is responsible for generating a set of manifests for the `base` and/or `landscape` target repositories. These components often correspond to a Gardener sub-project, such as [`gardener/gardener`](https://github.com/gardener/gardener) or various extensions like [`gardener/gardener-extension-provider-openstack`](https://github.com/gardener/gardener-extension-provider-openstack).

To ensure a smooth deployment flow, a single conceptual component may be split into smaller ones to properly manage dependencies. For example, deploying the Gardener core involves two components:

- A `gardener-operator` component, which installs the Operator Helm chart.
- A `garden` component, which creates the Garden resource in the runtime cluster.

From a technical standpoint, each component implements the `component` interface shown below. GLK invokes this implementation when running the `generate base` or `generate landscape` commands.

```go
// Interface is the components interface that each component must implement.
type Interface interface {
	// Name returns the component name.
	Name() string
	// GenerateBase generates the component base dir.
	GenerateBase(Options) error
	// GenerateLandscape generates the component landscape dir.
	GenerateLandscape(LandscapeOptions) error
}
```

In addition to generating manifests, components can also handle migrations in two ways:
- **Manifest Modification**: By modifying existing manifests in the target repositories (e.g., moving a value from a deprecated field to a new one).
- **In-Cluster Migration**: By creating Kubernetes `Job`s that execute migration tasks within the cluster.

All generated manifests include basic configuration, reasonable defaults, and inline comments to help operators discover and understand available configuration options. GLK preserves any modifications made by operators through a three-way merge strategy. It maintains a copy of each originally generated manifest in a `.glk` system directory, enabling GLK to detect and merge operator changes with newly generated content.

**Initial Scope**

The following components are part of the initial delivery scope:

| Component Name                                          | Description                                                                                                                                |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `flux`                                                  | Flux deployment and controller configuration (mainly [Source controller](https://fluxcd.io/flux/components/source/))                       |
| `gardener-operator`                                     | [`HelmRelease`](https://fluxcd.io/flux/components/source/) for `gardener-operator`                                                         |
| `gardener-garden`                                       | [`Kustomization`](https://fluxcd.io/flux/components/kustomize/kustomizations/) for `garden` resource                                       |
| `virtual-garden-access `                                | [`Kustomization`](https://fluxcd.io/flux/components/kustomize/kustomizations/) for generating a Kubeconfig for Flux for the virtual garden |
| `garden-config`                                         | [`Kustomization`](https://fluxcd.io/flux/components/kustomize/kustomizations/) for configuration in the virtual garden                     |
| `extension-networking-{calico,cilium}`                  | [`Kustomization`](https://fluxcd.io/flux/components/kustomize/kustomizations/) for networking `Extension` resources                        |
| `extension-os-{gardenlinux,suse-chost}`                 | [`Kustomization`](https://fluxcd.io/flux/components/kustomize/kustomizations/) for OS `Extension` resources                                |
| `extension-provider-{alicloud,azure,aws,gcp,openstack}` | [`Kustomization`](https://fluxcd.io/flux/components/kustomize/kustomizations/) for provider `Extension` resources                          |
| `extension-runtime-gvisor`                              | [`Kustomization`](https://fluxcd.io/flux/components/kustomize/kustomizations/) for runtime `Extension` resource                            |
| `extension-shoot-cert-service`                          | [`Kustomization`](https://fluxcd.io/flux/components/kustomize/kustomizations/) for certificate service `Extension` resource                |
| `extension-shoot-dns-service`                           | [`Kustomization`](https://fluxcd.io/flux/components/kustomize/kustomizations/) for DNS service `Extension` resource                        |
| `extension-shoot-networking-problemdetector`            | [`Kustomization`](https://fluxcd.io/flux/components/kustomize/kustomizations/) for network problem detector `Extension` resource           |
| `extension-shoot-oicd-service`                          | [`Kustomization`](https://fluxcd.io/flux/components/kustomize/kustomizations/) for OIDC service `Extension` resource                       |

**Acceptance Criteria**

GLK should contain components of common interest for the community. Throughout the development of the project, this soft requirement must be redefined, such as:
- Component owners are obliged to implement component updates and migrations.
- A test environment continuously verifies new component versions.
- Adding a new component must follow to be defined requirements of GLK; needs approval of GLK maintainers.

### Versions and OCI References

A key feature of GLK is its ability to resolve version and OCI information for the manifests it generates. Landscape operators can choose from three resolution strategies:

- **Default Versions**: GLK can use the default OCI image references and component versions embedded in its release (see this [example](https://github.com/gardener/gardener-landscape-kit/blob/main/componentvector/components.yaml)). An optional `ReleaseBranch` update strategy instructs GLK to download the latest version information from its own release branch. This is useful for applying patch or minor component updates without needing a new GLK release. This strategy is intended to help the community get started with Gardener and operate non-productive landscapes.

- **OCM Component Descriptors**: [OCM](https://ocm.software/) component descriptors can be used to let GLK resolve OCI references in generated manifests. In addition, this strategy may produce [image vector overwrites](https://github.com/gardener/gardener/blob/master/docs/deployment/image_vector.md) and [image vector component overwrites](https://github.com/gardener/gardener/blob/master/docs/deployment/image_vector.md#image-vectors-for-dependent-components) for Gardener and extensions. This is the recommended approach for productive landscapes that use OCM for releases and component transport.

- **Manual**: This strategy gives operators full control over GLK's version vector ((see this [example](https://github.com/gardener/gardener-landscape-kit/blob/main/componentvector/components.yaml)).
This allows using tools like [Renovate](https://github.com/renovatebot/renovate) to manage version updates according to a custom policy.

### Configuration API

A configuration file is required for GLK. Below is an excerpt of the planned API.

```yaml
apiVersion: landscape.config.gardener.cloud/v1alpha1
kind: LandscapeKitConfiguration
# git: # Version control information for GLK to produce Flux configuration
#   url: https://github.com/<org>/<repo>
#   ref:
#     branch: <branch-name>
#   paths:
#     base: ./base
#     landscape: ./
# components: # Control of enabled and disabled components
#   exclude:
#   - component-name
#   include:
#   - component-name 
# ocm: # optional - to resolve OCI references with OCM information
#   repositories:
#   - <repo-url>
#   rootComponent:
#     name: <component-name>
#     version: <component-version>
#   originalRefs: true
# versionConfig: # optional - version information source when OCM is not used 
#   componentsVectorFile: ./versions.yaml
#   defaultVersionsUpdateStrategy: ReleaseBranch
```

## Drawbacks

The following drawbacks have been identified for the Gardener Landscape Kit:

- **Lack of Unified Landscape Abstraction**: GLK does not provide a unified abstraction layer for landscape management. As a result, operators must interact with the APIs of individual Gardener projects to understand the configuration and available options. This can increase the operational burden, especially for beginners.
- **Opinionated Structure May Limit Adoption**: The directory layout and component model enforced by GLK are opinionated and may not align with the structure of existing landscapes. This can make migration to GLK challenging or unattractive for operators with established setups.

## Alternatives

One alternative is to build a dedicated landscape operator that introduces an additional abstraction layer for managing Gardener landscapes. However, internal proof-of-concept work indicated that Gardener's existing abstractions are already sufficient for most use cases. Adding another layer would likely result in duplicated functionality and increased complexity for both developers and operators.

## Appendix

Parts of the Gardener Landscape Kit have already been implemented to PoC purposes - see [`gardener/gardener-landscape-kit`](https://github.com/gardener/gardener-landscape-kit).
