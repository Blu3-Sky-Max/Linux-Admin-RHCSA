# 💾 Linux Storage: Partitioning & Persistent Mounting

**Author:** Usman O. Olanrewaju (Blu3 Sky)  
**Date:** 2025/01/24  
**Environment:** Fedora (Host), Rocky Linux (Lab), Framework 13

---

## 📝 Lab Overview
This lab documents the process of managing block devices, creating partitions, and configuring the filesystem for persistence. This is a core skill required for the **RHCSA** certification.
![Partitioning Process](Images-Videos/storage-management.jpeg)
### 1. Disk Identification
First, I used `lsblk -f` to identify the target disk (e.g., `/dev/sda`) and verify if any file systems or UUIDs were already present.
**Output:**
```bash 
$ lsblk -f
NAME        FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
/dev/sda
```
### 2. Disk Partitioning with `parted`
I used the `parted` utility for low-level partitioning. 
*Note: Commands like `mkpart` must be executed within the `parted` interactive shell.*

**Steps taken:**
- `mklabel gpt`: Set the partition table type.
- `mkpart`: Defined the partition type and size.
```bash
# parted /dev/sda
(parted) mklabel gpt
(parted) mkpart primary  xfs 1MiB   18.4MiB
(parted) mkpart seconday ext4 18.4MiB 37.2MiB
(parted) quit
```
### 3. Formating with `mkfs`
After creating the partition, I formatted it with the ext4 filesystem. 
**steps taken:** 
```bash
mkfs.ext4 /dev/sda2
mkfs.xfs -f /dev/sda1 
```
### 4. Persistent Mounting (/etc/fstab)
To ensure the drive mounts automatically on system boot, I first found the partition's UUID with lsblk -f.
**output:** 
```bash
$ lsblk -f
NAME        FSTYPE FSVER  LABEL         UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
/dev/sda
└─/dev/sda1 xfs                  f24fd142-6bdd-4b9e-9941-2fc584dd53e3
└─/dev/sda2 ext4    1.0          d0040f42-c802-4c14-a1f4-66cbad5681f8
```
To ensure the drive mounts automatically on system boot, I added the **UUID** of the partition to the `/etc/fstab` configuration file.

**Configuration Format:**
`UUID=f24fd142-6bdd-4b9e-9941-2fc584dd53e3   /data xfs defaults 0 0`


### 5. Verification
I tested the mount configuration without rebooting by unmounting the drive and running the mount-all command:
```bash
sudo umount /data
sudo mount -a
lsblk -f
 ```
 ***Final Ouput:***
 ```bash
 $ lsblk -f
NAME         FSTYPE  FSVER  LABEL     UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
/dev/sda
└─/dev/sda1  xfs                   f24fd142-6bdd-4b9e-9941-2fc584dd53e3    18.2G     5%     /data
└─/dev/sda2  ext4                  d0040f42-c802-4c14-a1f4-66cbad5681f8
```

## Case 2: Box Storage (VirtualBox Disk)

Before partitioning, a secondary virtual disk was attached via VirtualBox settings.

### 📸 VirtualBox storage configuration:  
![VirtualBox Setup](Images-Videos/Vbox-Disk-Setup.jpeg)

The disk appeared inside the guest as a new block device.

### 📸 Disk identification using lsblk:  
![Disk identification](Images-Videos/vbox-mounting.jpeg)

### 📸 Persistent Mount Configuration
![vbox persistence](Images-Videos/Vbox-persistence.jpeg)
