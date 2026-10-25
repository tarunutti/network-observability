# network-observability
The Network Observability Operator in OpenShift helps you understand what is happening with network traffic inside your Kubernetes/OpenShift cluster. It helps answer questions such as:
- Which application or pod is communicating with another pod?
- Which namespaces are generating the most network traffic?
- Is traffic flowing between two services as expected?
- Which pods are sending or receiving unusually high amounts of traffic?
- Is a connection being dropped, and where should you investigate?
- How does network traffic move between pods, nodes, and external endpoints?
Since you're working with OpenShift platform operations and GitOps, I'll explain this from both an architectural and implementation perspective, including the operator's components, packet-flow visibility, installation, configuration, security, troubleshooting, and production considerations.
1. What is Network Observability?
Imagine an OpenShift cluster with these applications:
Frontend namespace

frontend pod

HTTP traffic · TCP connections

OpenShift network

OVN-Kubernetes · nodes · pod networking

Service-to-service traffic

Backend namespace

backend pod and database


Network Observability

Collects network-flow information and makes it available for analysis through the OpenShift console's network topology and traffic views, with metrics and optional integrations.


Without network observability, you may know that an application is slow, but you might not know whether it is communicating with an unexpected destination, generating excessive traffic, or experiencing network drops.

With it, you can investigate the network relationships and traffic patterns among workloads.

One important distinction: network observability is not the same as application tracing. It can show that one workload communicates with another, but it does not automatically tell you which application function or code line caused a slow request.
2. What exactly is the Network Observability Operator?
The Network Observability Operator is a Red Hat OpenShift operator that deploys and manages components for collecting, processing, and presenting network-flow information.

It is commonly associated with eBPF-based network-flow collection using NetObserv, along with flow processing and visualization.

The operator manages the lifecycle of its components using Kubernetes custom resources (CRs). Rather than manually maintaining every collector and processing component, you declare the desired configuration, and the operator reconciles the deployment.

Its major responsibilities include:
- Collecting network flows from supported nodes and network paths.
- Enriching flows with Kubernetes metadata such as namespaces, pods, and workloads where available.
- Processing, filtering, and optionally sampling flow records.
- Exposing network-flow data through supported console views and metrics.
- Supporting optional integrations for exporting flow data to external systems.
It is primarily a network-flow observability tool, not a replacement for Prometheus, Loki, Tempo, or a network policy engine.
3. Architecture: how the components work together
For OpenShift 4.20, the main architecture is a pipeline managed by the FlowCollector custom resource. The documented pipeline uses an eBPF agent, a flow-processing component, and a storage or metrics backend. 

1. OpenShift workloads and nodes

Pods · Services · Nodes · Network interfaces

2. eBPF agent

DaemonSet · observes supported network events and produces flow records

3. Flowlogs-pipeline

Enriches records with Kubernetes metadata, filters flows and generates metrics

Loki

Stores queryable flow logs


Prometheus

Stores time-series metrics



OpenShift Console — Network Traffic

Topology · Traffic flows · Overview and charts


These are the logical stages; not every installation uses both Loki and Prometheus as flow-data destinations in the same way. Exact components and settings depend on the selected FlowCollector configuration.
Component-by-component explanation

1. eBPF agent

It runs as a DaemonSet, meaning the agent runs on the eligible nodes rather than as one central collector. eBPF lets the agent observe supported networking activity in the Linux kernel without requiring you to modify application code.

The agent creates network-flow records and can support additional features such as TCP round-trip-time measurements, DNS tracking, and packet-drop tracking, subject to configuration and support.

2. Flowlogs-pipeline

This is the processing stage. It receives flow records, enriches them with Kubernetes information, applies configured filtering, and generates metrics for supported network-flow dimensions.

For example, it can associate an IP address with a pod, namespace, or node when the required mapping is available.

3. Loki — optional flow-log storage

Loki can retain the network-flow records so you can query historical traffic. This is particularly useful when investigating whether a communication pattern changed before an incident.

Loki is not automatically installed simply because you install the Network Observability Operator. It must be deployed and configured when using the Loki integration.

4. Prometheus — flow metrics

Prometheus stores time-series metrics, such as aggregated bytes, packets, and flow counts for the configured dimensions. This is useful for dashboards, trends, and alerting.

Metrics provide efficient aggregation, while flow records provide more detailed connection-level context. Prometheus metrics alone do not replace detailed flow-log history.

5. Console plugin

The plugin integrates the network traffic interface into the OpenShift web console. It presents the collected data in topology, traffic-flow tables, and overview graphs.

If the operator's components are healthy but the console page is missing, investigate the console plugin deployment and its registration separately.

The architecture and the responsibilities of the main components are documented in the official OpenShift 4.20 guides. 


