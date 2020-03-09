
- Contribution Name: `Scale Testing`
- Implementation Owner: [@srv-twry](https://github.com/srv-twry)
- Start Date: 2020-03-09
- Target Date: 
- RFC PR: [linkerd/gsoc#0006](https://github.com/linkerd/gsoc/pull/6)
- Linkerd Issue: [linkerd/linkerd2#3895](https://github.com/linkerd/linkerd2/issues/3895)
- Reviewers: 

# Summary

[summary]: #summary

This project aims to automate the scale testing infrastructure to run reproducible scale tests at a non-trivial scale. It will allow measuring the impact of future changes and gain visibility into any performance regressions. 

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

- Identification of performance regressions automatically
- Linkerd strives to maintain it's blazing fast speed and performance. Addition of new features can sometimes have a negative impact on its performance. Frequent addition of new features means that there is little room for manual testing. Hence there is a need to automate the scale testing infrastructure to run reproducible scale tests. 
- Once implemented, the scale testing framework will:
    - Create a Kubernetes cluster on desired cloud service provider and install Linkerd and other tools in it.
    - Automatically add a sample workload to the cluster.
    - Record cluster, control plane and data plane metrics during test
        - Success rate
        - Latency and throughput
        - Dashboard load times
        - `linkerd stat` responsiveness
        - Prometheus cpu/memory usage
    - Report on resource usage, Linkerd performance and potential errors encountered. 
    - Cleanup the Kubernetes cluster


# Design proposal (Step 2)

[design-proposal]: #design-proposal

# Prior art

[prior-art]: #prior-art

There have been attempts to benchmark Istio and Linkerd's performance:

- Istio Performance/Stability Testing: [https://github.com/istio/tools/tree/master/perf](https://github.com/istio/tools/tree/master/perf)
- Linkerd and Istio Performance Benchmark: [https://medium.com/@ihcsim/linkerd-2-0-and-istio-performance-benchmark-df290101c2bb](https://medium.com/@ihcsim/linkerd-2-0-and-istio-performance-benchmark-df290101c2bb)

# Unresolved questions

[unresolved-questions]: #unresolved-questions

# Future possibilities

[future-possibilities]: #future-possibilities
