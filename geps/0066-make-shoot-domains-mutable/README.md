# GEP-0066: Make Shoot Domains Mutable

## Summary
This GEP proposes to provide a mechanism to make both the internal and external domains of a Shoot mutable. In certain environments, the internal domain can even be removed completely. The migration process after changing, adding, or removing a domain is carried out as part of a CA rotation.

## Motivation
Every Gardener environment requires at least one DNS zone for managing "internal domains" of shoots. The internal domain can be configured only down to the seed level, hence it is under the control of the operator. This is especially useful when shoot owners can configure their own "external domain"/provider/zone for their shoot API servers (i.e., via `Shoot.spec.dns.domain`). The internal domain is used for all cluster-critical communication (e.g., kubelet/gardener-node-agent to API server) to ensure robustness and prevent disruptions caused by user misconfigurations.

Currently, an existing shoot's internal domain cannot be changed. However, in real life, there are several cases where domain names need to be changed also for existing shoots, for example because of a wrong configuration in the first place, or due to compliance reasons.

Some Gardener setups don't grant users access to the Shoot API, but offer a different ("proxy") API (e.g., STACKIT Kubernetes Engine). In those setups, the environment has full control over the external domain. As it is not configurable by the shoot owners, the environment can make sure that the external domain is valid and available. Having an internal domain in such an environment is not strictly necessary, at least not for the sake of robustness and availability.

As the internal domain can currently only be configured down to the seed level, all shoots on a seed share the same internal domain zone. Consequently, if that zone is public, a shoot's internal domain will be publicly exposed, even if the shoot resides in a private network zone.

Providing a way to manage these domains for existing shoots would improve the user experience for all of the described cases. Internal and external domains should be mutable; the internal domain should even be optional in cases where the external domain is sufficient.

### Goals
- Make a Shoot's external domain mutable.
- Make a Seed's internal domain configuration mutable.
- Make the internal domain optional on the Seed level, for example for Gardener setups where the customer doesn't have access to the Shoot API and therefore can't change the external domain.
- Support a seamless migration of a Shoot's internal and external domain via the CA rotation mechanism.
- Enforce that each Shoot has at least one valid domain.
- Protect the ability to modify a Shoot's domains to prevent undesired/accidental changes.

### Non-Goals
- Allowing Shoots without any valid domain.
- Replacing the DNS extension architecture.

## Proposal

### Modify a Shoot's External Domain
To apply modifications of  a Shoot's internal and external domain, the existing CA rotation mechanism is reused.

**Flow of Operation: Change the External Domain for a Single Shoot**
- An authorized user modifies the field `Shoot.spec.dns.domain` according to their needs **AND** adds the annotation `gardener.cloud/operation=rotate-ca-start` to trigger the migration.
- The two-phase CA rotation is started.
- In the `PREPARING` phase, the DNS records for new domain names are created and appended to the server certificates, while the old domain names are still kept in place to allow a seamless transition. The new internal domain is written to `Shoot.status.advertisedAddresses` with the name `internal`; the old internal domain is added to `Shoot.status.advertisedAddresses` with the name `obsolete-internal`. So both the new and the old internal domain are recorded as in use. If the Shoot uses the default ServiceAccount token issuer — which is derived from the internal domain — the new issuer becomes the primary one (used to mint new tokens), and the old issuer is added to the kube-apiserver's `serviceAccountConfig.acceptedIssuers`, so tokens already issued with the old `iss` claim remain valid during the transition.
- When the `PREPARED` phase is reached, the cluster is available through its new domain names. Users need to ensure that they use the new domain and the credentials to access the cluster from now on.
- In the `COMPLETING` phase, the obsolete DNS records are deleted and the corresponding domains are removed from the server certificates. The obsolete internal domain is removed from `Shoot.status.advertisedAddresses`, leaving only the new one. Likewise, the old issuer is removed from the kube-apiserver's `serviceAccountConfig.acceptedIssuers` (see `PREPARING`). Note that bound/projected tokens refresh automatically, but tokens acquired through the token request API will be invalid after the migration. Users must take care of renewing those tokens.

Adding the triggering annotation in the same step as the actual domain modifications is mandatory.

### Modify the Internal Domain for an Entire Seed
The internal domain configuration is part of a Seed's spec in `Seed.spec.dns.internal` and affects all Shoots on this Seed. A modification of this structure's `domain` field requires a CA rotation of all Shoots. Since the CA rotation is a process that must be executed in a planned manner, adding the changed information to the Seed spec must not automatically trigger the CA rotation for the affected Shoots.

To make the Shoot owners aware of the need for a CA rotation, a new constraint will be introduced on the Shoot level to provide the corresponding information.

To allow disabling the internal domain, a new field named `Seed.spec.dns.internalDomainEnabled` is introduced. The default value is `true`, to keep the Shoot spec backward compatible. A modification of this field also requires a CA rotation of all Shoots.

The internal domain may only be disabled if every Shoot on the Seed has a valid external domain, so each Shoot keeps at least one valid domain.

