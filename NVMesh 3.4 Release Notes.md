# NVMesh 3.4 Release Notes![][image1]

**For internal distribution only\!**

# Permanent Locations

The source of truth is [NVIDIA Gitlab](https://gitlab-master.nvidia.com/excelero/nvmesh-documentation/-/tree/3.4.0?ref_type=heads). Generate the [MD Version](https://gitlab-master.nvidia.com/excelero/nvmesh-documentation/-/blob/3.4.0/NVMesh%203.4%20Release%20Notes.md?ref_type=heads) by downloading from the [Google Doc Version](https://docs.google.com/document/d/1XyMPUr_EL7FPmKyGQMowB77b5nky2bHuRfNvFMJflhg/edit?usp=sharing).

# Change log

| Version | Date | Release | NVMesh Artifacts | Autodeploy Artifacts | Changes / Bug fixes |
| ----- | ----- | ----- | ----- | ----- | ----- |
| 1.0 | 2025-02-22 (TBD final) | 3.4.0 | Do we need to list the artifacts here? |  | 1st release TBD: Add link |

# General

The main objective of the NVMesh 3.4.0 release is to provide 3-way mirroring to improve NVLustre resilience for DS9 scale.

# Release Details

This release comprises the following packages (TBD: do we need to list those here or just link to the trackers and avoid the repetition?):

*  **Target, Client and Base:**   
  * [https://urm.nvidia.com/artifactory/sw-ngc-nvmesh-generic-local/3.3.2/el/8.10/x86\_64/4.18.0-553.51.1.el8.1746466718.fd884b6339.x86\_64/nvmesh-base-3.3.2-237.el8\_10.1.1089.x86\_64.rpm](https://nam11.safelinks.protection.outlook.com/?url=https%3A%2F%2Furm.nvidia.com%2Fartifactory%2Fsw-ngc-nvmesh-generic-local%2F3.3.2%2Fel%2F8.10%2Fx86_64%2F4.18.0-553.51.1.el8.1746466718.fd884b6339.x86_64%2Fnvmesh-base-3.3.2-237.el8_10.1.1089.x86_64.rpm&data=05%7C02%7Ckecohen%40nvidia.com%7Cda6563145a704b93d2c708de155939c8%7C43083d15727340c1b7db39efd9ccc17a%7C0%7C0%7C638971670170753919%7CUnknown%7CTWFpbGZsb3d8eyJFbXB0eU1hcGkiOnRydWUsIlYiOiIwLjAuMDAwMCIsIlAiOiJXaW4zMiIsIkFOIjoiTWFpbCIsIldUIjoyfQ%3D%3D%7C0%7C%7C%7C&sdata=AGz0Mn6SlFYxctChdhfc4aAr0%2FvUl949hvmaWVsX6ZQ%3D&reserved=0)  
  * [https://urm.nvidia.com/artifactory/sw-ngc-nvmesh-generic-local/3.3.2/el/8.10/x86\_64/4.18.0-553.51.1.el8.1746466718.fd884b6339.x86\_64/nvmesh-target-3.3.2-237.el8\_10.1.1089.x86\_64.rpm](https://nam11.safelinks.protection.outlook.com/?url=https%3A%2F%2Furm.nvidia.com%2Fartifactory%2Fsw-ngc-nvmesh-generic-local%2F3.3.2%2Fel%2F8.10%2Fx86_64%2F4.18.0-553.51.1.el8.1746466718.fd884b6339.x86_64%2Fnvmesh-target-3.3.2-237.el8_10.1.1089.x86_64.rpm&data=05%7C02%7Ckecohen%40nvidia.com%7Cda6563145a704b93d2c708de155939c8%7C43083d15727340c1b7db39efd9ccc17a%7C0%7C0%7C638971670170779320%7CUnknown%7CTWFpbGZsb3d8eyJFbXB0eU1hcGkiOnRydWUsIlYiOiIwLjAuMDAwMCIsIlAiOiJXaW4zMiIsIkFOIjoiTWFpbCIsIldUIjoyfQ%3D%3D%7C0%7C%7C%7C&sdata=RmNosJwcNG8MyX0Fy8PMxFcsuufUn8wYOKfCVoqusAM%3D&reserved=0)  
  * [https://urm.nvidia.com/artifactory/sw-ngc-nvmesh-generic-local/3.3.2/el/8.10/x86\_64/4.18.0-553.51.1.el8.1746466718.fd884b6339.x86\_64/nvmesh-client-3.3.2-237.el8\_10.1.1089.x86\_64.rpm](https://nam11.safelinks.protection.outlook.com/?url=https%3A%2F%2Furm.nvidia.com%2Fartifactory%2Fsw-ngc-nvmesh-generic-local%2F3.3.2%2Fel%2F8.10%2Fx86_64%2F4.18.0-553.51.1.el8.1746466718.fd884b6339.x86_64%2Fnvmesh-client-3.3.2-237.el8_10.1.1089.x86_64.rpm&data=05%7C02%7Ckecohen%40nvidia.com%7Cda6563145a704b93d2c708de155939c8%7C43083d15727340c1b7db39efd9ccc17a%7C0%7C0%7C638971670170795236%7CUnknown%7CTWFpbGZsb3d8eyJFbXB0eU1hcGkiOnRydWUsIlYiOiIwLjAuMDAwMCIsIlAiOiJXaW4zMiIsIkFOIjoiTWFpbCIsIldUIjoyfQ%3D%3D%7C0%7C%7C%7C&sdata=pEiqK7vk4jefcBh4xtXEKumKkmXYv4kEaJGE2crUWxw%3D&reserved=0)  
  *    
*  **Mgmt & Utils**  
  * [https://urm.nvidia.com/artifactory/sw-ngc-nvmesh-generic-local/3.3.2/el/8.10/x86\_64/nvmesh-utils-3.3.2-43.el8\_5.1.1578.x86\_64.rpm](https://nam11.safelinks.protection.outlook.com/?url=https%3A%2F%2Furm.nvidia.com%2Fartifactory%2Fsw-ngc-nvmesh-generic-local%2F3.3.2%2Fel%2F8.10%2Fx86_64%2Fnvmesh-utils-3.3.2-43.el8_5.1.1578.x86_64.rpm&data=05%7C02%7Ckecohen%40nvidia.com%7Cda6563145a704b93d2c708de155939c8%7C43083d15727340c1b7db39efd9ccc17a%7C0%7C0%7C638971670170814818%7CUnknown%7CTWFpbGZsb3d8eyJFbXB0eU1hcGkiOnRydWUsIlYiOiIwLjAuMDAwMCIsIlAiOiJXaW4zMiIsIkFOIjoiTWFpbCIsIldUIjoyfQ%3D%3D%7C0%7C%7C%7C&sdata=G4e4Cv40cO5HK%2BhB9d9We%2FPEoooWbsnB%2B4dBis3WYYM%3D&reserved=0)  
  * [https://urm.nvidia.com/artifactory/sw-ngc-nvmesh-generic-local/3.3.2/el/8.10/x86\_64/nvmesh-management-3.3.2-43.el8\_5.1.1578.x86\_64.rpm](https://nam11.safelinks.protection.outlook.com/?url=https%3A%2F%2Furm.nvidia.com%2Fartifactory%2Fsw-ngc-nvmesh-generic-local%2F3.3.2%2Fel%2F8.10%2Fx86_64%2Fnvmesh-management-3.3.2-43.el8_5.1.1578.x86_64.rpm&data=05%7C02%7Ckecohen%40nvidia.com%7Cda6563145a704b93d2c708de155939c8%7C43083d15727340c1b7db39efd9ccc17a%7C0%7C0%7C638971670170831974%7CUnknown%7CTWFpbGZsb3d8eyJFbXB0eU1hcGkiOnRydWUsIlYiOiIwLjAuMDAwMCIsIlAiOiJXaW4zMiIsIkFOIjoiTWFpbCIsIldUIjoyfQ%3D%3D%7C0%7C%7C%7C&sdata=d5zV48FE%2B31gQGmK2%2F%2FubtNs9mexib5sOmrl7Tp4v0k%3D&reserved=0)  
  *    
* **Other packages:**  
  * Upgrade-agent: [https://urm.nvidia.com/artifactory/sw-ngc-nvmesh-generic-local/3.3.2/el/8.10/x86\_64/nvmesh-upgrade-agent-3.3.2-2.el8\_5.1.1562.x86\_64.rpm](https://nam11.safelinks.protection.outlook.com/?url=https%3A%2F%2Furm.nvidia.com%2Fartifactory%2Fsw-ngc-nvmesh-generic-local%2F3.3.2%2Fel%2F8.10%2Fx86_64%2Fnvmesh-upgrade-agent-3.3.2-2.el8_5.1.1562.x86_64.rpm&data=05%7C02%7Ckecohen%40nvidia.com%7Cda6563145a704b93d2c708de155939c8%7C43083d15727340c1b7db39efd9ccc17a%7C0%7C0%7C638971670170849963%7CUnknown%7CTWFpbGZsb3d8eyJFbXB0eU1hcGkiOnRydWUsIlYiOiIwLjAuMDAwMCIsIlAiOiJXaW4zMiIsIkFOIjoiTWFpbCIsIldUIjoyfQ%3D%3D%7C0%7C%7C%7C&sdata=gvg1xS9pzt39%2FpS62Cn2v7OXlq9%2BbA%2BLxSVYfSOgF0o%3D&reserved=0)  
  * Monitor: [https://urm.nvidia.com/artifactory/sw-ngc-nvmesh-generic-local/3.3.2/el/8.10/x86\_64/nvmesh-monitor-3.3.2-237.linux.1.262.x86\_64.rpm](https://nam11.safelinks.protection.outlook.com/?url=https%3A%2F%2Furm.nvidia.com%2Fartifactory%2Fsw-ngc-nvmesh-generic-local%2F3.3.2%2Fel%2F8.10%2Fx86_64%2Fnvmesh-monitor-3.3.2-237.linux.1.262.x86_64.rpm&data=05%7C02%7Ckecohen%40nvidia.com%7Cda6563145a704b93d2c708de155939c8%7C43083d15727340c1b7db39efd9ccc17a%7C0%7C0%7C638971670170865156%7CUnknown%7CTWFpbGZsb3d8eyJFbXB0eU1hcGkiOnRydWUsIlYiOiIwLjAuMDAwMCIsIlAiOiJXaW4zMiIsIkFOIjoiTWFpbCIsIldUIjoyfQ%3D%3D%7C0%7C%7C%7C&sdata=3EJPda2A5DwMfVRhc43yPvL%2FfJPI5%2FMlpdNRApxPPb4%3D&reserved=0)  
  * Interop DB:  [https://urm.nvidia.com/artifactory/sw-ngc-nvmesh-generic-local/3.3.2/el/8.10/x86\_64/nvmesh-interopdb-3.3.2-11.el8\_5.1.1583.x86\_64.rpm](https://nam11.safelinks.protection.outlook.com/?url=https%3A%2F%2Furm.nvidia.com%2Fartifactory%2Fsw-ngc-nvmesh-generic-local%2F3.3.2%2Fel%2F8.10%2Fx86_64%2Fnvmesh-interopdb-3.3.2-11.el8_5.1.1583.x86_64.rpm&data=05%7C02%7Ckecohen%40nvidia.com%7Cda6563145a704b93d2c708de155939c8%7C43083d15727340c1b7db39efd9ccc17a%7C0%7C0%7C638971670170880797%7CUnknown%7CTWFpbGZsb3d8eyJFbXB0eU1hcGkiOnRydWUsIlYiOiIwLjAuMDAwMCIsIlAiOiJXaW4zMiIsIkFOIjoiTWFpbCIsIldUIjoyfQ%3D%3D%7C0%7C%7C%7C&sdata=nEyVs4DTaDqi2TvVDo%2FzqLMUC2PbrvjhWVNTMCueEyU%3D&reserved=0)

# New Features / Enhancements

## 3-way Mirroring (NVMESH-xxxx)

3-way mirroring…

* [NVMESH-2843](https://jirasw.nvidia.com/browse/NVMESH-2843) \- Lays the foundational infrastructure for executing hot upgrades in a controlled, managed fashion across NVMesh clusters.  
* [NVMESH-5774](https://jirasw.nvidia.com/browse/NVMESH-5774) \- Focuses on enhancing the usability and manageability of the mNDU system, with specific attention to InteropDB integration

## Managed NDU Optimizations

* TBD: concurrent updates

## VPG Reserved Space Reduction ([NVMESH-5186](https://jirasw.nvidia.com/browse/NVMESH-5186))

VPG Reserved Space…

## TOMA Field Resilience Improvements

## NVMesh Exporter Enhancements

## REST API for Metadata Management

## Integrated OTEL Support

## NVMesh CSI Driver Update, version1.9.2

This CSI Driver supports both this version of NVMesh upstream and the following earlier versions, NVMesh 2.7.2-HF16+, NVMesh 3.3.1-HF7+ and NVMesh 3.3.2-HF3+.

### New Features

* (TBD) [NVMESH-5286](https://nam11.safelinks.protection.outlook.com/?url=https%3A%2F%2Fjirasw.nvidia.com%2Fbrowse%2FNVMESH-5286%3FatlOrigin%3DeyJpIjoiYjM0MTA4MzUyYTYxNDVkY2IwMzVjOGQ3ZWQ3NzMwM2QiLCJwIjoianN3LWdpdGxhYlNNLWludCJ9&data=05%7C02%7Ctleibo%40nvidia.com%7Ce63eec6506c745015ccd08dda9dee183%7C43083d15727340c1b7db39efd9ccc17a%7C0%7C0%7C638853496952927447%7CUnknown%7CTWFpbGZsb3d8eyJFbXB0eU1hcGkiOnRydWUsIlYiOiIwLjAuMDAwMCIsIlAiOiJXaW4zMiIsIkFOIjoiTWFpbCIsIldUIjoyfQ%3D%3D%7C0%7C%7C%7C&sdata=aMgeTEdzCvQSfWFuH%2Fqr4vkUlcCGRl3R5JdMDLnwX%2F0%3D&reserved=0) \- feat(cryptsetup open options): Allow encryption to not be a noisy neighbor  
* 

### Bug Fixes

* (TBD) [NVMESH-5516](https://nam11.safelinks.protection.outlook.com/?url=https%3A%2F%2Fjirasw.nvidia.com%2Fbrowse%2FNVMESH-5516%3FatlOrigin%3DeyJpIjoiYjM0MTA4MzUyYTYxNDVkY2IwMzVjOGQ3ZWQ3NzMwM2QiLCJwIjoianN3LWdpdGxhYlNNLWludCJ9&data=05%7C02%7Ctleibo%40nvidia.com%7Ce63eec6506c745015ccd08dda9dee183%7C43083d15727340c1b7db39efd9ccc17a%7C0%7C0%7C638853496953137087%7CUnknown%7CTWFpbGZsb3d8eyJFbXB0eU1hcGkiOnRydWUsIlYiOiIwLjAuMDAwMCIsIlAiOiJXaW4zMiIsIkFOIjoiTWFpbCIsIldUIjoyfQ%3D%3D%7C0%7C%7C%7C&sdata=F3ZNcFgum7eOk6GTnckLhN%2FuNXbKX0TsbYffNa7E%2BEk%3D&reserved=0) \-  fix(encryption): wrong block device mapped into pod ()

### Compatibility

* NVMesh 2.7.2, 3.3.1, 3.3.2, 3.4.0  
* (TBD) Kubernetes 1.22 \- 1.31

# CLI Changes

### Command Changes

* **–description** added to **user** **create**/**update** and to **config-profile create**/**update** commands

### Alpha Features

The following features are considered Alpha and not intended for use by end-users:

* An alpha feature has been added enabling mass or bulk operations on objects using shell-style ranges such as “**volume create –name joevol-{prod,stage}-{0..100} …**”. The use case for this is primarily testing related. This should not be used for production operations.

* **upgrade-agent, upgrade, upgrade-step, component, release, platform** \- these are new commands related to NDU. These are partial implementations that are not qualified for production operations.

# Resolved Issues

1. (TBD) [NVMESH-5871](https://jirasw.nvidia.com/browse/NVMESH-5871) \- Volume stuck in detaching state 

# Known Issues (TBD)

| Ticket | Description | Workaround & Comments |
| ----- | :---- | :---- |
| [NVMESH-4924](https://jirasw.nvidia.com/browse/NVMESH-4924) | Delete a large number of volumes in parallel may cause a management crash due to NodeJS running out of memory. The default memory limit for NodeJS is 4GB. | A manual workaround is to increase the memory limit by adding the flag \--max-old-space-size (default is 4GB). For example, to increase to 15 GB, use: services/services\_common:228 in start\_node\_entity()  startupcommand="nohup node \--max-old-space-size=15000 $MDIR/$nodestartupfile &\> $outfile & echo \\$\!" |
| [NVMESH-6039](https://jirasw.nvidia.com/browse/NVMESH-6039) | Support for external drives, SATA and NVMe-over-fabrics, has been disabled. |  |
| **New (3.3.0 HF1)** [NVMESH-6139](https://jirasw.nvidia.com/browse/NVMESH-6139) | When upgrading the upgrade agent package on Ubuntu package, the postun (post-uninstall) script from the old package is executed. In version 3.3.0, this script is faulty and accidentally deletes the /opt/nvmesh/upgradeagent folder, which causes the installation to fail. | Upgrade agent: for an upgrade from 3.3.0 to 3.3.0 HF1, uninstall the upgrade agent and install the new upgrade agent, for Ubuntu only |
| **3.3.1 HF3** |  |  |
| [NVMESH-6485](https://jirasw.nvidia.com/browse/NVMESH-6485)  | Rarely, new volumes remain in a rebuilding state | Restart TOMA TBD: instructions how to identify the relevant TOMA |
| [NVMESH-6552](https://jirasw.nvidia.com/browse/NVMESH-6552)  | Volume remains attached to client if deleted while mounted | Reboot client node Side note: there is an alternative workaround of sending in a IOCTL to the nvmesh client to “force detach”. Contact NVMesh SRE for instructions. |
| [NVMESH-6560](https://jirasw.nvidia.com/browse/NVMESH-6560)  | A newly installed TOMA does not join RAFT | Stop the target Delete it from the management Restart management Start the target |
| [NVMESH-6565](https://jirasw.nvidia.com/browse/NVMESH-6565)  | Management presents a Target incorrectly as healthy when it is down | Restart Toma |
| [NVMESH-6569](https://jirasw.nvidia.com/browse/NVMESH-6569)  | Init-encryption fails if submitted during Target/Toma restart, with Error="Boot time mismatch" and Retriable=OFF | Retry Init-encryption, the flag will be set to on in an upcoming version |
| [NVMESH-6600](https://jirasw.nvidia.com/browse/NVMESH-6600) | Allocation is allowed on offline drives even when the flag is not set | The error only happens under specific conditions. Changing the conditions can be used to workaround the issue. It should not affect any current workflow. |

# 

# Documentation (TBD)

1. [NVMesh 3.3.0 CLI Guide](https://nvidia-my.sharepoint.com/:w:/p/yaniv/EbrIruh8AkZDkX8shENh9pcBGQApzea5Q0YSypTHsMIfFg?e=jP3ylM)

2. [NVMesh 3.3.0 Module Params](https://nvidia-my.sharepoint.com/:w:/p/yaniv/EUAex7O6SxhMq-87XXNOBz8ByNZiKjYGkF3ItVX_jfMu5w?e=jVeC0n)

   Module	Module Param Name		Module Param Description

   Client 		nic\_io\_stats\_block\_size	Block-size to use for NIC iostats.json, default of 4KB.

   Client 		jam\_log\_metrics\_period	Jam periodic metrics logging interval \[sec\]; Defaults to 0, which disables logging.

   Common	io\_stats\_sizes\_hist		Sizes of the buckets for iostats histogram, default of 8 buckets, sizes 1 to 128, power of 2\.

   Server		nic\_io\_stats\_block\_size	Block-size to use for NIC iostats.json, default of 4KB.

3. [REST API](https://gitlab-master.nvidia.com/excelero/nvmesh-documentation/-/blob/3.2.0/NVMesh%203.2.0%20REST.html?ref_type=heads) 

4. [NVMesh 3.3.0 User Guide](https://nvidia-my.sharepoint.com/:w:/p/yaniv/EXdFk9hA0kRPlOo-yJ2GDGUBh-63PgETjgakHMEPwhNfCg?e=yXNGoH)

[Documentation Repository](https://gitlab-master.nvidia.com/excelero/nvmesh-documentation)

# Support Matrix Update (TBD)

The updated NVMesh support matrix is available at [NVMesh Support Matrix](https://confluence.nvidia.com/display/NSV/NVMesh+Support+Matrix).

**Note:** Kernels from 6.8.0 up until 6.14.6 suffer from a workqueue crash in ​​cma\_netevent\_work\_handler, as described [here](https://bugzilla.redhat.com/show_bug.cgi?id=2363273). NVMesh is incompatible with these kernels ([NVMESH-6447](https://jirasw.nvidia.com/browse/NVMESH-6447)) as is. The NVMesh team has inserted a patched version of the relevant non-NVMesh kernel modules to fix this issue and make NVMesh compatible.

# Upgrade

Upgrading from this version to future versions will be best conducted using the mNDU feature, see above for more details.

Upgrading from previous versions is **limited to NVMesh 3.2.0 HF2**. This upgrade must be a cold update, as follows:

1. It should be started from a functional system from the target's perspective, i.e., there should be an active TOMA RAFT leader.  
2. Stop all management instances.  
3. Wait at least 60 seconds to ensure that all updates sent from management have been digested by the TOMAs.  
4. Ensure that there is still an active TOMA RAFT leader. If not, restart the whole process, i.e., restart at least one management instance, and go back to step 1\.  
5. Stop all TOMAs, targets and clients.  
6. Update all the software.  
7. Restart management with the new version.  
8. Restart clients and targets, which will also start TOMAs, with the new version.

[image1]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAGUAAAAbCAIAAADqNtYpAAACiUlEQVR4Xu2WMWgTURiAC+9wiMGUEHXJ5mx2h67N4hBdnDpKIEuUJHdtitAKpkRwULBbJjtoSIa0l17aq7XUUiWlbQZRKIKDFS1oaIVAMTXx5f6715d3l9TXLiV5Hz/hz//+e/A+3nt3A00BDwNsQdAV4YsP4YsP4YsP4YsP4YsP4YuPM/mK60h5ffoYUxE747nnTL4UFcklpOBYOE2k166xM557OHzNbN1JLJlLledQfMl5d0xXrtvVOEYP+nq+MQTbJ6FLzNCPX5WZT7eLn+8+Wr2k6KaC8WKbxMQ866g3fR3WD2RjSbKGPv58CcXy7rOYsf4HK+j7fgVXJt8M/m0c0Q8mVcNFCZV3pkjx4Ypkl8X4Qh3AQ36/n+R0cz6fx7kkSTgPBoOk7nK5fD4f5BjcQD9FJiEVr9eLf1VVpYc64ewrvzkCS3q8fgUqU9Yi6TalaBQ1s/j7z97oYqsSW0b1eg2Kq7VJUG8Pu69CocBU0uk0ySORCD0Eud1XOBy25mjrpPOm9aDjUBecfWH2Dj7AquLWEZt4e3l0obV3SI/pi4oJ7fjYKvoFs15CG1+ftmZYu9h6P/yfL7fbjSupVIpuaLavk/zl9eXxeHAeCARGDEKhED3ahY6+AHnZXNuX3VmovNq8BZXp9zcy5ZvJRSmlX/1W2yaPJNePjbzYGiZ1GrgTuXzlcjnoQcYJIm12XwzVahU64S9Ostksos4pPYppNBrRaBQfz0wmQzcAJ/gCnpStV5519ABlDoVn0Rj1SYG3T1Lz0T3A4dH+uMreYr1z33fhnnFD4Ui867h753fuxzTj06xdUD/6orHfX1zRd776EOGLD+GLD+GLD+GLD+GLD+GLD+GLj3+c8AzGnIp8/QAAAABJRU5ErkJggg==>