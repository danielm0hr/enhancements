---
title: containerd-windows-container-runtime
authors:
  - "@selansen"
reviewers:
  - "@openshift/openshift-team-windows-containers"
approvers:
  - "@aravindhp"
  - "@mrunalp"
creation-date: 2021-11-19
last-updated: 2021-12-05
status: implementable
---

# containerd - Windows container runtime

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [OpenShift-docs](https://github.com/OpenShift/OpenShift-docs/)

## Summary

This enhancement intends to allow customers to bring up Windows nodes with containerd
as the default runtime from OpenShift 4.11 onwards. When customers upgrade clusters to
OpenShift 4.11, container runtime will be migrated from Docker to containerd

## Motivation

In Kubernetes, the kubelet talks to a container runtime using the Container Runtime Interface. From
Kubernetes 1.24 onwards dockershim will be removed from kubelet code via upstream [remove dockershim from kubelet](https://github.com/kubernetes/kubernetes/pull/97252).
At present WMCO uses Docker as the default runtime. The goal is to make containerd the default
runtime  for Windows containers and move away from Docker before dockershim has been decoupled from kubelet.

### Goals

* Make containerd the default container runtime on Windows nodes.
* When WMCO is upgraded from 5.x.z to 6.x.z, the runtime on all existing Windows nodes will be migrated to
  containerd.  

### Non-Goals

* Uninstalling Docker runtime is not part of this enhancement.
* Allowing the user to choose a container runtime is not supported.
* Downgrade is not supported.

## Proposal

To make containerd the default runtime, containerd should be started as a Windows
service before starting kubelet as Windows service. While starting kubelet, the parameters `container-runtime` and
`container-runtime-endpoint` should be configured. The remaining WMCO workflow of bringing Windows nodes into the
Kubernetes cluster has no impact other than the proposed changes. MachineSet and BYOH (Bring Your Own Host) upgrade
details are discussed in the upgrade section. There is no difference between MachineSet and BYOH on how we enable
containerd as the default runtime.

### Golden Image

vSphere and BYOH use golden images to create Windows VMs. Customers can create a new Windows golden image excluding Docker
runtime or continue to use the same Windows image with Docker. Once containerd becomes the default runtime, WMCO will
support Windows golden image with or without Docker.

### Containerd Migration plan

Containerd will be the default runtime in OpenShift 4.11 for Windows. For development and testing during the
OpenShift 4.10 / WMCO 5.0.0 cycle, we plan to introduce a WMCO CLI option which allows us to choose the runtime.
This CLI option will be used in the release of the WMCO community operator for OpenShift 4.10 to allow customers
to try out Windows nodes with containerd.

### User Stories

Stories can be found within the [Windows Containers: containerd epic](https://issues.redhat.com/browse/WINC-505)

### Justification
We have explored containerd, CRI-O, and Mirantis dockershim as alternative runtimes. After looking into business and
engineering details, we conclude containerd stands out to be the best solution for WMCO.

Advantage of using containerd:
* Containerd is widely adopted and supported by the open-source community.
* With minimal effort, we get new features and bug fixes from the community and enterprise.
* Microsoft is a major contributor in Windows containerd development and moved to containerd as the default
runtime for their Kubernetes offerings.
* Most of the Windows-supported Kubernetes orchestrators already moved to containerd.

Disadvantage of using other runtimes:
* Mirantis supported dockershim will continue to use Docker.
* From the business point of view subscribing to Mirantis for dockershim support is not an ideal situation as Mirantis
is our key competitor.
* The amount of time and engineering efforts involves in making CRI-O to work for Windows is huge. This doesn't
do business justification due to the high cost and time.
* Less traction from the community to develop CRI-O for Windows.

## Design Details

We plan to ship containerd 1.6.0 with WMCO. Containerd will be built as a submodule and bundled into the WMCO package.
As part of the WMCO Windows instance configuration workflow, Containerd service will be started before kubelet starts as kubelet
requires containerd to be running. Once containerd is started as Windows service, the remaining services will start as the same
way as they do today. Kubelet's dependency on Docker runtime will be removed and replaced by containerd. A separate folder will be
created under C:\k\ to store containerd config and related files. In the BYOH case, even after containerd becomes the default
runtime Docker will continue to run. Users can choose to uninstall Docker or use it for non-Kubernetes related requirements.
As Docker service is handled by the user, it is up to the user to decide if Docker should continue to run or not.
When it comes to MachineSet, the user can choose a VM image that doesn't have Docker and containerd will be the only runtime.
If the MachineSet uses a VM image that comes with Docker, containerd will run along with Docker service. But Kubernetes will
use only containerd. Customers can stop the Docker service if it is no longer needed. Even if Docker service gets started at any time,
it can co-exist with containerd.

Steps to install containerd as service:
- scp containerd/related executables into Windows VM
- copy the files into `C:\k\bin` folder
- create containerd config file
- register containerd as service
- start containerd as a Windows service
- make kubelet service dependent on containerd service instead of Docker

## Network changes
Current CNI/IPAM will be used for containerd and no changes will be made to HNS-Network and HNS-Endpoint creation
steps. The config file which will be used by containerd will point to the same CNI/IPAM executables.

## WMCO CLI option
WMCO CLI option 'runtime' will be used to choose containerd as the runtime during the OpenShift 4.10 / WMCO 5.0.0
development cycle.  In the 4.10 community operator, it will be specified as `--runtime containerd`. In the absence of
this flag, Docker will become the default runtime and this is how we will release WMCO 5.0.0. We will remove this
flag in 4.11 and make containerd the default runtime in WMCO 6.0.0. At any given point in time, we have one runtime
support and will not allow switching between runtime.

## Containerd logging
Containerd can be started with parameters in which we can enable logging and specify the file path to
log warnings/errors. Log files will be stored at c:\var\log\containerd. Pod logs will be stored in the
same location (C:\var\log\pods)

## Upgrade
WMCO supports upgrade. Please refer to the [WMCO-Upgrade](https://github.com/openshift/enhancements/blob/master/enhancements/windows-containers/windows-machine-config-operator-upgrades.md)
enhancement proposal document to learn more about WMCO upgrade.

The procedure for an upgrade that includes migration to containerd is as follows:
1) MachineSet
   * As part of upgrade Machine object gets deleted, resulting in the drain and deletion of the Windows node.
   * New Windows VM will be created.
   * The upgraded WMCO instance will configure Windows VM with containerd as the default runtime.
   * Once the Windows node joins the Kubernetes cluster, the user can deploy pods on the Windows node.
2) BYOH ( Bring Your Own Host)
   * Upgrade process will uninstall kubelet, kube-proxy, CNI, and hybrid-overlay components that were installed by WMCO.
   * Windows OS-specific configurations like HNS-Network that were created by WMCO instance would be deleted.
   * Once cleanup is done, WMCO instance will install containerd along with kubelet, kube-proxy, CNI and
     hybrid-overlay.
   * Kubelet will start as a service. Containerd will be the default runtime.
   * Once the Windows node joins the Kubernetes cluster, the user can deploy pods on the Windows node.
   * If the Docker service is present, it will continue to run.

### Risks and Mitigations

* If the cluster is upgraded, and the new version introduces an issue due to containerd
  that should be addressed because Docker runtime support won't be available in kubelet. We work closely with the
  containerd open source community and try to resolve it as soon as possible. We aim to fix any critical issue in the
  upstream first and then update the containerd submodule to point to the latest containerd upstream version. If there
  is any critical issue that needs to be addressed as soon as possible, as an exception we carry patch(es) and then
  follow up with upstream.
* As we don't support downgrade, reverting to an older version is not possible. To overcome
  any issues, we introduce this feature as part of the 5.0 community operator so that this feature
  can be well tested before we make it default in WMCO 6.0 release. We will use two test jobs to test all e2e test cases
  in the 4.10 community branch(OKD) and 4.10 branch(ccm OCP)to capture any regression.
  This will help us to identify any issues well before containerd becomes default runtime.
* containerd doesn't support image-pull-progress-deadline as of now. There is a work in progress PR, [support image pull progress timeout](https://github.com/containerd/containerd/pull/6150),
  that addresses this shortcoming. Until this gets merged, if Windows image pull takes more time than the default value, we will
  run into to image pull timeout error. To overcome this issue, Customers can bake required base container images into
  Windows golden image. If customers still notice timeout error, they should log in to Windows VM and install `ctr or crictl`
  command-line tool. Users can use one of these tools to pull the container image manually. Upon successful image download,
  the user can create pods.
* Currently Windows_exporter has been used to collect metrics from Windows node. We do
  have containerd support in [Windows exporter v0.16.0](https://github.com/prometheus-community/Windows_exporter/releases/tag/v0.16.0)
  As of now, there is no feature parity mismatch between Docker and containerd.

#### Removing a deprecated feature
Once containerd becomes the default runtime, Docker will no longer be needed in Kubernetes stack. This is discussed in
[design section](#design-details).

### API Extensions

N/A

### Operational Aspects of API Extensions

N/A

#### Failure Modes

Failure to start containerd service in Windows node will put the node into 'not ready' state. We will not be able to
schedule any pods into the Windows node.

#### Support Procedures

If containerd service doesn't get started, kubelet will not be in ready state. Windows node will not be able to schedule
any pods. Containerd logs should be collected via must-gather script. We will check containerd service status and update the
WMCO event log if the service is not up and running. In addition, we will be generating a Kubernetes event.

### Test Plan

We will use one OKD job and one ccm OCP job to test WMCO with the containerd as the runtime using the --runtime CLI option.
This is for Openshift 4.10 and in OpenShift 4.11 we will migrate all the jobs to containerd.
All other e2e tests will run as is, covering the currently supported platforms with the Docker runtime. We should make sure
there is no regression due to containerd runtime. Containerd is agnostic to the platform so testing on any platform should be fine.

### Graduation Criteria

This enhancement started as tech preview.

#### Dev Preview -> Tech Preview

The community WMCO 5.0.0 will be released with containerd as the runtime allowing customers to preview the feature using OKD / OCP 4.10.

#### Tech Preview -> GA

Red Hat certified WMCO 6.0.0 will be released against OCP 4.11 with containerd as the default runtime removing all support for Docker.

### Upgrade / Downgrade Strategy

The upgrade is already discussed in the design section. Downgrades are [not supported](https://github.com/operator-framework/operator-lifecycle-manager/issues/1177)
by OLM.

### Version Skew Strategy
We plan to maintain parity with the upstream [containerd](https://github.com/containerd/containerd/releases)
As part of the existing submodules update process, containerd will also be updated to a newer version.

## Implementation History

v1: Initial Proposal

## Alternatives

There are few alternatives, but they are either not cost-effective or depend on the competitor's less modular
components.
* Implementing CRI-O runtime for Windows involves huge engineering effort and there is no community
  support (most community supporters already moved to containerd).
* There is an effort going on to continue to use dockershim and Docker runtime. As kubelet is going to
  remove dockershim specific code, we still have to come up with the design change to make it work from Kubernetes 1.24
  onwards.

## Drawbacks

There are few drawbacks moving to containerd runtime. They are discussed in the [risks section](#risks-and-mitigations).
But we address them with mitigation plan.