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

## Interaction with other features
The e2e conformance tests proposed would be carried out outside the Linkerd CI. These tests would make use of the kubectl and linkerd binaries to interact with the cluster and with various features Linkerd can perform. The testing process can be configured easily using a .yaml definition file (more details explained under “implementation details”). Such an approach gives the users the flexibility in diagnosing the state of their cluster w.r.t. Linkerd. 

This project also proposes an optional sample distributed application - “MovieChat” that can exercise various features of Linkerd and also surface issues regarding various APIs, protocols and topologies such as streaming, websockets, MySQL and Redis. Currently, Linkerd does not have a sample application that implements these and would benefit from testing for conformance. We initially start off by modifying the existing emojivoto application to have feature flags to enable/disable features such as MySQL, Redis, gRPC, websockets, etc. This is important because emojivoto is heavily used in the getting started process.

## Implementation Details
The test suite shall be executed as a job on the cluster using [Sonobuoy](https://sonobuoy.io/). It is a great tool when it comes to conformance validation, and its [plugin model](https://sonobuoy.io/docs/master/plugins/) can allow us to easily extend functionality beyond k8s conformance validtion. Sonobuoy also saves developers the overhead of configuring the desired objects such as Namespaces, ServieAccounts, ClusterRoleBindings, Jobs, RBAC, etc.

In the Dockerfile, we define that the following must happen in the container running as a job:
- Setup and install all the binaries/tools required by the testing environment - linkerd, kubectl, mysql, etc.
- Copy the test configuration file conformance_config.yaml which is responsible for describing how the tests must run (see below for implementation details).
- Copy _run.sh_ file that initiates the tests using the go test cmd.
- Copy the test suite
- ENTRYPOINT / CMD that executes _./run.sh_

> Note: When generating a sonobuoy plugin, the plugin definition file sets the start-up command as ./run.sh by default. If we set the container’s ENTRYPOINT/CMD as go test, this may get overridden to ./run.sh because of how sonobuoy generates plugins. The generated plugin definition file may however be modified to not override the startup command of the container, but this may have a negative impact on the user experience.

In order to run the sonobuoy plugin, users/testers shall be required to carry out the following steps:
```bash
# setup docker registry
$ export USER=<your public registry>
$ docker build . -t $USER/progress-reporter:v0.1
$ docker push $USER/progress-reporter:v0.1

# configure plugin
$ sonobuoy gen plugin \
--name=l5d-conformance \
--image $USER/l5d-conformance:v0.1 > l5d-conformance.yaml

# run plugin
$ sonobuoy run --plugin l5d-conformance.yaml

# collect output
outfile=$(sonobuoy retrieve) && \
mkdir results && tar -xf $outfile -C results
```

**Bonus**

If the users do not want to do any of this, we can provide a default docker image with some predefined test configurations - Eg : _gcr.io/linkerd.io/conformance:v1.0.0_, and also host a plugin definition file that makes use of this image - _https://run.linkerd.io/path/to/plugin.yaml_. This approach is great if users quickly want to get the conformance validation tool up and running.

The users would then simply have to run :
```bash
$ sonobuoy run --plugin https://run.linkerd.io/path/to/plugin.yaml
```

Users shall also be allowed to run these tests without having to depend on Sonobuoy. Much like the [Kubernetes conformance validation process](https://github.com/kubernetes/kubernetes/tree/master/cluster/images/conformance) we can provide a yaml configuration file that can be used to run a pod that runs these tests on the cluster. This yaml configuration file is responsible for setting up things like namespace, service account, ClusterRoleBindings, the test runner pod, etc. ([See example](https://github.com/kubernetes/kubernetes/tree/master/cluster/images/conformance))

### Test Configuration
The conformance test suite proposed in this project shall be made customizable as per the users’ preferences, which they may mention in a .yaml file. Such an approach not only enhances user experience but can also give the testers greater insights on their cluster and Linkerd.

This configuration file shall be unmarshalled into an object during runtime so that tests may read the desired properties from it and can run accordingly. This feature gives users the flexibility of running tests according to their preference. Proposed structure : 
```yaml
installOptions:
    bools: <array>
        # linkerd install flags that are booleans
        # useful for install config like ha
    values: 
        # linkerd install flags that hold values
        # useful for properties like external certs, mtls
testOptions:
    # duration for which tests must wait for pods to come to running state
    wait: <int>
    
    # lookback time used for tracing
    tracingLookback: <String>
    
    # service used for tracing
    tracingService: <String>

    # array of test names that must be executed
    # If empty, run all tests
    enableTests: <array>

    # whether to inject MySQL pod 
    injectMySQL: <bool>                                                                     

    # whether to inject Redis pods
    injectRedis: <bool>
   
   # users may be allowed to run these tests on
   # services other than the default movieChat by 
   # mentioning the namespace in which they are deployed
   # However, protocol / topology related tests might fail
   # if running on non-default namespace
   # GOOD-TO-HAVE feature
   workloadNamespace: <String> 

   # RBAC configuration to be used if on GKE
   # If left empty, use default RBAC settings as
   # mentioned here
   gkeRbacConfig: <string>

```

This is great for things like setting installation options and additional test preferences such as which tests to run, wait time, lookback time for tracing, whether to inject MySQL / Redis pods, etc. An example configuration file :
```yaml
installOptions:
    bools:
        - control-plane-tracing
        - ha
        - disable-h2-upgrade
    values:
        identity-external-issuer: ./certs/ca.crt
        identity-issuer-certificate-file : ./certs/issuer.crt
        identity-issuer-key-file: issuer.key
testOptions:
    wait: 10000
    tracingLookback: 1h
    tracingService: nginx
    enableTests:
        - proxy-injection
        - tap
        - distributed-tracing
        - canary
        - ingress
    injectMySQL: true
    injectRedis: true
    workloadNamespace: moviechat
    gkeRbacConfig: ./path/to/config

```

### Testing Methodologies for various features
The conformance test suite is intended to be run against an installation of Linkerd. Passing this test suite would give the user the assurance that a given configuration of Linkerd works (as expected) with a given version of Kubernetes and its cluster configuration. Users may easily configure install options and test preference using the test configuration file provided.

This section includes write ups regarding testing methodologies for some primary features:

**1. Auomatic Proxy Injection**

Proxy injection process works by adding a `linkerd.io/inject: enabled` annotation to pod template / namespace. This triggers an admission webhook that injects the `linkerd-proxy` container to the targeted deployments. This process shall be tested in 7 steps:
- Check if `inject` command (with and without `--manual` flag) outputs the desired configuration with the appended annotation.
- Wait for the pods to come to a _Running_ state for a certain duration, after which the test shall fail.
- Check if the pods actually have the `linkerd-proxy` and the `linkerd-init` container
- Check if the linkerd-proxy container is reachable by sending a `GET` request to liveness/readiness probe - to ensure service discovery is functional.
- Check for any `SkipInject` errors in events
- Check if uninject command updates the annotation to `linkerd.io/inject: disabled`
- Wait for the pods to come to a _Running_ state for a certain duration, after which the test shall fail.
- Ensure that the pods no longer have the `linkerd-init` and `linkerd-proxy` containers.

**2. linkerd tap cmd**

This test will work by issuing the `linkerd tap` cmd on various resources of the sample application. The output returned shall be checked for any errors. For e.g - GKE may require extra RBAC configurations and permissions. Tap shall be tested by issuing a tap command using the `-o json` flag and performing checks on the obtained JSON :
- Check for gRPC status codes
- Check authority
- Check for tls settings
- Check httpStatus

**3. linkerd stat cmd**

- Check if prometheus pod(s) is/are available
- Check if controller pods are up and running
- Check if the application pods are up and running
- Issue stat cmd and validate output by checking for success rate and various latencies 
> Note: This test may have to be put on hold due to https://github.com/linkerd/linkerd2/pull/3693

**4. linkerd edges cmd**

- Check if application pods are up and running
- Check if application deployment has desired replicas
- Get application pods IPs
- Issue linkerd edge cmd
- Validate if the desired edges are returned

**5. linkerd routes cmd**
- Check if application pods are up and running
- Issue linkerd routes cmd
- Validate the returned output by counting instances of desired route substrings using `strings.Count`

**6. Distributed Tracing**
Much like the emojivoto application, our MovieChat application shall be configured with the OpenConsensus Agent library to support tracing information in the requests. The following checks must be carried out:
- Install the collector and check if it is up and running in the tracing namespace.
- Install jaeger and check if it is up and running in the tracing namespace.
- Confirm the status of MovieChat application - should be Running
- Configure the application to use `oc-collector.tracing:55678` address and check if it happened successfully using kubectl
- Check if Jaeger returns traces by making a `GET` request on the jaeger backend at `/api/traces` endpoint on port 16686. (lookback and service parameters shall be made configurable via CLI flags)

**7. Canary Release**
- Check and install flagger and wait until a certain duration for the status of the deployment to change to _Running_, after which the test fails.
- Check if the application is up and _Running_. We shall be making use of the web component to test Canary release.
- Configure and deploy a release. Example:
```yaml
apiVersion: flagger.app/v1alpha3
kind: Canary
metadata:
  name: web
  namespace: movie
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  service:
    port: 9898
  canaryAnalysis:
    interval: 10s
    threshold: 5
    stepWeight: 10
    metrics:
    - name: request-success-rate
      threshold: 99
      interval: 1m
```
- Check status of canary/web to ensure it is Initialized
- Trigger an update by changing the image of deployment/web
- Wait until status of canary/web is Succeeded
- Verify metrics by issuing stat cmd

**8. Ingress**

It is essential to test ingress to ensure that the ingress controller re-writes incoming headers to the internal service name. This process ensures that service discovery is working. This test shall include deploying various types of ingress controllers (Nginx, Traefik and Ambassador) :
- Check and install flagger and wait until a certain duration for the status of the deployment to change to “Running”, after which the test fails.
- Deploy an ingress control. An example from the docs:
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web-ingress
  namespace: emojivoto
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header l5d-dst-override $service_name.$namespace.svc.cluster.local:$service_port;
      grpc_set_header l5d-dst-override $service_name.$namespace.svc.cluster.local:$service_port;

spec:
  rules:
  - host: example.com
    http:
      paths:
      - backend:
          serviceName: web-svc
          servicePort: 8080
```
The configuration may change according to the application.

- Issue a curl cmd on the external IP to check if desired response is returned. This shall be done by wrapping the curl bash cmd as a method in Golang using `command.exec()`.

### Protocol related tests
It is important to note that these tests could fail if the workloadNamespace property of the test config is set to anything other than the default (moviechat/emojivoto) application.

**1. gRPC Streaming**

The movie chat application or emojivoto shall leverage gRPC streaming. It is essential to ensure that streaming services are not affected by Linkerd - under normal conditions as well as during stress tests. To test gRPC streaming, we shall make use of a curl replacement for gRPC called grpcurl (https://github.com/fullstorydev/grpcurl). grpcurl is a command-line utility like curl that can easily invoke unary as well as streaming methods in gRPC services. For example :
```bash
grpcurl -plaintext <url>:<port> ServiceName.MethodName
```

The above cmd simply sends an empty request to the gRPC service, invokes MethodName method and outputs the response. 

Similarly, streaming can be tested by running:
```
grpcurl -plaintext <url>:<port> -d @ ServiceName.StreamingMethod
```

grpcurl accepts a `-d` flag to pass the request body. Using `@` as the request body allows us to test streaming interactively by using stdin as the request body. 
This shall be done by wrapping the grpcurl bash cmd as a method in Golang using `command.exec()`
Assertion can easily be performed on the output fetched by running a grpcurl cmd.

**2. Websockets**

Similar to gRPC streaming, it is essential to check if Linkerd does not break services that utilise websocket connections. In our sample application, we create a chat server that handles socket connections to and fro from the web application. To test this, we can ping the websocket endpoints using curl. For example:

```bash
curl --include --no-buffer \
    --header "Connection: close" \
    --header "Upgrade: websocket" \
    http://localhost:3000/socket
```

This cmd would return a response from the websocket server from the /socket endpoint where websockets shall be served.

**3. MySQL and Redis***

Port-forwarding can be initiated on the pods serving these applications. Once done, the tests shall make sample queries to these to ensure that appropriate output is returned. Additionally, users may be allowed to mention if these pods must be injected or not in the test configuration file.

### MovieChat App
The plan is to initially start off by modifying the emojivoto app to have feature flags that enable/disable features required from a conformance validation perspective. As we develop these tests, new features must be added to emojivoto as required. 

If time permits, the we shall develop a sample application MovieChat as proposed below: 

<img src="https://i.imgur.com/3GBKOQC.jpg"/>

The application mainly consists of 6 different components / microservices:

1. Web - responsible for providing a web based UI for users to interact with, a HTTP Server and a gRPC Client . The web UI client allows the users to:
    - Send a Movie ID request to the gRPC server and get back a MovieName Response. [Unary gRPC request]
    - Send a request to the gRPC server to get a stream of all movie names stored in the database. [Server Streaming]
    - Send a stream of MovieID and MovieName and store them in the database. [Client Streaming]
    - Send a stream of MovieIDs and receive a stream of MovieName [bi-directional streaming]
    - Chat with other users about their favourite movies! [Redis, websockets]
2. Movie API - a gRPC server responsible for querying the MySQL database to store and retrieve movie data. 
3. Chat server - handles socket connections to update chat UI in real-time.
4. Redis - Saves the chat history so that the chat server can render it on the web UI. This is refreshed at regular intervals.
5. MySQL - responsible for storing Movie related data.
6. Traffic generator - responsible for interacting with the web to simulate various features.

The k8s configuration files shall be managed using kustomize. Users shall be allowed to fire up this application via docker-compose as well.

## Corner Cases

It is possible to have prior knowledge regarding which use-cases may need to be configured differently depending on the environment. For eg, tap requires extra RBAC configurations on GKE. Instead of having the corresponding tests for tap failing, users may be allowed to explicitly mention the environment these tests shall run on, so that test cases may take a different path, i.e the tests may be programmed conditionally. For eg, In case of GKE, the tests may take an additional step that sets up RBAC and verifies them before issuing a tap cmd . This would give the users more insights and would have a good impact on UX.
Similarly, any known corner cases related to the infrastructure of the cluster can be mapped to the way the test suites must be written.
  
Users may be allowed to mention such specifications in the test configuration file as mentioned above.

## Dependencies on tools / libraries 

**External dependencies (We use these tools as a wrapper for our tests so users may interact with it):**
- Docker
- Sonobuoy (check Unresolved questions)

**Internal dependencies (tools required inside the container environment):**
- Kubectl
- Linkerd
- MySQL
- Redis
- curl
- grpcurl

**Tools and libraries required for writing the tests:**

Writing the tests using golang seems a great fit for our use cases for 3 reasons:
- Managing output and assertion
- The k8s APIs and ecosystem built around go would make it simpler to write the logic
- Testing frameworks like ginkgo make it easier to bootstrap, organize and manage test suites semantically - this would improve contributor experience by preserving readability and maintainability of the test suite. Ginkgo also allows us to use custom reporters that may help us store the results of these tests.
Nevertheless, sticking to the standard testing framework also gives us great control over the flow and management of output / assertion. 

**Libraries/Modules required for MovieChat application:**

All microservices (except Web UI, which will be written using JavaScript/React) shall be written using golang. The following libraries may be required:
- [grpc/grpc-go](https://github.com/grpc/grpc-go)
- [net/http](https://golang.org/pkg/net/http/)
- [googollee/go-socket](https://github.com/googollee/go-socket.io)
- [go-redis/redis](https://github.com/go-redis/redis)
- [go-sql-driver/mysql](https://github.com/go-sql-driver/mysql)
- [go-yaml/yaml](https://github.com/go-yaml/yaml)

## Use Cases

This project introduces an extensive e2e test suite that validates a cluster by interacting with various features of Linkerd, mainly those that involve communicating with a data plane. The e2e tests for conformance shall carry out checks for the following: 
1. Automatic proxy injection
2. linkerd tap, routes, edges, stat cmds
3. Distributed tracing
4. Canary release
5. Ingress
6. gRPC streaming (client/server/bi-directional streaming and unary)
7. Websockets
8. MySQL
9. Redis

Before the test suite is run, the following actions must be carried out in sequence:
1. Check if kubectl binary is available
2. Check if linkerd binary is available
3. Check if connection to cluster can be established via kubectl
4. Check if linkerd control plane is installed
5. Install the sample application

Once the test suite is complete (success / failure ) all resources created for the test are destroyed if not immediately after each test.


## Goals
The goal of this project is to :
1. develop a sample application (that comprises microservices) that can exercise various features of Linkerd, mainly that involve the data plane. (from https://github.com/linkerd/linkerd2/issues/2079) This application shall also exercise various protocols and topologies that could benefit from deeper testing and surface issues around streaming, websockets, MySQL and Redis.
2. provide an extensive e2e test suite that can make use of this application and validate many properties of the cluster, to help a user test for conformance w.r.t Linkerd, i.e to check if the cluster has been configured to work correctly over a longer period of time.

## Non-goals
It is a non-goal for this project to provide an application that is expected to be a part of Linkerd application architecture (control / data plane). This project in no way shall directly affect the way Linkerd functions. Unlike the existing integration tests, the e2e tests for conformance in this project are not expected to be a part of the Linkerd CI workflow either. These tests shall run as a stand-alone component such that it can interact with a k8s cluster.

## Deliverables

### Must have (shall be completed by end of GSoC):
- An e2e test suite that can perform conformance validation for the following features:
    1. Automatic proxy injection
    2. Linkerd tap, stat, routes, edges cmd
    3. Distributed tracing
    4. Canary release
    5. Ingress
    6. gRPC streaming (client/server/bi-directional streaming and unary)
    7. Websockets
    8. MySQL
    9. Redis
- A sample emojivoto app with feature flags that can enable/disable features required for conformance vaidation.

### Good-to-have (if extra time remains, these shall be worked on):
- The conformance validation testing suite shall also cover the following features :
    1. Telemetry and metrics
    2. Getting route based metrics (Service profiles, retries, timeouts, fault injection)
    3.Checking if data plane proxies return 503
    4. mTLS
    5. Cluster security and introspection
    6. Data plane PSP
    7. Elastic search
- A new sample application - MovieChat - that has all the features required from conformance validation perspective


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

This project proposes that tests be written in golang mainly due to how easy it is to manage outputs and assertions, organize tests and maintain readability. Having the option to use client-go k8s API could be another plus point. However, shell script may be a good tool too to write these tests, but we should also consider factors such as contributor experience. Would love to have the community’s opinion regarding this.


# Future possibilities

[future-possibilities]: #future-possibilities

As a Linkerd user, it is important to check if Linkerd works as expected even outside of the emojivoto / any other sample application. Setting the workloadNamespace property in the test configuration file gives the users this flexibility :
- During runtime, the value for this property is picked up from the unmarshalled object, and a `config.linkerd.io/e2e: conformance` label is appended to the namespace config
- The tests then hit the k8s API with a label selector to annotate all objects (deployments) under this namespace with the `linkerd.io/inject: enable` annotation to trigger auto-injection.
- It is crucial for linkerd cmds like stat, route and edge to support label selectors, hence, a patch for this must be made.
> Note: stat recently started supporting -l label selector flag. A similar approach must be applied to route and edge.