4. What is eBPF, and why does the operator use it?
eBPF (extended Berkeley Packet Filter) is a Linux kernel technology that allows verified programs to observe or act on selected kernel events.

In network observability, it allows supported network events to be observed without installing a packet-capture tool inside every application container.

Consider this example:
- Frontend pod: 10.128.2.15
- Backend pod: 10.129.3.21
- Destination port: 8080
- Protocol: TCP
The agent can collect relevant flow information, and the pipeline can enrich it with the associated Kubernetes workloads.

You may then see information similar to this in a traffic-flow view:
Field	Example
Source namespace	frontend
Source pod	frontend-7c8d9
Destination namespace	backend
Destination pod	backend-5f7d2
Destination port	8080
Protocol	TCP
Bytes / packets	Observed or aggregated counts
Flow timing	Based on the flow record

These values are illustrative, not actual data from your cluster.
Does it capture every packet?
Not necessarily. The agent has a configurable sampling interval.

For example, if the sampling interval is 50, the default configuration described in the OpenShift 4.20 documentation samples approximately one packet in 50. A value of 1 means sampling every packet for that sampling mechanism. Higher sampling intervals generally reduce resource consumption but may reduce the detail and accuracy of derived statistics. 

Also, network-flow records are not the same as a full packet capture. You should not assume that the operator stores entire packet payloads, HTTP request bodies, or complete application conversations.
5. What can you observe with the operator?
The following are the main capabilities to understand before implementing it.

A. Network topology

Shows communication relationships between namespaces, workloads, pods, services, and other supported network entities.

Example: Discover that a frontend namespace communicates with five backend namespaces, although the application design expects only two.



B. Traffic-flow analysis

Lets you inspect flows and filter by fields such as source/destination namespace, workload, address, port, protocol, and traffic direction when those fields are available.

Example: Investigate which workloads communicate with a database on TCP port 5432.



C. Traffic metrics

Helps identify traffic-volume changes and unusual trends using generated metrics and console graphs.

Example: Determine which namespace is responsible for a sudden increase in network bytes.



D. Network performance investigation

Optional features can expose TCP round-trip-time information, DNS activity, and packet drops. Feature requirements and availability depend on the OpenShift and Network Observability versions.



E. Network-policy investigation

Depending on enabled features and platform support, you can use flow information to investigate traffic affected by NetworkPolicy and supported OVN-Kubernetes network controls. Observing a flow does not, by itself, mean the operator enforces or replaces a policy.



These features and their supported configurations are covered in the OpenShift 4.20 Network Observability documentation. 

6. How is it different from your other observability tools?
Since you are learning Prometheus, Loki, Tempo, OpenTelemetry, and OpenShift Service Mesh, this distinction is important.
Tool	Primary question it answers
Network Observability	Which workloads are communicating, and what network traffic is flowing?
Prometheus	What is the value or trend of a metric over time?
Loki	What do the application or system logs say?
Tempo	Which services and operations participated in a distributed request trace?
OpenTelemetry Collector	How do we receive, process, and export telemetry from supported sources?
Service Mesh / Istio	How do we manage service-to-service traffic, security, and routing?
NetworkPolicy	Which pod-level network connections should be allowed or denied?

Example: an application is slow
Suppose your frontend takes eight seconds to load a page.
Step 1 — Network Observability

Check whether the frontend is communicating with the expected backend, whether traffic volume has changed, and whether available TCP latency or packet-drop data points to a network issue.



Step 2 — Loki

Search application logs for database timeouts, connection errors, retries, or slow-query messages.



Step 3 — Prometheus

Check CPU, memory, request latency metrics, error rates, pod restarts, and saturation.



Step 4 — Tempo

Inspect a distributed trace to identify whether the delay occurred in the frontend, backend, database call, or another downstream operation, assuming tracing instrumentation is configured.



These tools complement one another. A network-flow view does not prove that the network caused the eight-second delay, and TCP round-trip time is not the same as HTTP response latency.

7. How to install Network Observability on OpenShift 4.20
The following is a practical installation path. The exact operator channel and version should match your cluster's supported catalog and release.
Prerequisites
Before installing, verify:
- You have suitable cluster-admin privileges for the initial installation and configuration.
- OVN-Kubernetes is the cluster network plugin.
- Your cluster architecture and node operating systems meet the operator requirements.
- If you want persistent flow-log storage in Loki, a compatible Loki deployment is available. The OpenShift 4.20 installation guide specifies Loki Operator 5.7 or later for the documented integration.
- You have planned the resource requirements and retention for your expected traffic volume. 
  
Check the cluster:
oc get clusterversion
oc get network.config.openshift.io cluster \
  -o jsonpath='{.status.networkType}{"\n"}'

oc get nodes -o wide
oc get pods -n openshift-network-operator