**Flow of Operation for Entire Seeds:**
- The operator modifies `Seed.spec.dns.internal.domain` or `Seed.spec.dns.internalDomainEnabled` according to their needs.
- If the gardenlet detects the mismatch, it adds the new constraint to make the Shoot owners aware of the need for a CA rotation.
- Whenever a CA rotation is triggered for a specific Shoot on this Seed, the internal domain of this Shoot is migrated to the new internal domain.

### Securing Control Over the Domain Configuration
For the case that only operators should be able to modify the external domain configuration of a Shoot, a new custom RBAC verb named `modify-spec-domain` is introduced, which protects the `Shoot.spec.dns.domain` field, similar to the `modify-spec-tolerations-whitelist` and `mark-self-hosted` verbs provided through the [`CustomVerbAuthorizer`](https://github.com/gardener/gardener/blob/0025fc1765c6fdb9106249bb1754108acedb4362/docs/concepts/apiserver-admission-plugins.md#customverbauthorizer).

The domain configuration on the Seed level can already only be modified by operators.

### Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Worker nodes are unavailable due to broken domain name resolution. | Low | High | The old DNS records and certificate SANs are kept during the first phase of the CA rotation ("prepare"). Only when all worker nodes are rolled and available under the new domain configuration, the obsolete DNS records and certificate SANs are removed. |
| The domain configuration is inconsistent or invalid. | Low | High | The validation enforces that each Shoot has at least one valid domain. The CA rotation is only triggered, if the domain configuration is consistent and valid. |
| Any user that has access to the Shoot resource can modify a cluster's internal or external domain. | Mid | High | A new custom RBAC verb is introduced, which is only granted to the `gardener.cloud:admin` role. The verb is checked every time a modification of the domain configuration in a Shoot spec is detected. Unauthorized modifications are rejected. |


## Design Details

### API Changes
- **Shoot Spec**: Allow the modification of `spec.dns.domain`.
- **Shoot Status**: Allow a new entry with name `obsolete-internal` in the `status.advertisedAddresses` while a CA rotation with domain migration is ongoing. Add a new constraint that detects a mismatch between the Seed's and the Shoot's internal domain.
- **Seed Spec**: Add the new field `spec.dns.internalDomainEnabled` to indicate the need for an internal domain for the Shoots on this Seed.
- **Admission**: Add the custom `modify-spec-domain` RBAC verb.

### Extension of the CA Rotation Mechanism
- **Phase 1 (Prepare)**: Deploy both old and new `DNSRecord` resources. Update APIServer SANs to include both. Issue new kubeconfigs with the new domain. Add the `obsolete-internal` domain into the `status.advertisedAddresses`.
- **Phase 2 (Complete)**: Clean up the old `DNSRecord` resources and remove the old domain from the SANs. Remove the `obsolete-internal` domain from `status.advertisedAddresses`.

If applicable, the ServiceAccount token issuer configuration must be updated as described in [Modify a Shoot's External Domain](#modify-a-shoots-external-domain).

### Extension of the Admission Mechanism
- implement or extend an admission plugin that checks the new RBAC verb
- the `ShootDNS` admission plugin might be a good fit

### Validation
- ensure a Shoot has at least one valid domain (either internal or external)
- enforce that domain changes only occur in the same API request that prepares a CA rotation
- the `ShootDNS` admission plugin might be a good fit

## Drawbacks
- increases the complexity of the reconciliation logic in Botanist

## Alternatives

### Options for Authorizing Shoot Domain Changes

This GEP proposes to introduce a new RBAC verb to protect domain configuration from modifications by the owners. Here are valid alternatives:

1. **No Dedicated Restriction**: Rely solely on the mandatory simultaneous CA rotation as the guard, and let anyone with permission to update the Shoot change the domains.
2. **Global Configuration**: Use a global configuration option instead of a permission to allow/disallow the modification of the domain configuration on the shoot level in the entire Gardener installation.

### Modify the Internal Domain per Shoot

A broader variant of this GEP's idea would be to allow enabling/disabling the internal domain at the Shoot level. Therefore, corresponding fields are introduced to the Shoot spec:
- `Shoot.spec.dns.internal`
- `Shoot.spec.dns.internalDomainEnabled`

Those fields can only be modified in conjunction with triggering a CA rotation, just like a modification of the Shoot's external domain.

This variant allows having Shoots with different internal domains (or no internal domain at all) on the same Seed, for example if Shoots of multiple customers with individual requirements share the same Seed.

### Other Alternatives

The following alternatives were considered as not applicable:
- **Manual Migration**: High risk of downtime and configuration errors.
- **Separate Rotation Resource**: Over-complicates the user experience compared to extending the already existing CA rotation mechanism.
- **Automatic CA Rotation on Domain Change**: A CA rotation has impact well beyond DNS. Requiring the annotation keeps users explicitly in control of a disruptive operation and guards against accidental/unintended domain changes silently triggering a full rotation.
