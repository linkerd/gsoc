- Contribution Name: `Alert Manager Integration`
- Implementation Owner: @christyjacob4
- Start Date: 2020-03-03
- Target Date: (is there a target date of completion?)
- RFC PR: [linkerd/gsoc#0000](https://github.com/linkerd/gsoc/pull/0000)
- Linkerd Issue: [linkerd/linkerd2#0000](https://github.com/linkerd/linkerd2/issues/0000)
- Reviewers: (Who should review the code deliverable? ex.@olix0r)

# Summary

[summary]: #summary

Linkerd uses Prometheus for collecting metrics from various endpoints. Out of the box, Prometheus is an opensource monitoring solution that gathers time series based numerical data. Your services need to expose an endpoint (/metrics) from which Prometheus can scrape metrics. The prometheus monitoring suite comes with tools that help enhance its capabilities like Grafana and AlertManager.

The current implementation of the Control Plane makes use of Prometheus for scraping endpoints and aggregating metrics.

The primary goal of this project is to integrate the Alertmanager into the current Control Plane so that Prometheus can Provide out of the box alerts to the preferred channels.

AlertManager can be used for silencing alerts, routing and sending alerts to preferred channels like slack, emails, pagerduty etc.

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

- By default, the Linkerd Control Plane doesn't have an alerting mechanism. Prometheus gathers all the metrics which can be visualised through Grafana, but in order to send these alerts to channels, we need to Integrate Alertmanager with Prometheus as part of the control plane.

* Once this RFC is implemented, users will have the option to install **Alertmanager** in the control plane using the following option

```sh
bin/linkerd install --alertmanager | kubectl apply -f -
```

- The optional **Alertmanager** installation will come with some default alerts
  which users can configure to appropriate receivers like
  pagerduty, email, slack and users will be able to customise the receivers by using a ```--receivers``` flag in the cli.

```sh
bin/linkerd install --alertmanager --receivers alert_manager_receivers.yml | kubectl apply -f -
```

- Users will also be able to specify custom rules along with the installation using a ```--prometheusRules ``` flag in the cli.

```sh
bin/linkerd install --alertmanager --prometheusRules prometheus_rules.yml | kubectl apply -f -
```

- Integration with Service Profiles will allow users to configure rules and alerts for per route metrics providing extremely fine grain alerting mechanisms for their apps.


# Design proposal (Step 2)

[design-proposal]: #design-proposal

**Note**: We suggest waiting to do this work, until Step 1 of the process is complete and you have has received feedback.

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features
- It is reasonably clear how the contribution would be implemented
- Corner cases are dissected by example
- Dependencies on libraries, tools, projects or work that isn't yet complete
- Use Cases
- Goals
- Non-Goals
- Deliverables

# Prior art

[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does this functionality exist in other software and what experience has their community had?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some
  relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other
software, provide readers of your RFC with a fuller picture. If there is no prior art, that is
fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from
other software.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?

# Future possibilities

[future-possibilities]: #future-possibilities

- If time permits, work on the identity and RBAC for the dashboard which will allow modifying the Prometheus rules and Alert Manager Configs directly from the **Linkerd Dashboard**.
- Allow users to add Prometheus rules using the **Linkerd Dashboard** and inject the new rules into the prometheus deployment, similar to how linkerd inject works.
- Allow users to add Alertmanager receivers using **Linkerd Dashboard** and inject them.