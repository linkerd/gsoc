- Contribution Name: Controller Auto-Update Operator
- Implementation Owner: [@Matei207](https://github.com/Matei207/)
- Start Date: 2020-03-30
- Target Date: 2020-08-24 
- RFC PR: 
- Linkerd Issue: [linkerd/linkerd2#1903](https://github.com/linkerd/linkerd2/issues/1903)
- Reviewers: 

# Summary

[summary]: #summary

RFC aimed at the design and implementation of a new controlled for Linkerd that will follow the operator pattern. The new controller is thought of as an auto-updating mechanism to simplify the process of upgrading to a new edge release, as well as eliminate any human-made errors that arise from it. It will make use of a new custom resource definition that will represent versioning for Linkerd. On each released version, the operator will restart the proxies. This should pave the way for new features to be built with automation and ease-of-use in mind.

# Problem Statement

[problem-statement]: #problem-statement

Linkerd follows a high-velocity release-train, the versions are as of now broken into two categories: stable, and edge releases respectively. Following convention, stable releases are infrequent and introduce new features, fixes or policies. On the other side of the spectrum, edge releases follow a weekly pattern; they deliver bug fixes, new features and enhancements, but in smaller (and less stable) increments([*](https://linkerd.io/2/edge/)). Consequently, users that choose to adopt the more-frequent edge release cycle will also have to upgrade to newer versions more frequently in order to make use of new features or to make use of new fixes. This RFC proposes the addition of a new controller to Linkerd, that follows the Kubernetes [operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/), and that will auto-update the control-plane once new edge releases are made available.

Currently, according to [documentation](https://linkerd.io/2/tasks/upgrade/#upgrade-the-data-plane), in order to perform an upgrade, the adopter must upgrade both the control, and the data plane. In order to upgrade the control-plane, the `linkerd cli` (if used) must also be upgraded, then, through `linkerd upgrade`, upgrade manifests can be rendered and applied to the cluster. The data plane component can be upgraded by re-deploying the pods (which boils it down to a simple `kubectl` command), the `proxy-injector` will ensure that the correct proxy version is used. Although simple in theory, the [documentation](https://linkerd.io/2/tasks/upgrade/#upgrade-the-control-plane) does reference more complications to the process, such as multi-stage upgrades, upgrading through `helm` (and not through `linkerd`), as well as ensuring TLS certificates are not expired. Furthermore, when applying the manifests, the right `kubectl` command must be used to make sure "(...) existing configuration and mTLS secrets are retained" ([*](https://linkerd.io/2/tasks/upgrade/#with-linkerd-cli)). This means that although an upgrade is simple in practice, many things can go wrong during a manual upgrade, doing this on a weekly (or even bi-weekly, monthly, etc) basis will inevitably introduce errors for a number of adopters and/or developers. Moreover, by implementing this RFC, future work can also be enabled -- or at the very least, considered.

Future work opportunities, as well as how the current state of the system will change, depends on the chosen implementation for this RFC. Optimistically, the introduction of automatic versioning for the control-plane will impact the workflows around the interaction with the system -- less feature implementations and edge cases will have to be considered for the `linkerd cli`, with less people upgrading as often, the workflow around upgrading can be bespoke to the release type (i.e when upgrading to stable, follow through using the normal procedure, otherwise, use the operator). More implementation agnostic changes would come in the form of test behaviour (additional integration tests and unit tests can be made to ensure non-trivial features such as backwards compatibility) and an overall simplification of bootstrapping Linkerd, which should adhere to the process of simplicity yet diverse choice. In a nutshell, optimistically, this will allow adopters and developers to cross a weekly task off their list and piggyback on it for future work. Pesimistically, this can raise the complexity if not implemented well, as well as introduce a new range of issues and displeasures from the community.

# Design proposal

[design-proposal]: #design-proposal

By following the [operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/), at its core, the controller must *track* version changes, *observe* the state of the cluster, and *update* the control plane (subsequently updating the proxies too). The rest of this section will be broken down into sub-sections targeting the each specific behviour.

### 1. Tracking version changes
---

* As [previously discussed](https://github.com/linkerd/gsoc/pull/5#discussion_r392028320), version changes can be tracked by polling `https://versioncheck.linkerd.io/version.json`. Since edge versions are released weekly, predicting when the next version will be released is trivial. `prior-art 1` introduces a proof of concept application that makes use of a CRD to track version changes. This RFC wishes to leverage that in the implementation (otherwise it would be hard to make use of the operator pattern).

* Tracking version changes then becomes a problem of how to create the custom resource: the initial suggestion I had in mind going forward was to have a `worker` as part of the operator that polls the version-check file at specific intervals (e.g using `go wait.until`) and creates a new custom resource every time the version changes. Custom resources will be created then picked up by the operator and processed on the next iteration of the feedback loop. Another idea would be to create a `cronJob` as part of the `linkerd install` phase. This would be a simple bash script that pulls down the most recent version (using the API) and creates a new custom resource. This might not be ideal, however, depending on the chosen implementation.

* **Dependencies**: creating and registering the definition of the resource is quite an easy task. Managing it and programatically dealing with it might not be. `prior-art 1` uses `Kubebuilder` to work with the custom resource programatically, another option is to use the `K8s code-builder` project. From personal experience, `Kubebuilder` should be used -- eliminates the need of managing the cache and running into a `go.mod` dependency failure; to my understanding, using `code-builder` would require a cache (and with it wrapper types around the kube object) which would not adhere to the design principle of simplicity.

* The control-plane and proxies are all annotated with the current version in use (which is taken from the associated `linkerd-config` configMap), validating the new version would again be quite simple to do once the version can be represented by the custom resource.


### 2. Observing the state of the cluster
---

* In my opinion, this is the harder part of the RFC. When observing the state of the cluster, there needs to be care around how different objects with different statuses are handled. A new custom resource should be created for each new version; when a new custom resource is created (or updated), the operator will automatically pick it up, validate it, update the global version to be used by the control plane and proxies, and restart the pods. `prior-art 1` suggests the source of truth in this scenario should be the configmap, another suggestion would be to have the new custom resource as the new source of truth for versioning -- this means the custom resource will be registered (and created) as part of the install process. The latter is a worse approach, since it would force all adopters into using the operator, in my mind the adoption of the operator should be voluntary.

* *Interactions with other features*: this would interact with the configmap object and Kubernetes objects (pods). Kubernetes objects will need to be registered and cached in order to be operated on (e.g restarting pods), the `linkerd-config` configMap will also be edited (depending on the approach taken).

* Regardless how the version custom resource will be created, it will need to be registered and cached by the operator.

### 3. Updating the control-plane
---

* When the new version is picked up, provided it is newer than the currently used version, to update the control-plane becomes trivial. Restart the pods which will pull down the new proxy img (depending on what the source of truth is, this means getting the version from the custom resource itself or from the configMap).
* The main difficulty here is judging whether the version is newer or not, there needs to be a mechanism in place to validate that the version is correct.

* **Edge cases**: consider for example that in the implementation, on a create/update event of the custom resource, the version is assigned is read and then assigned to the configMap. If a previous custom resource would be updated, this will consequently mean upgrading to an inferior version. Since controllers should be stateless, one way around it would be to cache the most recent versions on start-up (by polling the API), have logic based on the `creation_ts` field of the resource spec (e.g if newer, replace, otherwise ignore it), or take the `prior-art 1` suggestion and use an `active` field (this in turns means deactivating/deleting all previous custom resources).


#### Goals:
1. Provide a functioning piece of software that is simple and integrates with the Linkerd ecosystem by the agreed target date.
2. Extend the functionality of Linkerd to pave the way for new ideas and features that can piggyback on this implementation.
3. Build an efficient operator that is up to the Linkerd [standards](https://linkerd.io/2/faq/#how-fast-is-linkerd)

#### Deliverables:
* An operator that actively watches for new released version and updates the control-plane accordingly;
* Option to opt-in to auto-updates through the use of annotations/labels -- this would adhere to the design principles and also allow for extended functionality;
* Rigurous validation to ensure nothing will break as a result of automating upgrades;
* Rolling updates of proxies (leveraging Kubernetes' main advantages).

# Prior art

1. [PoC on this issue](https://github.com/ihcsim/linkerd-config) -- [ref](https://github.com/linkerd/gsoc/pull/5)
---
* The PoC implements this issue the way I thought I would, but in a different way. I have heard of `Kubebuilder` before but I have never personally used it. In my own implementation of an operator, I built my own cache (see `prior-art 2`), generated the clientset for my custom resource using the `K8s code-generator` (which was not very straight-forward to work with) and created wrapper objects for the Kubernetes resources in order to preserve the information that I needed. I think the approach taken here is much cleaner and easier to understand, less complex and less of a nightmare to manage and debug. The `README.md` highlights future work as well as current worries, I'll explore them here.
    - `ownerReference`: once we update the `linkerd-config` configMap, the owner reference will also be changed. This means the configMap is now owned by the custom resource, from my research it seems that this is the gc mechanism for `K8s`, it is stated [here](https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/#controlling-how-the-garbage-collector-deletes-dependents) that garabe collection can be automatic, or manual. In this case, we can opt in for the configMap to be orphaned should the custom resource be deleted. This would in my mind fix the problem.
    - `mTLS`: again, it is underlined that that the custom resource can be used to manage the configMap, I think this is not a good option, as previously stated, I think the operator should be opt in, and tying adopters to its implementation might "raise some eyebrows". The alternative would be to automatically inject the trust in the resource on its creation. If we create the resource programatically (i.e NOT through a `cronJob`) then we can inject the trust the same way it is being done in the `linkerd check` command (provided the operator will have the right RBAC policies in place).
    - multiple versions: this was tackled in the last paragraph of the design proposal, the most recent resource might not be the most recent version. Since this is meant to auto-update, it should target the most recent version (if adopters would like to rollback, assumption is they can do so manually). There can be an `active` boolean field, on creation of the resource we can also pass in the date of the release, or cache the most recent version on start-up and use that as the source of truth when it comes to which version should be the most recent.


2. [kube-batch](https://github.com/kubernetes-sigs/kube-batch)
---
* `kube-batch` used the caching design previously mentioned. It makes use of the `K8s code-gen`, maintains a cache of the system (complete with informers), every time an informer picks a CRUD event, the `K8s` object is converted to the wrapper objects and added to the cache through a set of handlers. I have done something similar in a personal project. The default `kube-scheduler` also works similarily. I liked the approach taken, however, I find that it might be a bit too much for the task at hand. If anything, it should serve as inspiration on a different alternative to dealing with resources, caching and reconciling state in Kubernetes.

3. [Coreos update operator](https://github.com/coreos/container-linux-update-operator/tree/master/pkg/operator)
---

* Similar fashion as `prior-art 1`. The component of interest is the `update-operator` (the other component runs as a daemon). This is interesting to me from the point of view of leader election, although I doubt the Linkerd Auto Update operator will need more than one replica at any point.


# Unresolved questions

1. What will happen when a custom resource is deleted? I am unsure if there should be any validation done, any action taken, etc.
2. Managing custom resources with version clash -- there are a few ways I thought about doing this, I am curious to see what the rest of the community/mentorship thinks.
3. Should updating/managing the configMap be transactional? What happens if the operator fails partly through, will it affect anything? Should we deep copy the configMap and create a new one? This is all related to the trickiness of managing the state and statuses in Kubernetes as I have previously mentioned. If this is not an issue, please disregard.
4. `prior-art 1` asserts: "Currently, the controller relies on `time.Sleep()` to delay the restarting of the pods, so that the proxy-injector has time to pick up the updates to the `linkerd-config` configmap" -- agreed a better way to handle this must be found, not exactly sure what should be done in this instance though, a bit uncertain. Would like to discuss with other people.

- Will update as the RFC develops and feedback is given.

# Future possibilities

- Work in progress, for now. 