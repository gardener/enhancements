# GEP-0066: Make Shoot Domains Mutable

## Summary
This GEP proposes to provide a mechanism to make both the internal and external domains of a Shoot mutable. In certain environments, the internal domain can even be removed completely. The migration process after changing, adding, or removing a domain is carried out as part of a CA rotation.

## Motivation
Every Gardener environment requires at least one DNS zone for managing "internal domains" of shoots. The internal domain is especially useful, when shoot owners can configure their own "external domain"/provider/zone for their shoot API servers (i.e., via Shoot.spec.dns). It is used for all cluster-critical communication (e.g., kubelet/gardener-node-agent to API server) to ensure robustness and prevent disruptions caused by user configurations.

Currently, an existing shoot's internal domain cannot be changed. Although, in real life, there are several cases, where domain names need to be changed also for existing shoots, for example because of a wrong configuration in the first place, or due to compliance reasons.

Some Gardener setups don't grant users access to the Shoot API, but offer a different ("proxy") API (e.g., STACKIT Kubernetes Engine). In those setups, the environment has full control over the external domain. As it is not configurable by the shoot owners, the environment can make sure, that the external domain is valid and available. Having an internal domain in such an environment is not strictly necessary, at least not in behalf of robustness and availability. 

As the internal domain is currently configured per seed, all shoots on a seed share the same internal domain zone. Consequently, if that zone is public, a shoot's internal domain will be publicly exposed, even if the shoot resides in a private network zone. 

A native way to manage these domains would improve the user experience for all of the described cases. Internal and external domains should be mutable, the internal domain should even be optional in cases, where the external domain is sufficient.

### Goals
- Support changing both the internal and external domain of existing Shoots.
- Make the internal domain optional, for example for gardener setups where the customer doesn't have access to the Shoot API and therefore can't change the DNS
- Enforce that each Shoot has at least one valid domain.
- Protect the ability to modify any domain to prevent undesired/accidental changes.
- Support a seamless migration of domains via the CA rotation mechanism.
- Support seed-wide domain migrations.

### Non-Goals
- Allowing Shoots without any valid domain.
- Replacing the DNS extension architecture.

## Proposal

### Migration Mechanism for Single Shoots
Because the internal and external domains are part of a Shoot's network setup, the existing CA rotation mechanism can be reused for domain migrations.

**Flow of Operation for Single Shoots:**
- The operator modifies the domain configuration in the Shoot spec according to their needs **AND** adds the annotation `gardener.cloud/operation=rotate-ca-start` to trigger the migration.
- The two-phase CA rotation is started.
- In the `PREPARING` phase, the new DNS records are created and appended to the server certificates, while the old ones are still kept in place to allow a seamless transition.
- In the `COMPLETING` phase, the obsolete DNS records deleted and the corresponding domains are removed from the server certificate.

Adding the triggering annotation in the same step as the actual domain modifications is mandatory.

Make the internal domain optional for cases, where the external domain is fully under the environments control and the internal domain is not desired. Introduce the flag `spec.dns.internalDomain.enabled` in the Shoot API to indicate if the internal domain is actually needed for the Shoot. The flag is `true` by default (or if absent).

If the internal domain is disabled for a Shoot, its external domain is used instead.

### Migration Mechanism for Whole Seeds
For cases where all Shoots on a Seed are affected (for example because of compliance requirements), introduce a way to provide the change information through the Seed spec. 

Since the CA rotation is a process that must be executed in a planned manner, adding the change information to the Seed spec must not trigger the CA rotation for the affected Shoots.

**Flow of Operation for Whole Seeds:**
- The operator modifies the list of valid internal domains in the Seed spec according to their needs **AND** notifies all shoot owners that they should trigger a CA rotation.
- Whenever a Shoot CA rotation is triggered, the internal domain of this Shoot is migrated to the first entry in the Seed's list of valid internal domains (or removed, if the internal domain disabled for this Shoot).

### Securing Control Over the Domain Configuration
Only users with the role `gardener.cloud:admin` must be able to modify the domain configuration of a Shoot or a Seed. Therefore, a new custom RBAC verb named `modify-spec-domains` is introduced, similar to the [`modify-spec-tolerations-whitelist`](https://github.com/gardener/gardener/blob/0025fc1765c6fdb9106249bb1754108acedb4362/docs/concepts/apiserver-admission-plugins.md#customverbauthorizer).

### Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Worker nodes are unavilable due to broken domain name resolution. | Low | High | The old DNS records and certificate SANs are kept during the first phase of the CA rotation ("prepare"). Only when all worker nodes are rolled and available under the new domain configuration, the obsolete DNS records and certificate SANs are removed. |
| The domain configuration is inconsistent or invalid. | Low | High | The validation enforces that each Shoot has at least one valid domain. The CA rotation is only triggered, if the domain configuration is consitent and valid. |
| Any user that has access to the Shoot or Seed CRD can modify a cluster's internal or external domain. | Mid | High | A new custom RBAC verb is introduced, which is only granted to the `gardener.cloud:admin` role. The verb is checked every time a modification of the domain configuration in a Shoot or Seed spec is detected. Unauthorized modifications are rejected. |

## Design Details

### API Changes
- **Shoot Spec**: Add the configuration flag `spec.dns.internalDomain.enabled` to enable/disable the internal domain.
- **Seed Spec**: Replace the single value for the internal domain with a list of multiple internal domains, designating the first entry as the primary value. The primary value is used for newly created Shoots and whenever a CA rotation of an existing Shoot allows to migrate the Shoot's internal domain.
- **Admission**: Add the custom `modify-spec-domains` RBAC verb.

### Extension of the CA Rotation Mechanism
- **Phase 1 (Prepare)**: Deploy both old and new `DNSRecord` resources. Update APIServer SANs to include both. Issue new kubeconfigs with the new domain.
- **Phase 2 (Complete)**: Cleanup old `DNSRecord` resources and remove the old domain from SANs.

### Extension of the Admission Mechanism
- implement or extend a admission plugin that checks the new RBAC verb
- the `ShootDNS` admission plugin might be a good fit

### Validation
- ensure a Shoot has at least one valid domain (either internal or external)
- enforce that domain changes only occur in the same API request that prepares a CA rotation
- the `ShootDNS` admission plugin might be a good fit

## Drawbacks
- increases the complexity of the reconciliation logic in Botanist
- tight coupling between DNS management and CA rotation

## Alternatives
- **Manual Migration**: High risk of downtime and configuration errors.
- **Separate Rotation Resource**: Over-complicates the user experience compared to extending the already existing CA rotation mechanism.
