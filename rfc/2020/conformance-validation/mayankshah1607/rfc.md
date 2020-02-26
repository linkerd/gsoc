- Contribution Name: Conformance Validation
- Implementation Owner: [@mayankshah1607](https://github.com/mayankshah1607)
- Start Date: 2020-02-23 (24th Feb, 2020)
- Target Date: 
- RFC PR:
- Linkerd Issue: https://github.com/linkerd/linkerd2/issues/1096
- Reviewers: 

# Summary

[summary]: #summary
This project proposes a new test suite that shall be used for conformance validation. The new test suite shall be used to validate non-trivial network communication (HTTP, gRPC, websocket) among stateless and stateful workloads in the Linkerd data plane. The correctness of the interaction between the Linkerd control plane and the Kubernetes API Server will also be tested. This shall be done by carrying out extensive e2e tests of linkerd features using a sample distributed application (data plane).

# Problem Statement (Step 1)

[problem-statement]: #problem-statement
Linkerd has an extensive check suite that validates a cluster is ready to install Linkerd and that the installation was successful. These checks are, unfortunately, static checks. Because of the wide number of ways a Kubernetes cluster can be configured, users want a way to validate their specific install for conformance over time. The proposed project tackles this problem by allowing users to deploy sample workloads to their cluster and carry out extensive e2e (conformance) tests. 

# Design proposal (Step 2)

[design-proposal]: #design-proposal


# Prior art

[prior-art]: #prior-art
- Sample sonobuoy plugins - https://github.com/vmware-tanzu/sonobuoy/tree/master/examples/plugins
- K8s e2e tests - https://github.com/kubernetes/kubernetes/tree/master/test/e2e
- K8s sonobuoy image - https://github.com/kubernetes/kubernetes/tree/master/cluster/images/conformance
- Some sample applications developed by linkerd for testing:
    - https://github.com/BuoyantIO/emojivoto
    - https://github.com/BuoyantIO/booksapp
    - https://github.com/linkerd/linkerd-examples
    - https://github.com/BuoyantIO/slow_cooker/


# Unresolved questions

[unresolved-questions]: #unresolved-questions


# Future possibilities

[future-possibilities]: #future-possibilities

