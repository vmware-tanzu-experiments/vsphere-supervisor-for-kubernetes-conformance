# Known Test Failures

This page aims to summarize the causes of known test failures when running
Kubernetes conformance tests against a Supervisor cluster running vSphere Pods.
None of these are a consideration when running TKG clusters in a Supervisor, which
are fully-conformant upstream Kubernetes clusters.

For each category of test failure, we will clearly identify the impact on
applications in terms of the application itself, its portability and
what alternative approaches we recommend.

Note that all of these test failures are due to the inherent elevated
security architecture of vSphere and ESXi, not a fault with the product.

## Index

- [Host namespace access is not possible](#host-namespace-access-is-not-possible)
- [HostPath Volumes are limited in scope](#hostpath-volumes-are-limited-in-scope)
- [Only CSI-based PersistentVolumes are supported](#only-csi-based-persistentvolumes-are-supported)
- [NodePort Services are not supported](#nodeport-services-are-not-supported)
- [SchedulerPreemption is not supported](#scheduler-preemption-is-not-supported)

## Host namespace access is not possible

### What's the issue?

Linux Containers can be granted access to the [namespaces of the node](
https://kubernetes.io/docs/concepts/policy/pod-security-policy/#host-namespaces)
they're running on. This is a kind of privileged contract between a
container and its host.

ESXi allows no such weakening of isolation between VMs or vSphere Pods and
the host. As such, there is no functional equivalent of HostNetwork,
HostIPC, HostPID or HostPorts.

### What's the impact?

The following container config in pod specs is ignored:

- `hostNetwork: true` and any associated `hostPorts`
- `hostIPC: true`
- `hostPID: true`

Since the config is ignored, you can run vSphere pods with these options.
However, a pod with any of these options is almost certainly not a good
candidate for a vSphere Pod.

### Are there workarounds?

No. These options are useful for pods that perform monitoring functions or
extend a Linux node in some way. By definition a strongly-isolated pod
should not be attempting to share anything with the host it's running on.

## HostPath Volumes are limited in scope

### What's the issue?

[HostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)
is a means of bind-mounting state from a node into a pod. This can include
files, directories, sockets, character devices, block devices and virtual
filesystems such as `/sys` or `/proc`.

ESXi does not have any means of bind-mounting its state into a vSphere Pod.
However, there are limited use cases for containers in a pod accessing
data about other containers in the same pod. We support one of these
use cases: accessing logs.

### What's the impact?

The only permitted use-case for a HostPath Volume is for one container
in a pod to be able to see the logs of other containers in the pod.
This is achieved by mounting `/var/log/pods` and it can only be mounted
read-only. Eg:

```
...
volumes:
  - name: container-logs
    hostPath:
      path: /var/log/pods
      type: Directory
      readOnly: true
```

Any other hostPath definition will be rejected by the scheduler.

### Are there workarounds?

That depends on why the container needs a hostPath volume. Note that the
Kubernetes documentation states "[This is not something that most Pods will need](
https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)".

If the reason is to access a socket or virtual filesystem on the host,
the answer is no. By definition a strongly-isolated pod shares nothing
with its host or other pods.

If the container needs to share state with other containers in the pod,
it can use a [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
volume type, which writes to the pod's ephemeral disk. The size of this
disk can be [specified in the container spec](
https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#setting-requests-and-limits-for-local-ephemeral-storage)
with the `ephemeral-storage` clause.

## Only CSI-based PersistentVolumes are supported

### What's the issue?

VMs, and by implication vSphere Pods, manage their own storage independent of the
ESXi hosts they're deployed to. They can either access virtual block devices on
Datastores or they mount their own network attached storage. There is no concept
in which the ESXi host can share any part of its state with the VMs or vSphere
Pods it's managing.

[Local](https://kubernetes.io/docs/concepts/storage/volumes/#local) Volumes
are very similar in concept to HostPath volumes in that they allow a pre-determined
mount on a node to be made available to containers in a pod. The main difference
is that a local Volume is defined as a PersistentVolume type.
There is no equivalent concept in ESXi.

### What's the impact?

Any Pod that references a PersistentVolume of local type will be rejected by
the scheduler.

See [here](https://kubernetes.io/docs/concepts/storage/volumes/#local) for an example

### Are there workarounds?

No. PersistentVolumes for vSphere Pods must be defined explicitly and are
then in use exclusively by that pod via a PersistentVolumeClaim.

## NodePort Services are not supported

### What's the issue?

[NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport)
is a type of Kubernetes Service that allows a container to be accessed on a
pre-determined port at the IP Address of any of the nodes in the cluster.

This is a useful feature if the cluster doesn't have any kind of external
load balancer. However, access to the Service depends on the availability
of a single node, so is not a reliable way of exposing a Service in production.

ESXi does not allow any application traffic on its management IP.
As such there is no functional equivalent of a NodePort Service for vSphere Pods.

### What's the impact?

Any Service definition of type NodePort is ignored. Eg:

```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
...
```

### Are there workarounds?

Yes. Use a Service of type LoadBalancer if you have an external load balancer
configured for the cluster. vSphere with Tanzu assumes you have such a load balancer
already configured and uses it to load balance control plane traffic.

Services of type [ClusterIP](
https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)
work just fine with vSphere pods, but those are only useful for communication
inside of the cluster.

You can also deploy an [Ingress Controller](
https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
and use layer 7 load balancing instead if you're looking to load-balance HTTP traffic.

## SchedulerPreemption is not supported

### What's the issue?

Supervisor cluster does not support Preemption based on PriorityClass during
vSphere Pod scheduling.

### What's the impact?

Preemption of lower-priority pods when higher-priority pods fail to schedule,
is not supported on the Supervisor Cluster.

### Are there workarounds?

Manually evicting lower-priority pods, to ensure there are enough resources for
higher priority pods.
