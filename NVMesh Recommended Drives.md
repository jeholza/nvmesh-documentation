# NVMesh Recommended Drives

# **Revision History**

This document is also stored in this git repository: [https://gitlab-master.nvidia.com/excelero/nvmesh-documentation.git](https://gitlab-master.nvidia.com/excelero/nvmesh-documentation.git).

| Version | Date | Modified By | Description |
| ----- | ----- | ----- | :---- |
| [25.06](https://docs.google.com/document/d/1d-ZFTyzTHXnlX40iwuDBH4MhzTFs28NqDe5hI2hQORs/edit?usp=sharing) | 2025-06-19 | [Yaniv Romem IL](mailto:yaniv@nvidia.com) | First version, includes only validated drives |
| [25.07](https://docs.google.com/document/d/14aVBIaF6ihCKNC5MWc1xGTHWE5kQgSRxG87cpzzsCFc/edit?usp=sharing) | 2025-07-02 | [Yaniv Romem IL](mailto:yaniv@nvidia.com) | Added multiple Gen5 drives in validation |
| [25.12](https://docs.google.com/document/d/1F4KFGfKgC71bCh_-db-LeUHHGnTaF1yy-f3D4QSx3og/edit?usp=sharing) | 2025-12-21 | [Yaniv Romem IL](mailto:yaniv@nvidia.com) | Kioxia CD8-P removed after Kioxia reported 4k+8 support dropped. Kioxia CM7 validated. Samsung PM9D3a validated. |
| 26.01 | 2026-01-26 | [Yaniv Romem IL](mailto:yaniv@nvidia.com) | Separated Gen4 and Gen5 and moved to table format. Added Revision History instead of creating new documents. Solidigm PS1010 in validation and PS1030 validated. Putting in a git repo in md format for safe-keeping. |
| 26.02 | 2026-02-10 | [Yaniv Romem IL](mailto:yaniv@nvidia.com) | Added PS1010 after Solidigm confirmed that the controller and driver are the same as for PS1030 with the only difference being the overprovisioning. |

# **Source of Truth**

Anything that is EC-certified in the following list that is also PCI G4/5 is recommended for NVMesh usage. The following confluence link is the source of truth for all validations.

* [https://confluence.nvidia.com/display/NSV/NVMe+Devices](https://confluence.nvidia.com/display/NSV/NVMe+Devices)

For erasure-coding, it is critical that the drive has 4k+8 support, a.k.a., long blocks, protected blocks, 4104 blocks.

SED functionality is also recommended.

# **Preferred Drives**

## **Gen5 Drives**

The preferred drives in alphabetical vendor order as follows. Non-certified drives are marked in red.

| Vendor | Model | Comments |
| :---- | :---- | :---- |
| Dapustor | H5300 |  |
| Kioxia | CM7 |  |
| Micron | 6550 ION | Supports 4k+64. May need a special firmware for 4k+8. |
| Samsung | PM1743 |  |
|  | PM1753 | Currently in validation with high probability of passing. |
|  | PM9D3a |  |
| Solidigm | PS1010 | Requires firmware G75YG154. |
|  | PS1030 |  |

## **Gen4 Drives**

The preferred drives in alphabetical vendor order as follows. Non-certified drives are marked in red.

| Vendor | Model | Comments |
| :---- | :---- | :---- |
| Exascend | PD4 |  |
| Kioxia | CD8 | Kioxia has confirmed via email that the CD8 drive has 4k+8 support, so this is low risk. |
| Micron | 6500 ION | Requires special firmware. |
| Phison | EPW5970 (X1) |  |
| Samsung | PM1733 |  |
|  | PM1735 |  |
| Solidigm | P55xx |  |
|  | P56xx |  |

# **Wording for External NCPs**

The preferred drives are as follows in alphabetical order. Gen5 drives are preferred over Gen4.

1. Gen5: Dapustor H5300, Kioxia CM7, Samsung PM1743 and PM9D3a, Solidigm PS1010/1030  
2. Gen4: Exascend PD4, Micron 6500 ION, Phison EPW5970, Samsung PM1733 and PM1735, Solidigm P55xx and P56xx

Additional drives for consideration, currently under NVIDIA validation, for which final approval by NVIDIA will be needed:

1. Gen5: Samsung PM1753

Other drives for consideration for which approval by NVIDIA will be needed, but not currently in validation:

1. Gen5: Micron 6550 ION  
2. Gen4: Kioxia CD8

