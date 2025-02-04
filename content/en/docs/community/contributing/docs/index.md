---
title: 'Docs'
linkTitle: 'Docs'
weight: 2
description:
---

Bugs affecting more than one service commonly surface only when the whole system is running. Our continuous integration system builds and runs a suite of integration tests against Spinnaker nightly on real cloud provider infrastructure to detect these bugs.

> **Access the CI system here**: [https://builds.spinnaker.io](https://builds.spinnaker.io)
>
> Viewers must be a member of the `build-cops` GitHub Team.

## Build Cop

The **build cop** responsibilities include:

- Triage integration test failures on `master` and the 3 most recent release branches
- [Clean up](#cleaning-orphaned-resources) orphaned resources across target cloud providers
- Route [new GitHub issues](https://github.com/spinnaker/spinnaker/issues) to the appropriate SIG
  (applying GitHub labels as appropriate). You can find the full list of SIGs in the
  [governance repo](https://github.com/spinnaker/governance/blob/master/sig-index.md)
- Observe any systemic problems raised in the [#general](https://spinnakerteam.slack.com/messages/general/)
  and [#dev](https://spinnakerteam.slack.com/messages/dev/) Slack channels
- Log observations and corrective actions taken in the [rotation log](https://docs.google.com/document/d/1T0kifZ0C7zSIKOy2McKfmDnwvmRuU5Z3t81Tly6kH1M/edit#)

## Process Structure

The CI system comprises both _jobs_, which do a specific task, and _flows_, which invoke a series of jobs.

[`Flow_BuildAndValidate`](https://builds.spinnaker.io/job/Flow_BuildAndValidate/) is the primary entry point for the `master` branch flow.

`Flow_BuildAndValidate_<version>` is the the entry point for the respective top-level release. It is a copy of the primary `Flow_BuildAndValidate` when that release was cut. Top-level release flows work off their respective `release-1.ABC.x` branches.

As its name implies, `Flow_BuildAndValidate` builds and tests the whole Spinnaker system. It follows this general process:

### 1. [Build_PrimaryArtifacts](https://builds.spinnaker.io/job/Build_PrimaryArtifacts/)

1. `git checkout` all services
1. Constructs a BOM from the most recent commit on the target branch
1. Builds a Docker container and a Debian package of each Spinnaker microservice.
1. Builds additional supporting artifacts:

- `halyard`
- `spin-cli`
- Changelog

1. Publishes the BOM under the following names:

- _With the floating tag:_ `<branchName>-latest-unvalidated` (e.g. `master-latest-unvalidated`)
- _With a fixed tag:_ `<branchName>-<timestamp>` (e.g. `master-20191213154039`)

1. Publishes the [changelog](https://gist.github.com/spinnaker-release/4f8cd09490870ae9ebf78be3be1763ee)

### 2. [Validate_BomAndReportMultiPlatform](https://builds.spinnaker.io/job/Validate_BomAndReportMultiPlatform/)

For uninteresting reasons, this job must wrap the following `ValidateBomMultiPlatform` in order to aggregate its results.

### 3. [ValidateBomMultiPlatform](https://builds.spinnaker.io/job/Validate_BomMultiPlatform/)

This "Multi-configuration project" specifies the same test(s) to run across different environments. This confirms
Spinnaker works whether deployed as a single VM or in a Kubernetes cluster, for instance.

1. Starts Halyard in a new VM
1. Connects to this instance and executes a series of `hal config` steps, including account setup for the managed cloud provider(s).
1. Deploys the configuration with `hal deploy apply`.
1. Invokes [`citest`](https://github.com/google/citest) integration tests against the new Spinnaker instance.
   - `citest` invokes a command to Spinnaker, and then uses the underlying cloud provider's CLI to confirm the expected changes were made. For example, using `gcloud` to confirm a GCE server group was created or deleted.

## Cleaning Orphaned Resources

Occasionally, integration tests fail in a way that is either undesirable or difficult to automatically clean up. Build cops should periodically ensure these orphaned resources are deleted from the following locations:

- [`spinnaker-community` GCP project](https://console.cloud.google.com/home/dashboard?organizationId=912934373776&project=spinnaker-community)
  - [Instance Groups](https://console.cloud.google.com/compute/instanceGroups/list?organizationId=912934373776&project=spinnaker-community&instanceGroupsTablesize=50&instanceGroupsTablequery=%255B%257B_22k_22_3A_22name_22_2C_22t_22_3A10_2C_22v_22_3A_22_5C_22gcp*_5C_22_22%257D%255D) named `gcp<testName>-*`
  - [VMs](https://console.cloud.google.com/compute/instances?organizationId=912934373776&project=spinnaker-community&instancessize=50&instancesquery=%255B%257B_22k_22_3A_22name_22_2C_22t_22_3A10_2C_22v_22_3A_22_5C_22jenkins-validate-bom*_5C_22_22%257D%255D) named `jenkins-validate-bom-*`
  - [Load balancers](https://console.cloud.google.com/net-services/loadbalancing/loadBalancers/list?project=spinnaker-community&organizationId=912934373776&filter=%255B%257B_22k_22_3A_22Name_22_2C_22t_22_3A10_2C_22v_22_3A_22_5C_22gcp*_5C_22_22%257D%255D) named `gcp<testName>-*`
  - [Managed certificates](https://console.cloud.google.com/net-services/loadbalancing/advanced/sslCertificates/list?project=spinnaker-community&organizationId=912934373776&sslCertificateTablesize=50&sslCertificateTablequery=%255B%257B_22k_22_3A_22domain_22_2C_22t_22_3A10_2C_22v_22_3A_22_5C_22localhost_5C_22_22%257D%255D) that are _**not**_ `builds.spinnaker.io` (!)

## Deleting Obsolete Artifacts

The following jobs assist in removing old artifacts created during the build process:

- [Admin_DailyJanitor](https://builds.spinnaker.io/view/5%20Admin/job/Admin_DailyJanitor/)
- [Admin_AuditBoms](https://builds.spinnaker.io/view/5%20Admin/job/Admin_AuditBoms/)
- [Admin_DeleteObsoleteArtifacts](https://builds.spinnaker.io/view/5%20Admin/job/Admin_DeleteObsoleteArtifacts/)

## Troubleshooting Playbook

Check whether the failure happened during the _build_ or the _test_ phase:

1. Click the failing Flow.

   ![](troubleshooting - base - 10 - flow.png)

1. Click for the most recent failing build.

   ![](troubleshooting - base - 20 - mostRecent.png)

1. Click through to the failing phase.

   ![](troubleshooting - base - 30 - phase.png)

### Build Failures

1.  The build phase uses many subshells to perform its work in parallel. Use the `Console Output`
    to help narrow down which step of the build has failed, and use the collected logs to view more information
    on what specificially went wrong.

        ![](troubleshooting - build - 10 - consoleOutput.png)

1.  The Console Output prints out after each completion how much work is still remaining.

    ![](troubleshooting - build - 20 - buildSteps.png)

1.  Frequently, the build error will be printed out directly to the Console Output, but sometimes this output can be hard to read. View the raw file directly using the Build Artifacts link from Step 1.

    ![](troubleshooting - build - 30 - failedOutput.png)

#### Common Build Failures

##### Bintray Conflicts

If an artifact is uploaded to the Bintray repository but never published
(either because of a transient Bintray error or an interrupted build), you'll
get an error like this:

> Bintray API Request 'create version 0.20.0-20200512192702' failed with HTTP response 409 Conflict

Follow these steps to delete the artifact and resolve the issue:

1. Navigate to the specific version [in the Bintray
   repository](https://bintray.com/beta/#/spinnaker-releases/jars?tab=packages)

1. Click on the Spinnaker repository that had the failure. (If you don't see
   it, click to the next page; there are only 10 items per page for some reason.)

1. Click on the specific version that had the issue.

1. Click "Actions" in the upper right and select "Edit".

1. On the next page, click the "Delete" link in the upper right. It
   will look like nothing happened, but after 10 seconds or so, the page will
   refresh and the version will be gone.

Now that the conflict has been removed, you can restart the build.

### Test Failures

1. View the Test Results Overview.

   ![](troubleshooting - test - 10 - testResultsOverview.png)

1. Identify the failing test.

   ![](troubleshooting - test - 20 - failingTest.png)

1. Identify which step in the test is failing.

   ![](troubleshooting - test - 30 - failingStep.png)

1. It can sometimes help to view the last call that was made prior to that stage failing.

   ![](troubleshooting - test - 40 - failingDetails.png)

### Connecting to the Jenkins VM

Members of the `jenkins-debuggers@spinnaker.io` group have access to SSH directly to the Jenkins VM. You can connect to the instance with this command:

```bash
$ gcloud compute ssh --project spinnaker-community jenkins-transfer --zone us-central1-f --ssh-flag "-L 4040:test-jenkins:8080"
```

The extra `--ssh-flag` establishes a tunnel to the `test-jenkins` instance, which is used to trigger some integration tests. You can view this instance at [http://localhost:4040](http://localhost:4040) after the connection is established.

#### Change to `jenkins` user

All processes are run as the `jenkins` user and most of the useful links are in `/home/jenkins`. Switch to it with:

```bash
$ sudo su - jenkins
```
