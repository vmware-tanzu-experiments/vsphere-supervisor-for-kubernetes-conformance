# Known Test Failures for v1.19

This page aims to summarize the causes of known test failures when running
this version of Kubernetes conformance tests against a Supervisor cluster
running vSphere Pods.

- Test: `Kubectl cluster-info should check if Kubernetes master services is included in cluster-info`

	Failure Reason: [Test has been updated to use inclusive terminology](https://github.com/kubernetes/kubernetes/commit/ab129349acadb4539cc8c584e4f9a43dd8b45761)

	Fixed: `v1.20`

