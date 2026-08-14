# Incident 01 — RHEL Root Filesystem Full & Dynamic Disk Troubleshooting

## Overview

While building a multi-server RHEL lab environment, the root filesystem (`/`) reached its storage capacity.

The existing Linux LVM volume group had no free extents available, and no additional physical disk was available to add to the volume group.

The system was configured as a dual-boot environment with Windows and RHEL on separate physical disks. The second disk contained Windows storage that had unused capacity, so I investigated whether part of that capacity could be safely reallocated to Linux.

---

## Initial Storage Assessment

I first analyzed the existing storage layout using Linux disk and LVM utilities.

```bash
lsblk
sudo fdisk -l
sudo pvs
sudo vgs
sudo lvs
df -h
```

The investigation showed that:

* The RHEL root filesystem was nearly/full.
* The existing LVM volume group had no free space available.
* There was no additional physical disk available for immediate expansion.
* A significant amount of unused capacity was available on the Windows disk.
* The Windows and Linux installations were located on separate physical disks.

Because the Windows disk contained critical EFI/recovery partitions, I intentionally avoided modifying that disk.

---

## Attempted Windows Space Reallocation

I attempted to reduce the Windows volume and leave approximately **200 GB as Unallocated space**.

Windows successfully performed the shrink operation and displayed the 200 GB as Unallocated space.

However, after booting into RHEL, the newly released space was not available in Linux as expected.

This indicated that the problem was not simply the absence of free disk space, but rather how the Windows disk was structured.

---

## Root Cause Investigation

Further inspection showed that the Windows disk was configured as a **Microsoft Dynamic Disk using LDM (Logical Disk Manager)** rather than a traditional Basic GPT disk.

This created a compatibility issue with the way the available space was represented and exposed to Linux.

The goal therefore became:

> Convert the Windows Dynamic Disk to a Basic Disk while preserving the existing partitions and data.

---

## Bootable Partition Management Environment

To perform the disk conversion, I used a bootable partition-management environment.

### Secure Boot Issue

The system initially refused to boot the external bootable environment because Secure Boot was enabled.

I temporarily disabled Secure Boot in the firmware settings to allow the maintenance environment to boot.

This was performed specifically to access the disk-management environment and was not intended as a permanent security configuration change.

---

## Storage Visibility Issue — Intel VMD

After successfully booting into the partition-management environment, another issue appeared:

The internal NVMe disks were not visible, while the bootable media itself was detected.

I investigated the system storage configuration and identified that **Intel VMD (Volume Management Device)** was enabled.

The bootable environment did not have the required storage support/configuration to access the NVMe device while VMD was enabled.

I temporarily disabled VMD in the firmware settings and booted the maintenance environment again.

After disabling VMD:

* The internal NVMe disks became visible.
* The existing Windows and Linux partitions were detected.
* The disk could be managed by the partition-management environment.

---

## Resolution

The affected Windows Dynamic Disk was converted from:

```text
Microsoft Dynamic Disk
        ↓
Basic Disk
```

The existing partitions and data were preserved during the conversion.

After the conversion, the storage layout behaved as expected from both Windows and Linux.

I then returned to Windows and reduced the Windows volume by approximately **200 GB**, leaving the released capacity as:

```text
Unallocated Space
```

The system was then booted back into RHEL.

The newly available space was now visible to Linux and could be used for Linux storage expansion.

---

## Verification

I verified the resulting storage configuration using:

```bash
lsblk
sudo fdisk -l
sudo pvs
sudo vgs
sudo lvs
df -h
```

The verification confirmed that:

* The Windows disk was no longer using the Dynamic Disk layout.
* The existing Linux partitions remained available.
* The newly released capacity was visible to Linux.
* The storage was ready to be incorporated into the Linux storage architecture.

---

## Lessons Learned

This incident highlighted several important system-administration considerations:

1. **Storage capacity planning is important.**
   A root filesystem should have sufficient capacity for future growth, especially in a server lab containing multiple services and virtual machines.

2. **Disk layout matters when working across operating systems.**
   Windows Dynamic Disk/LDM storage behaves differently from traditional Basic GPT partitioning and can complicate cross-platform storage management.

3. **Firmware configuration can affect storage visibility.**
   Secure Boot and Intel VMD can affect the ability of external maintenance environments to boot correctly and access NVMe storage.

4. **Troubleshooting should be performed layer by layer.**

```text
Filesystem
    ↓
LVM
    ↓
Partition Table
    ↓
Disk Management Technology
    ↓
Firmware Storage Configuration
    ↓
Boot Environment
```

5. **Changes to production-like storage should always be preceded by a verified backup.**

---

## Outcome

The incident was resolved without rebuilding the RHEL environment.

The previously unavailable Windows capacity was successfully made available to Linux, providing additional storage for the continued development and expansion of the RHEL server lab.

The newly available capacity was subsequently used to expand the RHEL storage environment.

**Related documentation:**
See `Incident-02-RHEL-LVM-Root-Expansion.md` for the procedure used to extend the Linux LVM and root filesystem.
