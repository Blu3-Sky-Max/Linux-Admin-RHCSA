# Swap Space Management

**Author:** Usman O. Olanrewaju (Blu3 Sky)

**Date:** 2026/04/23

**Focus:** Swap space creation and management — partitions, LVM volumes, labels, priority, persistence, and teardown

---

## Table of Contents

1. [Swap Fundamentals](#1-swap-fundamentals)
2. [Creating Swap on a Partition](#2-creating-swap-on-a-partition)
3. [Persistent Swap — Editing fstab](#3-persistent-swap--editing-fstab)
4. [Adding a Logical Volume to the Swap Pool](#4-adding-a-logical-volume-to-the-swap-pool)
5. [Turning Off Swap](#5-turning-off-swap)
6. [Labeling Partitions After mkpart](#6-labeling-partitions-after-mkpart)

---

## 1. Swap Fundamentals

Swap extends the effective memory of the system. When free physical memory drops below a threshold, the kernel begins moving idle pages from RAM to swap space — freeing room for active processes. This is called **swap out**. When a process needs data that was paged out, the kernel retrieves it from swap back into RAM — this is called **swap in**.

Swap does not make a system faster. It prevents out-of-memory crashes by giving the kernel a pressure valve. Heavy swap usage is a signal that the system needs more RAM.


### 1.1 Check current swap state

```bash
$ sudo swapon 
NAME      TYPE      SIZE USED PRIO
/dev/dm-1 partition   2G   0B   -2
``` 
### 1.2 Confirm pool size with free

```bash
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           1.4Gi       1.1Gi        71Mi        83Mi       497Mi       343Mi
Swap:          2.0Gi       245Mi       1.8Gi
```

## 2. Creating Swap on a Partition

### 2.1 Inspect available partitions

```bash

$ lsblk /dev/sda
NAME                                           MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                                              8:0    1 58.6G  0 disk 
├─sda1                                           8:1    1  476M  0 part 
│ ├─Usmanlvm-Blue--Sky                         253:2    0  351M  0 lvm  /usman-vfat
│ └─Usmanlvm-Blue--lv                          253:3    0  990M  0 lvm  /home/blu3sky/usman-xfs
├─sda2                                           8:2    1  476M  0 part 
│ └─Usmanlvm-Blue--lv                          253:3    0  990M  0 lvm  /home/blu3sky/usman-xfs
├─sda3                                           8:3    1  476M  0 part 
│ └─Usmanlvm-Blue--lv                          253:3    0  990M  0 lvm  /home/blu3sky/usman-xfs
├─sda4                                           8:4    1  476M  0 part 
│ └─Blue--Sky--Max-Blue--Sky--Max--Workacholic 253:4    0  300M  0 lvm  /usman-ext4
├─sda5                                           8:5    1  1.9G  0 part 
└─sda6                                           8:6    1  572M  0 part 

```
 
### 2.2 Initialize swap without a label

```bash
$ sudo mkswap /dev/sda5
Setting up swapspace version 1, size = 1.9 GiB (1998581760 bytes)
no label, UUID=84df0a7e-eadd-4ee5-a31d-49890f1d38b8
``` 
### 2.3 Initialize swap with a label 

```bash 
$ sudo mkswap -L usman-swap /dev/sda6 
Setting up swapspace version 1, size = 572 MiB (599781376 bytes)
LABEL=usman-swap, UUID=bd19a56c-865e-48ce-ab81-53e7b809e738
```
> Labels make fstab entries human-readable and survive device renaming. Always prefer labeling swap partitions in any environment you manage long-term.


## 3.  Persistent Swap — Editing fstab

### 3.1 Add entries to /etc/fstab

Both `LABEL=` and `UUID=` are valid identifiers. Use `pri=` to set swap priority; higher number means higher priority. The kernel uses higher-priority swap first.

```bash
LABEL=usman-swap  swap swap pri=1,nofail  0 0 
UUID=84df0a7e-eadd-4ee5-a31d-49890f1d38b8 swap swap pri=1,nofail 0 0
```

### 3.2 Activate all swap entries in fstab

```bash
$ sudo swapon -a 
```

### 3.3 Verify swap pool
```bash
$ sudo swapon -s 
Filename				Type		Size		Used		Priority
/dev/dm-1                               partition	2097148		0		-2
/dev/sda6                               partition	585724		0		1
/dev/sda5                               partition	1951740		0		1
```

### 3.4 Confirm pool size with free
```bash
$ free -h 
               total        used        free      shared  buff/cache   available
Mem:           3.6Gi       2.0Gi       789Mi       100Mi       1.1Gi       1.6Gi
Swap:          4.4Gi          0B       4.4Gi
```


## 4. Adding a Logical Volume to the Swap Pool
Any LV can be repurposed as swap. If the LV has an existing filesystem signature, `mkswap` will warn and wipe it automatically.

### 4.1 Initialize LV as swap; no label

```bash
$ sudo mkswap /dev/Blue-Sky-Max/Blue-Sky-Max-Workacholic 
mkswap: /dev/Blue-Sky-Max/Blue-Sky-Max-Workacholic: warning: wiping old ext4 signature.
Setting up swapspace version 1, size = 300 MiB (314568704 bytes)
no label, UUID=63f42c97-0339-4173-a727-35c521d94761
```
### 4.2 Initialize LV as swap; with label

```bash 
$ sudo mkswap -L usman-lvm-swap /dev/Usmanlvm/Blue-Sky 
mkswap: /dev/Usmanlvm/Blue-Sky: warning: wiping old vfat signature.
Setting up swapspace version 1, size = 351 MiB (368046080 bytes)
LABEL=usman-lvm-swap, UUID=93b05571-be83-48fd-8ee2-fbefc01817b5
```
### 4.3 Add persistence entries to fstab
For LVM volumes, either the device path or UUID works. Device paths for LVM are stable by name so both are acceptable.

```bash 
/dev/Usmanlvm/Blue-Sky  swap swap pri=1,nofail  0 0
UUID=63f42c97-0339-4173-a727-35c521d94761 swap swap pri=2,nofail  0 0 
```

### 4.4 Activate new pool  
```bash
$ sudo swapon -a 
```
### 4.5  verify
 
```bash
$ sudo swapon -s
Filename				Type		Size		Used		Priority
/dev/dm-1                               partition	2097148		0		-2
/dev/sda6                               partition	585724		0		1
/dev/sda5                               partition	1951740		0		1
/dev/dm-2                               partition	359420		0		1
/dev/dm-4                               partition	307196		0		2
```

### 4.6 Confirm pool size with free
```bash
$ free -h 
               total        used        free      shared  buff/cache   available
Mem:           3.6Gi       2.0Gi       759Mi       100Mi       1.1Gi       1.6Gi
Swap:          5.1Gi          0B       5.1Gi
```

## 5. Turning off Swap 

### 5.1 Deactivate specific swap devices
```bash
$ sudo swapoff /dev/sda6 ; sudo swapoff /dev/dm-2 
```

### 5.2  Verify pool reduced

```bash
$ sudo swapon -s
Filename				Type		Size		Used		Priority
/dev/dm-1                               partition	2097148		0		-2
/dev/sda5                               partition	1951740		0		1
/dev/dm-4                               partition	307196		0		2
```
#### 5.2.1 Confirm pool size with free
```bash
$ free -h 
               total        used        free      shared  buff/cache   available
Mem:           3.6Gi       2.0Gi       762Mi       100Mi       1.1Gi       1.6Gi
Swap:          4.2Gi          0B       4.2Gi
```

### 5.2 Deactivate all swap

```bash
$ sudo swapoff -a
```
> **Always edit `/etc/fstab` after teardown** — remove or comment out all swap entries. If the entries remain and the device is gone, the next boot will hang trying to activate swap that does not exist.


### 5.3 Wipe swap signatures
```bash
$ sudo wipefs -a /dev/sda5  /dev/sda6 /dev/Blue-Sky-Max/Blue-Sky-Max-Workacholic /dev/Usmanlvm/Blue-Sky
/dev/sda5: 10 bytes were erased at offset 0x00000ff6 (swap): 53 57 41 50 53 50 41 43 45 32
dev/sda6: 10 bytes were erased at offset 0x00000ff6 (swap): 53 57 41 50 53 50 41 43 45 32
/dev/Blue-Sky-Max/Blue-Sky-Max-Workacholic: 10 bytes were erased at offset 0x00000ff6 (swap): 53 57 41 50 53 50 41 43 45 32
/dev/Usmanlvm/Blue-Sky: 10 bytes were erased at offset 0x00000ff6 (swap): 53 57 41 50 53 50 41 43 45 32
```
### 5.4 Activiate the system swap on 
```bash
$ sudo swapon -a
```

## 6. Labeling Partitions After mkpart 
`parted` does not write filesystem labels,it only sets partition metadata. Labels are applied by the filesystem tool after formatting. This section covers how to format and label a partition post-creation.

### 6.1 Confirm partitions have no filesystem
```bash
$ lsblk /dev/sda5 /dev/sda6 -f
NAME FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sda5                
sda6                                                                        
                        
```
### 6.2 Format the partitions
```bash 
$ sudo mkfs.xfs /dev/sda5 ; sudo mkfs.ext4 /dev/sda6
meta-data=/dev/sda5              isize=512    agcount=4, agsize=121984 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=1
         =                       exchange=0  
data     =                       bsize=4096   blocks=487936, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=0
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
mke2fs 1.47.1 (20-May-2024)
Creating filesystem with 146432 4k blocks and 36640 inodes
Filesystem UUID: 35ffe675-dfbc-45a1-b955-6009002eeda7
Superblock backups stored on blocks: 
	32768, 98304

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done


```

### 6.3 Label an XFS partition
```bash 
$ sudo xfs_admin -L usman-label-xfs /dev/sda5
writing all SBs
xfs_admin: truncating label length from 15 to 12
new label = "usman-label-"

```

> **XFS labels are capped at 12 characters.** Any label longer than 12 characters will be silently truncated. Plan your label names accordingly.

### 6.4 Label an ext4 partition

```bash
$ sudo mke2fs -L usman-label-ext4 /dev/sda6 
mke2fs 1.47.1 (20-May-2024)
/dev/sda6 contains a ext4 file system
	created on Fri Apr 24 12:45:39 2026
Proceed anyway? (y,N) y
Creating filesystem with 146432 4k blocks and 36640 inodes
Filesystem UUID: a9b4da91-fab6-417c-bf1d-d79971018617
Superblock backups stored on blocks: 
	32768, 98304

Allocating group tables: done                            
Writing inode tables: done                            
Writing superblocks and filesystem accounting information: done
```
> `mke2fs -L` reformats the partition to apply the label. If you want to relabel an existing ext4 filesystem **without** reformatting, use `tune2fs -L <label> /dev/sdX` instead.
 
### 6.5 Verify labels

```bash 
$ lsblk -f /dev/sda5 /dev/sda6 
NAME FSTYPE FSVER LABEL            UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sda5 xfs          usman-label-     cae256f6-9d5b-4997-a9b6-a3a82f904cb4                
sda6 ext2   1.0   usman-label-ext4 a9b4da91-fab6-417c-bf1d-d79971018617                
```


## Key Concepts and Common Mistakes

### Swap does not speed up the system

Swap prevents crashes under memory pressure — it does not improve performance. If a system is regularly using swap heavily, the fix is more RAM, not more swap.

### A mounted device cannot be added to swap

You must unmount a partition or LV before running `mkswap` on it. The workflow is always:
 **umount --> mkswap -->  swapon**.

### Labels can be used in fstab

Both `LABEL=` and `UUID=` are valid fstab identifiers for swap. Labels are human-readable but must be unique on the system. UUIDs are always unique and preferred in production.

### Always wipe after swapoff

`wipefs -a` removes the swap signature from the device. Without this step the signature persists on disk and the kernel may attempt to activate the swap again on next boot or device insertion.

### XFS label limit is 12 characters

`xfs_admin -L` silently truncates labels longer than 12 characters. Use `tune2fs -L` to relabel ext4 without reformatting.

---

### Command Quick Reference

| Task | Command |
|---|---|
| Check active swap | `sudo swapon` |
| Check swap usage | `sudo swapon -s` |
| Check memory and swap | `free -h` |
| Initialize swap (no label) | `sudo mkswap /dev/sdX` |
| Initialize swap (with label) | `sudo mkswap -L <label> /dev/sdX` |
| Activate all fstab swap | `sudo swapon -a` |
| Deactivate specific swap | `sudo swapoff /dev/sdX` |
| Deactivate all swap | `sudo swapoff -a` |
| Wipe swap signature | `sudo wipefs -a /dev/sdX` |
| Label XFS partition | `sudo xfs_admin -L <label> /dev/sdX` |
| Label ext4 (reformat) | `sudo mke2fs -L <label> /dev/sdX` |
| Relabel ext4 (no reformat) | `sudo tune2fs -L <label> /dev/sdX`|
