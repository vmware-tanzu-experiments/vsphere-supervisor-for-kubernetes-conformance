# Known Test Failures for v1.20

This page aims to summarize the causes of known test failures when running
this version of Kubernetes conformance tests against a Supervisor cluster
running vSphere Pods.

- Test: `ServiceAccounts should mount projected service account token`

	Failure Reason:
	Spherelet was incorrectly processing token requests with no audience members,
	leading to failures in volume mount.

	Impact:
	Using vSphere pods with token requests with no audience members would fail.

	Fixed: `v1.21`

- Test: `Variable Expansion should fail substituting values in a volume subpath with absolute path`

	Failure Reason:
	Spherelet was not setting the expected error message for container failures due to
	Variable Expansion failure for volume subpath.

	Impact:
	Error message for vSphere Pod with variable Expansion failures will not contain `ErrCreateContainerConfig`.

	Fixed: `v1.21`

- Test: `Variable Expansion should fail substituting values in a volume subpath with backticks`
	
	Failure Reason:
	Spherelet was not setting the expected error message for container failures due to
	Variable Expansion failure for volume subpath.

	Impact:
	Error message for vSphere Pod with variable Expansion failures will not contain `ErrCreateContainerConfig`.

	Fixed: `v1.21`
