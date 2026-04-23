# Logical Volume Management Storage Management 

**Author:** Usman O. Olanlanrewaju (Blu3 Sky)

**Date:** 2026/04/14

**Focus:** Partition tables, Physical Volumes, Volume Groups, Logical Volumes — full lifecycle

**Modify:2026/04/23 

---

## Table of Contents

1. [Partition Table Fundamentals](#1-partition-table-fundamentals)
2. [LVM Architecture](#2-lvm-architecture)
3. [Disk Inspection](#3-disk-inspection)
4. [Partition Table Operations with parted](#4-partition-table-operations-with-parted)
5. [Creating and Managing Partitions](#5-creating-and-managing-partitions)
6. [Physical Volume Management](#6-physical-volume-management)
7. [Volume Group Management](#7-volume-group-management)
8. [Logical Volume Management](#8-logical-volume-management)
9. [Extending the Storage Pool](#9-extending-the-storage-pool)
10. [Renaming and Reorganising Volumes](#11-renaming-and-reorganising-volumes)
11. [Resizing Logical Volumes](#10-resizing-logical-volumes)
12. [Formatting Logical Volumes](#12-formatting-logical-volumes) 
13. [Teardown — Removing LVM Objects](#12-teardown--removing-lvm-objects)
14. [Key Concepts and Common Mistakes](#13-key-concepts-and-common-mistakes)

---

## 1. Partition Table Fundamentals

A partition table tells the OS how the disk is divided. There are two formats:

| Feature | MBR (msdos) | GPT |
|---|---|---|
| Max partitions | 14 usable (3 primary + 11 logical) | 128 |
| Max disk size | 2 TB | 18 EB |
| Redundancy | None | Backup table at end of disk |
| parted label name | `msdos` | `gpt` |

> **MBR and msdos are the same thing.** MBR is the industry name. `msdos` is what `parted` calls it because IBM introduced the format with MS-DOS. Same format, two names.

---

## 2. LVM Architecture

LVM sits between raw disks and the filesystem. It does not create storage — it makes existing storage flexible to manage.That all it does 

```
HDD/SSD(/dev/sda, /dev/sdb) -->  Physical Volumes (PV)  -->  Volume Group (VG)  -->   Logical Volumes (LV)
(Physical Disks)                        (LVM labels)       (storage pool/container)        (usable slices)
``` 

**The key insight:** A VG is just a pool. Add more PVs(Physical Volumes) to grow the pool. Carve LVs(Logical Volumes)  from that pool. The total space is always bounded by real physical disks.

**Physical Extent (PE)** — the allocation unit inside a VG. Set with `-s` at `vgcreate` time. All sizes are rounded up to the nearest PE boundary. Default is 4 MiB.

---


## 3. Disk Inspection

Understanding what storage is available before touching anything.

### 3.1  List all block devices and their mount points
```bash
$ lsblk
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             8:0    1 58.6G  0 disk               #here we go this is my disk
sr0            11:0    1 1024M  0 rom
vda           252:0    0   20G  0 disk
├─vda1        252:1    0    1M  0 part
├─vda2        252:2    0    1G  0 part /boot
└─vda3        252:3    0   19G  0 part
  ├─rhel-root 253:0    0   17G  0 lvm  /
  └─rhel-swap 253:1    0    2G  0 lvm  [SWAP]
```

### 3.2  List partition details with fdisk 
```bash 
$ sudo fdisk -l 
Disk /dev/vda: 20 GiB, 21474836480 bytes, 41943040 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 7F73B667-7507-4DAD-B9AF-A4B9AD234904

Device       Start      End  Sectors Size Type
/dev/vda1     2048     4095     2048   1M BIOS boot
/dev/vda2     4096  2101247  2097152   1G Linux extended boot
/dev/vda3  2101248 41940991 39839744  19G Linux LVM


#truncated output
```

### 3.3 Check current LVM state before starting any lab
```bash  
$ sudo pvs; sudo vgs; sudo lvs; sudo vgdisplay -v
[sudo] password for blu3sky:
  PV         VG   Fmt  Attr PSize   PFree
  /dev/vda3  rhel lvm2 a--  <19.00g    0
  VG   #PV #LV #SN Attr   VSize   VFree
  rhel   1   2   0 wz--n- <19.00g    0
  LV   VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root rhel -wi-ao---- <17.00g
  swap rhel -wi-ao----   2.00g
  --- Volume group ---
  VG Name               rhel
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <19.00 GiB
  PE Size               4.00 MiB
  Total PE              4863
  Alloc PE / Size       4863 / <19.00 GiB
  Free  PE / Size       0 / 0
  VG UUID               kEPqDb-Zz4U-Ku4K-Sh0g-NXSQ-5Wjh-7CcV0D

  --- Logical volume ---
  LV Path                /dev/rhel/swap
  LV Name                swap
  VG Name                rhel
  LV UUID                4B2JIj-Zazt-iMWF-f6if-xLmA-YYaO-y4mWC6
  LV Write Access        read/write
  LV Creation host, time localhost.localdomain, 2026-01-27 12:04:43 -0500
  LV Status              available
  # open                 1
  LV Size                2.00 GiB
  Current LE             512
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:1

  --- Logical volume ---
  LV Path                /dev/rhel/root
  LV Name                root
  VG Name                rhel
  LV UUID                3Qixkt-4Yfn-DbaC-d6FV-vuyk-9BA9-VG1LtD
  LV Write Access        read/write
  LV Creation host, time localhost.localdomain, 2026-01-27 12:04:43 -0500
  LV Status              available
  # open                 1
  LV Size                <17.00 GiB
  Current LE             4351
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0

  --- Physical volumes ---
  PV Name               /dev/vda3
  PV UUID               14gTwR-Z3Wu-cUSw-r2eB-tWXH-EIlf-ASTD1z
  PV Status             allocatable
  Total PE / Free PE    4863 / 0
```
--- 

## 4. Partition Table Operations with parted

### 4.1 Print existing partition table

```bash
$ sudo parted /dev/sda print
Model: VendorCo ProductCode (scsi)
Disk /dev/sda: 62.9GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start  End  Size  Type  File system  Flags
``` 
### 4.2 Create a GPT partition table — interactive mode
Use interactive mode when you want to inspect results between each step.

```bash 
$ sudo parted /dev/sda 
GNU Parted 3.6
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mklabel
New disk label type? gpt                                                  
Warning: The existing disk label on /dev/sda will be destroyed and all data on this disk will be
lost. Do you want to continue?
Yes/No? yes                                                               
(parted) print                                                            
Model: VendorCo ProductCode (scsi)
Disk /dev/sda: 62.9GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start  End  Size  File system  Name  Flags

(parted) quit                                                             
Information: You may need to update /etc/fstab.

```

### 4.3 Create a GPT partition table — single command
```bash

$ sudo parted /dev/sda mklabel gpt ; sudo parted /dev/sda print
Warning: The existing disk label on /dev/sda will be destroyed and all data on this disk will be
lost. Do you want to continue?
Yes/No? yes                                                               
Information: You may need to update /etc/fstab.

Model: VendorCo ProductCode (scsi)                                        
Disk /dev/sda: 62.9GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start  End  Size  File system  Name  Flags
```

> **Warning:** `mklabel` destroys any existing partition table. This is intentional and irreversible. Always verify the target device before running.

---

## 5. Creating and Managing Partitions

### 5.1 Create partitions — interactive mode
Using 500MB each for each partition 

```bash
$ sudo parted /dev/sda 
GNU Parted 3.6
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mkpart                                                           
Partition name?  []? lvm                                                  
File system type?  [ext2]?    # i pressed enter and the system use the default file system ext2             
Start? 1                                                                  
End? 500                                                                  
(parted) print                                                            
Model: VendorCo ProductCode (scsi)
Disk /dev/sda: 62.9GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End    Size   File system  Name  Flags
 1      1049kB  500MB  499MB  ext2         lvm
```
### 5.2 Adding a second and third partition back-to-back (start each where the previous ended):

The LVM flag marks a partition for use by LVM. It is metadata only — it does not format the partition.

``` bash
(parted) mkpart
Partition name?  []? Usman-lvm
File system type?  [ext2]? xfs
Start? 501
End? 1001
(parted) mkpart
Partition name?  []? Usman-lvm1
File system type?  [ext2]? ext4
Start? 1002
End? 1503
(parted) print                                                            
Model: VendorCo ProductCode (scsi)
Disk /dev/sda: 62.9GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size   File system  Name        Flags
 1      1049kB  500MB   499MB  ext2         lvm
 2      501MB   1001MB  500MB  xfs          Usman-lvm
 3      1002MB  1503MB  500MB  ext4         Usman-lvm1

``` 
### 5.3 Set the LVM flag on a partition
```bash
(parted) mkpart
Partition name?  []? usman-lvm2
File system type?  [ext2]? set 4 lvm on
parted: invalid token: set
File system type?  [ext2]?
Start? 1504
End? 2005
(parted) set 4 lvm on
(parted) print
Model: VendorCo ProductCode (scsi)
Disk /dev/sda: 62.9GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size   File system  Name        Flags
 1      1049kB  500MB   499MB               lvm
 2      501MB   1001MB  500MB               Usman-lvm
 3      1002MB  1503MB  500MB               Usman-lvm1
 4      1504MB  2005MB  501MB  ext2         usman-lvm2  lvm

(parted)
```
### 5.4 Create a partition from the command line (no interactive mode)
```bash
$ sudo parted /dev/sdb  mkpart primary 1 151
Information: You may need to update /etc/fstab.
```
### 5.4 Delete partitions

```bash
$ sudo parted /dev/sdb rm 1; sudo parted /dev/sdb rm 2; sudo parted /dev/sdb print
Information: You may need to update /etc/fstab.
Information: You may need to update /etc/fstab.

Partition Table: msdos
Number  Start  End  Size  Type  File system  Flags
```
> **Note:** I switched to a different USB stick at this point — `sda` now refers to the new stick.

### 5.5 Verify with lsblk

```bash
$ lsblk
NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda             8:0    1 58.6G  0 disk 
├─sda1          8:1    1  476M  0 part 
├─sda2          8:2    1  477M  0 part 
└─sda3          8:3    1  477M  0 part 
sr0            11:0    1 1024M  0 rom  
vda           252:0    0   20G  0 disk 
├─vda1        252:1    0    1M  0 part 
├─vda2        252:2    0    1G  0 part /boot
└─vda3        252:3    0   19G  0 part 
  ├─rhel-root 253:0    0   17G  0 lvm  /
  └─rhel-swap 253:1    0    2G  0 lvm  [SWAP]
```
### 5.6 In-memory partition table (kernel view — refreshed without reboot)

```bash
$ cat /proc/partitions 
major minor  #blocks  name

 252        0   20971520 vda
 252        1       1024 vda1
 252        2    1048576 vda2
 252        3   19919872 vda3
  11        0    1048575 sr0
 253        0   17821696 dm-0
 253        1    2097152 dm-1
   8        0   61440000 sda
   8        1     487424 sda1
   8        2     488448 sda2
   8        3     488448 sda3
```

--- 

## 6. Physical Volume Management

A Physical Volume (PV) is a disk or partition with an LVM label written on it. This label is how LVM recognises the device as part of its managed storage.

### 6.1 Create a Physical Volume

```bash
$ sudo pvcreate /dev/sda1 /dev/sda2 -v
  Wiping signatures on new PV /dev/sda1.
  Wiping signatures on new PV /dev/sda2.
  Set up physical volume for "/dev/sda1" with 974848 available sectors.
  Zeroing start of device /dev/sda1.
  Writing physical volume data to disk "/dev/sda1".
  Physical volume "/dev/sda1" successfully created.
  Set up physical volume for "/dev/sda2" with 976896 available sectors.
  Zeroing start of device /dev/sda2.
  Writing physical volume data to disk "/dev/sda2".
  Physical volume "/dev/sda2" successfully created.
  Creating directory "/etc/lvm/devices/backup/"
```

> `vgcreate` also calls `pvcreate` internally. If you run `vgcreate` on a raw disk, it will initialise the PV automatically — you do not need a separate `pvcreate` step.

### 6.2 Inspect Physical Volumes

```bash
$ sudo pvs
  PV         VG   Fmt  Attr PSize   PFree  
  /dev/sda1       lvm2 ---  476.00m 476.00m
  /dev/sda2       lvm2 ---  477.00m 477.00m
  /dev/vda3  rhel lvm2 a--  <19.00g      0 
``` 

## 7. Volume Group Management

A Volume Group (VG) is a storage pool built from one or more PVs. LVs are carved from this pool.

### 7.1 Create a Volume Group

```bash
$ sudo vgcreate -vs 9 Usmanlvm /dev/sda1 /dev/sda2
  Wiping signatures on new PV /dev/sda1.
  Wiping signatures on new PV /dev/sda2.
  Adding physical volume '/dev/sda1' to volume group 'Usmanlvm'
  Adding physical volume '/dev/sda2' to volume group 'Usmanlvm'
  Creating volume group backup "/etc/lvm/backup/Usmanlvm" (seqno 1).
  Volume group "Usmanlvm" successfully created
``` 
`-s` sets the Physical Extent size (allocation unit). Default is 4 MiB.


### 7.2 Inspect Volume Groups

```bash
$ sudo vgs
  VG       #PV #LV #SN Attr   VSize   VFree  
  Usmanlvm   2   0   0 wz--n- 936.00m 936.00m # we have 936 MB free to use 
  rhel       1   2   0 wz--n- <19.00g      0 
```
```bash
$ sudo vgdisplay Usmanlvm
  --- Volume group ---
  VG Name               Usmanlvm
  System ID
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               0
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               936.00 MiB
  PE Size               9.00 MiB
  Total PE              104
  Alloc PE / Size       101 / 909.00 MiB
  Free  PE / Size       3 / 27.00 MiB
  VG UU                 CUr3tT-mJE2-DeEH-ZPed-ky30-jhGI-8ekKUr
```

--- 

## 8. Logical Volume Management

A Logical Volume (LV) is a usable slice of the VG pool. It behaves like a regular block device — you format it and mount it.

### 8.1 Create a Logical Volume

Creating without name and adding 200MB to the VG Usmanlvm

#### 8.1.1  Without a name: LVM auto-assigns lvol0, lvol1, lvol2...

```bash
$ sudo lvcreate -vL 200 Usmanlvm 
  Rounding up size to full physical extent 207.00 MiB
  Creating logical volume lvol0
  Archiving volume group "Usmanlvm" metadata (seqno 1).
  Activating logical volume Usmanlvm/lvol0.
  activation/volume_list configuration setting not defined: Checking only host tags for Usmanlvm/lvol0.
  Creating Usmanlvm-lvol0
  Loading table for Usmanlvm-lvol0 (253:2).
  Resuming Usmanlvm-lvol0 (253:2).
  Wiping known signatures on logical volume Usmanlvm/lvol0.
  Initializing 4.00 KiB of logical volume Usmanlvm/lvol0 with value 0.
  Logical volume "lvol0" created.
  Creating volume group backup "/etc/lvm/backup/Usmanlvm" (seqno 2).
``` 

#### 8.1.2 With a name and specific size
Creating name `Blue-lv` and adding 700MB to the VG Usmanlvm

```bash
$ sudo lvcreate -L 700 -n Blue-lv Usmanlvm
  Rounding up size to full physical extent 702.00 MiB
  Logical volume "Blue-lv" created.
```

### 8.2 Inspect Logical Volumes & Size available 

#### 8.2.1 Viewing Logical Volumes 
 
```bash 

$ sudo lvs Usmanlvm
  LV      VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  Blue-lv Usmanlvm -wi-a----- 702.00m                                                    
  lvol0   Usmanlvm -wi-a----- 207.00m    
```

#### 8.2.2 Viewing the Size Available 

```bash 
$ sudo vgs; sudo pvs
  VG       #PV #LV #SN Attr   VSize   VFree  
  Usmanlvm   3   2   0 wz--n-   1.37g 495.00m
  rhel       1   2   0 wz--n- <19.00g      0 
  PV         VG       Fmt  Attr PSize   PFree  
  /dev/sda1  Usmanlvm lvm2 a--  468.00m  27.00m
  /dev/sda2  Usmanlvm lvm2 a--  468.00m      0 
  /dev/sda3  Usmanlvm lvm2 a--  468.00m 468.00m
  /dev/vda3  rhel     lvm2 a--  <19.00g      0 
```
---

## 9. Extending the Storage Pool

This is the standard workflow when a VG is full and you need more space.

### 9.1 Extend a Volume Group — add a PV to the pool
```bash
$ sudo vgextend Usmanlvm /dev/sda3
  Physical volume "/dev/sda3" successfully created.
  Volume group "Usmanlvm" successfully extended
```
### 9.2 Verify the the pool size 
```bash
$ sudo vgs; sudo pvs
  VG       #PV #LV #SN Attr   VSize   VFree
  Usmanlvm   3   2   0 wz--n-   1.37g 495.00m
  rhel       1   2   0 wz--n- <19.00g      0
  PV         VG       Fmt  Attr PSize   PFree
  /dev/sda1  Usmanlvm lvm2 a--  468.00m  27.00m
  /dev/sda2  Usmanlvm lvm2 a--  468.00m      0
  /dev/sda3  Usmanlvm lvm2 a--  468.00m 468.00m
  /dev/vda3  rhel     lvm2 a--  <19.00g      0
```
### 9.3 Move a PV to a different VG

#### 9.3.1 Remove from current VG, then add to target VG 
```bash
$ sudo vgreduce -f  Usmanlvm /dev/sda3
  Removed "/dev/sda3" from volume group "Usmanlvm"
```
***Note:*** `vgreduce` succeeds here because no LV data is sitting on /dev/sda3. If any LV had extents allocated on it, the command would fail with "still in use" — you would need to pvmove the data off first before removing it

--- 

## 10. Renaming and Reorganising Volumes   

### 10.1  Rename a Logical Volume
```bash
$ sudo lvrename Usmanlvm lvol0 Blue-Sky
[sudo] password for blu3sky:
Sorry, try again.
[sudo] password for blu3sky:
  Renamed "lvol0" to "Blue-Sky" in volume group "Usmanlvm"
```
#### 10.1.1 Viewing Changes 
```bash 
$ sudo lvs Usmanlvm
  LV       VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  Blue-Sky Usmanlvm -wi-a----- 414.00m
  Blue-lv  Usmanlvm -wi-a----- 909.00m
```
 
 
## 11. Resizing Logical Volumes


### 11.1  Grow two LVs in a single chained command
 
Adding Size 200 MB each to each lvs 

```bash 
$ sudo lvresize -L +200 /dev/Usmanlvm/lvol0; sudo lvresize -L +200 /dev/Usmanlvm/Blue-lv
  Rounding size to boundary between physical extents: 207.00 MiB.
  Size of logical volume Usmanlvm/lvol0 changed from 207.00 MiB (23 extents) to 414.00 MiB (46 extents).
  Logical volume Usmanlvm/lvol0 successfully resized.
  Rounding size to boundary between physical extents: 207.00 MiB.
  Size of logical volume Usmanlvm/Blue-lv changed from 702.00 MiB (78 extents) to 909.00 MiB (101 extents).
  Logical volume Usmanlvm/Blue-lv successfully resized.
```

> **PE rounding explained:** With a 9 MiB PE size, +200 MiB becomes +207 MiB (23 × 9). The requested size is always rounded up to the next full PE boundary — LVM will never allocate a partial extent; the PE i set at the vgcreate is 9 and that the unit allocated to each block so mutiply 9 *23  same as the second one 

#### 11.1.1  Viewing increment 
```bash
$ sudo lvs
  LV      VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  Blue-lv Usmanlvm -wi-a----- 909.00m                                                    
  lvol0   Usmanlvm -wi-a----- 414.00m                                                    
  root    rhel     -wi-ao---- <17.00g                                                    
  swap    rhel     -wi-ao----   2.00g   
```
### 11.2 Shrink an LV
Reducing Blue-lv 300MB, `-L`  sets the absolute size (not a delta)
```bash
$ sudo lvreduce -L 300 /dev/Usmanlvm/Blue-lv
  Rounding size to boundary between physical extents: 306.00 MiB.
  No file system found on /dev/Usmanlvm/Blue-lv.
  Size of logical volume Usmanlvm/Blue-lv changed from 909.00 MiB (101 extents) to 306.00 MiB (34 extents).
  Logical volume Usmanlvm/Blue-lv successfully resized.
```
#### 11.2.1 Viewing Changes
```bash
$ sudo lvs  Usmanlvm
  LV       VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  Blue-Sky Usmanlvm -wi-a----- 414.00m
  Blue-lv  Usmanlvm -wi-a----- 306.00m
```
LVM cannot allocate partial extents. 300 MiB ÷ 9 MiB PE = 33.3, so LVM rounds up to 34 extents — giving you 306 MiB.
 
### 11.3 Create a second VG on a new partition
```bash
$ sudo vgcreate -s 5 Blue-Sky-Max /dev/sda4 
  Physical volume "/dev/sda4" successfully created.
  Volume group "Blue-Sky-Max" successfully created
```
#### 11.3.1 Adding more Size 
```bash
$ sudo vgextend Blue-Sky-Max /dev/sda3
  Volume group "Blue-Sky-Max" successfully extended
```

#### 11.3.2 Viewing Changes
```bash
$ sudo vgs ; sudo pvs
  VG           #PV #LV #SN Attr   VSize   VFree  
  Blue-Sky-Max   1   0   0 wz--n- 475.00m 475.00m
  Usmanlvm       3   2   0 wz--n-   1.37g 684.00m
  rhel           1   2   0 wz--n- <19.00g      0 
  PV         VG           Fmt  Attr PSize   PFree  
  /dev/sda1  Usmanlvm     lvm2 a--  468.00m 234.00m
  /dev/sda2  Usmanlvm     lvm2 a--  468.00m 162.00m
  /dev/sda3  Usmanlvm     lvm2 a--  468.00m 288.00m
  /dev/sda4  Blue-Sky-Max lvm2 a--  475.00m 475.00m
  /dev/vda3  rhel         lvm2 a--  <19.00g      0 
```
#### 11.4 Creating new LV to Second VG
Using `-l` (lowercase) to allocate by extent count rather than size.
```bash
$ sudo lvcreate -l 10  -n Blue-Sky-Max-Workacholic Blue-Sky-Max
  Logical volume "Blue-Sky-Max-Workacholic" created.
```
> **`-l` vs `-L`:** `-l 10` allocates 10 extents. With a PE size of 5 MiB: 10 × 5 = 50 MiB total. `-L 50` would produce the same result here. Use `-l` when thinking in extents, `-L` when thinking in bytes.

## 12. Formatting Logical Volumes 
LVs behave like raw block devices. Before they can hold data, they must be formatted with a filesystem, mounted, and optionally added to `/etc/fstab` for persistent mounting.

### 12.1 Format with ext4, XFS, and VFAT
```bash
$ sudo mkfs.ext4 /dev/Blue-Sky-Max/Blue-Sky-Max-Workacholic ; sudo mkfs.xfs -f /dev/Usmanlvm/Blue-lv; sudo mkfs.vfat /dev/Usmanlvm/Blue-Sky  
mke2fs 1.47.1 (20-May-2024)
Creating filesystem with 51200 1k blocks and 12824 inodes
Filesystem UUID: de6e6ebc-e9ba-43dd-b6bd-b66b6e5a739d
Superblock backups stored on blocks: 
	8193, 24577, 40961

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

meta-data=/dev/Usmanlvm/Blue-lv  isize=512    agcount=4, agsize=63360 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=1
         =                       exchange=0  
data     =                       bsize=4096   blocks=253440, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=0
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
mkfs.fat 4.2 (2021-01-31)
```
> **RHEL 10 — XFS minimum size requirement:** `mkfs.xfs` enforces a hard minimum of **300 MB** on RHEL 10. Attempting to format a smaller LV or partition will fail with `Filesystem must be larger than 300MB`. This is not documented in the Gor Ghori textbook, which assumes smaller lab partitions. Always size XFS targets at 400 MB or larger to have margin after PE rounding.

### 12.2 Create mount points
```bash 
$ sudo mkdir /usman-ext4 ~/usman-xfs /usman-vfat
[sudo] password for blu3sky: 
``` 
### 12.3 Mount manually

```bash 
$ sudo mount /dev/Usmanlvm/Blue-lv ~/usman-xfs; sudo mount /dev/Usmanlvm/Blue-Sky /usman-vfat/; sudo mount /dev/Blue-Sky-Max/Blue-Sky-Max-Workacholic /usman-ext4 
```
### 12.3.1 Verify mounts 
```bash 
$ lsblk /dev/sda
NAME                                           MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                                              8:0    1 58.6G  0 disk 
├─sda1                                           8:1    1  476M  0 part 
│ ├─Usmanlvm-Blue--Sky                         253:3    0  351M  0 lvm  /usman-vfat
│ │                                                                     /usman-vfat
│ └─Usmanlvm-Blue--lv                          253:4    0  990M  0 lvm  /home/blu3sky/usman-xfs
├─sda2                                           8:2    1  476M  0 part 
│ └─Usmanlvm-Blue--lv                          253:4    0  990M  0 lvm  /home/blu3sky/usman-xfs
├─sda3                                           8:3    1  476M  0 part 
│ └─Usmanlvm-Blue--lv                          253:4    0  990M  0 lvm  /home/blu3sky/usman-xfs
└─sda4                                           8:4    1  476M  0 part 
  └─Blue--Sky--Max-Blue--Sky--Max--Workacholic 253:2    0   50M  0 lvm  /usman-ext4
                                                                        /usman-ext4
``` 
### 12.4 Persist mounts — edit /etc/fstab

Both device paths and UUIDs are valid in fstab. UUIDs are preferred in production because they survive devicerenaming (e.g. sda -->  sdb after a reboot). For LVM volumes, the device path is stable by name, so either works. The `nofail` option prevents a boot hang if the device is unavailable.

```bash
/dev/Usmanlvm/Blue-Sky                         /usman-vfat                vfat      defaults,nofail     0 0
/dev/Usmanlvm/Blue-lv                          /home/blu3sky/usman-xfs    xfs       defaults,nofail     0 0
UUID=de6e6ebc-e9ba-43dd-b6bd-b66b6e5a739d       /usman-ext4               ext4       defaults,nofail    0 0

```
### 12.5 Unmount all three filesystems

```bash
$ sudo umount /dev/Usmanlvm/Blue-lv ; sudo umount /dev/Usmanlvm/Blue-Sky; sudo umount /dev/Blue-Sky-Max/Blue-Sky-Max-Workacholic

```
#### 12.5.1 Verify unmounted 
```bash
$ lsblk /dev/sda 
NAME                                           MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                                              8:0    1 58.6G  0 disk 
├─sda1                                           8:1    1  476M  0 part 
│ ├─Usmanlvm-Blue--Sky                         253:3    0  351M  0 lvm  
│ └─Usmanlvm-Blue--lv                          253:4    0  990M  0 lvm  
├─sda2                                           8:2    1  476M  0 part 
│ └─Usmanlvm-Blue--lv                          253:4    0  990M  0 lvm  
├─sda3                                           8:3    1  476M  0 part 
│ └─Usmanlvm-Blue--lv                          253:4    0  990M  0 lvm  
└─sda4                                           8:4    1  476M  0 part 
  └─Blue--Sky--Max-Blue--Sky--Max--Workacholic 253:2    0   50M  0 lvm  

``` 

### 12.6 Mount via fstab and verify persistence
```bash 

$ sudo mount -a 
[sudo] password for blu3sky: 
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
mount: /mnt: fsconfig system call failed: /dev/sr0: Can't open blockdev.
       dmesg(1) may have more information after failed mount system call.
```

> The `sr0` error is expected — it is the optical drive entry in fstab with no disc inserted. It does not affect the LV mounts. Run `systemctl daemon-reload` to sync systemd with the updated fstab if needed.

#### 12.6.1 Verify persistent mounts
```bash 
$ lsblk /dev/sda 
NAME                                           MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                                              8:0    1 58.6G  0 disk 
├─sda1                                           8:1    1  476M  0 part 
│ ├─Usmanlvm-Blue--Sky                         253:3    0  351M  0 lvm  /usman-vfat
│ └─Usmanlvm-Blue--lv                          253:4    0  990M  0 lvm  /home/blu3sky/usman-xfs
├─sda2                                           8:2    1  476M  0 part 
│ └─Usmanlvm-Blue--lv                          253:4    0  990M  0 lvm  /home/blu3sky/usman-xfs
├─sda3                                           8:3    1  476M  0 part 
│ └─Usmanlvm-Blue--lv                          253:4    0  990M  0 lvm  /home/blu3sky/usman-xfs
└─sda4                                           8:4    1  476M  0 part 
  └─Blue--Sky--Max-Blue--Sky--Max--Workacholic 253:2    0   50M  0 lvm  /usman-ext4
```

## 13. Teardown — Removing LVM Objects
Always remove in reverse order: **UMOUNT --> LV -->  VG -->  PV --> Wipe disk**.
Skipping steps or removing out of order will leave stale LVM metadata on disk. That metadata persists across reboots — the kernel reads it on the next boot and resurfaces ghost VGs. Always run `wipefs -a` as the final step.

### 13.1 UNMOUNT
```bash
$ sudo umount /dev/Usmanlvm/Blue-lv ; sudo umount /dev/Usmanlvm/Blue-Sky; sudo umount /dev/Blue-Sky-Max/Blue-Sky-Max-Workacholic
```
### 13.2 Remove all LVs 

``` bash
$ sudo lvremove /dev/Usmanlvm/Blue-lv -y
  Logical volume "Blue-lv" successfully removed.
``` 
### 13.3 Removing a VG (vgremove -y handles this automatically)
```bash
$ sudo vgremove Usmanlvm -y
  Logical volume "Blue-Sky" successfully removed.
  Volume group "Usmanlvm" successfully removed
``` 
#### 13.3.1 Viewing Changes
```bash
$ sudo vgs
  VG           #PV #LV #SN Attr   VSize   VFree  
  Blue-Sky-Max   2   1   0 wz--n- 950.00m 900.00m
  rhel           1   2   0 wz--n- <19.00g      0
```
### 13.4  Remove PV labels

PVs must be detached from its VG with `vgreduce` or `vgremove` before `pvremove` can wipe it. You cannot remove a PV that is still a member of a VG.
```bash
$ sudo pvremove /dev/sda1 /dev/sda2
  Labels on physical volume "/dev/sda1" successfully wiped.
  Labels on physical volume "/dev/sda2" successfully wiped.
```

### 13.5 Wipe file-system signatures from all disks used in the lab
```bash
$ sudo wipefs -a /dev/sda2 /dev/sda2

```

> **Always run `wipefs -a` after teardown.** LVM writes metadata directly to the partition. Without this step, the signatures survive and the kernel will re-import the old VGs the next time the disk is inserted — surfacing volume groups and LVs that no longer exist. This is especially important when reusing the same disk across multiple labs.

### 13.6  Confirm clean state
```bash
$ sudo pvs; sudo vgs; sudo lvs
  PV         VG   Fmt  Attr PSize   PFree
  /dev/vda3  rhel lvm2 a--  <19.00g    0

  VG   #PV #LV #SN Attr   VSize   VFree
  rhel   1   2   0 wz--n- <19.00g    0
```
## 14. Key Concepts and Common Mistakes

### LVM does not create free space

LVM is a management layer. The total storage is always bounded by real physical disks. Adding a PV is the only way to grow a VG.

### PE size is not disk size

`vgcreate -s 16` sets the allocation unit to 16 MiB — it does not add 16 MiB of storage. The actual space comes from the disk(s) you pass to `vgcreate`. PE size is locked after creation and cannot be changed on an existing VG.

### vgcreate runs pvcreate internally

If the target disk has no PV label yet, `vgcreate` will initialise it automatically. A separate `pvcreate` step is optional but explicit and preferred in production for clarity.

### + vs absolute sizing in lvresize

`-L +200` adds 200 MiB to the current size. `-L 300` sets the size to exactly 300 MiB (rounded to PE boundary). Confusing these two will give unexpected results.


### XFS has a 300 MB minimum on RHEL 10

`mkfs.xfs` enforces a hard minimum filesystem size of 300 MB on RHEL 10. This is not negotiable and cannot be overridden with a flag. Always provision XFS targets at 400 MB or more.

### Stale LVM metadata persists across reboots

LVM metadata is written directly to the PV, not held in memory. If you do not run `wipefs -a` after teardown, the kernel will re-import old VGs on the next boot or disk insertion. Always wipe as the final teardown step.

--- 

### Command quick-reference

| Task | Command |
|---|---|
| List block devices | `lsblk` |
| Inspect partition table | `sudo parted /dev/sdX print` |
| Create partition table | `sudo parted /dev/sdX mklabel gpt` |
| Create partition | `sudo parted /dev/sdX mkpart primary 1 500` |
| Remove partition | `sudo parted /dev/sdX rm <number>` |
| Set LVM flag | `sudo parted /dev/sdX set <number> lvm on` |
| Initialise PV | `sudo pvcreate /dev/sdX` |
| Create VG | `sudo vgcreate -s <PE_size> <vg_name> /dev/sdX` |
| Extend VG | `sudo vgextend <vg_name> /dev/sdX` |
| Remove PV from VG | `sudo vgreduce <vg_name> /dev/sdX` |
| Create LV (sized) | `sudo lvcreate -L <size> -n <name> <vg_name>` |
| Create LV (extents) | `sudo lvcreate -l <count> -n <name> <vg_name>` |
| Grow LV | `sudo lvresize -L +<size> /dev/<vg>/<lv>` |
| Shrink LV | `sudo lvreduce -L <size> /dev/<vg>/<lv>` |
| Rename LV | `sudo lvrename <vg_name> <old_name> <new_name>` |
| Remove LV | `sudo lvremove /dev/<vg>/<lv> -y` |
| Remove VG | `sudo vgremove <vg_name> -y` |
| Remove PV label | `sudo pvremove /dev/sdX` |
| Wipe disk signatures | `sudo wipefs -a /dev/sdX` |
| Inspect PVs | `sudo pvs` |
| Inspect VGs | `sudo vgs` |
| Inspect LVs | `sudo lvs` |
| Full VG detail | `sudo vgdisplay <vg_name>` | 
