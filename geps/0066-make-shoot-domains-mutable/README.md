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
- An authorized user modifies the domain configuration in the Shoot spec according to their needs **AND** adds the annotation `gardener.cloud/operation=rotate-ca-start` to trigger the migration.
- The two-phase CA rotation is started.
- In the `PREPARING` phase, the DNS records for new domain names are created and appended to the server certificates, while the old domain names are still kept in place to allow a seamless transition. The new internal domain is prepended to `status.internalDomains`, so both the new (first) and the old internal domain are recorded as in use. If the Shoot uses the default ServiceAccount token issuer — which is derived from the internal domain — the new issuer becomes the primary one (used to mint new tokens), and the old issuer is added to the kube-apiserver's `serviceAccountConfig.acceptedIssuers`, so tokens already issued with the old `iss` claim remain valid during the transition.
- When the `PREPARED` phase is reached, the cluster is available through its new domain names. This must be checked, additionally to all the new credentials.
- In the `COMPLETING` phase, the obsolete DNS records are deleted and the corresponding domains are removed from the server certificates. The obsolete internal domain is removed from `status.internalDomains`, leaving only the new one. Likewise, the old issuer is removed from the kube-apiserver's `serviceAccountConfig.acceptedIssuers` (see `PREPARING`). Note that bound/projected tokens refresh automatically, but legacy long-lived secret-based tokens carrying the old issuer would then be invalidated.

Adding the triggering annotation in the same step as the actual domain modifications is mandatory.

To make the internal domain optional, introduce the flag `spec.dns.internalDomain.enabled` in the Shoot API. It indicates, if the internal domain is actually needed for the Shoot. The flag is `true` by default (or if absent).

If the internal domain is disabled for a Shoot, its external domain is used in all places where the internal domain was used before, instead.

### Migration Mechanism for Entire Seeds
For cases where all Shoots on a Seed are affected (for example because of compliance requirements), introduce a way to provide the changed domain configuration through the Seed spec. 

Since the CA rotation is a process that must be executed in a planned manner, adding the changed information to the Seed spec must not automatically trigger the CA rotation for the affected Shoots.

**Flow of Operation for Entire Seeds:**
- The operator modifies the list of valid internal domains in the Seed spec according to their needs **AND** notifies all Shoot owners that they should trigger a CA rotation.
- Whenever a CA rotation is triggered for a specific Shoot on this Seed, the internal domain of this Shoot is migrated to the first entry in the Seed's list of valid internal domains.

Therefore the Seed API is extended to provide a list of internal domains instead of a single value. The first entry in this list is the one that must be applied to new or reconciled Shoots, while the other entries are also valid. This pattern is already used in other places.

Disabling the internal domains of all Shoots on a Seed is out of scope, this can only be done through the Shoot API.

### Securing Control Over the Domain Configuration
Only operators must be able to modify the domain configuration of a Shoot. Therefore, a new custom RBAC verb named `modify-spec-domains` is introduced, which protects the `Shoot.spec.dns.internalDomain` field, similar to the [`modify-spec-tolerations-whitelist`](https://github.com/gardener/gardener/blob/0025fc1765c6fdb9106249bb1754108acedb4362/docs/concepts/apiserver-admission-plugins.md#customverbauthorizer).

The domain configuration on the Seed level is already restricted to operators.

### Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Worker nodes are unavilable due to broken domain name resolution. | Low | High | The old DNS records and certificate SANs are kept during the first phase of the CA rotation ("prepare"). Only when all worker nodes are rolled and available under the new domain configuration, the obsolete DNS records and certificate SANs are removed. |
| The domain configuration is inconsistent or invalid. | Low | High | The validation enforces that each Shoot has at least one valid domain. The CA rotation is only triggered, if the domain configuration is consitent and valid. |
| Any user that has access to the Shoot resource can modify a cluster's internal or external domain. | Mid | High | A new custom RBAC verb is introduced, which is only granted to the `gardener.cloud:admin` role. The verb is checked every time a modification of the domain configuration in a Shoot spec is detected. Unauthorized modifications are rejected. |
| An operator removes an internal domain from a Seed that is still in use by Shoots. | Low | High | Seed admission rejects removing an internal domain that any Shoot on the Seed still records in `status.internalDomains`. |

## Design Details

### API Changes
- **Shoot Spec**: Add the configuration flag `spec.dns.internalDomain.enabled` to enable/disable the internal domain.
- **Shoot Status**: Add `status.internalDomains`, the list of internal (base) domains currently in use by the Shoot, maintained by gardenlet during reconciliation. Under normal operation it holds a single entry; during a domain migration it holds both the new and the old internal domain, with the new (primary) domain as the **first** entry. This field lets the Seed admission determine which internal domains are still in use.
- **Seed Spec**: Replace the single value for the internal domain with a list of multiple internal domains, designating the first entry as the primary value. The primary value is used for newly created Shoots and whenever a CA rotation of an existing Shoot allows to migrate the Shoot's internal domain.
- **Admission**: Add the custom `modify-spec-domains` RBAC verb.

### Extension of the CA Rotation Mechanism
- **Phase 1 (Prepare)**: Deploy both old and new `DNSRecord` resources. Update APIServer SANs to include both. Issue new kubeconfigs with the new domain.
- **Phase 2 (Complete)**: Clean up the old `DNSRecord` resources and remove the old domain from the SANs. 

If applicable, the ServiceAccount token issuer configuration must be updated as described in [Migration Mechanism for Single Shoots](#migration-mechanism-for-single-shoots).

### Extension of the Admission Mechanism
- implement or extend a admission plugin that checks the new RBAC verb
- the `ShootDNS` admission plugin might be a good fit

### Validation
- ensure a Shoot has at least one valid domain (either internal or external)
- enforce that domain changes only occur in the same API request that prepares a CA rotation
- reject removing an internal domain from a Seed's list while it is still in use, i.e. while any Shoot scheduled on that Seed still lists it in `status.internalDomains`. This mirrors the existing `ValidateDefaultDomainsChangeForSeed` check for default (external) domains in the Seed validator admission plugin.
- the `ShootDNS` admission plugin might be a good fit

## Drawbacks
- increases the complexity of the reconciliation logic in Botanist
- tight coupling between DNS management and CA rotation

## Alternatives
- **Manual Migration**: High risk of downtime and configuration errors.
- **Separate Rotation Resource**: Over-complicates the user experience compared to extending the already existing CA rotation mechanism.
- **Automatic CA Rotation on Domain Change**: A CA rotation has impact well beyond DNS. Requiring the annotation keeps users explicitly in control of a disruptive operation and guards against accidental/unintended domain changes silently triggering a full rotation.
