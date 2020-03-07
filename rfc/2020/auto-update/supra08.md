- Contribution Name: Auto-Update
- Implementation Owner: [@supra08](https://github.com/supra08)
- Start Date: 2020-03-7 (7th Mar, 2020)
- Target Date: 
- RFC PR: 
- Linkerd Issue: [linkerd/linkerd2#1903](https://github.com/linkerd/linkerd2/issues/1903)
- Reviewers: 

# Summary

[summary]: #summary

The aim of this proposal is to automate the update procedure by implementing a *Kubernetes operator* which will poll a *version-check API* and auto update the control plane and the data plane, also validating its successful completion.

# Problem Statement (Step 1)

[problem-statement]: #problem-statement
* `Linkerd`, being an active open source project has a high frequency of updates and releases. Maintaining compatible versions in the control and data planes and keeping up with the edge releases is a necessary requirement. Currently the upgrade procedure is primarily manual, with `Linkerd CLI` to be updated and injected with the updated YAML. Maintaining version consistency in the controller pods and data plane proxies requires re-injecting the updated configurations and the developer has to keep up with the edge releases.

* This whole problem can be broken down into two subtasks:
    * First task requires an *operator resource* that takes care of watching the *upstream update registry* (provided with an API) and carries out the necessary tasks, decoupling user intervention.
    * The second task revolves around planning the *update schemes* (choose ordering edge releases, etc), validate the successful completion of the update and handle cases when the update brings about breaking changes to the infrastructure (integration checks.

* Auto-update feature will ultimately remove the heavylifting from the developer of keeping track of the releases. With this implemented, security will be leveraged and updated as soon as a patch is released. System wide version consistency will be automatically managed with automatic rollback in case of failures.

# Design proposal (Step 2)

[design-proposal]: #design-proposal

# Prior art

[prior-art]: #prior-art

# Unresolved questions

[unresolved-questions]: #unresolved-questions

# Future possibilities

[future-possibilities]: #future-possibilities
