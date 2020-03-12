- Contribution Name: `Alert Manager Integration`
- Implementation Owner: @christyjacob4
- Start Date: 2020-03-03
- Target Date: 
- RFC PR: [linkerd/gsoc#0000](https://github.com/linkerd/gsoc/pull/0000)
- *Linkerd* Issue: [linkerd/linkerd2#1726](https://github.com/linkerd/linkerd2/issues/1726)
- Reviewers: 

# Summary

[summary]: #summary

*Linkerd* uses Prometheus for collecting metrics from various endpoints. Out of the box, Prometheus is an open source monitoring solution that gathers time series based numerical data. Your services need to expose an endpoint (/metrics) from which Prometheus can scrape metrics. The Prometheus monitoring suite comes with tools that help enhance its capabilities like Grafana and AlertManager. The current implementation of the Control Plane makes use of Prometheus for scraping endpoints and aggregating metrics.

The primary goal of this project is to integrate the Alertmanager into the current Control Plane so that Prometheus can provide out of the box alerts to the preferred channels.
AlertManager can be used for grouping alerts, silencing alerts, routing and sending alerts to preferred channels like slack, emails, pagerduty etc. 
This will allow users and teams to keep track of critical events that occur on their deployments with ease. 

# Problem Statement (Step 1)

[problem-statement]: #problem-statement

- By default, the *Linkerd* Control Plane doesn't have an alerting mechanism. Prometheus gathers all the metrics which can be visualised through Grafana, but in order to send these alerts to channels, we need to integrate Alertmanager with Prometheus as part of the control plane.

* Once this RFC is implemented, users will have the option to install **Alertmanager** in the control plane using the `addon-config` flag in `linkerd install`


- The optional **Alertmanager** installation will come with some default alerts
  which users can configure to appropriate channels like slack, email, pagerduty, wechat etc.

- Users will also be able to specify custom rules for Prometheus as part of the configuration for `--addon-config` in `config.yaml`.

- Integration with Service Profiles will allow users to define rules and alerts for per route metrics providing extremely fine grain alerting mechanisms for their apps.

A POC for this has been done in [linkerd/linkerd2#1726](https://github.com/linkerd/linkerd2/pull/4124)

# Design proposal (Step 2)

The Integration of Alertmanager with the Existing Control Plane is a multi step process. It can be divided into the following stages

---
- ## Providing an option to include AlertManager in the installation
The *Linkerd* install command is currently used to generate a helm chart based on various flags and options that are passed to it. The helm chart is then applied using `kubectl apply` .
With the recent addition of [**Tracing**](https://github.com/linkerd/linkerd2/pull/3955) to *Linkerd* installation as an add-on, we have a framework to add more modules as part of the Control Plane installation.

The plan is to pass a configuration file `config.yaml` in the `--addon-config` flag along with `linkerd install`, that would allow the user to install *Linkerd* with Alertmanager configured out-of-the-box, as an addon. So the complete command would look like this 

```sh
linkerd install --addon-config config.yaml | kubectl apply -f -
```

This will simply enable Alertmanager with a default, publicly available [**webhook receiver**](#webhook-configuration) and configure Prometheus with a few default rules.  

> This will enable the user to 
>  * Check if the default Prometheus rules are firing.  
>  * Check if Alertmanager is able to receive alerts from Prometheus.
>  * Validate if Alertmanager is able to forward alerts to receivers.

Additional configuration for a more specific Alertmanager install can be specified in the [`config.yaml`](#installation-yaml) file. This would contain all the configurations required to setup a full feldged Alertmanager install like enabling the default webhook flag, path to Prometheus files, path to Alertmanager Receivers file etc. The documentation for the `config.yaml` file is given [**here**](#installation-yaml).

For the Alertmanager to function properly, it should have the correct iptables and roles, this is done by the proxy injector and RBAC settings. More information about the implementation of the Alertmanager Helm chart can be found in this [**section**](#alertmanager-helm-chart). 

---
- ## Webhook configuration 

[Webhook.site](https://webhook.site/) is an open source project that allows you to instantly get a unique, random URL that you can use to test and debug Webhooks and HTTP requests. Their API is public, free to use and doesn’t require authentication.  

A unique webhook URL will be generated for the user during the `linkerd install` process and this URL will be configured as the default receiver in the Alertmanager configuration as mentioned in the previous step. 

This involves requesting a token by making a `POST` request to `https://webhook.site/token` with the following parameters.
```json
{
  "default_status": 200,
  "default_content": "Hello world!",
  "default_content_type": "text/html",
  "timeout": 0
}
```
The response contains a **uuid**, which can be used in conjunction with the base url, to access the webhook endpoint at https://webhook.site/{token.uuid}
```json
{
  "redirect": false,
  "alias": null,
  "timeout": 0,
  "premium": true,
  "uuid": "9981f9f4-657a-4ebf-be7c-1915bedd4775",
  "ip": "127.0.0.1",
  "user_agent": "Paw\/3.1.8 (Macintosh; OS X\/10.14.6) GCDHTTPRequest",
  "default_content": "Hello world!",
  "default_status": 200,
  "default_content_type": "text\/plain",
  "premium_expires_at": "2019-10-22 10:52:20",
  "created_at": "2019-09-22 10:52:20",
  "updated_at": "2019-09-22 10:52:20"
}
```

An Exmaple of alerts that were triggered from Prometheus (as part of my POC) can be found [here](https://webhook.site/#!/c21467b9-7ce4-4389-998b-8d91578433f3/ff66d8b1-c206-49a7-b5f8-b44af28a022f/1)

> Privacy can be a concern here as some users may not want their private alerts to go to a public webhook. The default behaviour can be easily overridden using the `config.yaml` file passed to the `--addon-config` flag as shown below

```yaml
publicWebhook: false 
...
Additional Configuration Items.
...
```

---
- ## Installation YAML

The user can pass a yaml file along with the `--addon-config` flag to override the default settings we provide.
```sh
linkerd install --addon-config config.yaml | kubectl apply -f -
```

The yaml file will be unmarshalled during installation and will overwrite the default values in `values.yaml`

The template of the `yaml` file can be defined as follows 

```yaml
# Flag to enable/disable sending alerts to Webhook.site
publicWebhook: true [ default = false ]

# This section contains configuration items specific to Alertmanager addon component
alertmanager:

  # Name of the Alertmanager component, re-used across helm chats 
  name: alertmanager [ default = alertmanager ]

  # Flag to determine if Alertmanager should be enabled/disabled.
  enabled: true [ default = false ]

  # Image name of a stable release of Alertmanager, that works best with the current version of Prometheus
  image: prom/alertmanager:v0.19.0 [ default = prom/alertmanager:v0.19.0 ]

  # Path to the file containing the configMap for Alertmanager Deployment
  config: path/to/config/file.yaml [ default = "" ]

# This section contains configuration items specific to Prometheus component
prometheus:

  # Path to file containing Prometheus Rules
  rules: path/to/rules/file.yaml [ default = "" ]

  # Path to the swagger spec file that will be used to generate a service profile and provide per route metrics
  swaggerSpecPath: path/to/swagger/spec [ default = "" ]

  # Path to the protobuf spec file that will be used to generate a service profile and provide per route metrics
  protobufSpecPath: path/to/protobuf/spec [ default = "" ]
```


---
- ## Prometheus Helm Chart
The Alertmanager installation will come out of the box with some Prometheus rules, and having a `config.yaml` will allow the user to override the default rules. 

The rule file that is mentioned in `config.yaml` will need to be read, unmarshalled and then embedded in the Prometheus ConfigMap. 

We need to inform Prometheus about the existence of a Alertmanager service and this is done by adding an alerting mechanism in the Prometheus ConfigMap in `charts/linkerd2/templates/prometheus.yaml` file 

```yaml
alerting:
  alertmanagers:
  - scheme: http
    static_configs:
    - targets: ['{{.Values.alertmanager.name}}:9093']
```

As part of `linkerd install` we can also check the syntax of the Prometheus Rules that are passed in the `config.yaml` file using a [**utility**](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/#syntax-checking-rules) that Prometheus provides. This will allow us to gracefully exit in case there are syntax errors in the Prometheus rules.  

The Prometheus rules can be [reloaded](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/#configuring-rules) at runtime by sending a `SIGHUP` to the Prometheus process. This feature could be exploited to allow the user to modify the rules during runtime. An example use case could be when we add the support for changing these rules from the *Linkerd* dashboard.

Information about the default rules can be found in the next section.

---
- ## Default Rules

Rules for Prometheus are written using [PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/).
By default, we can have rules for the following necessary conditions.

1. Error Reloading Prometheus Configuration
```yaml
- alert: ErrorReloadingPrometheusConfiguration
  expr: prometheus_config_last_reload_successful != 1
  for: 3m
  labels:
    severity: error
  annotations:
    summary: "Error Reloading Prometheus Configuration"
    description: "Prometheus configuration reload (instance {{ $labels.instance }})"
```
2. Prometheus unable to connect to Alertmanager 
```yaml
- alert: PrometheusUnableToConnectToAlertmanager
  expr: prometheus_notifications_alertmanagers_discovered < 1
  for: 3m
  labels:
    severity: error
  annotations:
    summary: "Prometheus unable to connect to Alertmanager"
    description: "Prometheus not connected to alertmanager (instance {{ $labels.instance }})"
```
3.  Error Reloading AlertManager Configuration
```yaml
- alert: ErrorReloadingAlertManagerConfiguration
  expr: alertmanager_config_last_reload_successful != 1
  for: 5m
  labels:
    severity: error
  annotations:
    summary: "Error Reloading AlertManager Configuration"
    description: "AlertManager configuration reload (instance {{ $labels.instance }})"
```
#### Other Conditions that may need default alerts include:
- #### Unusual network traffic In/Out
- #### Unusual Disk read/write rate
- #### Node running out of memory (only x % Memory Available).
- #### Node out of disk space (only x % disk space left).
- #### High CPU Load
- #### Too many Context Switches
- #### Swap memory filling up
- #### Success Rate on a particular route is below 90%
- #### RPS on an endpoint is going too high (endpoint abuse)
- #### P50, P95 and P99 latency based alerts 
---
- ## Alertmanager Helm Chart
The Alertmanager helm chart will be implemented in a way similar to other *Linkerd* control plane elements. 

The Alertmanmager config from `config.yaml` file will be unmarshalled and embedded into the Alertmanager Helm chart as seen below.

```yaml
---
###
### Alert Manager
###
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: linkerd-{{.Values.alertmanager.name}}-config
  namespace: {{.Values.global.namespace}}
data:
  alertmanager.yml: |-
    <include contents of the configuration file>
---
kind: Service
apiVersion: v1
metadata:
  name: {{.Values.alertmanager.name}}
  namespace: {{.Values.global.namespace}}
  labels:
    {{.Values.global.controllerComponentLabel}}: {{.Values.alertmanager.name}}
    {{.Values.global.controllerNamespaceLabel}}: {{.Values.global.namespace}}
  annotations:
    {{.Values.global.createdByAnnotation}}: {{default (printf "linkerd/helm %s" .Values.global.linkerdVersion) .Values.global.cliVersion}}
spec:
  type: ClusterIP
  selector:
    {{.Values.global.controllerComponentLabel}}: {{.Values.alertmanager.name}}
  ports:
  - name: admin-http
    port: 9093
    targetPort: 9093
---
{{ if empty .Values.global.proxy.image.version -}}
{{ $_ := set .Values.global.proxy.image "Version" .Values.global.linkerdVersion -}}
{{ end -}}
{{ $_ := set .Values.global.proxy "workloadKind" "deployment" -}}
{{ $_ := set .Values.global.proxy "component" "linkerd-alertmanager" -}}
{{ include "linkerd.proxy.validation" .Values.global.proxy -}}
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    {{.Values.global.createdByAnnotation}}: {{default (printf "linkerd/helm %s" .Values.global.linkerdVersion) .Values.global.cliVersion}}
  labels:
    app.kubernetes.io/name: {{.Values.alertmanager.name}}
    app.kubernetes.io/part-of: Linkerd
    app.kubernetes.io/version: {{default .Values.global.linkerdVersion .Values.controllerImageVersion}}
    {{.Values.global.controllerComponentLabel}}: {{.Values.alertmanager.name}}
    {{.Values.global.controllerNamespaceLabel}}: {{.Values.global.namespace}}
  name: linkerd-{{.Values.alertmanager.name}}
  namespace: {{.Values.global.namespace}}
spec:
  replicas: 1
  selector:
    matchLabels:
      {{.Values.global.controllerComponentLabel}}: {{.Values.alertmanager.name}}
      {{.Values.global.controllerNamespaceLabel}}: {{.Values.global.namespace}}
      {{- include "partials.proxy.labels" .Values.global.proxy | nindent 6}}
  template:
    metadata:
      annotations:
        {{.Values.global.createdByAnnotation}}: {{default (printf "linkerd/helm %s" .Values.global.linkerdVersion) .Values.global.cliVersion}}
        {{- include "partials.proxy.annotations" .Values.global.proxy| nindent 8}}
      labels:
        {{.Values.global.controllerComponentLabel}}: {{.Values.alertmanager.name}}
        {{.Values.global.controllerNamespaceLabel}}: {{.Values.global.namespace}}
        {{- include "partials.proxy.labels" .Values.global.proxy | nindent 8}}
    spec:
      {{- include "linkerd.node-selector" . | nindent 6 }}
      containers:
      - image: {{.Values.alertmanager.image}}
        imagePullPolicy: {{.Values.global.imagePullPolicy}}
        name: {{.Values.alertmanager.name}}
        ports:
        - containerPort: 9093
          name: admin-http
        volumeMounts:
        - mountPath: /etc/alertmanager
          name: {{.Values.alertmanager.name}}-config
      - {{- include "partials.proxy" . | indent 8 | trimPrefix (repeat 7 " ") }}
      {{ if not .Values.global.cniEnabled -}}
      initContainers:
      - {{- include "partials.proxy-init" . | indent 8 | trimPrefix (repeat 7 " ") }}
      {{ end -}}
      serviceAccountName: linkerd-{{.Values.alertmanager.name}}
      volumes:
      - configMap:
          name: linkerd-{{.Values.alertmanager.name}}-config
        name: {{.Values.alertmanager.name}}-config
      - {{- include "partials.proxy.volumes.identity" . | indent 8 | trimPrefix (repeat 7 " ") }}

```
The `initContianers` is added here in a way similar to other *Linkerd* Components that configures iptables to forward the traffic correctly.

Since the Alertmanager config can contain sensitive information like passwords for different channels, the proposal is to use [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) to store the ConfigMap. 

Apart from this, the roles need to be defined for Alertmanager which is done using an RBAC file 

```yaml
---
###
### linkerd-alertmanager RBAC
###
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: linkerd-{{.Values.global.namespace}}-{{.Values.alertmanager.name}}
  labels:
    {{.Values.global.controllerComponentLabel}}: {{.Values.alertmanager.name}}
    {{.Values.global.controllerNamespaceLabel}}: {{.Values.global.namespace}}
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/proxy", "pods"]
  verbs: ["get", "list", "watch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: linkerd-{{.Values.global.namespace}}-{{.Values.alertmanager.name}}
  labels:
    {{.Values.global.controllerComponentLabel}}: {{.Values.alertmanager.name}}
    {{.Values.global.controllerNamespaceLabel}}: {{.Values.global.namespace}}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: linkerd-{{.Values.global.namespace}}-{{.Values.alertmanager.name}}
subjects:
- kind: ServiceAccount
  name: linkerd-{{.Values.alertmanager.name}}
  namespace: {{.Values.global.namespace}}
---
kind: ServiceAccount
apiVersion: v1
metadata:
  name: linkerd-{{.Values.alertmanager.name}}
  namespace: {{.Values.global.namespace}}
  labels:
    {{.Values.global.controllerComponentLabel}}: {{.Values.alertmanager.name}}
    {{.Values.global.controllerNamespaceLabel}}: {{.Values.global.namespace}}
```

---
- ## Service Profile Integration 

Service Profiles is an extremely useful *Linkerd* Feature that enables a developer to get metrics on a per route basis. A Service profile can be generated by *Linkerd* using the `linkerd profile` command and passed onto `kubectl apply`. 

`linkerd profile` can obtain route information in 4 different ways:
- Swagger
- Protobuf
- Auto-creation
- Template

Hence based on the availability of either of these specifications, we can derive per route metrics and create default Prometheus rules for the same. The `config.yaml` file will allow the specification of either a swagger file or a protobuf file. 
```yaml
publicWebhook: false
alertmanager:
  config: path/to/config/file.yaml 

prometheus:
  rules: path/to/rules/file.yaml
  swaggerSpecPath: path/to/swagger/spec
  protobufSpecPath: path/to/protobuf/spec
```
In the event that both of these are absent, we will revert to the [Auto-Creation](https://linkerd.io/2/tasks/setting-up-service-profiles/#auto-creation) method where we watch live traffic and generate a service profile from the `tap` data.

The information from the Service Profile will allow us to define per route metris and thus per route alerts can be configured out of the box.

As a starting point, this can easily be tested using *Linkerd's* own test apps [Emojivoto](https://github.com/BuoyantIO/emojivoto) and [BooksApp](https://github.com/BuoyantIO/booksapp). 

---
- ## Linkerd Check
The `linkerd check` command currently has a check to ensure that Prometheus can talk to the control Plane
```sh
√ [prometheus] control plane can talk to Prometheus
```
A similar check can be added to ensure that the Alertmanager Addon is functional and working as expected. The check will ensure two things
- Control Plane can talk to Alertmanager.
- Prometheus can talk to Alertmanager.

```sh
√ [alertmanager] control plane can talk to Alertmanager
√ [alertmanager] Prometheus can talk to Alertmanager
```

# Deliverables
- Ability to [install](#providing-an-option-to-include-AlertManager-in-the-installation) Alertmanager as an addon as part of the Control plane. 
- [Default](#default-rules) out-of-the-box alerts. 
- [Checks](#linkerd-check), to see that Alertmanager has been successfully installed and configured correctly
- Simple configuration of [alert channels](#alertmanager-helm-chart) (slack, email).
- [Service profile Integration](#Service-Profile-Integration) to provide per route alerts.
- [Default Webhook](#Webhook-configuration) to quickly test if the app is running. 
- Test cases for checking the functionalities of the `alertmanager` flag. 
- Prometheus Rule file that will be syntax checked using [Prometheus Syntax Checker](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/#syntax-checking-rules)
- Configure Alertmanager for [High Availability](https://prometheus.io/docs/alerting/alertmanager/#high-availability) Mode.

# Non Goals
- Alertmanager should not impact the performance of *Linkerd*.
- Alertmanager should not be a point of failure for any of the *Linkerd* Components.
- Alertmanager should not impact the the functioning of any of the *Linkerd* features.

# Prior art

[prior-art]: #prior-art

- Service Profile Integration in https://linkerd.io/2/tasks/books/#service-profiles
- [Grafana Alerts](https://grafana.com/docs/grafana/latest/alerting/notifications/) vs Alertmanager <br>
Grafana v4.0 onwards has support for basic Alerting features. I'd like to contrast the two , to show why Alertmanager is a better option. 
  - Grafana's alerting is focused on providing a simple UI, and it's features are pretty basic. Only single metric based, simple threshold alerts are supported.
  - Alertmanager can do real high availability
    - Alertmanager/Prometheus communicates alerts with a protocol specifically engineered for high availability
    - Gossip protocol for deduplication with multiple instances 
  - Alertmanager can deduplicate, group, silence & inhibit alerts
  - Alertmanager offers separation of concerns. In case of Grafana, when your dashboard goes down, your alerting goes down too, but not with Alertmanager
  - Alertmanager has its configuration in a config file which is much easier to operate, version, apply than a local sqlite database that has to be kept in sync if you have multiple instances of Grafana
- Prometheus is a full monitoring and trending system that includes built-in and active scraping, storing, querying, graphing, and alerting. [Graphite](https://graphiteapp.org/) is a passive time series logging and graphing tool. Other concerns like scraping and alerting, are addressed by [external components](https://graphite.readthedocs.io/en/latest/tools.html).


- [InfluxDB](https://www.influxdata.com/) with [Kapacitor](https://github.com/influxdata/kapacitor) is the closest comparison with Prometheus and Alertnanager. However, Prometheus offers a more powerful query language for graphing and alerting. The Prometheus Alertmanager additionally offers grouping, deduplication and silencing functionality.

- Prometheus can provide a dimensional data model where metrics are identified by a metric name and tags with built-in storage, graphing and alerting whereas [Nagios](https://www.nagios.org/) is a legacy IT infrastructure monitoring tool with a focus on server, network, and application monitoring.
- Insights by other community members https://www.reddit.com/r/devops/comments/afqye3/whats_your_monitoring_and_alerting_stack_look_like/

# Timeline

This timeline is created because the **GSoC** programme recommends it, but I’ll try to move as fast as I can. If I can do better than this, I certainly will. But this timeline is to create a **minimum viable goal** for myself so I can judge myself as **successful** or not at the end of the programme.
I will also be writing a blog post every fortnight to share things I’ve learnt, my progress and what I plan to do next.

| Dates | Tasks |
| --- | --- |
| | *Community bonding period begins `( April 27th )`* |
| `April 27th -  May 4th` | Familiarise with the *Linkerd* team and fellow GSoC Students <br> Discussions about default Prometheus Rules and Alertmanager Receivers <br> Understand the per route metrics that need to be collected as part of Service Profile Integration |
| `May 4th -  May 11th` | Test the `linkerd profile` command extensively on Books and Emojivoto to understand the creation, working and usage of Service Profiles <br> Blog Post #1 |
| `May 11th -  May 18th` | Familiarise with other Linkerd components <br> Discuss about the project in detail and answer any remaining unresolved questions. |
| | *Community bonding period ends `( May 18th )`* |
| `May 18th -  May 25th` | Define Default Prometheus Rules and Alertmanager Receivers in `values.yaml` <br> Write wrapper functions to make API requests to Webhook.site to get the default webhook URL <br> Blog Post #2 |
| `May 25th -  June 1st` | Define Alertmanager Charts and Alertmanager RBAC yaml files <br> Enable Alertmanager as an addOn option using the default rules and receivers |
| `June 1st -  June 8th` | Define functions for reading the rules and config files and unmarshalling them from file paths passed in `config.yaml` <br> Blog Post #3 |
| `June 8th -  June 15th` |  Add support for `config.yaml` file to be passed to the `addon-config` flag. <br>  Perform extensive testing of the `config.yaml `functionality <br> |
| | *Phase #1 Evaluation Starts `( June 15th )`* |
| `June 15th - June 22nd` |  Write Integration Tests and define golden files for Alertmanager addOn functionality.<br> Add a check in `linkerd check` to ensure that Alertmanager has been successfully installed and correctly configured.<br> Blog Post #4 |
| `June 22nd -  June 29th` | Tests to ensure that `linkerd check` command is functional and update the golden files <br> Configure Alertmanager for High Availability Mode |
| `June 29th -  July 6th` | Extensive Testing of all the aforementioned functionality <br> Integration Test cases for Alertmanager in  High Availability mode <br> Blog Post #5 |
| `July 6th -  July 13th` |  Generate Service Profiles for apps with swagger spec, record their per-route metrics and define default Prometheus rules for them  |
| | *Phase #2 Evaluation Starts `( July 13th )`* |
| `July 13th -  July 20th` | Generate Service Profiles for apps with protobuf spec, record their per-route metrics and define default Prometheus rules for them <br> Blog Post #6 |
| `July 20th -  July 27th` | Auto generate Service Profiles for apps without swagger spec or protobuf spec and record their per-route metrics and define default Prometheus rules for them |
| `July 27th -  August 3rd` | Other Service Profile integration related action items  <br> Blog Post #7 |
| `August 3rd -  August 10th` | Integration Tests for Service Profile Integration <br> Buffer Time for any action items that may not have been completed |
| | *Final Evaluation Starts `( August 10th )`* |
| `August 10th -  August 17th` |   Discuss changes for allowing write access from Linkerd Dashboard <br> Design UI components to support Add/Modify/Delete Prometheus Rules and Alertmanager receivers <br> Blog Post #8 |



# Unresolved questions

[unresolved-questions]: #unresolved-questions

- I understand that `linkerd profile` can be used to generate Service Profiles based on protobufs, swagger specs, or by using tap to observe the web service. I would like to understand the kind of integration that is expected as part of [Out of the box alerts #1726](https://github.com/linkerd/linkerd2/issues/1726) . Does it mean that the `linkerd profile` command must be automatically run if the user installs Alertmanager, and the generated Service Profile needs to be part of the Alertmanager rules? 

- Should Alertmanager be installed as an Addon module like Trace [Tracing Add-on For Linkerd #3955](https://github.com/linkerd/linkerd2/pull/3955) or should it be a stand alone component like Prometheus, Grafana etc. ?

# Future possibilities

[future-possibilities]: #future-possibilities

- If time permits, work on the identity and RBAC for *Linkerd* web dashboard which will allow modifying the Prometheus rules and Alert Manager Configs directly from the **Linkerd Dashboard**.
- Allow users to add, delete and modify Prometheus rules using the **Linkerd Dashboard** and inject the new rules into the Prometheus deployment, similar to how `linkerd inject` command works.
- Allow users to add, delete and modify Alertmanager receivers and also configure them using **Linkerd Dashboard** and inject them.
- Handling the `linkerd upgrade` scenario and ability to upgrade Alertmanager as part of `linkerd upgrade`.