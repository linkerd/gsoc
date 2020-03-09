- Contribution Name: Scale Testing
- Implementation Owner:  ['kohsheen1234'](https://github.com/kohsheen1234)
- Start Date: 29-02-2020
- Target Date: 
- RFC PR:
- Linkerd Issue: [linkerd/linkerd2#3895](https://github.com/linkerd/linkerd2/issues/3895)
- Reviewers: 

[summary]: #summary

# Summary
This contribution aims to completely automate the scale test framework, to ensure the scale tests are repeatable by the community. Automating scale tests serve to provide visibility into the proxy's performance and reveal any performance degration.
# Problem Statement (Step 1)

[problem-statement]: #problem-statement

- Identify performance regression/potential errors with minimal manual intervention.
- Latency in the network communication can be a massive issue. When new features added to Linkerd, it might deteriorate the performance of Linkerd. So, it is important to understand if any new feature which has been added recently has a negative impact on Linkerd's performance. Manually checking every time if there has been a regression in the performance is not a convenient solution. This process can be automated, so that it is very simple to perform scale testing.
- Once this scale testing has been automated,the scale test framework will be able to :
  - Automatically add a sample workload to the cluster.
  - Record cluster,control pane and data plane metrics during the test.
    - Success rate
    - Latency and throughput
    - Dashboard load times
    - `linkerd stat` responsiveness
    - Prometheus cpu/memory usage
  - Report on resource usage, Linkerd performance and potential errors encountered.

# Design proposal (Step 2)

[design-proposal]: #design-proposal


# Prior art

[prior-art]: #prior-art

- Scale and performance testing is included in other service mesh. 
  - For instance, let's consider Istio https://istio.io/docs/ops/deployment/performance-and-scalability/. 
  - Github link - https://github.com/istio/tools/tree/release-1.4/perf
- Current scale-test of linkerd - https://github.com/linkerd/linkerd2/blob/master/bin/test-scale 


# Unresolved questions

[unresolved-questions]: #unresolved-questions



# Future possibilities

[future-possibilities]: #future-possibilities
