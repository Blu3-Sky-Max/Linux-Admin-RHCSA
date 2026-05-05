# Kernel Management

**Author:** Usman O. Olanrewaju (Blu3 Sky)

**Date:** 2026/05/04

**Focus:** Linux kernel architecture, package structure, the /boot and /proc filesystems, kernel installation on RHEL 10 and Fedora 44, and version verification

---

## Table of Contents

1. [What is the Linux Kernel](#1-what-is-the-linux-kernel)
2. [Kernel Packages](#2-kernel-packages)
3. [Inspecting the Running Kernel](#3-inspecting-the-running-kernel)
4. [The /boot Filesystem](#4-the-boot-filesystem)
5. [The /proc Virtual Filesystem](#5-the-proc-virtual-filesystem)
6. [Kernel Modules — /usr/lib/modules](#6-kernel-modules--usrlibmodules)
7. [Installing a New Kernel on RHEL 10](#7-installing-a-new-kernel-on-rhel-10)
8. [Installing a New Kernel on Fedora 44](#8-installing-a-new-kernel-on-fedora-44)
9. [Installing Kernel 7.0.3 on Fedora 44](#9-installing-kernel-703-on-fedora-44)

---

## 1. What is the Linux Kernel

The Linux kernel is the core component of the operating system. It manages hardware, enforces security, regulates system access, and handles processes, services, and application workloads. Every interaction between software and hardware passes through the kernel.

RHEL 10 source code packages are also available for those who wish to customize or recompile the kernel for specific needs. 

---

## 2. Kernel Packages

The kernel ships as a set of separate RPM packages. Each package serves a specific purpose.

| Package | Description |
|---|---|
| `kernel` | Contains no files — ensures all other kernel packages are accurately installed |
| `kernel-core` | Includes essential modules to provide core functionality |
| `kernel-devel` | Includes support for building external kernel modules |
| `kernel-headers` | Includes files to support the interface between the kernel and userspace libraries and programs |
| `kernel-modules` | Contains modules for common hardware devices |
| `kernel-modules-core` | Contains modules for core hardware devices |
| `kernel-modules-extra` | Contains modules for not-so-common hardware devices |
| `kernel-tools` | Includes tools to manipulate the kernel |
| `kernel-tools-libs` | Includes the libraries to support the kernel tools |

### 2.1 List all installed kernel packages

```bash
$ rpm -qa | grep kernel
kernel-modules-core-6.12.0-55.9.1.el10_0.x86_64
kernel-core-6.12.0-55.9.1.el10_0.x86_64
kernel-modules-6.12.0-55.9.1.el10_0.x86_64
kernel-modules-extra-6.12.0-55.9.1.el10_0.x86_64
kernel-6.12.0-55.9.1.el10_0.x86_64
kernel-modules-core-6.12.0-124.43.1.el10_1.x86_64
kernel-core-6.12.0-124.43.1.el10_1.x86_64
kernel-modules-6.12.0-124.43.1.el10_1.x86_64
kernel-modules-extra-6.12.0-124.43.1.el10_1.x86_64
kernel-tools-libs-6.12.0-124.43.1.el10_1.x86_64
kernel-tools-6.12.0-124.43.1.el10_1.x86_64
kernel-6.12.0-124.43.1.el10_1.x86_64
```

> Two kernel versions are installed simultaneously. RHEL never removes the previous kernel on upgrade — it keeps the old one as a fallback. If the new kernel fails to boot, GRUB2 presents the previous version as a recovery option.

---

## 3. Inspecting the Running Kernel

### 3.1 Check the system architecture

```bash
$ uname -m
x86_64
```

### 3.2 Check the running kernel version

```bash
$ uname -r
6.12.0-124.43.1.el10_1.x86_64
```

**Version string breakdown:**

```
6.12.0 - 124.43.1 . el10_1 . x86_64
|        |           |         |
|        |           |         └---> Architecture
|        |           └---> Distribution tag (el10 = Enterprise Linux 10)
|        └--->  Red Hat patch and build version
└----> Upstream kernel version (major.minor.patch)
```

> On Fedora, the distribution tag reads `fc44` or `fc43` — meaning Fedora Core 44 or 43. On RHEL it reads `el10` for Enterprise Linux version 10.


## 4. The /boot Filesystem

`/boot` is the home of the kernel and GRUB2. It holds every file needed to load the operating system.

### 4.1 List /boot contents

```bash
$ ls /boot/
config-6.12.0-124.43.1.el10_1.x86_64
config-6.12.0-55.9.1.el10_0.x86_64
efi
grub2
initramfs-0-rescue-5b510439f1e0470db9ce41063fb8db5f.img
initramfs-6.12.0-124.43.1.el10_1.x86_64.img
initramfs-6.12.0-124.43.1.el10_1.x86_64kdump.img
initramfs-6.12.0-55.9.1.el10_0.x86_64.img
initramfs-6.12.0-55.9.1.el10_0.x86_64kdump.img
loader
symvers-6.12.0-124.43.1.el10_1.x86_64.xz
symvers-6.12.0-55.9.1.el10_0.x86_64.xz
System.map-6.12.0-124.43.1.el10_1.x86_64
System.map-6.12.0-55.9.1.el10_0.x86_64
vmlinuz-0-rescue-5b510439f1e0470db9ce41063fb8db5f
vmlinuz-6.12.0-124.43.1.el10_1.x86_64
vmlinuz-6.12.0-55.9.1.el10_0.x86_64
```

**Key files in /boot:**

| File | Description |
|---|---|
| `vmlinuz-6*` | Compressed kernel executable |
| `config-6*` | Kernel build configuration |
| `initramfs-6*` | Initial RAM filesystem used for early boot |
| `symvers-6*` | Symbol versioning information for module compatibility |
| `System.map-6*` | Memory address mapping for kernel symbols |

> The `efi` and `grub2` subdirectories hold bootloader information specific to the firmware type — UEFI or BIOS respectively. The `loader` subdirectory stores configuration for the running and rescue kernels under `loader/entries/`.

### 4.2 BLS entry files — /boot/loader/entries

The Boot Loader Specification (BLS) stores each kernel's boot configuration as a separate `.conf` file. GRUB2 reads these to build the boot menu.

```bash
$ sudo ls -l /boot/loader/entries/
total 12
-rw-r--r--. 1 root root 498 Jan 27 12:08 5b510439f1e0470db9ce41063fb8db5f-0-rescue.conf
-rw-r--r--. 1 root root 477 Mar 12 19:46 5b510439f1e0470db9ce41063fb8db5f-6.12.0-124.43.1.el10_1.x86_64.conf
-rw-r--r--. 1 root root 470 Mar 12 19:46 5b510439f1e0470db9ce41063fb8db5f-6.12.0-55.9.1.el10_0.x86_64.conf
```

> The long hex prefix is the machine ID. Each file corresponds to one entry in the GRUB2 boot menu — one per installed kernel plus one rescue entry.

---

## 5. The /proc Virtual Filesystem

`/proc` is a virtual filesystem. Its contents are generated in memory at system boot, updated dynamically during runtime, and destroyed at system shutdown. No data in `/proc` is written to disk. System utilities including `top`, `ps`, `uname`, `free`, `uptime`, and `w` all read from `/proc` to produce their output.

### 5.1 Check memory and CPU information

```bash
$ cat /proc/meminfo; cat  /proc/cpuinfo
MemTotal:        3740252 kB
MemFree:          446700 kB
MemAvailable:    1569444 kB
Buffers:               8 kB
Cached:          1363548 kB
SwapCached:         2844 kB
Active:          1766696 kB
Inactive:         980720 kB
Active(anon):    1386596 kB
Inactive(anon):    97228 kB
Active(file):     380100 kB
Inactive(file):   883492 kB
Unevictable:       64480 kB
Mlocked:               0 kB
SwapTotal:       2097148 kB
SwapFree:        2076756 kB
Zswap:                 0 kB
Zswapped:              0 kB
Dirty:                 0 kB
Writeback:             0 kB
AnonPages:       1444872 kB
Mapped:           296964 kB
Shmem:            100244 kB
KReclaimable:     139208 kB
Slab:             265016 kB
SReclaimable:     139208 kB
SUnreclaim:       125808 kB
KernelStack:       10304 kB
PageTables:        21652 kB
SecPageTables:         0 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     3967272 kB
Committed_AS:    5842060 kB
VmallocTotal:   34359738367 kB
VmallocUsed:       28520 kB
VmallocChunk:          0 kB
Percpu:             4032 kB
HardwareCorrupted:     0 kB
AnonHugePages:   1019904 kB
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
FileHugePages:         0 kB
FilePmdMapped:         0 kB
CmaTotal:              0 kB
CmaFree:               0 kB
Unaccepted:            0 kB
Balloon:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
DirectMap4k:      175980 kB
DirectMap2M:     4018176 kB
DirectMap1G:     2097152 kB
processor   : 0
vendor_id   : AuthenticAMD
cpu family  : 26
model       : 36
model name  : AMD Ryzen AI 9 HX 370 w/ Radeon 890M
stepping    : 0
microcode   : 0xb204037
cpu MHz     : 1996.249
cache size  : 512 KB
physical id : 0
siblings    : 5
core id     : 0
cpu cores   : 5
apicid      : 0
initial apicid  : 0
fpu     : yes
fpu_exception   : yes
cpuid level : 16
wp      : yes
flags       : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx mmxext fxsr_opt pdpe1gb rdtscp lm rep_good nopl cpuid extd_apicid tsc_known_freq pni pclmulqdq ssse3 fma cx16 sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm cmp_legacy svm cr8_legacy abm sse4a misalignsse 3dnowprefetch osvw perfctr_core ssbd perfmon_v2 ibrs ibpb stibp ibrs_enhanced vmmcall fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid avx512f avx512dq rdseed adx smap avx512ifma clflushopt clwb avx512cd sha_ni avx512bw avx512vl xsaveopt xsavec xgetbv1 xsaves avx_vnni avx512_bf16 clzero xsaveerptr wbnoinvd arat npt lbrv nrip_save tsc_scale vmcb_clean flushbyasid pausefilter pfthreshold v_vmsave_vmload vgif vnmi avx512vbmi umip pku ospke avx512_vbmi2 gfni vaes vpclmulqdq avx512_vnni avx512_bitalg avx512_vpopcntdq rdpid movdiri movdir64b overflow_recov succor fsrm avx512_vp2intersect flush_l1d
bugs        : sysret_ss_attrs spectre_v1 spectre_v2 spec_store_bypass srso ibpb_no_ret spectre_v2_user
bogomips    : 3992.49
TLB size    : 1024 4K pages
clflush size    : 64
cache_alignment : 64
address sizes   : 48 bits physical, 48 bits virtual
power management:

processor   : 1
vendor_id   : AuthenticAMD
cpu family  : 26
model       : 36
model name  : AMD Ryzen AI 9 HX 370 w/ Radeon 890M
stepping    : 0
microcode   : 0xb204037
cpu MHz     : 1996.249
cache size  : 512 KB
physical id : 0
siblings    : 5
core id     : 1
cpu cores   : 5
apicid      : 1
initial apicid  : 1
fpu     : yes
fpu_exception   : yes
cpuid level : 16
wp      : yes


#truncated output
```


## 6. Kernel Modules — /usr/lib/modules

`/usr/lib/modules` contains the loadable kernel modules for each installed kernel version. Each version gets its own subdirectory.

### 6.1 List installed kernel versions

```bash
$ ls -l /usr/lib/modules
total 8
drwxr-xr-x. 7 root root 4096 Mar 12 19:46 6.12.0-124.43.1.el10_1.x86_64
drwxr-xr-x. 7 root root 4096 Jan 27 12:09 6.12.0-55.9.1.el10_0.x86_64
```

### 6.2 Inspect a kernel version subdirectory

```bash
$ ls -l /usr/lib/modules/6.12.0-55.9.1.el10_0.x86_64/
total 28476
lrwxrwxrwx.  1 root root       44 Mar 24  2025 build -> /usr/src/kernels/6.12.0-55.9.1.el10_0.x86_64
-rw-r--r--.  1 root root   249779 Mar 24  2025 config
drwxr-xr-x. 12 root root      131 Jan 27 12:05 kernel
-rw-r--r--.  1 root root   893620 Jan 27 12:09 modules.alias
-rw-r--r--.  1 root root   846315 Jan 27 12:09 modules.alias.bin
-rw-r--r--.  1 root root      344 Mar 24  2025 modules.block
-rw-r--r--.  1 root root     9665 Mar 24  2025 modules.builtin
-rw-r--r--.  1 root root    12284 Jan 27 12:09 modules.builtin.alias.bin
-rw-r--r--.  1 root root    11990 Jan 27 12:09 modules.builtin.bin
-rw-r--r--.  1 root root    86199 Mar 24  2025 modules.builtin.modinfo
-rw-r--r--.  1 root root   315868 Jan 27 12:09 modules.dep
-rw-r--r--.  1 root root   412099 Jan 27 12:09 modules.dep.bin
-rw-r--r--.  1 root root      405 Jan 27 12:09 modules.devname
-rw-r--r--.  1 root root      175 Mar 24  2025 modules.drm
-rw-r--r--.  1 root root       34 Mar 24  2025 modules.modesetting
-rw-r--r--.  1 root root     1641 Mar 24  2025 modules.networking
-rw-r--r--.  1 root root    96054 Mar 24  2025 modules.order
-rw-r--r--.  1 root root      975 Jan 27 12:09 modules.softdep
-rw-r--r--.  1 root root   500549 Jan 27 12:09 modules.symbols
-rw-r--r--.  1 root root   594999 Jan 27 12:09 modules.symbols.bin
lrwxrwxrwx.  1 root root        5 Mar 24  2025 source -> build
-rw-r--r--.  1 root root   275084 Mar 24  2025 symvers.xz
-rw-r--r--.  1 root root  8932873 Mar 24  2025 System.map
drwxr-xr-x.  2 root root        6 Mar 24  2025 systemtap
drwxr-xr-x.  2 root root        6 Mar 24  2025 updates
drwxr-xr-x.  2 root root       40 Jan 27 12:05 vdso
-rwxr-xr-x.  1 root root 15865976 Mar 24  2025 vmlinuz
drwxr-xr-x.  2 root root        6 Mar 24  2025 weak-updates
```

---

## 7. Installing a New Kernel on RHEL 10

This section covers installing a Fedora 44 kernel on a RHEL 10 system. I used advanced procedure  here for lab purposes.

### 7.1 Confirm running kernel before install

```bash
$ uname -r
6.12.0-124.43.1.el10_1.x86_64
```

### 7.2 Download the Fedora kernel packages

Source: [https://koji.fedoraproject.org/koji/packageinfo?packageID=8](https://koji.fedoraproject.org/koji/packageinfo?packageID=8)

![Fedora Koji download page](./fedorarhel.jpeg)

Download the following packages for the target version:

```bash
$ ls kernel-*
kernel-6.19.14-300.fc44.x86_64.rpm
kernel-core-6.19.14-300.fc44.x86_64.rpm
kernel-modules-6.19.14-300.fc44.x86_64.rpm
kernel-modules-core-6.19.14-300.fc44.x86_64.rpm
```

### 7.3 First attempt — dependency failure

```bash
$ rpm -ivh kernel-*
error: Failed dependencies:
    kernel-modules-core-uname-r = 6.19.14-300.fc44.x86_64 is needed by kernel-6.19.14-300.fc44.x86_64
    kernel-modules-core-uname-r = 6.19.14-300.fc44.x86_64 is needed by kernel-core-6.19.14-300.fc44.x86_64
    kernel-modules-core-uname-r = 6.19.14-300.fc44.x86_64 is needed by kernel-modules-6.19.14-300.fc44.x86_64
```

> `rpm -ivh` resolves dependencies only from the packages you provide. If any required package is missing from the local set, the install fails. Download all required packages first, then install together.

### 7.4 Install all packages together

```bash
$ sudo rpm -ivh kernel-*
[sudo] password for blu3sky:
Verifying...                          ################################# [100%]
Preparing...                          ################################# [100%]
Updating / installing...
   1:kernel-modules-core-6.19.14-300.f################################# [ 25%]
   2:kernel-core-6.19.14-300.fc44     ################################# [ 50%]
   3:kernel-modules-6.19.14-300.fc44  ################################# [ 75%]
   4:kernel-6.19.14-300.fc44          ################################# [100%]
Generating grub configuration file ...
Adding boot menu entry for UEFI Firmware Settings ...
done
```

![Kernel installation progress](./kernel44usman.jpeg)

> GRUB2 configuration is regenerated automatically after kernel installation. The new kernel appears as the first entry in the boot menu.

### 7.5 Verify all installed kernels

```bash
$ sudo dnf list installed | grep kernel
kernel.x86_64                    6.12.0-55.9.1.el10_0       @anaconda
kernel.x86_64                    6.12.0-124.43.1.el10_1     @@commandline
kernel.x86_64                    6.19.14-300.fc44           @System
kernel-core.x86_64               6.12.0-55.9.1.el10_0       @anaconda
kernel-core.x86_64               6.12.0-124.43.1.el10_1     @@commandline
kernel-core.x86_64               6.19.14-300.fc44           @System
kernel-modules.x86_64            6.12.0-55.9.1.el10_0       @anaconda
kernel-modules.x86_64            6.12.0-124.43.1.el10_1     @@commandline
kernel-modules.x86_64            6.19.14-300.fc44           @System
kernel-modules-core.x86_64       6.12.0-55.9.1.el10_0       @anaconda
kernel-modules-core.x86_64       6.12.0-124.43.1.el10_1     @@commandline
kernel-modules-core.x86_64       6.19.14-300.fc44           @System
kernel-modules-extra.x86_64      6.12.0-55.9.1.el10_0       @anaconda
kernel-modules-extra.x86_64      6.12.0-124.43.1.el10_1     @@commandline
kernel-tools.x86_64              6.12.0-124.43.1.el10_1     @@commandline
kernel-tools-libs.x86_64         6.12.0-124.43.1.el10_1     @@commandline
```

> The fc44 kernel shows `@System` as the repository — meaning it was installed manually from local RPM files, not from a configured DNF repository.

### 7.6 Reboot and confirm

```bash
$ reboot
```

```bash
$ uname -r
6.19.14-300.fc44.x86_64
```


## 8. Installing a New Kernel on Fedora 44

This section covers the initial state of the Fedora 44 machine and the kernel present before upgrading.

### 8.1 Check system info


```bash
$ hostnamectl
     Static hostname: Sky.usman.O.Olanrewaju
           Icon name: computer-vm
             Chassis: vm 🖴
          Machine ID: eaf6762190f8406d95c38dc9cd0a3321
             Boot ID: 4050d733a34c457fbc55ffc770033db9
      Virtualization: kvm
    Operating System: Fedora Linux 44 (Workstation Edition)
         CPE OS Name: cpe:/o:fedoraproject:fedora:44
      OS Support End: Wed 2027-05-19
OS Support Remaining: 1y 1w 6d
              Kernel: Linux 6.19.14-300.fc44.x86_64
        Architecture: x86-64
     Hardware Vendor: QEMU
      Hardware Model: Standard PC _Q35 + ICH9, 2009_
    Hardware Version: pc-q35-10.1
    Firmware Version: 1.17.0-9.fc43
       Firmware Date: Tue 2025-06-10
        Firmware Age: 10month 3w 3d
```


### 8.2 List installed kernel packages on Fedora 44

```bash
$ rpm -qa | grep ^kernel
kernel-core-6.19.10-300.fc44.x86_64
kernel-modules-core-6.19.10-300.fc44.x86_64
kernel-modules-6.19.10-300.fc44.x86_64
kernel-modules-extra-6.19.10-300.fc44.x86_64
kernel-6.19.10-300.fc44.x86_64
kernel-tools-libs-6.19.14-300.fc44.x86_64
kernel-core-6.19.14-300.fc44.x86_64
kernel-modules-core-6.19.14-300.fc44.x86_64
kernel-modules-6.19.14-300.fc44.x86_64
kernel-modules-extra-6.19.14-300.fc44.x86_64
kernel-6.19.14-300.fc44.x86_64
kernel-tools-6.19.14-300.fc44.x86_64
kernel-headers-6.19.6-300.fc44.x86_64
```


## 9. Installing Kernel 7.0.3 on Fedora 44

Kernel 7.0.3 is not yet in the official Fedora repositories. This install uses locally downloaded RPM files passed directly to `dnf`.

### 9.1 Install using dnf with local packages

```bash
$ sudo dnf install kernel*
[sudo] password for blue:
Updating and loading repositories:
Repositories loaded.
Package                            Arch    Version              Repository        Size
Upgrading:
 kernel-tools                      x86_64  7.0.3-200.fc44       @commandline  859.3 KiB
   replacing kernel-tools          x86_64  6.19.14-300.fc44     updates       927.5 KiB
 kernel-tools-libs                 x86_64  7.0.3-200.fc44       @commandline   30.2 KiB
   replacing kernel-tools-libs     x86_64  6.19.14-300.fc44     updates        30.2 KiB
Installing:
 kernel                            x86_64  7.0.3-200.fc44       @commandline    0.0   B
 kernel-core                       x86_64  7.0.3-200.fc44       @commandline   99.1 MiB
 kernel-modules                    x86_64  7.0.3-200.fc44       @commandline   99.4 MiB
 kernel-modules-core               x86_64  7.0.3-200.fc44       @commandline   72.4 MiB
 kernel-modules-extra              x86_64  7.0.3-200.fc44       @commandline    4.4 MiB
 kernel-tools-libs-devel           x86_64  7.0.3-200.fc44       @commandline   12.3 MiB

Transaction Summary:
 Installing:         6 packages
 Upgrading:          2 packages
 Replacing:          2 packages

Total size of inbound packages is 203 MiB. Need to download 0 B.
After this operation, 287 MiB extra will be used (install 288 MiB, remove 958 KiB).
Is this ok [y/N]: y
Running transaction
[ 1/12] Verify package files                    100% |  15.0   B/s |   8.0   B |  00m01s
[ 2/12] Prepare transaction                     100% |  57.0   B/s |  10.0   B |  00m00s
[ 3/12] Installing kernel-core-7.0.3            100% |  85.9 MiB/s |  29.6 MiB |  00m00s
[ 4/12] Installing kernel-modules-core-7.0.3    100% |  80.0 MiB/s |  73.0 MiB |  00m01s
[ 5/12] Installing kernel-modules-7.0.3         100% |  13.2 MiB/s |  99.8 MiB |  00m08s
[ 6/12] Upgrading kernel-tools-libs-7.0.3       100% | 211.1 KiB/s |  30.6 KiB |  00m00s
[ 7/12] Upgrading kernel-tools-7.0.3            100% |   8.0 MiB/s | 865.7 KiB |  00m00s
[ 8/12] Installing kernel-modules-extra-7.0.3   100% | 667.8 KiB/s |   4.4 MiB |  00m07s
[ 9/12] Installing kernel-7.0.3                 100% |   4.2 KiB/s | 124.0   B |  00m00s
[10/12] Installing kernel-tools-libs-devel-7.0.3 100% |  71.0 MiB/s |  12.3 MiB |  00m00s
[11/12] Removing kernel-tools-6.19.14           100% |   1.0 KiB/s |  44.0   B |  00m00s
[12/12] Removing kernel-tools-libs-6.19.14      100% |   0.0   B/s |   2.0   B |  00m32s
Warning: skipped OpenPGP checks for 8 packages from repository: @commandline
Complete!
```

![Kernel 7.0.3 installation](./Installing-fc-7.0.3-.jpeg)

### 9.2 Confirm new kernel in GRUB environment

```bash
$ cat /boot/grub2/grubenv
# GRUB Environment Block
# WARNING: Do not edit this file by tools other than grub-editenv!!!
env_block=512+1
saved_entry=eaf6762190f8406d95c38dc9cd0a3321-7.0.3-200.fc44.x86_64
menu_auto_hide=1
boot_success=1
######################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################
```


### 9.3 Reboot and verify

```bash
$ reboot
```

```bash
$ hostnamectl
     Static hostname: Sky.usman.O.Olanrewaju
           Icon name: computer-vm
             Chassis: vm 🖴
          Machine ID: eaf6762190f8406d95c38dc9cd0a3321
             Boot ID: ccfd8ececb79410080b3a20055f13389
      Virtualization: kvm
    Operating System: Fedora Linux 44 (Workstation Edition)
         CPE OS Name: cpe:/o:fedoraproject:fedora:44
      OS Support End: Wed 2027-05-19
OS Support Remaining: 1y 1w 6d
              Kernel: Linux 7.0.3-200.fc44.x86_64
        Architecture: x86-64
     Hardware Vendor: QEMU
      Hardware Model: Standard PC _Q35 + ICH9, 2009_
    Hardware Version: pc-q35-10.1
    Firmware Version: 1.17.0-9.fc43
       Firmware Date: Tue 2025-06-10
        Firmware Age: 10month 3w 4d
```


```bash
$ uname -r
7.0.3-200.fc44.x86_64
```

---

## Key Concepts and Common Mistakes

### RHEL keeps the previous kernel on upgrade

Unlike most packages, kernel upgrades do not replace the old version — both are kept. This ensures you always have a fallback if the new kernel fails to boot. GRUB2 lists all installed kernels at boot.

### /proc is not on disk

Everything under `/proc` is generated in memory at boot and gone at shutdown. Never try to back it up or write to it directly.

### rpm -ivh requires all dependencies locally

`rpm` does not resolve dependencies from repositories — it only sees what you pass to it. If installing a kernel with `rpm -ivh kernel-*`, all required packages must be present in the current directory. Use `dnf` when possible as it handles dependencies automatically.

### The fc44 kernel tag means Fedora, not RHEL

Installing a Fedora kernel on RHEL is unsupported in production. It works in a lab environment for learning purposes but should never be done on a production system.

---

### Command Quick Reference

| Task | Command |
|---|---|
| List installed kernel packages | `rpm -qa \| grep kernel` |
| Check running kernel version | `uname -r` |
| Check system architecture | `uname -m` |
| Check full system info | `hostnamectl` |
| List /boot contents | `ls /boot/` |
| List BLS boot entries | `sudo ls -l /boot/loader/entries/` |
| View memory info | `cat /proc/meminfo` |
| View CPU info | `cat /proc/cpuinfo` |
| List kernel module versions | `ls -l /usr/lib/modules` |
| List all installed kernels | `sudo dnf list installed \| grep kernel` |
| Install kernel from local RPMs | `sudo rpm -ivh kernel-*` |
| Install kernel via dnf | `sudo dnf install kernel*` |
| View GRUB environment | `cat /boot/grub2/grubenv` |
