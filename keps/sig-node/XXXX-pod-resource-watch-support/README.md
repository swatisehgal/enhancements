# Enabling Watch support in Kubelet endpoint for pod resource assignment

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Observability](#observability)
    - [Node Feature Discovery](#node-feature-discovery)
    - [Topology aware scheduling](#topology-aware-scheduling)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Proposed API](#proposed-api)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Alpha to Beta Graduation](#alpha-to-beta-graduation)
    - [Beta to G.A Graduation](#beta-to-ga-graduation)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature enablement and rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Alternatives](#alternatives)
- [Alternatives](#alternatives)
  - [Add a new endpoint](#add-a-new-endpoint)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [X] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements](https://github.com/kubernetes/enhancements/issues/606)
- [X] (R) KEP approvers have approved the KEP status as `implementable`
- [X] (R) Design details are appropriately documented
- [X] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [X] (R) Graduation criteria is in place
- [X] (R) Production readiness review completed
- [X] Production readiness review approved
- [X] "Implementation History" section is up-to-date for milestone
- ~~ [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io] ~~
- [X] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

This document presents an addition to the kubelet pod resources endpoint (pod resources API) which allows third party consumers to learn about the compute device allocation, thus, alongside the existing pod resources API endpoint, properly evaluate the node capacity.

## Motivation

### Goals

* Enable node monitoring agents to receive events corresponding to updates wrt allocation of resources as well as allocatable compute resources on a node, in order
to allow proper calculation of the node compute resource utilization. This is to provide an event-based mechanism to the clients of the API that rely on the API as opposed to the current way of having to poll the API.

## Proposal

### User Stories

#### Observability
As a _Cluster Administrator_, I provide a set of devices from various vendors in my cluster.  Each vendor independently maintains their own agent, so I run monitoring agents only for devices I provide.  Each agent adheres to to the [node monitoring guidelines](https://docs.google.com/document/d/1_CdNWIjPBqVDMvu82aJICQsSCbh2BR-y9a8uXjQm4TI/edit?usp=sharing), so I can use a compatible monitoring pipeline to collect and analyze metrics from a variety of agents, even though they are maintained by different vendors.

In addition to above, I would like to gather information on how dynamic the cluster is with respect to compute resource availablity and allocation to pods.

#### Node Feature Discovery
Enable the Node Feature Discovery to [expose hardware topology information](https://github.com/kubernetes-sigs/node-feature-discovery/issues/333).
Faclitate a more event driven design to ensure that NFD-topology Updater updates the NodeResourceTopology CRs on every deviceplugin and/or pod creation/modification/deletion event as opposed to the current approach of polling the API endpoints. 

#### Topology aware scheduling

This interface can be used to track down allocated resources with information about the NUMA topology of the worker node in general way. This interface can be used to the available resources on the worker node. The kubelet is the best source of information because it manages concrete resources assignment. The information can then be used in NUMA aware scheduling. Combining the information reported by the List API, which pertains the current allocation, with the information reported by the GetAllocatableResources API, monitoring agent can reliably report the compute device utilization and availability.

The endpoints in podresource API need to be polled whereas the ability to receive events corresponding to updates wrt allocation of resources and/or allocatable compute resources on a node can be used to further optimize the calculation of the node compute resource utilization.

### Risks and Mitigations

This API is read-only, which removes a large class of risks. The aspects that we consider below are as follows:
- What are the risks associated with the API service itself?
- What are the risks associated with the data itself?

| Risk                                                      | Impact        | Mitigation |
| --------------------------------------------------------- | ------------- | ---------- |
| Too many requests risk impacting the kubelet performances | High          | Implement rate limiting and or passive caching, follow best practices for gRPC resource management. |
| Improper access to the data | Low | Server is listening on a root owned unix socket. This can be limited with proper pod security policies. |


## Design Details

### Proposed API

We propose to add a new gRPC service to the Kubelet. This gRPC service would be listening on a unix socket at `/var/lib/kubelet/pod-resources/kubelet.sock` and return information about the kubelet's assignment of devices to containers.

This gRPC service returns information about
- the devices which kubelet knows about from the device plugins.
- the kubelet's assignment of devices and cpus with NUMA id to containers.
The GRPC service obtains this information from the internal state of the kubelet's Device Manager and CPU Manager respectively.
The GRPC Service exposes these endpoints:
- `List`, which returns a single PodResourcesResponse, enabling monitor applications to poll for resources allocated to pods and containers on the node.
- `GetAllocatableResources`, which returns AllocatableResourcesResponse, enabling monitor applications to learn about the the allocatable resources available on the node.
- `ListAndWatch`, which returns a stream of ListAndWatchPodResourcesResponse, enabling monitor applications to be notified of new resource allocation, release or resource allocation updates, using the `action` field in the response.
- `ListAndWatchAllocatableResources`, which returns a stream of ListAndWatchAllocatableResourcesResponse, enabling monitor applications to be notified of addition/modification/deletion of allocatable resources on a node, using the `action` field in the response.

This is shown in proto below:
```protobuf
// PodResourcesLister is a service provided by the kubelet that provides information about the
// node resources consumed by pods and containers on the node
service PodResourcesLister {
    rpc List(ListPodResourcesRequest) returns (ListPodResourcesResponse) {}
    rpc GetAllocatableResources(AllocatableResourcesRequest) returns (AllocatableResourcesResponse) {}
    rpc ListAndWatch(ListAndWatchPodResourcesRequest) returns (stream ListAndWatchPodResourcesResponse) {}
    rpc ListAndWatchAllocatableResources(ListAndWatchAllocatableResourcesRequest) returns (stream ListAndWatchAllocatableResourcesResponse) {}
}

message AllocatableResourcesRequest {}

// AvailableResourcesResponses contains informations about all the devices known by the kubelet
message AllocatableResourcesResponse {
    repeated ContainerDevices devices = 1;
    repeated int64 cpu_ids = 2;
    repeated ContainerMemory memory = 4;
}

// ListPodResourcesRequest is the request made to the PodResources service
message ListPodResourcesRequest {}

// ListPodResourcesResponse is the response returned by List function
message ListPodResourcesResponse {
    repeated PodResources pod_resources = 1;
}

// ListAndWatchPodResourcesRequest is the request made to the Watch PodResourcesLister service
message ListAndWatchPodResourcesRequest {}

enum WatchPodAction {
    ADDED = 0;
    DELETED = 1;
    UPDATED = 2;
}

// ListAndWatchPodResourcesResponse is the response returned by Watch function
message ListAndWatchPodResourcesResponse {
    WatchPodAction action = 1;
    string uid = 2;
    repeated PodResources pod_resources = 3;
}

// ListAndWatchAllocatableResourcesRequest is the request made to the Watch PodResourcesLister service
message ListAndWatchAllocatableResourcesRequest {}

enum WatchResourceAction {
    ADDED = 0;
    DELETED = 1;
    UPDATED = 2;
}

// ListAndWatchAllocatableResourcesResponse is the response returned by Watch function
message ListAndWatchAllocatableResourcesResponse {
    WatchResourceAction action = 1;
    repeated ContainerDevices devices = 1;
    repeated int64 cpu_ids = 2;
    repeated ContainerMemory memory = 4;
}

// PodResources contains information about the node resources assigned to a pod
message PodResources {
    string name = 1;
    string namespace = 2;
    repeated ContainerResources containers = 3;
}

// ContainerResources contains information about the resources assigned to a container
message ContainerResources {
    string name = 1;
    repeated ContainerDevices devices = 2;
    repeated int64 cpu_ids = 3;
    repeated ContainerMemory memory = 4;
}

// Topology describes hardware topology of the resource
message TopologyInfo {
	repeated NUMANode nodes = 1;
}

// NUMA representation of NUMA node
message NUMANode {
	int64 ID = 1;
}

// ContainerDevices contains information about the devices assigned to a container
message ContainerDevices {
    string resource_name = 1;
    repeated string device_ids = 2;
    TopologyInfo topology = 3;
}
```

Using the `ListAndWatch` endpoint, client applications will be notified of the pod resource allocation changes as soon as possible.
This captures state of the pods' resources after any resource allocation change (addition/modification/deletion).
NOTE: The response does not capture or indicate explicitly the differnce from the previous response received and (if required) the client
would be responsible for evaluating that. 
However, the state of a pod will not be sent up until the first resource allocation change, which can be pod deletion in the worst case.
Client applications who need to have the complete resource allocation picture thus need to consume both `List` and `ListAndWatch` endpoints.

To keep the implementation simple as possible, the kubelet does *not* store any historical list of changes.

In order to make the resource accounting on the client side, safe and easy as possible the `ListAndWatch` implementation
will guarantee ordering of the event delivery in such a way that the capacity invariants are always preserved, and the value
will be consistent after each event received - not only at steady state.
Consider the following scenario with 10 devices, all allocated: pod A with device D1 allocated gets deleted, then
pod B starts and gets device D1 again. In this case `ListAndWatch` will guarantee that `DELETE` and `ADDED` events are delivered
in the correct order.

To properly evaluate the amount of allocatable compute resources on a node, client applications can use the `GetAlloctableResources` endpoint.
Applications can use `List` and `ListAndWatch` to learn about the current allocation of these resources, and thus the current amount of free (unallocated)
compute resources.
Applications can also subscribe to `ListAndWatchAllocatableResources` to receive stream events corresponding to change in the resources
availability (e.g. CPUs onlined/offlined, devices added/removed). We anticipate these changes to happen very rarely.
Similar to `ListAndWatch`, implementation of `ListAndWatchAllocatableResources` will also guarantee ordering of the event delivery.

### Test Plan

Given that the API allows observing what device has been associated to what container, we need to be testing different configurations, such as:
* Pods without devices assigned to any containers.
* Pods with devices assigned to some but not all containers.
* Pods with devices assigned to init containers.
* Test the above cases of device assignment in the context of Watch support.
- ...

We have identified two main ways of testing this API:
- Unit Tests which won't rely on gRPC. They will test different configurations of pods and devices.
- Node E2E tests which will allow us to test the service itself. The Tests will cover the both a list-only client and a list-and-watch client.
- Node E2E tests to showcase podresource watch support (ListAndWatch and ListAndWatchAllocatableResources).

E2E tests are explicitly not written because they would require us to generate and deploy a custom container.
The infrastructure required is expensive and it is not clear what additional testing (and hence risk reduction) this would provide compare to node e2e tests.

### Graduation Criteria

#### Alpha
- [X] Implement the new service API.
- [X] Ensure proper e2e node tests are in place.
#### Alpha to Beta Graduation
- [X] The new API endpoints is consumed by other public software components (e.g. NFD).
- [X] No major bugs reported in the previous cycle.
- [X] External clients are using this capability in their solutions
    Topology aware Scheduling is one of the primary use cases of `ListAndWatch` and `ListAndWatchAllocatableResources` podresource endpoints. As part of this initiative an exporter populates CRs per node to expose the information of resources available per NUMA. Currently, Pod Resource API `List` and `GetAllocatableResources` API endpoints are used to obtain resource allocation of running pods along with the underlying hardware topology (NUMA) information
    but the goal is to determine the available resources optimally in an event based fashion rather than having to poll the API. Topology aware scheduler can be configured such that users can create custom exporters or use already existing exporters to expose the NodeResourceTopology information as CRs and then [Topology aware Scheduler](https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/pkg/noderesourcetopology) uses this information to make a NUMA aware placement decision leading to the reduction of occurrence of Topology affinity Errors highlighted in the issue [here](https://github.com/kubernetes/kubernetes/issues/84869).
    Examples of two such exporters are:
     - [Node feature Discovery](https://github.com/kubernetes-sigs/node-feature-discovery) for exposing resource topology information as part of the initiative here: [Introducing NFD Topology Updater exposing Resource hardware Topology info through CRs](https://github.com/kubernetes-sigs/node-feature-discovery/pull/525).
     - [Resource Topology Exporter](https://github.com/k8stopologyawareschedwg/resource-topology-exporter)

#### Beta to G.A Graduation
- [X] Allowing time for feedback.
- [X] Risks have been addressed.

### Upgrade / Downgrade Strategy

With gRPC the version is part of the service name.
Old versions and new versions should always be served and listened by the kubelet.

To a cluster admin upgrading to the newest API version, means upgrading Kubernetes to a newer version as well as upgrading the monitoring component.

To a vendor changes in the API should always be backwards compatible.

Downgrades here are related to downgrading the plugin

### Version Skew Strategy

Kubelet will always be backwards compatible, so going forward existing plugins are not expected to break.

## Production Readiness Review Questionnaire
### Feature enablement and rollback

* **How can this feature be enabled / disabled in a live cluster?**
  - [X] Feature gate (also fill in values in `kep.yaml`).
    - Feature gate name: `KubeletPodResourcesWatchSupport`.
    - Components depending on the feature gate: N/A.

* **Does enabling the feature change any default behavior?** No
* **Can the feature be disabled once it has been enabled (i.e. can we rollback the enablement)?** Yes, through feature gates.
* **What happens if we reenable the feature if it was previously rolled back?** The API is stateless, so no recovery is needed, clients can just consume the data.
* **Are there any tests for feature enablement/disablement?** e2e tests will demonstrate that when the feature gate is disabled, the API returns the appropriate error code.

### Rollout, Upgrade and Rollback Planning

* **How can a rollout fail? Can it impact already running workloads?** The new API may report inconsistent data, or may cause the kubelet to crash.
* **What specific metrics should inform a rollback?** Not Applicable, metrics wouldn't be available.
* **Were upgrade and rollback tested? Was upgrade->downgrade->upgrade path tested?** Not Applicable.
* **Is the rollout accompanied by any deprecations and/or removals of features,  APIs, fields of API types, flags, etc.?** No.

### Monitoring requirements
* **How can an operator determine if the feature is in use by workloads?**
  - Look at the `pod_resources_endpoint_requests_total` metric exposed by the kubelet.
  - Clients are connected to the podresources unix socket, for example  bychecking which containers mount the podresources socket path.
* **What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?**
  - [X] Metrics
    - Metric name: `pod_resources_endpoint_requests_total`, `pod_resources_endpoint_requests_list`, `pod_resources_endpoint_requests_get_allocatable`, `pod_resources_endpoint_errors_list`, `pod_resources_endpoint_errors_get_allocatable`, `pod_resources_endpoint_requests_list_and_watch` and `pod_resources_endpoint_requests_list_and_watch_allocatable_resources`.
    - Components exposing the metric: kubelet

* **What are the reasonable SLOs (Service Level Objectives) for the above SLIs?** N/A
* **Are there any missing metrics that would be useful to have to improve observability if this feature?** As part of this feature enhancement, per-API-endpoint resources metrics are being added; to observe this feature the `pod_resources_endpoint_requests_list_and_watch` and `pod_resources_endpoint_requests_list_and_watch_allocatable_resources` metrics should be used.


### Dependencies

* **Does this feature depend on any specific services running in the cluster?** Not aplicable.

### Scalability

* **Will enabling / using this feature result in any new API calls?** No.
* **Will enabling / using this feature result in introducing new API types?** No.
* **Will enabling / using this feature result in any new calls to cloud provider?** No.
* **Will enabling / using this feature result in increasing size or count of the existing API objects?** No.
* **Will enabling / using this feature result in increasing time taken by any operations covered by [existing SLIs/SLOs][]?** No. Feature is out of existing any paths in kubelet.
* **Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?** In 1.18, DDOSing the API can lead to resource exhaustion. It is planned to be addressed as part of G.A.
Feature only collects data when requests comes in, data is then garbage collected. Data collected is proportional to the number of pods on the node.

### Troubleshooting

* **How does this feature react if the API server and/or etcd is unavailable?**: No effect.
* **What are other known failure modes?** No known failure modes
* **What steps should be taken if SLOs are not being met to determine the problem?** N/A

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos

## Implementation History

- 2022-02-18: KEP extracted from [previous iteration](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2043-pod-resource-concrete-assigments) and updated with the relevant details pertaining to watch support with the introduction of `ListAndWatch` and `ListAndWatchAllocatableResources`. 

## Alternatives

### Add a new endpoint
* Pros:
  * No changes to existing APIs
* Cons:
  * Requires the client to consume two APIs
  * This work nicely fits in the boundaries and purpose of the podresources API
  * The changes proposed in this KEP are very low-risk and backward compatible