Expected network type for the documented setup:
OVNKubernetes


If your cluster uses another primary network plugin, do not assume this implementation is supported.
Step 1 — Install the operator
In the OpenShift console:
1. Navigate to Ecosystem → Software Catalog (OperatorHub).
2. Search for Network Observability Operator.
3. Open the operator and select Install.
4. Choose the appropriate supported channel and installation settings for your cluster.
5. Enable the option to expose operator-recommended cluster monitoring for the operator namespace when appropriate.
6. Complete the installation and verify its status.
The namespace commonly used in the official examples is netobserv. Your organization may use a different approved namespace. 

Step 2 — Understand the FlowCollector custom resource
The most important resource is FlowCollector.

Its API group and version in the OpenShift 4.20 examples are:
apiVersion: flows.netobserv.io/v1beta2
kind: FlowCollector
metadata:
  name: cluster


Important points:
- It is cluster-scoped, not namespaced.
- Only one FlowCollector is supported for the cluster.
- Its name must be cluster.
- It tells the operator how to configure the collection pipeline.
- You normally create it after installing the operator, using the provided API or the setup wizard.
You do not create one FlowCollector for every application namespace. A single collector can observe traffic across the cluster, subject to configuration, platform support, and access controls. 

Step 3 — Create the initial configuration
The simplest recommended starting point is the console's FlowCollector setup wizard. This helps configure the fields appropriate to your selected storage backend.

For reference, here is a starting template based on the documented 4.20 API. It enables the eBPF agent, uses a sampling interval of 50, and enables the console plugin. It does not configure Loki storage, so do not treat it as a complete persistent flow-log deployment.

apiVersion: flows.netobserv.io/v1beta2
kind: FlowCollector
metadata:
  name: cluster
spec:
  namespace: netobserv

  deploymentModel: Service

  agent:
    type: eBPF
    ebpf:
      sampling: 50
      privileged: false
      features: []

  processor:
    addZone: false
    subnetLabels:
      openShiftAutoDetect: true
      customLabels: []
    consumerReplicas: 3

  consolePlugin:
    enable: true

  loki:
    enable: false

  exporters: []



Before applying this template: confirm that the installed operator exposes the expected API version and that the fields match the installed version. This is an illustrative baseline, not a universally production-ready sizing recommendation. If your deployment requires packet-drop tracking or other features that need elevated privileges, you must explicitly plan for those requirements. 

Verify the installed API before creating the resource:
oc api-resources | grep -i flowcollector

oc get crd flowcollectors.flows.netobserv.io

oc explain flowcollector.spec


After validating the template against the installed CRD, save it as flowcollector.yaml and apply it:
oc apply -f flowcollector.yaml
oc get flowcollector cluster -o yaml


If you already have a FlowCollector, inspect and update that resource instead of creating another.
8. Configure Loki for historical network-flow data
If you want to retain detailed network-flow records, configure a compatible Loki integration. Your Loki deployment can be separate from the Network Observability namespace.

For example:
- Network Observability: netobserv
- Loki: netobserv-loki
- LokiStack instance: loki
The names above are examples. The actual names must match your installed resources.

A relevant part of the FlowCollector configuration looks like this:
spec:
  namespace: netobserv

  loki:
    enable: true
    mode: LokiStack
    lokiStack:
      name: loki
      namespace: netobserv-loki

  consolePlugin:
    enable: true


This is a configuration fragment, not a complete replacement FlowCollector manifest. Merge it into your existing resource and validate it against the installed API. Your LokiStack must exist, be healthy, and be configured for the supported integration.

The important distinction is that enabling Loki integration does not install or configure all of Loki's storage, object-storage credentials, retention, and access controls for you. Those must be addressed separately. 

9. Verify that the pipeline is working
After applying the configuration, check the operator and its managed components.
# Find the operator and related workloads
oc get pods -A | grep -i netobserv

# Check the FlowCollector
oc get flowcollector cluster

# Inspect its desired and observed state
oc describe flowcollector cluster

# Check the collector namespace
oc get pods -n netobserv

# Check services and deployments
oc get svc,deploy,ds -n netobserv


Depending on the deployment, you should find the eBPF agent DaemonSet, the flow-processing components, and the console plugin components. Some resources may be in a namespace other than netobserv if you configured a different deployment namespace.
Verify each layer independently
Layer	What to check
Operator	Installed successfully and reconciles resources
FlowCollector	Resource accepted and components reconciled
eBPF agent	Expected pods running on eligible nodes
Flow processor	Ready replicas, healthy logs, no persistent errors
Loki, if enabled	Healthy LokiStack and working integration
Metrics	Relevant metrics available through the configured monitoring path
Console plugin	Plugin registered and network traffic page available

