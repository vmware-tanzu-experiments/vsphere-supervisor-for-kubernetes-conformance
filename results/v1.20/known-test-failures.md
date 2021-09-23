# Known Test Failures for v1.20

This page aims to summarize the causes of known test failures when running
this version of Kubernetes conformance tests against a Supervisor cluster
running vSphere Pods.

- Test: `ServiceAccounts should mount projected service account token`

	Failure Reason: Bug

	Fixed: `v1.21`

- Test: `Variable Expansion should fail substituting values in a volume subpath with absolute path`

	Failure Reason: Bug

	Fixed: `v1.21`

- Test: `Variable Expansion should fail substituting values in a volume subpath with backticks`
	
	Failure Reason: Bug

	Fixed: `v1.21`
