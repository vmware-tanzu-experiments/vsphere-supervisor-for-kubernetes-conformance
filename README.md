# vSphere Supervisor for Kubernetes Conformance

## Overview

In the vSphere 7.0 release, VMware added a Kubernetes control plane to vSphere
called "Supervisor". The Supervisor control plane is enabled on a per vSphere
cluster basis and performs multiple functions:

1. The ability to apply the Kubernetes desired state model with controllers and
   CRDs natively to vSphere compute, storage and networking. This includes the
   ability to use Supervisor to manage the lifecycle of Tanzu Kubernetes Grid
   (TKG) clusters using declarative configuration.
1. Paravirtual capabilities that enhance the integration with virtual
   Kubernetes clusters such as single sign-on, RBAC, persistent volumes,
   load balancing and high availability.
1. The ability to manage ESXi hosts as if they are Kubernetes nodes and
   therefore deploy Kubernetes applications directly to vSphere using
   hardware-isolated "native" pods.

The Supervisor itself and the TKG clusters it can create are fully-conformant
upstream Kubernetes. However, the ability for the Supervisor to run pods
directly on the Hypervisor as strongly-isolated first class
citizens comes with its own conformance considerations and caveats.

This repository is dedicated to providing comprehensive information on
the Conformance of the Supervisor when you're running applications as
vSphere native pods. This includes:

- Links to blogs and articles on the topic
- The raw conformance results themselves
- Information to help you simply and easily understand the caveats

## Supervisor and vSphere Native Pods

VMware is not the only company to see the value in hardware-isolated pods.
Projects such as Kata Containers or Firecracker typically use KVM or QEMU in
Linux to provide a hardware-isolated sandbox with a custom CRI implementation
that interfaces with Kubelet. vSphere Native pods use a custom VMX abstraction
in ESXi called CRX that boots instantly.

### Benefits

#### Security

- Native pods run their own kernel effectively creating a single failure
  domain. Issues such as kernel panics, inode starvation, buffer cache
  contention can only impact the pod experiencing the problem. It also has
  the potential to allow for kernel customization at the pod level, such
  as loading of kernel modules specific to the application.
- Security breaches via privilege escalation are significantly harder.
  This means that the pod is harder to break into _and_ harder to break out of.
- Storage and network isolation. Shared state on bind-mounted storage is a
  significant security risk if not carefully managed. Native pods do not share
  storage and have their own TCP/IP stack.

#### Performance

- Native pods collapse an entire layer of CPU scheduling allowing a hypervisor
  to make better optimizations. When we launched vSphere 7.0, we were able to
  show that native pods can even be faster than bare metal!
- A vSphere admin gets all of the same performance analysis with a native pod
  that they would with a VM
- The isolation should ensure that a native pod gets very consistent
  performance as none of its resource is shared

#### Resource utilization

- Native pods only consume resource in vSphere when they're running. Virtual
  Kubernetes nodes reserve capacity from the hypervisor even when no pods are
  running. So even though a native pod may have a higher footprint than a regular
  pod, the net effect on the vSphere cluster resources may be significantly smaller.

### Functional Benefits

The sweet spot for Native Pods are:

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
The biggest different between vSphere native pods and other strongly-isolated
pod implementations is that ESXi is the node, not Linux.

While Kubernetes has well-defined APIs and extension mechanisms, the ABI of
the Linux Kubernetes node is an implied interface that a lot of Kubernetes
extensions make use of. Many extensions deploy privileged containers to Linux
nodes via DaemonSets that add capabilities to the node that containers
running on that node can make use of. These extensions often depend on the
contract between the containers running on the node and the node itself to
function.

ESXi is not Linux and the contract between VMX/CRX and the ESXi node is very
different to that of a container on Linux. As such, the biggest limitation of
vSphere native pods is that the ability to deploy cross-cutting extensions to
the Supervisor is limited. The best alternative is to deploy sidecar
containers into native pods, which makes architectural sense given that a
native pod is very like a single-pod node in concept.

## Conformance

The whole point of the GitHub repository is to provide a touch point with the
community using our products around the topic of Supervisor conformance.

The Supervisor when running native pods does not pass all of the Kubernetes
conformance tests. This is all due to the inherent security of ESXi and CRX.
To be clear, TKG clusters that are managed by Supervisor _are_ fully conformant
upstream Kubernetes. This only applies to Supervisor running native pods.

### Links

This blog post goes into a lot of detail on the security-oriented design
and architecture of ESXi and how that impacts our ability to pass conformance.

This document is a more concise summary of which tests we fail and why.

We provide all of our raw conformance results from running with Supervisor
with native pods in the same format as the CNCF conformance GitHub.

If you have any questions on the data presented here or want to see content
added, please raise an issue.






