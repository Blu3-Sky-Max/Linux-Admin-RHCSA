
# SELinux Management

**Author:** Usman O. Olanrewaju (Blu3 Sky)

**Date:** 2026/06/01

**Focus:** SELinux architecture (MAC vs DAC), contexts, modes, policy management, file and port labeling, Boolean toggling, copy/move context behavior, and persistent context mapping via `semanage` and `restorecon`

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [SELinux Terminology](#2-selinux-terminology)
3. [SELinux Management Packages and Commands](#3-selinux-management-packages-and-commands)
4. [Security Contexts](#4-security-contexts)
5. [Viewing and Controlling SELinux State](#5-viewing-and-controlling-selinux-state)
6. [Querying Full Runtime Status](#6-querying-full-runtime-status)
7. [Modifying SELinux Contexts with chcon](#7-modifying-selinux-contexts-with-chcon)
8. [Persistent Context Mapping with semanage and restorecon](#8-persistent-context-mapping-with-semanage-and-restorecon)
9. [Managing SELinux Port Labels](#9-managing-selinux-port-labels)
10. [Copy and Context Inheritance](#10-copy-and-context-inheritance)
11. [Move and Context Retention](#11-move-and-context-retention)
12. [Boolean Toggle](#12-boolean-toggle)
13. [Key Takeaways](#13-key-takeaways)

---


## 1. Core Concepts

### 1.1 What SELinux Is

*Security Enhanced Linux* (SELinux) is an implementation of the **Mandatory Access Control** (MAC) security architecture. It provides flexible, granular, and policy-driven access controls integrated directly into the Linux kernel — enabling the kernel to enforce security decisions beyond the traditional permission model.

The goal of SELinux is to limit the damage that can occur due to unauthorized user or program access. If a service is compromised, SELinux confines the damage to only those resources explicitly permitted to that service by policy.

### 1.2 DAC vs MAC

SELinux complements, not replaces, the standard Linux **Discretionary Access Control** (DAC) model. The key difference is who controls access:

| Model | Who Controls Access |
|-------|---------------------|
| **DAC** | The resource owner. A user who creates a file decides who can read, write, or execute it — even if granting that access introduces a security risk |
| **MAC (SELinux)** | A system-wide policy. Ownership and DAC permissions alone are not sufficient. Even if a user owns a file, SELinux denies access unless the policy explicitly permits it |

MAC limits what a subject (user or process) can do with an object (resource), even when DAC would otherwise allow it. This model prevents compromised services or accounts from accessing resources beyond their defined scope.

### 1.3 Policy Enforcement and Protection Model

SELinux enforces a set of authorization rules collectively known as a *policy*. When a subject attempts to access an object, SELinux examines the **security contexts** (also called *labels*) assigned to both and determines whether the action is allowed. Any action not explicitly permitted is denied by default.

For example, if the `httpd` process is compromised, the attacker can only access files and resources that SELinux policy permits `httpd` — not arbitrary system files or another service's data.

RHEL ships with three predefined policies:

| Policy | Description |
|--------|-------------|
| **targeted** | Confines selected system services while letting other processes run unconfined. Default policy in RHEL |
| **mls** | Multi-Level Security — strict hierarchical controls for highly regulated environments |
| **minimum** | A lightweight variant of targeted that protects only a limited set of services |

### 1.4 Access Vector Cache (AVC)

SELinux access decisions are cached in the **Access Vector Cache** (AVC). Each time a process attempts an action, SELinux first checks the AVC for a cached decision. If one exists, it is used immediately without re-evaluating the full policy. This improves performance while maintaining strict enforcement.

---

## 2. SELinux Terminology

### 2.1 Subject

A *subject* is a user or process attempting to access an object. Examples: `system_u` for SELinux system processes, `unconfined_u` for users not restricted by policy. The subject is stored in **field 1** of the SELinux context.

### 2.2 Object

An *object* is any resource a subject attempts to access — files, directories, file systems, hardware devices, network interfaces, ports, pipes, and sockets. Examples: `object_r` for general objects, `system_r` for system-owned objects, `unconfined_r` for objects not bound by policy.

### 2.3 Access

An *access* is an action performed by a subject on an object — reading a file, writing to a directory, executing a program, or listening on a network port.

### 2.4 Policy

A *policy* is the comprehensive rule set SELinux enforces system-wide. SELinux does not rely on user discretion; it consults the policy for every access decision. Any action not explicitly allowed is denied.

### 2.5 Context and Label

A *context* (or *label*) is a tag that stores security attributes for subjects and objects. Every SELinux context consists of four fields:

```
user : role : type (or domain) : level
```

SELinux uses this information to make all access control decisions.

### 2.6 SELinux User

SELinux defines several user identities, each authorized for specific roles and security levels. Linux users are mapped to SELinux users through policy rules. For example, a Linux user mapped to `user_u` runs in confined domains and is restricted from `su`, `sudo`, and executing programs from home directories.


### 2.7 Role

A *role* is an attribute of SELinux's Role-Based Access Control (RBAC) model. It classifies which subjects may access which domains or types. Example roles: `user_r` for ordinary users, `sysadm_r` for administrators, `system_r` for processes that initiate under the system role. The role is stored in **field 2** of the context.

### 2.8 Type Enforcement

*Type enforcement* (TE) is the primary access control mechanism in SELinux. It defines rules that control how processes running in specific domains interact with objects labeled with particular types — files, directories, devices, network resources. Access decisions are evaluated by comparing the SELinux contexts of both subject and object.

### 2.9 Type and Domain

A *type* classifies objects (files, directories, devices) based on their security requirements. Objects sharing common security characteristics are grouped under the same type. Examples: `user_home_dir_t` for objects in user home directories, `usr_t` for most objects under `/usr`. The type is stored in **field 3** of a file context.
SELinux policy rules specify how domains interact with object types, which domain transitions are permitted, and how domains interact with each other. Access is allowed only if explicitly permitted — all other attempts are denied by default.

### 2.10 Level

A *level* is used by Multi-Level Security (MLS) and Multi-Category Security (MCS). It is a `sensitivity:categ
ory` pair defining the security level within a context. In RHEL, the targeted policy enforces MCS, not full MLS. Under MCS, all processes operate at sensitivity level `s0` but may be assigned 0 to 1023 categories for additional isolation. A range like `s0-s0:c0.c1023` means sensitivity `s0`, categories `c0` through `c1023`.

### 2.11 User Context Example

```bash
$ id -Z
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```
This output shows the current user is mapped to `unconfined_u` — meaning no SELinux policy restrictions apply to processes launched by this user. In a default RHEL configuration, this is true for all Linux users including `root`. Under this default, SELinux enforcement targets system services and daemons, not interactive user sessions.


## 3. SELinux Management Packages and Commands

Managing SELinux requires several packages. These provide the tools to inspect mode, contexts, policy, and Booleans:

| Package | Provides |
|---------|----------|
| `libselinux-utils` | `getenforce`, `setenforce`, `getsebool` |
| `policycoreutils` | `sestatus`, `setsebool`, `restorecon` |
| `policycoreutils-python-utils` | `semanage` |
| `setools-console` | `seinfo`, `sesearch` |

For graphical alert viewing and debugging, the `setroubleshoot-server` package provides the SELinux Alert Browser.


### 3.1 Installing setools-console

`setools-console` provides `seinfo` and `sesearch`, which are required for querying the policy database.

```bash 
$ sudo dnf install setools-console
Updating Subscription Management repositories.
Last metadata expiration check: 0:00:39 ago on Tue 02 Jun 2026 03:14:56 AM +03.
Dependencies resolved.
======================================================================================================================
 Package                    Architecture      Version                 Repository                                 Size
======================================================================================================================
Installing:
 setools-console            x86_64            4.6.0-3.el10            rhel-10-for-x86_64-baseos-rpms             53 k

Transaction Summary
======================================================================================================================
Install  1 Package

Total download size: 53 k
Installed size: 139 k
Is this ok [y/N]: y
Downloading Packages:
setools-console-4.6.0-3.el10.x86_64.rpm                                                75 kB/s |  53 kB     00:00    
----------------------------------------------------------------------------------------------------------------------
Total                                                                                  74 kB/s |  53 kB     00:00     
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                              1/1 
  Installing       : setools-console-4.6.0-3.el10.x86_64                                                          1/1 
  Running scriptlet: setools-console-4.6.0-3.el10.x86_64                                                          1/1 
Installed products updated.

Installed:
  setools-console-4.6.0-3.el10.x86_64                                                                                 

Complete!
``` 

### 3.2 Management Command Reference

| Command | Category | Description |
|---------|----------|-------------|
| `getenforce` | Mode | Displays the current operating mode |
| `setenforce` | Mode | Switches mode between enforcing and permissive at runtime |
| `sestatus` | Mode | Shows full SELinux runtime status and Boolean values |
| `grubby` | Mode | Updates GRUB2 boot loader arguments persistently |
| `chcon` | Context | Changes file context (does not survive filesystem relabeling) |
| `restorecon` | Context | Restores default contexts from the policy database |
| `semanage` | Context / Policy / Boolean | Changes contexts persistently, manages policy database, manages Booleans |
| `seinfo` | Policy | Provides information on policy components |
| `sesearch` | Policy | Searches rules within the policy database |
| `getsebool` | Boolean | Displays Boolean values and their current settings |
| `setsebool` | Boolean | Modifies Boolean values temporarily or permanently |
| `sealert` | Troubleshooting | Reads AVC denial logs and suggests fixes (setroubleshoot-server) |


### 3.3 Querying the Policy Database

#### 3.3.1 Listing SELinux Users

```bash
$ sudo seinfo -u 

Users: 8
   guest_u
   root
   staff_u
   sysadm_u
   system_u
   unconfined_u
   user_u
   xguest_u
``` 
> 8 SELinux user identities are defined in the policy database. `unconfined_u` is the default for all Linux users including root in a standard RHEL installation.

#### 3.3.2 Listing SELinux Types
```bash
$ sudo seinfo -t | head

Types: 4718
   NetworkManager_dispatcher_chronyc_script_t
   NetworkManager_dispatcher_chronyc_t
   NetworkManager_dispatcher_cloud_script_t
   NetworkManager_dispatcher_cloud_t
   NetworkManager_dispatcher_console_script_t
   NetworkManager_dispatcher_console_t
   NetworkManager_dispatcher_console_var_run_t
   NetworkManager_dispatcher_custom_t
```
> 4,718 types are defined in the targeted policy database — one for each security classification group of objects and processes.

#### 3.3.3 Listing SELinux Roles

```bash
$ sudo seinfo -r

Roles: 15
   auditadm_r
   container_user_r
   dbadm_r
   guest_r
   logadm_r
   nx_server_r
   object_r
   secadm_r
   staff_r
   sysadm_r
   system_r
   unconfined_r
   user_r
   webadm_r
   xguest_r
```

> 15 roles are defined in the policy. `object_r` is the default role assigned to all non-process objects (files, directories, ports).

### 3.4 Viewing User-to-SELinux User Mappings

```bash 
$ sudo semanage login -l 
[sudo] password for blue: 

Login Name           SELinux User         MLS/MCS Range        Service

__default__          unconfined_u         s0-s0:c0.c1023       *
root                 unconfined_u         s0-s0:c0.c1023       *
```

> Column 1 is the Linux login name. Column 2 is the SELinux user that login maps to. Column 3 is the MLS/MCS security range. The `*` in column 4 means all services. `__default__` covers all Linux users not explicitly listed — they all map to `unconfined_u`.




## 4. Security Contexts

### 4.1 Context for Processes

The `-Z` flag on `ps` reveals the SELinux label for each running process. The context format is `user:role:domain:level`.

```bash
$ ps -eZ | head
LABEL                               PID TTY          TIME CMD
system_u:system_r:init_t:s0           1 ?        00:00:02 systemd
system_u:system_r:kernel_t:s0         2 ?        00:00:00 kthreadd
system_u:system_r:kernel_t:s0         3 ?        00:00:00 pool_workqueue_release
system_u:system_r:kernel_t:s0         4 ?        00:00:00 kworker/R-rcu_gp
system_u:system_r:kernel_t:s0         5 ?        00:00:00 kworker/R-sync_wq
system_u:system_r:kernel_t:s0         6 ?        00:00:00 kworker/R-kvfree_rcu_reclaim
system_u:system_r:kernel_t:s0         7 ?        00:00:00 kworker/R-slub_flushwq
system_u:system_r:kernel_t:s0         8 ?        00:00:00 kworker/R-netns
system_u:system_r:kernel_t:s0        10 ?        00:00:00 kworker/0:0H-events_highpri
```
Context field breakdown for process labels:

```
system_u  :  system_r  :  init_t  :  s0
[field 1]    [field 2]    [field 3]   [field 4]
SELinux user  role         domain      sensitivity level
```
### 4.2 Context for Files and Directories

The `-Z` flag on `ls` reveals the SELinux label for files and directories. For files, field 3 is a *type*; for process contexts it is a *domain* — same field position, different name depending on whether the subject is a file or a process.

```bash
$ ls -lZ /etc/passwd
-rw-r--r--. 1 root root system_u:object_r:passwd_file_t:s0 2567 May 24 23:58 /etc/passwd
```
#### 4.2.1 Security conetxt for dir
```bash
$ ls -ldZ Downloads
drwxr-xr-x. 2 blue blue unconfined_u:object_r:user_home_t:s0 6 May 10 22:33 Downloads
``` 
> `s0` is the sensitivity level — the fourth field of every context. In a standard targeted policy deployment using MCS, `s0` is the only sensitivity level used.


### 4.3 Context for Ports

SELinux also labels network ports. A service attempting to bind to a port not permitted by policy will be denied — regardless of whether firewall rules allow the traffic.

```bash 
$ sudo semanage port -l  | head
SELinux Port Type              Proto    Port Number

afs3_callback_port_t           tcp      7001
afs3_callback_port_t           udp      7001
afs_bos_port_t                 udp      7007
afs_fs_port_t                  tcp      2040
afs_fs_port_t                  udp      7000, 7005
afs_ka_port_t                  udp      7004
afs_pt_port_t                  tcp      7002
afs_pt_port_t                  udp      7002
```


Column 1 is the SELinux port type, column 2 is the protocol, and column 3 lists the associated port numbers.

### 4.4 SELinux Booleans Filesystem

Booleans are on/off switches that enable or disable conditional rules within the SELinux policy without recompiling or reloading it. Their values are stored as virtual files in `/sys/fs/selinux/booleans/`, one file per Boolean.

```bash 
$ ll /sys/fs/selinux/booleans/ | head 
total 0
-rw-r--r--. 1 root root 0 Jun  1 19:58 abrt_anon_write
-rw-r--r--. 1 root root 0 Jun  1 19:58 abrt_handle_event
-rw-r--r--. 1 root root 0 Jun  1 19:58 abrt_upload_watch_anon_write
-rw-r--r--. 1 root root 0 Jun  1 19:58 auditadm_exec_content
-rw-r--r--. 1 root root 0 Jun  1 19:58 authlogin_nsswitch_use_ldap
-rw-r--r--. 1 root root 0 Jun  1 19:58 authlogin_radius
-rw-r--r--. 1 root root 0 Jun  1 19:58 authlogin_yubikey
-rw-r--r--. 1 root root 0 Jun  1 19:58 cdrecord_read_content
-rw-r--r--. 1 root root 0 Jun  1 19:58 cluster_can_network_connect
``` 

> A standard RHEL system has hundreds of Booleans. Boolean manual pages are provided by the `selinux-policy-doc` package and can be browsed with `man -K <boolean_name>`.


## 5. Viewing and Controlling SELinux State

### 5.1 The Configuration File

The primary SELinux configuration file is `/etc/selinux/config`. It controls the activation mode and loaded policy name. Changes here require a **system reboot** to take effect.


```bash 
$ cat /etc/selinux/config 

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
# See also:
# https://docs.fedoraproject.org/en-US/quick-docs/getting-started-with-selinux/#getting-started-with-selinux-selinux-states-and-modes
#
# NOTE: In earlier Fedora kernel builds, SELINUX=disabled would also
# fully disable SELinux during boot. If you need a system with SELinux
# fully disabled instead of SELinux running with no policy loaded, you
# need to pass selinux=0 to the kernel command line. You can use grubby
# to persistently set the bootloader to boot with selinux=0:
#
#    grubby --update-kernel ALL --args selinux=0
#
# To revert back to SELinux enabled:
#
#    grubby --update-kernel ALL --remove-args selinux
#
SELINUX=enforcing  
# SELINUXTYPE= can take one of these three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted

```

The `SELINUX` directive controls the activation mode:

| Mode | Behavior |
|------|----------|
| `enforcing` | SELinux is active and denies or allows actions based on policy rules |
| `permissive` | SELinux is active but only logs policy violations — nothing is blocked. Useful for troubleshooting |
| `disabled` | SELinux is completely off — no policy is loaded |

> To fully disable SELinux at the kernel level (not just via config), use `grubby` to pass `selinux=0` as a kernel argument — see Section 5.4.

### 5.2 Getting the Active Mode

`getenforce` reports the current runtime mode — not what is set in the config file, but what is actually active right now.

```bash 
$ getenforce
Enforcing
```
### 5.3 Switching Modes at Runtime

`setenforce` changes the mode immediately without a reboot. It toggles between `enforcing` and `permissive` only — it cannot disable SELinux at runtime.

```bash
$ sudo setenforce permissive 
[sudo] password for blue: 
```
#### 5.3.1 Verify Permissive Mode
```bash
$ getenforce
Permissive
```
#### 5.3.2  Switching using binary 
```bash
$ sudo setenforce 1
```
#### 5.3.3 Verifying 
```bash
$ getenforce
Enforcing
```


> Use `permissive` mode when troubleshooting a service that appears non-functional due to SELinux denials. Always switch back to `enforcing` after resolving the issue. You can use `1` for enforcing and `0` for permissive as numeric arguments.

### 5.4 Disabling SELinux at Boot via grubby

`grubby` modifies the GRUB2 boot loader configuration persistently. Adding `selinux=0` as a kernel argument disables SELinux entirely at boot — independent of the `/etc/selinux/config` setting.


#### 5.4.1 Confirm Kernel Version
```bash 
 $ uname -r
6.12.0-211.18.1.el10_2.x86_64
```
#### 5.4.2 Initial Boot Entry State
```bash 
$ sudo cat  /boot/loader/entries/4e8d59cb84f34f1e90ecac4543c3db88-6.12.0-211.18.1.el10_2.x86_64.conf
title Red Hat Enterprise Linux (6.12.0-211.18.1.el10_2.x86_64) 10.2 (Coughlan)
version 6.12.0-211.18.1.el10_2.x86_64
linux /vmlinuz-6.12.0-211.18.1.el10_2.x86_64
initrd /initramfs-6.12.0-211.18.1.el10_2.x86_64.img $tuned_initrd
options root=/dev/mapper/rhel-root ro crashkernel=2G-64G:256M,64G-:512M resume=UUID=b9fcef3a-852c-4635-85a5-8ae383e086cd rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet $tuned_params
grub_users $grub_users
grub_arg --unrestricted
grub_class rhel
```

#### 5.4.3 Add selinux=0 to Kernel Arguments
```bash 
$ sudo grubby --update-kernel ALL --args selinux=0 
```

#### 5.4.4 Verify the Argument Was Added
```bash 
$ sudo cat  /boot/loader/entries/4e8d59cb84f34f1e90ecac4543c3db88-6.12.0-211.18.1.el10_2.x86_64.conf
title Red Hat Enterprise Linux (6.12.0-211.18.1.el10_2.x86_64) 10.2 (Coughlan)
version 6.12.0-211.18.1.el10_2.x86_64
linux /vmlinuz-6.12.0-211.18.1.el10_2.x86_64
initrd /initramfs-6.12.0-211.18.1.el10_2.x86_64.img $tuned_initrd
options root=/dev/mapper/rhel-root ro crashkernel=2G-64G:256M,64G-:512M resume=UUID=b9fcef3a-852c-4635-85a5-8ae383e086cd rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet $tuned_params selinux=0  # HERE WE GO
grub_users $grub_users
grub_arg --unrestricted
grub_class rhel
```

> `selinux=0` is now appended to the kernel options line. A reboot would bring the system up with SELinux completely disabled.

#### 5.4.5 Remove the Kernel Argument

```bash
$ sudo grubby --update-kernel ALL --remove-args selinux=0 
```

#### 5.4.6 Verify Removal

```bash 
$ sudo cat  /boot/loader/entries/4e8d59cb84f34f1e90ecac4543c3db88-6.12.0-211.18.1.el10_2.x86_64.conf
title Red Hat Enterprise Linux (6.12.0-211.18.1.el10_2.x86_64) 10.2 (Coughlan)
version 6.12.0-211.18.1.el10_2.x86_64
linux /vmlinuz-6.12.0-211.18.1.el10_2.x86_64
initrd /initramfs-6.12.0-211.18.1.el10_2.x86_64.img $tuned_initrd
options root=/dev/mapper/rhel-root ro crashkernel=2G-64G:256M,64G-:512M resume=UUID=b9fcef3a-852c-4635-85a5-8ae383e086cd rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet $tuned_params
grub_users $grub_users
grub_arg --unrestricted
grub_class rhel
```
> `selinux=0` is removed. The system will boot with SELinux enabled again. Note: to disable SELinux permanently via config file, edit `/etc/selinux/config` and set `SELINUX=disabled`, then reboot.


## 6. Querying Full Runtime Status

### 6.1 Running sestatus

`sestatus` provides the complete SELinux runtime picture — status, filesystem mount, root directory, loaded policy name, current mode, config file mode, and policy settings.

```bash
$ sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
```
> When invoked with `-v`, `sestatus` also reports the SELinux contexts for the current process, the `systemd` (init) process, the controlling terminal, and key system files — using the file list defined in `/etc/sestatus.conf`.

### 6.2 The sestatus Configuration File

`/etc/sestatus.conf` defines which files and processes `sestatus -v` includes in its context report:

```bash 
$ cat /etc/sestatus.conf 
[files]
/etc/passwd
/etc/shadow
/bin/bash
/bin/login
/bin/sh
/sbin/agetty
/sbin/init
/sbin/mingetty
/usr/sbin/sshd
/lib/libc.so.6
/lib/ld-linux.so.2
/lib/ld.so.1

[process]
/sbin/mingetty
/sbin/agetty
/usr/sbin/sshd
```

## 7. Modifying SELinux Contexts with chcon

`chcon` changes the SELinux context of a file or directory immediately at runtime. Changes made with `chcon` do **not** survive a filesystem relabeling — if `restorecon` or a full relabel is run, the system reverts to the context defined in the policy database. Use `semanage fcontext` for persistent changes (see Section 8).

### 7.1 Creating Test Files

```bash
$ mkdir usman-se1
$ touch usman-se1/usman-se-file1
```

### 7.2 Checking Initial Contexts

```bash 
$ ls -lZ usman-se1/usman-se-file1 ; ls -ldZ usman-se1 
-rw-r--r--. 1 blue blue unconfined_u:object_r:user_home_t:s0 0 Jun  2 04:23 usman-se1/usman-se-file1
drwxr-xr-x. 2 blue blue unconfined_u:object_r:user_home_t:s0 28 Jun  2 04:23 usman-se1
``` 
Both the directory and its file inherit `user_home_t` from the home directory.

### 7.3 Changing User and Type with chcon

Get available users and types to pick from:

```bash 
$ seinfo -u  ;  echo ; seinfo -t | tail 

Users: 8
   guest_u
   root
   staff_u
   sysadm_u
   system_u
   unconfined_u
   user_u
   xguest_u

   zookeeper_election_port_t
   zookeeper_election_server_packet_t
   zookeeper_leader_client_packet_t
   zookeeper_leader_port_t
   zookeeper_leader_server_packet_t
   zope_client_packet_t
   zope_port_t
   zope_server_packet_t
   zos_remote_exec_t
   zos_remote_t
```
Change the SELinux user to `user_u` and type to `etc_t` recursively:

```bash 
$ sudo chcon -u user_u -t etc_t usman-se1 -R 
```
#### 7.3.1 Verify

```bash
$ ls -lZ usman-se1/usman-se-file1 ; ls -ldZ usman-se1 
-rw-r--r--. 1 blue blue user_u:object_r:etc_t:s0 0 Jun  2 04:23 usman-se1/usman-se-file1
drwxr-xr-x. 2 blue blue user_u:object_r:etc_t:s0 28 Jun  2 04:23 usman-se1
```
> Changing the SELinux user on files is rarely needed in real-world administration. Type enforcement is what primarily controls access decisions. The user field matters mainly for user confinement scenarios.



## 8. Persistent Context Mapping with semanage and restorecon

`semanage fcontext` writes a context mapping to the SELinux policy database. Once written, `restorecon` applies it to the filesystem. This combination is the correct and persistent approach — unlike `chcon`, a `semanage`-applied context survives filesystem relabeling.

Context definitions for system-installed and user-created files are stored in the `file_contexts` and `file_contexts.local` policy files located in `/etc/selinux/targeted/contexts/files/`.

### 8.1 Adding a Custom File Context to the Policy Database

#### 8.1.1 Check Initial Context

```bash
$ ls -lZ usman-se1/usman-se-file1 ; ls -ldZ usman-se1 
-rw-r--r--. 1 blue blue user_u:object_r:etc_t:s0 0 Jun  2 04:23 usman-se1/usman-se-file1
drwxr-xr-x. 2 blue blue user_u:object_r:etc_t:s0 28 Jun  2 04:23 usman-se1
``` 

#### 8.1.2 Add the Policy Mapping

```bash 
$ sudo semanage fcontext -as guest_u -t public_content_t '/home/blue/usman-se1(/.*)?' 
[sudo] password for blue:
```

> The regular expression `(/.*)?` tells `semanage` to include all files and subdirectories under the target path. Without it, only the directory itself is matched — no recursion. Omit it when you only need to label the top-level object.

#### 8.1.3 Verify the Policy Entry

```bash
$ sudo semanage fcontext -lC
SELinux fcontext                                   type               Context

/home/blue/usman-se1(/.*)?                         all files          guest_u:object_r:public_content_t:s0 
``` 

#### 8.1.4 Verify via the Local Policy File

```bash
$ cat /etc/selinux/targeted/contexts/files/file_contexts.local
# This file is auto-generated by libsemanage
# Do not edit directly.

/home/blue/usman-se1(/.*)?    guest_u:object_r:public_content_t:s0
``` 

> The local policy database is stored under `/etc/selinux/targeted/contexts/files/`. This file is auto-generated by `libsemanage` — do not edit it directly.


### 8.2 Deliberately Breaking the Context with chcon

`chcon` can intentionally override the context — useful to simulate a mislabeling and then demonstrate `restorecon` recovery:
```bash
$ sudo chcon -vt httpd_sys_content_t  usman-se1/ -R 
changing security context of 'usman-se1/usman-se-file1'
changing security context of 'usman-se1/'
 
``` 

#### 8.2.1 Verify the Broken Context

```bash 
$ ls -lZ usman-se1/usman-se-file1 ; ls -ldZ usman-se1 
-rw-r--r--. 1 blue blue user_u:object_r:httpd_sys_content_t:s0 0 Jun  2 04:23 usman-se1/usman-se-file1
drwxr-xr-x. 2 blue blue user_u:object_r:httpd_sys_content_t:s0 28 Jun  2 04:23 usman-se1
``` 
The type has been overridden to `httpd_sys_content_t` by `chcon`.

### 8.3 Restoring the Context from the Policy Database

`restorecon` reads the policy database and reapplies the correct context from the `semanage`-defined mapping:

```bash 
$ sudo restorecon -Rv usman-se1 
Relabeled /home/blue/usman-se1 from user_u:object_r:httpd_sys_content_t:s0 to user_u:object_r:public_content_t:s0
Relabeled /home/blue/usman-se1/usman-se-file1 from user_u:object_r:httpd_sys_content_t:s0 to user_u:object_r:public_content_t:s0
```

#### 8.3.1 Verify Full Restoration

```bash 
$ ls -lZ usman-se1/usman-se-file1 ; ls -ldZ usman-se1 
-rw-r--r--. 1 blue blue user_u:object_r:public_content_t:s0 0 Jun  2 04:23 usman-se1/usman-se-file1
drwxr-xr-x. 2 blue blue user_u:object_r:public_content_t:s0 28 Jun  2 04:23 usman-se1
``` 

> Only the **type** changed back to `public_content_t`. The **user** field (`user_u`) was set earlier with `chcon -u` and `semanage` preserves it as-is. `restorecon` restores the type and role from the policy mapping — the SELinux user component is kept from what was last applied.


## 9. Managing SELinux Port Labels

SELinux enforces which ports a service is permitted to bind to. Even if a firewall rule allows the traffic, SELinux will deny the bind if the port is not labeled with the correct type for that service.

### 9.1 Listing Port Contexts

Filter for `http_port_t` to see which TCP/UDP ports are permitted for the HTTP service:
```bash
$ sudo semanage port -l | grep http_port
http_port_t                    tcp      9005, 80, 81, 443, 488, 8008, 8009, 8443, 9000
http_port_t                    udp      80, 443
pegasus_http_port_t            tcp      5988
```

### 9.2 Adding a Port Label

Add port `8007/tcp` to the `http_port_t` type so an HTTP service can bind to it:

```bash
$ sudo semanage port -at http_port_t  -p tcp 8007
```
#### 9.2.1 Verify 
```bash
$ sudo semanage port -l | grep ^http_port
http_port_t                    tcp      8007, 9005, 80, 81, 443, 488, 8008, 8009, 8443, 9000
http_port_t                    udp      80, 443
``` 
Port `8007` is now prepended to the list — `httpd` can now legally bind to it.

### 9.3 Removing a Port Label

```bash 
$ sudo semanage port -dp tcp 8007 
[sudo] password for blue: 
```
#### 9.3.1 Verify

```bash
$ sudo semanage port -l | grep ^http_port
http_port_t                    tcp      9005, 80, 81, 443, 488, 8008, 8009, 8443, 9000
http_port_t                    udp      80, 443
``` 
Port `8007` is removed. Attempts by `httpd` to bind to it will now be denied by SELinux.


## 10. Copy and Context Inheritance

When copying files, SELinux context behavior depends on whether `--preserve=context` is used. Understanding this is critical for avoiding silent mislabeling when deploying files to new locations.

### 10.1 Copy Without Preserve — Inherits Destination Context

Create a file in the home directory and confirm its context:

```bash
$ pwd
/home/blue
$ touch usman-selinux 
```
### 10.2 verify context 
```bash 
$ ls -lZ usman-selinux 
-rw-r--r--. 1 blue blue unconfined_u:object_r:user_home_t:s0 0 Jun  2 05:32 usman-selinux
```

### 10.3 Copy to `/tmp/` without `--preserve=context`:
```bash
$ sudo cp usman-selinux /tmp/
```
### 10.4  Verify Context in Destination
```bash 
$ ls -lZ /tmp/usman-selinux 
-rw-r--r--. 1 root root unconfined_u:object_r:user_tmp_t:s0 0 Jun  2 05:34 /tmp/usman-selinux
``` 
> The file inherited `user_tmp_t` — the context of the `/tmp` directory — not the original `user_home_t`. This is the default copy behavior: the destination directory's context wins.


### 10.5 Copy With --preserve=context — Retains Source Context

```bash
$ touch usman-sefile2
```

### 10.5.1 verify context  
```bash
$ ls -lZ usman-sefile2 
-rw-r--r--. 1 blue blue unconfined_u:object_r:user_home_t:s0 0 Jun  2 05:39 usman-sefile2
```

#### 10.5.2 Copy to `/tmp/` with `--preserve=context`: 
```bash
$ sudo cp usman-sefile2 --preserve=context /tmp/ 
```
#### 10.5.3 Verify Context in Destination


```bash
$ ls -lZ /tmp/usman-sefile2 
-rw-r--r--. 1 root root unconfined_u:object_r:user_home_t:s0 0 Jun  2 05:43 /tmp/usman-sefile2
``` 
> The context is `user_home_t` — the source file's original context was preserved despite being copied to `/tmp`. This is the correct approach when deploying files to a new location while retaining their intended security label.

#### 10.6 Tear down
```bash
$ sudo rm usman-sefile2 ; sudo rm /tmp/usman-sefile2
$ sudo rm usman-selinux ; sudo rm /tmp/usman-selinux 
``` 

> The same copy context behavior applies to directories — not just individual files.


## 11. Move and Context Retention

When a file or directory is **moved** (rather than copied), its SELinux context stays unchanged regardless of the destination directory's context. This is the opposite of the default copy behavior.

### 11.1 Moving a Directory


```bash
$ ls -ldZ Music/
drwxr-xr-x. 2 blue blue unconfined_u:object_r:audio_home_t:s0 6 May 10 22:33 Music/
```
```bash
$ mv Music /tmp/ 
```
### 11.1.1  Verify
```bash
$ ls -ldZ /tmp/Music
drwxr-xr-x. 2 blue blue unconfined_u:object_r:audio_home_t:s0 6 May 10 22:33 /tmp/Music
```
> `audio_home_t` is retained after moving into `/tmp`. A copy without `--preserve=context` would have changed it to `user_tmp_t`.


### 11.2 Moving a File

```bash
$ ls -lZ no-internet-in-school 
-rw-r--r--. 1 blue blue unconfined_u:object_r:user_home_t:s0 0 May 23 21:00 no-internet-in-school
``` 
```bash
$ mv no-internet-in-school /tmp
```
```bash
$ ls -lZ /tmp/no-internet-in-school 
-rw-r--r--. 1 blue blue unconfined_u:object_r:user_home_t:s0 0 May 23 21:00 /tmp/no-internet-in-school
```


> `user_home_t` is retained after the move. Context inheritance behavior for copy/move/archive:
>
> - **Copy to a different directory:** inherits destination context
> - **Copy overwriting an existing file:** inherits the overwritten file's context
> - **Move:** context remains unchanged
> - **Archive with tar:** use `--selinux` to preserve context within the archive


## 12. Boolean Toggle

Booleans let administrators enable or disable specific conditional rules in the SELinux policy dynamically — without recompiling or reloading the policy. Changes take effect immediately in both temporary and permanent modes.

The Boolean `ssh_use_tcpd` is used here to demonstrate both runtime and persistent toggling, with reboot verification to confirm behavior.

### 12.1 Checking Boolean State

Three ways to query a Boolean's current value:
```bash
$ sudo semanage boolean -l | grep ssh_use_tcp
ssh_use_tcpd                   (off  ,  off)  Allow ssh to use tcpd
```
```bash
$ getsebool -a | grep ssh_use_tcp 
ssh_use_tcpd --> off
``` 
```bash
$ sestatus -b  | grep ssh_use_tcp 
ssh_use_tcpd                                off
```
> `semanage boolean -l` shows two values in parentheses: `(runtime, persistent)`. Both being `off` means the Boolean is off now and will remain off after a reboot.


### 12.2 Enabling a Boolean at Runtime Only

```bash
$ sudo setsebool ssh_use_tcpd on 
```

#### 12.2.1 Verify Runtime Change

```bash
$ getsebool -a | grep ssh_use_tcp 
ssh_use_tcpd --> on
```
```bash
$ sudo semanage boolean -l | grep ssh_use_tcp
ssh_use_tcpd                   (on   ,  off)  Allow ssh to use tcpd
``` 
> The `semanage` output now shows `(on, off)` — runtime is `on`, but the persistent (post-reboot) value is still `off`.

### 12.3 Verifying Non-Persistence After Reboot

```bash
$ reboot
```  

#### 12.3.1  Verify state 
```bash
$ getsebool -a | grep ssh_use_tcp
ssh_use_tcpd --> off
```
```bash 
$ sudo semanage boolean -l | grep ssh_use_tcp
[sudo] password for blue: 
ssh_use_tcpd                   (off  ,  off)  Allow ssh to use tcpd
```

> After reboot, the Boolean reverted to `off` — confirming that `setsebool` without `-P` is runtime-only.

### 12.4 Enabling a Boolean Persistently

```bash
$ sudo setsebool -P ssh_use_tcpd 1
```
The `-P` flag writes the Boolean value to the policy database on disk:



#### 12.4.1 Verify Persistent State

```bash
$ sudo semanage boolean -l | grep ssh_use_tcp 
ssh_use_tcpd                   (on   ,   on)  Allow ssh to use tcpd
```
```bash
$ getsebool -a | grep ssh_use_tcp
ssh_use_tcpd --> on
``` 
> Both values are now `(on, on)` — the Boolean is active now and will survive a reboot.

### 12.5 Reboot
```bash
$ reboot
``` 

### 12.5 Verifying Persistence After Reboot

```bash
$ sudo semanage boolean -l | grep ssh_use_tcp 
[sudo] password for blue: 
ssh_use_tcpd                   (on   ,   on)  Allow ssh to use tcpd
``` 
```bash
$ getsebool -a | grep ssh_use_tcp
ssh_use_tcpd --> on
``` 


> The Boolean survived the reboot with both runtime and persistent values set to `on`. This confirms that `-P` writes to the policy database and the value persists across boots.

---

## 13. Key Takeaways

- **MAC sits above DAC.** SELinux adds a second layer of access control on top of standard Linux permissions. Even if DAC allows access, SELinux can deny it. Both must permit an action for it to succeed.
- **Default-deny.** Any action not explicitly permitted by the SELinux policy is denied. There is no implicit allow.
- **Context is everything.** Every subject (process) and object (file, port, socket) carries a four-field security label: `user:role:type:level`. Access decisions are made by comparing these labels against the active policy.
- **Type enforcement is the primary mechanism.** The type field (field 3) is what actually governs access in the targeted policy. The user and role fields matter more in confined user scenarios.
- **`chcon` is temporary, `semanage` is permanent.** Use `chcon` for quick one-off context changes that do not need to survive relabeling. Use `semanage fcontext` + `restorecon` for anything that must persist through a full relabel or system rebuild.
- **Copy inherits, move retains.** Copying a file to a new directory gives it the destination directory's context by default. Use `--preserve=context` to retain the source context. Moving a file always retains its original context regardless of destination.
- **Port labeling enforces service binding.** A service cannot bind to a port unless SELinux policy explicitly permits it. Firewall rules alone are not sufficient — the port must carry the correct SELinux type label too.
- **Boolean toggle without `-P` is runtime-only.** `setsebool` without `-P` changes the runtime state immediately but resets on reboot. Add `-P` to write the change to the policy database and make it persistent.
- **`permissive` mode is for troubleshooting only.** Switch to permissive when diagnosing SELinux-related service failures. Always switch back to `enforcing` after resolving the issue. Leaving a production system in permissive mode defeats the entire security model.
- **To fully disable SELinux at boot**, use `grubby --update-kernel ALL --args selinux=0`. Editing `/etc/selinux/config` to `SELINUX=disabled` and rebooting also works — but the `grubby` method operates at the kernel argument level, independent of the config file.

