# NVMesh 3.4 Release Notes <img src="./rn-media/NVIDIA_logo.png" style="width: 18%; height: auto;" alt="The NVIDIA logo." />

<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

**For internal distribution only.**

# Change log

| Version | Date | Release | Soul |
| :-----: | :-----: | :-----: | ----- |
| 1.0 | <nobr>2025-02-26</nobr> | 3.4.0 | First release of NVMesh 3.4.0. |

# General

The main objective of the NVMesh 3.4.0 release is to provide some key resilience-related improvements. The release of the first functionally complete version of managed non-disruptive upgrade (mNDU) is the main enhancement.

See [Release Index](https://nvidia.atlassian.net/wiki/spaces/NSV/pages/2831793511/Release+index) for package details.

# Functionality

## Managed NDU Completion and Optimizations

[NVMESH-2557](https://jirasw.nvidia.com/browse/NVMESH-2557) \- The scope of mNDU now includes upgrading management itself, the nvmesh-upgrader agents and the interopDB. The end-to-end upgrade is initiated by upgrading and restarting a single management. Then this upgraded management can be instructed to upgrade the rest of the cluster.
- For previous upgrade agents, i.e., prior to NVMesh 3.4.0, they will not auto-upgrade, so they will need to be restarted manually on all nodes running NVMesh.

[NVMESH-6594](https://jirasw.nvidia.com/browse/NVMESH-6594) \- Multiple clients can now be upgraded concurrently, i.e., in parallel instead of one by one. In addition, mNDU does not stop on a single upgrade failure. Instead, it stops after some user-set number of failures.

## TOMA Field Resilience Improvements

A few options have been added to the `toma_rpc` application to facilitate overcoming field issues. These options were added as a means to overcome reversion of specific TOMA issues that have been fixed in the interim. Nevertheless, these options may be useful in certain unexpected scenarios instead of restarting TOMAs and thus provide SREs with additional optionality.

[NVMESH-7068](https://jirasw.nvidia.com/browse/NVMESH-7068) \- Addition of a `toma_rpc` command to instruct the local TOMA to stop being the leader.

[NVMESH-7072](https://jirasw.nvidia.com/browse/NVMESH-7072) \- Addition of a `toma_rpc` command to instruct the TOMAs to resend all volume statuses to management to resynchronize them. The command can also be limited to a specific volume.

## NVMesh Observability Enhancements

Multiple enhancements have been made to the NVMesh exporter as described in the EPIC, [NVMESH-5849](https://jirasw.nvidia.com/browse/NVMESH-5849) providing enhanced observability for memory usage, VPG consumption, SERJIO usage and TOMA RAFT information.

Some `/proc` additions and changes have been implemented as part of these enhancements.

## REST API for Metadata Management

Prior to NVMesh 3.4.0, it was possible to insert arbitrary fields in volume definitions through REST CRUD operations as long as they did not collide with fields needed by management. To make this more robust, only management fields are allowed in the base volume object hereon. Now, user-defined fields can only be set within the volume's metadata section, [NVMESH-5320](https://jirasw.nvidia.com/browse/NVMESH-5320). 

Volume metadata can also be managed via the CLI, [NVMESH-7010](https://jirasw.nvidia.com/browse/NVMESH-7010).

## Integrated OTEL Support

Integrated management support for OpenTelemetry (OTEL) traces has been added. It runs auto-instrumentation for several adjunct components, i.e., MongoDB, Kafka and NodeJS, which generate a significant amount of traces. NVMesh management itself generates a small amount of traces reporting on the the internal management queue length.

## CPU Pinning and "Noisy Neighbor" Reduction

NVMesh 3.4.0 introduces new options for pinning IO of specific volumes to specific CPU cores. This is useful for machines running multiple applications requiring different volumes that are CPU-core separated, e.g., when a volume is used only by a specific container or VM and it is pinned to specific CPU cores. In that case, it makes sense to pin the IO to that volume the same cores, [NVMESH-6156](https://jirasw.nvidia.com/browse/NVMESH-6156).

## Performance Improvements for Ethernet Multi-Rail Environments

In Ethernet multi-rail environments, many connection attempts will fail. Improvements were made to reduce the affect this has during error situations so that reconnection and IO resumption is significantly faster, [NVMESH-7778](https://jirasw.nvidia.com/browse/NVMESH-7778).

## Perform Improvement to Drive Formatting

Drive formatting time on multi-drive servers is improved by performing multiple formats in parallel, [NVMESH-7337](https://jirasw.nvidia.com/browse/NVMESH-7337).

## NVMesh CSI Driver Update, version 1.9.2

This CSI Driver supports both this version of NVMesh upstream and the following earlier versions, NVMesh 2.7.2-HF16+, NVMesh 3.3.1-HF7+ and NVMesh 3.3.2-HF3+.

### New Features

[NVMESH-XXXX] \- TBD

### Bug Fixes

[NVMESH-XXXX] \-  TBD

### Compatibility

* NVMesh 2.7.2, 3.3.1, 3.3.2, 3.4.0
* (TBD) Kubernetes 1.22 \- 1.31

# CLI Changes

## Command Changes

* **–description** added to **user** **create**/**update** and to **config-profile create**/**update** commands

## Alpha Features

The following features are considered Alpha and not intended for use by end-users:

* An alpha feature has been added enabling mass or bulk operations on objects using shell-style ranges such as “**volume create –name test-volumes-{prod,stage}-{0..100} …**”. The use case for this is primarily testing related. This should not be used for production operations.

* **upgrade-agent, upgrade, upgrade-step, component, release, platform** \- these are new commands related to NDU. These are partial implementations that are not qualified for production operations.

# Resolved Issues

<!--
Template for new table entries
| [NVMESH-](https://jirasw.nvidia.com/browse/NVMESH-) | | |
-->
| Ticket | Description | Comments |
| :-----: | :---- | :---- |
| [NVMESH-5138](https://jirasw.nvidia.com/browse/NVMESH-5138) | Bug fix for Grace CPU when the IOMMU is enabled. | |
| [NVMESH-5412](https://jirasw.nvidia.com/browse/NVMESH-5412) | Improve performance for local drive operations.| |
| [NVMESH-5956](https://jirasw.nvidia.com/browse/NVMESH-5956) | Improve cold recovery to handle additional error cases of media errors.| |
| [NVMESH-6061](https://jirasw.nvidia.com/browse/NVMESH-6061) | Correct nvmesh-utils installation issue. | |
| [NVMESH-6337](https://jirasw.nvidia.com/browse/NVMESH-6337) | Reload systemd daemon as part of RPM post-install. | |
| [NVMESH-6338](https://jirasw.nvidia.com/browse/NVMESH-6338) | Error handling improvements to client service startup. | |
| [NVMESH-6554](https://jirasw.nvidia.com/browse/NVMESH-6554) | TOMA networking did not handle an EWOULDBLOCK return from a call to sendto. | |
| [NVMESH-6574](https://jirasw.nvidia.com/browse/NVMESH-6574) | Correct nvmesh_update kernel parsing. | |
| [NVMESH-6712](https://jirasw.nvidia.com/browse/NVMESH-6712) <br> [NVMESH-6802](https://jirasw.nvidia.com/browse/NVMESH-6802) <br> [NVMESH-7253](https://jirasw.nvidia.com/browse/NVMESH-7253) | Improve handling of detaching of deleted volumes during restarts. | |
| [NVMESH-6726](https://jirasw.nvidia.com/browse/NVMESH-6726) | Fix incorrect iostats latency units, off by 10x. | |
| [NVMESH-6786](https://jirasw.nvidia.com/browse/NVMESH-6786) | Fix SoftiWarp race condition that causes a kernel crash. | |
| [NVMESH-6788](https://jirasw.nvidia.com/browse/NVMESH-6788) | Fix a client crash when the IOMMU is enabled. | |
| [NVMESH-6837](https://jirasw.nvidia.com/browse/NVMESH-6837) | Improve connectivity times upon IP address change. | |
| [NVMESH-7022](https://jirasw.nvidia.com/browse/NVMESH-7022) | Avoid soft lockups and reduce the time to IO enabled when the IOMMU is enabled. | |
| [NVMESH-7054](https://jirasw.nvidia.com/browse/NVMESH-7054) | Prevent kernel crash in SoftiWarp upon a multi-disaster scenario. | |
| [NVMESH-7288](https://jirasw.nvidia.com/browse/NVMESH-7288) | Revert changes made that increased mNDU IO-disabled time. | |
| [NVMESH-7313](https://jirasw.nvidia.com/browse/NVMESH-7313) | Reduce redundant SIW trace message, "Nothing to receive". | |
| [NVMESH-7772](https://jirasw.nvidia.com/browse/NVMESH-7772) | Fix crash due to race condition in the target. | The bug may have been introduced in the development of 3.4.0, so may be redundant to note it. |
| [NVMESH-7778](https://jirasw.nvidia.com/browse/NVMESH-7778) | Improving handling of TCP_CLOSE in the SoftiWarp stack. | This improves error handling performance and IO disabled times when using SoftiWarp. |
| [NVMESH-7797](https://jirasw.nvidia.com/browse/NVMESH-7797) | Improve TOMA network path selection for RAFT messages to increase robustness.| |
| [NVMESH-](https://jirasw.nvidia.com/browse/NVMESH-) | | |

# Known Issues

| Ticket | Description | Workaround & Comments |
| ----- | :---- | :---- |
| [NVMESH-7269](https://jirasw.nvidia.com/browse/NVMESH-7269) | The manual upgrade option appears as an option in the CLI, while in practice it will be rejected as an incorrect option by management.| The manual mode is not a product feature, rather used for debug. |
| [NVMESH-7745](https://jirasw.nvidia.com/browse/NVMESH-7745) | When mNDU is performed on a client that is encrypting a volume, the encryption may fail. | Redo the encryption. |
| [NVMESH-7755](https://jirasw.nvidia.com/browse/NVMESH-7755) | When mNDU is performed on a client that is encrypting a volume, that volume may remain attached in limbo on the client, in the atom state indefinitely. | Reboot the node to clean the state. |
| [NVMESH-7826](https://jirasw.nvidia.com/browse/NVMESH-7826) | On nodes with IOMMU enabled, with some kernels and with some drives, unbinding and then binding a drive to an NVMe driver, either the built-in kernel one or NVMesh's driver, may cause corrupt memory writes. | This behavior is not related to NVMesh directly. <br><br> Newer kernels such as 6.8 and 6.14 do not appear to exhibit this behavior, so it might be isolated to a few kernels. <br><br> Using strict IOMMU, i.e., setting the kernel command line parameter `iommu.strict=1`, prevents this, but affects performance significantly and so is not recommended. |

[Documentation Repository](https://gitlab-master.nvidia.com/excelero/nvmesh-documentation)

# Support Matrix Update

The updated NVMesh support matrix is available at [NVMesh Support Matrix](https://confluence.nvidia.com/display/NSV/NVMesh+Support+Matrix).

**Note:** Kernels from 6.8.0 up until 6.14.6 suffer from a kernel workqueue crash in ​​cma\_netevent\_work\_handler, as described [here](https://bugzilla.redhat.com/show_bug.cgi?id=2363273). NVMesh is incompatible with these kernels ([NVMESH-6447](https://jirasw.nvidia.com/browse/NVMESH-6447)) as is. The NVMesh team has inserted a patched version of the relevant non-NVMesh kernel modules to fix this issue and make NVMesh compatible.

# Upgrade

Upgrading from this version to future versions will be best conducted using the mNDU feature, see above for more details.

Upgrading from versions prior to NVMesh 3.2.0-HF2 is not possible. Upgrading from 3.2.0-HF2 is with a cold upgrade. From NVMesh 3.3.0 and onwards, it is recommended to perform upgrades using mNDU. For these versions, hot upgrade is supported.
