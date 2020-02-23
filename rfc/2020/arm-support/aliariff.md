- Contribution Name: ARM Support
- Implementation Owner:
- Start Date: 2020-02-23
- Target Date:
- RFC PR:
- Linkerd Issue: [linkerd/linkerd2#1165](https://github.com/linkerd/linkerd2/issues/1165), [linkerd/linkerd2#3114](https://github.com/linkerd/linkerd2/issues/3114)
- Reviewers:

# Summary

[summary]: #summary

This project goal is to make Linkerd officially support ARM architecture (`arm` & `arm64`).

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

Currently, Linkerd is only supporting `amd64` architecture officially. Supporting other architectures will make Linkerd more powerful and attractive for the users.

After the project implemented, releasing process will have more steps:
- Build docker images in ARM architecture
- Integration test for the resulting images
- Publish the images

# Design proposal (Step 2)

[design-proposal]: #design-proposal

# Prior art

[prior-art]: #prior-art

This project is already done in many CNCF projects like:
- [kubernetes/kubernetes#17981](https://github.com/kubernetes/kubernetes/issues/17981)
- [grafana/grafana#13186](https://github.com/grafana/grafana/issues/13186)
- [containous/maesh#206](https://github.com/containous/maesh/issues/206)

And also proof that Linkerd can be run on ARM [linkerd/linkerd2#1165](https://github.com/linkerd/linkerd2/issues/1165#issuecomment-515470739)

# Unresolved questions

[unresolved-questions]: #unresolved-questions

# Future possibilities

[future-possibilities]: #future-possibilities
