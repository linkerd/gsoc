- Contribution Name: Scale Testing
- Implementation Owner:  ['kohsheen1234'](https://github.com/kohsheen1234)
- Start Date: 26-02-2020
- Target Date: 
- RFC PR:
- Linkerd Issue: [linkerd/linkerd2#3895](https://github.com/linkerd/linkerd2/issues/3895)
- Reviewers: 


[summary]: #summary

# Summary
This conrtibution aims to completely automate the scale tests, so that scale-testing can be performed very easily. Automating scale tests is done to ensure the performance of Linkerd is maintained.

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

- Identify performance regression/potential errors with minimal manual intervention.
- Latency in the network communication can be a massive issue. When new features added to Linkerd, it might detoirate the performance of Linkerd. So, it is important to understand if any new feature which has been added recently has a negative impact on Linkerd's performance. Manually checking every-time if there has been a regression in the performance is not a convinient solution. This process can be automated, so that it is very simple to perform scale testing.
- Once this scale testing has been automated, Linkerd will be able to :
  - Automatically add a sample workload to the cluster.
  - Record cluster,control pane and data plane metrics during the test.
    - Success rate
    - Latency and throughput
    - Dashboard load times
    - Linkerd stat responsiveness
    - Prometheus cpu/memory usage
  - Report on resource usage, Linkerd performance and potential errors encountered.

Please avoid discussing your proposed solution.

# Design proposal (Step 2)

[design-proposal]: #design-proposal

https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands

**Note**: We suggest waiting to do this work, until Step 1 of the process is complete and you have has received feedback.

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear
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

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.
