# Status Quo

There are many ways to build Kubernetes clusters. 
This page covers how AKS builds Kubernetes nodes.

## Overview

A user submits a description of an AKS cluster via REST API.
The user provides configuration like VM size, Linux or Windows OS,
and other parameters. 

AKS uses those parameters to choose a VM image. Our VM image 
doesn't contain user data, so it requires further customization on
first boot.

AKS takes the input from the user and passes it through functions
to generate first boot provisioning scripts for users. For Linux,
AKS applies these scripts using cloud init and Azure's custom script
extension.

AKS builds weekly VM images which cache as much as possible required for
all supported Kubernetes versions to reduce runtime downloads and improve
latency for users.

## Goals

We have several goals which warrant changes to this flow:

- We want to reduce provisioning latency for users. 
    - Using cloud init and CSE after VM boot adds a lot of latency.
      We should strive to bake as much as possible into our VHD.
- We want a single source of truth for versioned components on the node (VHD).
- We want to reduce maintenance cost of VHDs, and Agentbaker as a service.
- We want to allow in-place update of any node configuration.

## Problems

We have a number of issues with our current flow:

- CSE depends on waagent which does not start until fairly late in boot cycle.
  - Likely can save 5-10 seconds or more.
- We don't know which Kubernetes version a node will use until runtime, so we can't start kubelet earlier.
  - Kubelet start depends on CSE start, adding substantial latency.
- We do depend on data from cloud-init/cse, so we need another way to inject that earlier to the VM.
  - Potentially IMDS/userData
- CSE/cloud init as written are not known to be idempotent. This makes re-running it difficult.

We also have a number of technical issues unrelated to user-facing latency:
- VHD build is slow (~1hr)
  - Lots of time spent pulling binaries/images. Can we optimize?
- VHD release is slow (~1d)
- Releasing VHDs/service is still somewhat manual.
- Cutting release branches and doing hotfixes is error prone.
- Dependencies between code and VHD build make automatic releases difficult.

## Future?

To resolve some of these, I would like to delivers:
- 1 VHD per Kubernetes patch version, with Kubelet and all dependencies pre-installed/started.
- Node bootstrapping dependent only on IMDS for configuration.
- "Self contained VHDs" - Eliminate generation of cloud init/CSE via AgentBaker.
  - All scripts should be versioned and built into VHD.
- Node configuration daemon consuming a versioned config file.
  - This daemon would run on first boot and could be triggered manually for updates.