Then open the OpenShift console and look under Observe → Network Traffic.

Generate some legitimate application traffic if the cluster is idle. If the traffic view is empty, clear the default filters before concluding that collection is broken. The official installation guide specifically calls out filters as a possible reason for seeing no results. 

Red Hat Documentation



10. Common troubleshooting scenarios
Problem: Network Traffic is missing from the console

Check the console plugin configuration and pods, and verify that the FlowCollector has reconciled successfully.
oc get flowcollector cluster -o yaml
oc get pods -n netobserv
oc get consoles.operator.openshift.io cluster -o yaml


The last command inspects the OpenShift console operator configuration; it does not by itself confirm plugin registration.

Problem: No network flows are visible

Clear the UI filters, generate application traffic, check agent pods across the nodes, and inspect the flow-processing logs.
oc get ds,po -n netobserv -o wide
oc logs -n netobserv deployment/flowlogs-pipeline


Verify the actual deployment name first; it can vary by deployment and version.

Problem: Loki is enabled but historical flow data is missing

Check the LokiStack name and namespace, Loki health, the collector's integration configuration, and its logs.
oc get lokistack -A
oc get pods -A | grep -i loki
oc describe flowcollector cluster


Also inspect authentication, TLS, network connectivity, and Loki ingestion errors.

Problem: High CPU, memory, or storage usage

Review sampling, flow filters, processor replicas, traffic volume, and Loki retention. Sampling every packet can substantially increase processing and storage costs. Avoid reducing the sampling interval or enabling extra features indiscriminately.

The deployment names in the commands above are examples. Always discover the actual resources with oc get deployments,daemonsets,pods -n netobserv before running commands against a specific deployment.
11. Production design and security considerations
For a production cluster, plan these areas before enabling collection broadly.

Resource sizing: size the eBPF agents and processor for actual node count and traffic volume. A configuration that works in a development cluster may not be sufficient for a busy production cluster.

Sampling and filtering: start with the documented default sampling interval, observe resource consumption, and adjust using measured traffic patterns. Apply filters thoughtfully so you do not discard flows needed for incident investigations.

Storage and retention: if using Loki, size persistent storage and object storage where applicable, configure retention, and monitor ingestion errors. Loki is not a substitute for a backup strategy.

Privileges: the eBPF agent needs access to supported kernel networking information. Some features require privileged mode. Enable it only when required and approved, and understand the security implications.

Access control: restrict access to network-flow information. It may reveal workload relationships, internal addresses, service ports, and communication patterns. Configure appropriate OpenShift RBAC and any relevant Loki access controls.

Change management: a FlowCollector change can restart managed components. Apply changes through GitOps with validation and a rollback plan, and monitor the resulting pipeline after each change.

Disconnection and mirroring: for an air-gapped cluster, ensure the operator and required component images are mirrored from the appropriate supported release. Any Loki or other dependency must also have its images and dependencies mirrored. The operator itself does not eliminate the need to manage these dependencies.


Network Policies are required to control which components can communicate with the Network Observability components. They are a security boundary, not a mechanism for collecting network flows.

In OpenShift 4.20, Network Observability can automatically create and manage these policies through the Operator. Network policy management is enabled by default in the FlowCollector. 

1. Why do we need them?
Imagine your netobserv namespace contains the flow-processing pipeline. You don't want every pod in the cluster to be able to connect to its services unnecessarily.

Network Policies can restrict traffic to approved paths, such as:
- OpenShift Console → Network Observability services
- Flow-collection components → required processing services
- Flowlogs pipeline → LokiStack
- Monitoring components → permitted metrics endpoints
- Flowlogs pipeline → an external exporter, if configured
The exact rules depend on the deployment model and enabled integrations.
2. What happens if we disable them?
spec:
  networkPolicy:
    enable: false


This disables the FlowCollector-managed network policies. It does not disable flow collection or automatically open every connection.

If your cluster relies on these policies, disabling them can remove intended traffic restrictions. Conversely, if another policy blocks required communication, flows might not reach Loki or the console may fail to display data.
3. Do we need to write NetworkPolicy YAML manually?
Usually, no. For a standard installation, start with the Operator-managed policies.
spec:
  namespace: netobserv
  networkPolicy:
    enable: true


If Loki, Kafka, or an exporter is installed in a separate namespace with its own Network Policies, verify the required connections. You may need to add the destination namespace to additionalNamespaces or configure the destination-side policy accordingly. 

For example, if your exporter is deployed in my-kafka-exporter:
spec:
  networkPolicy:
    enable: true
    additionalNamespaces:
      - openshift-console
      - openshift-monitoring
      - my-kafka-exporter


Keep the default namespaces where required by your setup. This configuration is illustrative; confirm the installed Operator's default values and schema before applying it.
