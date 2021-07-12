# vSphere Supervisor for Kubernetes Conformance

## Overview

In the [vSphere 7.0 release](
https://blogs.vmware.com/vsphere/2020/03/vsphere-7-features.html),
VMware [added a Kubernetes control plane](
https://blogs.vmware.com/vsphere/2019/08/project-pacific-technical-overview.html)
to vSphere called "Supervisor". The Supervisor control plane is enabled on a per vSphere
cluster basis and performs multiple functions:

1. The ability to apply the Kubernetes [desired state model](
   https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)
   with controllers and
   [CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
   natively to vSphere compute, storage and networking. This includes the
   ability to use Supervisor to manage the lifecycle of
   [Tanzu Kubernetes Grid](https://tanzu.vmware.com/kubernetes-grid)
   (TKG) clusters [using declarative configuration](https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-360B0288-1D24-4698-A9A0-5C5217C0BCCF.html).
1. Paravirtual capabilities that enhance the integration with virtual
   Kubernetes clusters such as single sign-on, RBAC, persistent volumes,
   load balancing and high availability.
1. The ability to manage ESXi hosts as if they are Kubernetes nodes and
   therefore deploy Kubernetes applications directly to vSphere using
   [hardware-isolated "vSphere Pods"](
   https://blogs.vmware.com/vsphere/2020/05/vsphere-7-vsphere-pods-explained.html).

The Supervisor itself and the TKG clusters it can create are fully-conformant
upstream Kubernetes. However, the ability for the Supervisor to run pods
directly on the hypervisor as strongly-isolated first class
citizens comes with its own conformance [considerations and caveats](#limitations).

This repository is dedicated to providing comprehensive information on
the conformance of the Supervisor when you're running applications as
vSphere Pods. This includes:

- [Links](#Links) to blogs and articles on the topic
- The raw conformance results themselves
- Information to help you simply and easily understand the caveats

## Supervisor and vSphere Pods

VMware is not the only company to see the value in hardware-isolated pods.
Projects such as Kata Containers or Firecracker typically use KVM or QEMU in
Linux to provide a hardware-isolated sandbox with a custom CRI implementation
that interfaces with Kubelet.

vSphere Pods use a custom VMX abstraction in
[ESXi](https://www.vmware.com/products/esxi-and-esx.html) called "[CRX](
https://blogs.vmware.com/vsphere/2020/05/vsphere-7-vsphere-pods-explained.html)" that boots
instantly and is controlled by an ESXi-native implementation of Kubelet,
called "[Spherelet](
https://frankdenneman.nl/2020/03/06/initial-placement-of-a-vsphere-native-pod/)".
Pod management is handled by Spherelet on the ESXi node and container management
is managed by "Spherelet Agent" running in each pod.
Image pulling and resolution is done by Kubernetes controllers that run custom
CRXs which populate a single image cache shared by all of the ESXi nodes.

### Functional Benefits

#### Security

- vSphere Pods run their own kernel, effectively creating a single failure
  domain. Issues such as kernel panics, inode starvation, buffer cache
  contention can only impact the pod experiencing the problem. It also has
  the potential to allow for kernel customization at the pod level, such
  as loading of kernel modules specific to an application.
- [Security breaches via privilege escalation](
  https://www.youtube.com/watch?v=vTgQLzeBfRU) are significantly harder.
  This means that the pod is harder to break into _and_ harder to break out of.
- Storage and network isolation. Shared state on bind-mounted storage is a
  significant security risk if not carefully managed. vSphere Pods do not share
  storage and have their own TCP/IP stack.

#### Performance

- vSphere Pods collapse an entire layer of CPU scheduling allowing a hypervisor
  to make better optimizations. When we launched vSphere 7.0, we were able to
  show that vSphere Pods can [even be faster than bare metal](
  https://blogs.vmware.com/performance/2019/10/how-does-project-pacific-deliver-8-better-performance-than-bare-metal.html)!
- A vSphere admin gets all of the same performance analysis with a vSphere Pod
  that they would with a VM
- The isolation should ensure that a vSphere Pod gets very consistent
  performance as none of its resource is shared

#### Resource utilization

- vSphere Pods only consume resource in vSphere when they're running. Virtual
  Kubernetes nodes reserve capacity from the hypervisor even when no pods are
  running. So even though a vSphere Pod may have a higher footprint than a regular
  pod, the net effect on the vSphere cluster resources may be significantly smaller.

### Ideal Use Cases

The sweet spot for vSphere Pods are:

#### Long running services

- Services that are needed by multiple Kubernetes clusters where their
  lifecycle can be managed independently
- Services that have higher requirements for security, either because of
  sensitive data, auditing requirements etc.
- Services that require strong isolation for consistency of performance
- Services that are stateful and are not tolerant of being shut down to
  move between Kubernetes nodes

#### Highly elastic ephemeral jobs

- If you need to consume large amounts of vSphere resource ephemerally
  without reserving capacity.
  For example, analytics, data mining etc. vSphere scheduling will use
  capacity where it's available and migrate VMs where necessary.

### Limitations

As well as understanding the functional benefits, it's equally important to
understand the limitations.
The biggest difference between vSphere Pods and other strongly-isolated
pod implementations is that ESXi is the node, not Linux.

While Kubernetes has well-defined APIs and extension mechanisms, the ABI of
the Linux Kubernetes node is an implied interface that a lot of Kubernetes
extensions make use of. Many extensions deploy privileged containers to Linux
nodes via DaemonSets that add capabilities to the node that containers
running on that node can make use of. These extensions often depend on the
contract between the containers running on the node and the node itself to
function.

ESXi is not Linux and the contract between CRX and the ESXi node is
different to that of a container on Linux. As such, the biggest limitation of
vSphere Pods is that the ability to deploy cross-cutting extensions to
the Supervisor is limited. The best alternative is to deploy [sidecar containers](
https://kubernetes.io/docs/concepts/workloads/pods/#how-pods-manage-multiple-containers)
into vSphere Pods, which makes architectural sense given that a
vSphere Pod is very similar to a single-pod node in concept.

## Conformance

The whole point of this GitHub repository is to provide a touch point with the
community using our products around the topic of Supervisor conformance.

The Supervisor when running vSphere Pods does not pass all of the Kubernetes
conformance tests. This is all due to the inherent security architecture of ESXi and CRX.
To be clear, TKG clusters that are managed by Supervisor _are_ fully conformant
upstream Kubernetes. This only applies to Supervisor running vSphere Pods.

## Links

[This blog post](https://core.vmware.com/blog/tanzu-secure-by-design) 
goes into a lot of detail on the security-oriented design
and architecture of ESXi and how that impacts our ability to pass conformance.

[This document](known-test-failures.md) 
is a more concise summary of which tests we fail and why.

We run the Kubernetes conformance tests as part of our Supervisor build pipeline and 
will upload raw results to this repository for every release in the same format as 
the [CNCF conformance GitHub](https://github.com/cncf/k8s-conformance/blob/master/README.md).

## Contact

If you have any questions on the data presented here or want to see content
added, please raise an issue.
