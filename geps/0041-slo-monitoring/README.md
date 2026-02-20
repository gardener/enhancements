# GEP-0041: SLO Monitoring

## Table of Contents

- [GEP-0041: SLO Monitoring](#gep-0041-slo-monitoring)
  - [Table of Contents](#table-of-contents)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
  - [Proposal](#proposal)
    - [High level Architecture](#high-level-architecture)
    - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
      - [Seed inaccessibility](#seed-inaccessibility)
    - [Risks and Mitigations](#risks-and-mitigations)
  - [Design Details](#design-details)
  - [Drawbacks](#drawbacks)
  - [Alternatives](#alternatives)
  - [References](#references)

## Summary

<!--
This section is incredibly important for producing high-quality, user-focused
documentation such as release notes or a development roadmap. It should be
possible to collect this information before implementation begins, in order to
avoid requiring implementors to split their attention between writing release
notes and implementing the feature itself. GEP editors should help to ensure
that the tone and content of the `Summary` section is useful for a wide audience.

A good summary is probably at least a paragraph in length.
-->
Although Gardener and Kubernetes expose a lot of metrics, there is presently no easy way to evaluate the availability of a shoot from the customer (shoot owner/operator) perspective. In the context that Gardener is meant to run a multitude of Kubernetes clusters at scale, operating it is usually not an easy endeavor. Having such means to measure and monitor the shoot's reliability is a key aspect from both customer happiness and operational excellence.

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this GEP. Describe why the change is important and the benefits to users.
-->

Day-2 operation tooling is a must to help drive adoption. This also mimics industry best practices as described in the [SRE book](https://sre.google/sre-book/table-of-contents/).

This extension also enables Gardener operators and developers to make data-driven decisions about reliability vs feature prioritization while providing clear visibility into system health.

This would benefit multiple user personas, including:

- Operator: Having a ready to go extension to help with reliability and day-to-day operations
- Developers: Having SLOs is a great tool to help prioritize between reliability and feature issues.
- Users: Since SLOs are internal objectives meant for Gardener operators, they are normally not shared with the end users (shoot owners/operators). However, better reliability generally makes customers happy, which in turn drives user adoption.

### Goals

<!--
List the specific goals of the GEP. What is it trying to achieve? How will we
know that this has succeeded?
-->
- An SLO (service level objective) extension that provides a standardized framework for measuring and monitoring shoot cluster reliability, always from the customer perspective. By customer perspective, we want to focus on metrics that would reflect the customer's experience and satisfaction in operating their shoot clusters.

### Non-Goals

<!--
What is out of scope for this GEP? Listing non-goals helps to focus discussion
and make progress.
-->
- Provide SLAs to customers. SLOs are internal objectives for Gardener operators, and are not meant to be shared with customers (shoot owners/operators). However, such extension could be used to provide SLAs in the future, if desired.

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->

### High level Architecture

This extension delivers 4 core capabilities:

- **Default and custom SLI**: A set of predefined SLIs based on industry best practices for Kubernetes clusters and Gardener specifics, but always from the customer perspective. However, since the needs of each Gardener operator may vary, these defaults will be configurable. There should also be the ability to define custom SLIs based on specific metrics exposed by Gardener or the shoot clusters. However, those SLIs would need to be generic across the landscape.
- **Configurable SLOs**: The SLO parameters (SLI, threshold, ...) should be configurable. Then a templating engine would generate the necessary Prometheus recording rules and Perses dashboards.
- **SLO-based alerting**: Since we have the data to calculate SLO violations and burn rates (SRE best practice), we should also provide, as part of the extension, the capability to configure an Alertmanager based on those SLOs. Again, this should be configurable to fit the needs of each Gardener operator.
- **Monitoring infrastructure**: The extension should provide the necessary monitoring infrastructure to collect, store, and visualize SLO-related metrics. This includes Prometheus rules for SLI calculation, Perses dashboards for visualization, Prometheus alerts for SLO violations, Alertmanager to manage those alerts, etc.

The extension builds on the existing monitoring infrastructure (Prometheus operator, Perses operators, plutono annotations, ...), using a dedicated Prometheus instance in the runtime cluster to collect and aggregate SLO-specific metrics with minimal impact on the existing monitoring systems.

![SLO extension high-level architecture](./slo-extension-plan.png)

### Notes/Constraints/Caveats (Optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above?
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->
### Seed inaccessibility

In some cases, the seed cannot be reached from the runtime cluster, so Prometheus is not able to scrape (federate) metrics from the seed. This is the case when a seed is in a network segment (behind a firewall) that forbids incoming traffic from the runtime cluster. In those cases, gardenlet still works because its connection is initiated from the seed to the runtime/garden cluster (NAT or similar egress policy has to be allowed). Hence, since Prometheus is not able to scrape metrics from the seed, we would not be able to calculate SLOs based on those metrics.

To enable total coverage across a landscape, there are 2 possible solutions:

- **Push-based metrics**: In this approach, instead of relying on Prometheus `federate` (pull) mechanism, we could implement a flag in the seed's configuration to use Prometheus `RemoteWrite` capability to push metrics to the `garden-prometheus` in the runtime cluster. This would require changes in the gardenlet to configure prometheus properly. This also requires the prometheus operator to support `RemoteWrite` (work still ongoing in https://github.com/prometheus-operator/prometheus-operator/issues/6508)

- **Reverse VPN**: Another approach could be to establish a reverse VPN connection from the seed to the runtime cluster, allowing Prometheus to scrape metrics as if it were directly accessible. This would require setting up reversed VPN tunnels for each seed, similar to what we already do between seed<=>shoot clusters. Although this approach could also enable other use cases, it also adds complexity and operational overhead, so it should be carefully evaluated before being implemented.

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
Gardener ecosystem.
-->
Since the extension introduces additional monitoring components and configurations, there is a risk of increased resource consumption and potential conflicts with existing monitoring setups. Careful planning and testing are required to mitigate these risks.

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->
N/A

## Drawbacks

<!--
Why should this GEP _not_ be implemented?
-->
N/A

## Alternatives

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->
- **Reusing existing Prometheus instances**: Rejected due to risk of impacting other critical metrics. Since SLO calculations can be resource-intensive, isolating them ensures stability and separation of concerns

## References

- [Kubernetes's SLI / SLO](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/slos.md)
- [SRE Book](https://sre.google/sre-book/table-of-contents/) (why)
- [SRE Workbook](https://sre.google/workbook/table-of-contents/) (how)
