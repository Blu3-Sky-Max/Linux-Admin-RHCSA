# Network File System Management

**Author:** Usman O. Olanrewaju (Blu3 Sky)

**Date:** 2026-05-11

**Environment:** RHEL 10 (NFS Server) · Fedora 44 (SSHFS Client/Client) · VirtualBox KVM VMs

**Focus:** NFS · SSHFS · AutoFS — installation, configuration, persistence, and automounting

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [NFS Server Setup](#2-nfs-server-setup)
3. [Exporting NFS Shares](#3-exporting-nfs-shares)
4. [NFS Client Setup](#4-nfs-client-setup)
5. [NFS Persistence via fstab](#5-nfs-persistence-via-fstab)
6. [SSHFS — Client-Side Setup](#6-sshfs--client-side-setup)
7. [SSHFS Persistence via fstab](#7-sshfs-persistence-via-fstab)
8. [AutoFS — Concepts and Configuration](#8-autofs--concepts-and-configuration)
9. [AutoFS Direct Method — Server](#9-autofs-direct-method--server)
10. [AutoFS Direct Method — Client](#10-autofs-direct-method--client)
11. [AutoFS Indirect Method — Server](#11-autofs-indirect-method--server)
12. [AutoFS Indirect Method — Client (with pre-created mount point)](#12-autofs-indirect-method--client-with-pre-created-mount-point)
13. [AutoFS Indirect — Wildcard Mounting](#13-autofs-indirect--wildcard-mounting)
14. [AutoFS Indirect — Client (without pre-creating mount point)](#14-autofs-indirect--client-without-pre-creating-mount-point)
15. [AutoFS via /misc (built-in indirect map)](#15-autofs-via-misc-built-in-indirect-map)
16. [AutoFS Home Directory Automounting — Server](#16-autofs-home-directory-automounting--server)
17. [AutoFS Home Directory Automounting — Client](#17-autofs-home-directory-automounting--client)

---
 

## 1. Core Concepts

### 1.1 Network File System (NFS)

NFS is a distributed file system protocol that allows directories on a remote machine to be shared and accessed over a network as if they were local. It operates in a server/client model.

| Term | Definition |
|---|---|
| **NFS Server** | The machine that owns and exports a directory for network access. Making a directory available is called *exporting*. |
| **NFS Client** | Any machine that accesses an exported share. Attaching that share locally is called *mounting*. |
| **NFSv3** | Supports both UDP and TCP at the transport layer. |
| **NFSv4** | Uses TCP exclusively by default. RHEL 10 defaults to NFSv4.2. |

**Important:** NFS requires two network endpoints — a server and a client. Two users on the same machine do not need NFS; standard Unix file permissions and group membership handle that case. For lab practice, a single machine can run both server and client over the loopback interface (`127.0.0.1`), but RHCSA exam scenarios always involve two separate machines.

**NFS benefits:**
- Cross-platform: supports Linux, UNIX, and Windows (via additional services).
- Multiple clients can mount and access a single share simultaneously.
- Centralizes shared binaries and read-only data, reducing storage cost and admin overhead.
- Enables consolidation of user home directories on the server and exporting them to all clients — users maintain only one home directory regardless of which client they log into.
### 1.2 SSH Filesystem (SSHFS)

SSHFS mounts a remote directory over an SSH connection using FUSE (Filesystem in Userspace). It does not require a dedicated NFS server — any machine running `sshd` can act as the remote. Configuration is done entirely on the client side. Because it tunnels through SSH, all data is encrypted in transit.

### 1.3 AutoFS

AutoFS is a kernel-assisted service that mounts and unmounts network shares automatically at runtime on demand, and survives system reboots without `/etc/fstab` entries. Its core advantages over static fstab mounts are:

- **No root required for access.** Users can access AutoFS-managed NFS shares without `root` privileges. Both manual mounting and fstab mounting require root.
- **Prevents client hangs.** If the NFS server is down or unreachable, AutoFS prevents the client from hanging. Static fstab mounts can cause the client to hang indefinitely waiting for a server that is not available.
- **Automatic unmounting.** A share is unmounted after a configurable idle timeout (default: 300 seconds). With fstab, the share stays mounted until manually unmounted or the system shuts down — wasting kernel resources.
- **Wildcard and environment variable support.** AutoFS map files support wildcard characters (`*`) and environment variables. Static fstab does not.
- **No fstab conflicts.** Mounts managed by AutoFS must not be defined in `/etc/fstab` and must not be mounted or unmounted manually — doing so causes inconsistencies.

**How AutoFS works:**

```
System Boot --->  Read Maps --->  User Access (cd or ls) --->  Auto Mount
```

AutoFS runs a user-space daemon called `automount`. At boot, the daemon reads the master map and creates the mount point stubs but does not mount anything immediately. When a user accesses a path under a managed mount point (via `cd`, `ls`, or any file operation), the daemon mounts the requested share instantly. If the share sits idle past the timeout, `automount` unmounts it automatically.

## 2. NFS Server Setup

All steps in this section are performed on the **NFS server** machine. 

### 2.1 Install nfs-utils 
```bash 
$ sudo dnf install nfs-utils 
[sudo] password for blue: 
Last metadata expiration check: 0:21:46 ago on Sun 10 May 2026 10:40:10 PM +03.
Dependencies resolved.
=====================================================================================================
 Package                    Architecture       Version                      Repository          Size
=====================================================================================================
Installing:
 nfs-utils                  x86_64             1:2.8.2-3.el10               BaseOs             487 k
Installing dependencies:
 gssproxy                   x86_64             0.9.2-10.el10                BaseOs             118 k
 libev                      x86_64             4.33-14.el10                 BaseOs              56 k
 libnfsidmap                x86_64             1:2.8.2-3.el10               BaseOs              67 k
 libverto-libev             x86_64             0.3.2-10.el10                BaseOs              15 k
 rpcbind                    x86_64             1.2.7-3.el10                 BaseOs              63 k
 sssd-nfs-idmap             x86_64             2.10.2-3.el10                BaseOs              39 k

Transaction Summary
=====================================================================================================
Install  7 Packages

Total size: 845 k
Installed size: 2.0 M
Is this ok [y/N]: y
Downloading Packages:
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                             1/1 
  Installing       : libnfsidmap-1:2.8.2-3.el10.x86_64                                           1/7 
  Running scriptlet: rpcbind-1.2.7-3.el10.x86_64                                                 2/7 
  Installing       : rpcbind-1.2.7-3.el10.x86_64                                                 2/7 
  Running scriptlet: rpcbind-1.2.7-3.el10.x86_64                                                 2/7 
Created symlink '/etc/systemd/system/multi-user.target.wants/rpcbind.service' → '/usr/lib/systemd/system/rpcbind.service'.
Created symlink '/etc/systemd/system/sockets.target.wants/rpcbind.socket' → '/usr/lib/systemd/system/rpcbind.socket'.

  Installing       : libev-4.33-14.el10.x86_64                                                   3/7 
  Installing       : libverto-libev-0.3.2-10.el10.x86_64                                         4/7 
  Running scriptlet: gssproxy-0.9.2-10.el10.x86_64                                               5/7 
  Installing       : gssproxy-0.9.2-10.el10.x86_64                                               5/7 
  Running scriptlet: gssproxy-0.9.2-10.el10.x86_64                                               5/7 
  Running scriptlet: nfs-utils-1:2.8.2-3.el10.x86_64                                             6/7 
  Installing       : nfs-utils-1:2.8.2-3.el10.x86_64                                             6/7 
  Running scriptlet: nfs-utils-1:2.8.2-3.el10.x86_64                                             6/7 
Created symlink '/etc/systemd/system/multi-user.target.wants/nfs-client.target' → '/usr/lib/systemd/system/nfs-client.target'.
Created symlink '/etc/systemd/system/remote-fs.target.wants/nfs-client.target' → '/usr/lib/systemd/system/nfs-client.target'.

Warning: The unit file, source configuration file or drop-ins of gssproxy.service changed on disk. Run 'systemctl daemon-reload' to reload units.

Warning: The unit file, source configuration file or drop-ins of gssproxy.service changed on disk. Run 'systemctl daemon-reload' to reload units.

  Installing       : sssd-nfs-idmap-2.10.2-3.el10.x86_64                                         7/7 
  Running scriptlet: sssd-nfs-idmap-2.10.2-3.el10.x86_64                                         7/7 
Installed products updated.

Installed:
  gssproxy-0.9.2-10.el10.x86_64                     libev-4.33-14.el10.x86_64                        
  libnfsidmap-1:2.8.2-3.el10.x86_64                 libverto-libev-0.3.2-10.el10.x86_64              
  nfs-utils-1:2.8.2-3.el10.x86_64                   rpcbind-1.2.7-3.el10.x86_64                      
  sssd-nfs-idmap-2.10.2-3.el10.x86_64              

Complete!

``` 

Note that `rpcbind` is installed as a dependency and its socket and service units are automatically enabled via symlinks into `multi-user.target`. 

### 2.2 Create the shared directory
```bash
$ sudo mkdir /usman-Blue-Sky
```
### 2.3 Set permissions on the share
```bash 
$ sudo chmod  777 /usman-Blue-Sky/
``` 


### 2.4 Configure the firewall (initial) 
```bash 
$ sudo firewall-cmd --add-service nfs ; sudo firewall-cmd --reload 
success
success
``` 
This opens the NFS service port in the current runtime zone. The permanent rules are added later in [Section 3.2](#32-firewall-permanent-rules).

### 2.5 Enable and start nfs-server
```bash 
$ sudo systemctl enable nfs-server; sudo systemctl start nfs-server; 
Created symlink '/etc/systemd/system/multi-user.target.wants/nfs-server.service' → '/usr/lib/systemd/system/nfs-server.service'.
```

### 2.6 Verify nfs-server status

```bash 
$ sudo systemctl status nfs-server 
● nfs-server.service - NFS server and services
     Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; preset: disabled)
     Active: active (exited) since Sun 2026-05-10 23:19:46 +03; 3s ago
 Invocation: ef1f1089bfe142b8ac0dc5870e51de87
       Docs: man:rpc.nfsd(8)
             man:exportfs(8)
    Process: 17458 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
    Process: 17460 ExecStart=/bin/sh -c /usr/sbin/nfsdctl autostart || /usr/sbin/rpc.nfsd (code=exit>
    Process: 17480 ExecStart=/bin/sh -c if systemctl -q is-active gssproxy; then systemctl reload gs>
   Main PID: 17480 (code=exited, status=0/SUCCESS)
   Mem peak: 2.1M
        CPU: 24ms

May 10 23:19:46 Sky.Usman.O.Olanrewaju systemd[1]: Starting nfs-server.service - NFS server and serv>
May 10 23:19:46 Sky.Usman.O.Olanrewaju systemd[1]: Finished nfs-server.service - NFS server and serv>
lines 1-15/15 (END)

``` 
**Note:** `active (exited)` is the expected and correct status for `nfs-server.service`. This is not a failure — the service launches kernel threads during startup and then the launcher process exits. The NFS kernel threads remain running and serving requests in the background.



## 3. Exporting NFS Shares

### 3.1 Get the NFS client IP address

Before editing the exports file, determine the IPv4 address of the NFS client machine. The exports file must reference either the client's IP or its hostname.

### 3.2 Firewall permanent rules 
```bash
$ sudo firewall-cmd --permanent --add-service=nfs; sudo firewall-cmd --permanent --add-service=rpc-bind;sudo firewall-cmd --permanent --add-service=mountd ;sudo firewall-cmd --reload 
success
success
success
success
``` 
| Service | Purpose |
|---|---|
| `nfs` | Core NFS traffic (TCP/UDP 2049) |
| `rpc-bind` | RPC port mapper (TCP/UDP 111) — maps RPC services to ports |
| `mountd` | NFS mount daemon — handles initial client mount requests |

### 3.3 Edit /etc/exports
```bash 
$ sudo vi /etc/exports
```
Add the following line (using client IP):
 
```
/usman-Blue-Sky client-ip(rw,sync)
```
**Using hostname instead of IP:** If both machines are on the same subnet (first three octets of the IP address match — e.g., both are `192.168.888.x`), the client hostname can be used directly. Verify the server's hostname with `hostnamectl`:

### 3.3 Exporting 
```bash 
$ sudo exportfs -av 
exporting client-ip:/usman-Blue-Sky
```
note you can use the hostname of the nfs client but they have to be on same subnet i.e if your machine is at 192.183.133.43 then the client has to be 192.183.133.232. the first three dot ha
to be the same to use the hostname. 

***example*** 

```bash 
$ hostnamectl
     Static hostname: Sky.usman.O.Olanrewaju
           Icon name: computer-vm
             Chassis: vm 🖴
          Machine ID: eaf6762190f8406d95c38dc9cd0a3321
             Boot ID: a24effc85bc54a5cbba8dc47d17484b2
      Virtualization: kvm
    Operating System: Fedora Linux 44 (Workstation Edition)
         CPE OS Name: cpe:/o:fedoraproject:fedora:44
      OS Support End: Wed 2027-05-19
OS Support Remaining: 1y 1w 1d                             
              Kernel: Linux 7.0.3-200.fc44.x86_64
        Architecture: x86-64
     Hardware Vendor: QEMU
      Hardware Model: Standard PC _Q35 + ICH9, 2009_
    Hardware Version: pc-q35-10.1
    Firmware Version: 1.17.0-9.fc43
       Firmware Date: Tue 2025-06-10
        Firmware Age: 10month 4w 1d
```
In that case, the exports entry using the client hostname (`Sky`) would be:

```bash
$ sudo vi /etc/exports
``` 
Line for using hostname on the same subnet

```
/usman-Blue-Sky Sky(rw,sync)
```
### 3.4 Apply the exports

```bash
$ sudo exportfs -av
exporting client-ip:/usman-Blue-Sky
```
`exportfs -av` reads `/etc/exports` and makes all listed shares immediately available without restarting the NFS server. The `-a` flag processes all entries; `-v` prints verbose output showing what is being exported and to whom.

## 4. NFS Client Setup

All steps in this section are performed on the **NFS client** machine.

### 4.1 Install nfs-utils

```bash 
$ sudo dnf install nfs-utils
[sudo] password for blue:
Last metadata expiration check: 0:21:46 ago on Sun 10 May 2026 10:40:10 PM +03.
Dependencies resolved.
=====================================================================================================
 Package                    Architecture       Version                      Repository          Size
=====================================================================================================
Installing:
 nfs-utils                  x86_64             1:2.8.2-3.el10               BaseOs             487 k
Installing dependencies:
 gssproxy                   x86_64             0.9.2-10.el10                BaseOs             118 k
 libev                      x86_64             4.33-14.el10                 BaseOs              56 k
 libnfsidmap                x86_64             1:2.8.2-3.el10               BaseOs             

# truncated output
```  

### 4.2 Create the mount point
```bash 
$ sudo mkdir /usman-mounting-nfs 
[sudo] password for blue: 
```

### 4.3 Mount the NFS share
```bash 
$ sudo mount server-ip:/usman-Blue-Sky    /usman-mounting-nfs

```
**Troubleshooting — "No route to host":**

```bash
$ sudo mount server-ip:/usman-Blue-Sky /usman-mounting-nfs
mount.nfs: No route to host for 192.168.122.206:/usman-Blue-Sky on /usman-mounting-nfs
```
This error means the NFS server firewall is blocking the connection. Return to the server and ensure all three permanent firewall rules (nfs, rpc-bind, mountd) are applied and reloaded. After fixing the firewall, retry the mount.


#### 4.3.1 Second Try
```bash 
$ sudo mount server-ip:/usman-Blue-Sky    /usman-mounting-nfs
``` 
### 4.4 Verify with df
```bash 
$ df -h 
Filesystem                       Size  Used Avail Use% Mounted on
/dev/vda3                         18G  6.9G   11G  40% /
devtmpfs                         1.9G     0  1.9G   0% /dev
tmpfs                            2.0G   92K  2.0G   1% /dev/shm
tmpfs                            780M  1.7M  778M   1% /run
none                             1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
none                             1.0M     0  1.0M   0% /run/credentials/systemd-resolved.service
/dev/vda3                         18G  6.9G   11G  40% /home
tmpfs                            2.0G   36K  2.0G   1% /tmp
/dev/vda2                        2.0G  536M  1.3G  30% /boot
tmpfs                            390M  136K  390M   1% /run/user/1000
server-ip:/usman-Blue-Sky   17G  4.1G   13G  25% /usman-mounting-nfs # here we are 
tmpfs                            390M   36K  390M   1% /run/user/0
```
### 4.5 Verify with mount
```bash 
$ mount | grep usman-
192.168.122.206:/usman-Blue-Sky on /usman-mounting-nfs type nfs4 (rw,relatime,vers=4.2,rsize=524288,wsize=524288,namlen=255,hard,fatal_neterrors=none,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.122.241,local_lock=none,addr=192.168.122.206)
```   
The output confirms `nfs4` protocol, `vers=4.2`, and `proto=tcp` — consistent with NFSv4 defaults on RHEL 10. Any files or directories created under `/usman-mounting-nfs` on the client are immediately visible on the server under `/usman-Blue-Sky`, and vice versa.
 
## 5. NFS Persistence via fstab
 
Persistence configuration is done on the **NFS client**.

### 5.1 Edit /etc/fstab

the persistence is done on the NFS client 
```bash 
$ sudo vi /etc/fstab 
```
Add the following line:

```
server-ip:/usman-Blue-Sky /usman-mounting-nfs nfs -netdev 0 0
```

**Critical option — `_netdev`:** This instructs the system to wait for networking to be fully available before attempting to mount this share during boot. Without `_netdev`, the system may attempt to mount the NFS share before the network interface is up, causing a boot failure or hang.

**Mount point requirement:** The mount point (`/usman-mounting-nfs`) must exist and be empty before mounting. If it is in use or non-empty, the mount will fail.

### 5.2 Test the fstab entry
```bash 
$ sudo umount /usman-mounting-nfs ; sudo mount -a 
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
``` 
The hint is informational — the mount still succeeded. Run `systemctl daemon-reload` to sync systemd with the updated fstab.

### 5.3 Verify persistence
```bash
$ df -h 
Filesystem                       Size  Used Avail Use% Mounted on
/dev/vda3                         18G  6.9G   11G  40% /
devtmpfs                         1.9G     0  1.9G   0% /dev
tmpfs                            2.0G   92K  2.0G   1% /dev/shm
tmpfs                            780M  1.7M  778M   1% /run
none                             1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
none                             1.0M     0  1.0M   0% /run/credentials/systemd-resolved.service
/dev/vda3                         18G  6.9G   11G  40% /home
tmpfs                            2.0G   36K  2.0G   1% /tmp
/dev/vda2                        2.0G  536M  1.3G  30% /boot
tmpfs                            390M  140K  390M   1% /run/user/1000
tmpfs                            390M   36K  390M   1% /run/user/0
server-ip:/usman-Blue-Sky   17G  4.1G   13G  25% /usman-mounting-nfs
``` 
The share is back and will remount automatically on every reboot.

## 6. SSHFS — Client-Side Setup

SSHFS mounts a remote directory over SSH. All configuration is done on the **client** machine only. The server must have `sshd` running — no additional server-side packages are needed.

### 6.1 Install fuse-sshfs
 ```bash 
$ sudo dnf install fuse-sshfs 
Updating and loading repositories:
Repositories loaded.
Package                                Arch       Version                                Repository                Size
Installing:
 fuse-sshfs                            x86_64     0:3.7.5-3.fc44                         fedora               125.2 KiB
Installing weak dependencies:
 openssh-askpass                       x86_64     0:10.2p1-10.fc44                       updates               19.5 KiB

Transaction Summary:
 Installing:         2 packages

Total size of inbound packages is 85 KiB. Need to download 85 KiB.
After this operation, 145 KiB extra will be used (install 145 KiB, remove 0 B).
Is this ok [y/N]: y
[1/2] openssh-askpass-0:10.2p1-10.fc44.x86_64                                  100% |  59.3 KiB/s |  20.6 KiB |  00m00s
[2/2] fuse-sshfs-0:3.7.5-3.fc44.x86_64                                         100% | 147.5 KiB/s |  64.3 KiB |  00m00s
-----------------------------------------------------------------------------------------------------------------------
[2/2] Total                                                                    100% |  47.3 KiB/s |  84.9 KiB |  00m02s
Running transaction
[1/4] Verify package files                                                     100% | 200.0   B/s |   2.0   B |  00m00s
[2/4] Prepare transaction                                                      100% |  10.0   B/s |   2.0   B |  00m00s
[3/4] Installing openssh-askpass-0:10.2p1-10.fc44.x86_64                       100% | 517.2 KiB/s |  20.7 KiB |  00m00s
[4/4] Installing fuse-sshfs-0:3.7.5-3.fc44.x86_64                              100% | 347.5 KiB/s | 127.2 KiB |  00m00s
Complete!
``` 
`openssh-askpass` is installed as a weak dependency to handle SSH password prompts in graphical environments.

### 6.2 Create the mount point
```bash 
$ mkdir usman-sshfs
```
No `sudo` needed — SSHFS mounts in user space via FUSE.

### 6.3 Mount the remote directory via SSHFS
```bash 
sshfs blue@ip:/usman-Blue-Sky /home/blue/usman-sshfs/
blue@ip's password:
``` 
The syntax is `sshfs user@host:/remote/path /local/mount/point`. Both machines in this lab use the same username (`blue`), which is why the command references `blue` on both sides. SSHFS authenticates using SSH — it will prompt for a password or use SSH keys if configured.

### 6.4 Verify the SSHFS mount

```bash
$ df -h 
Filesystem                            Size  Used Avail Use% Mounted on
/dev/vda3                              18G  6.9G   11G  40% /
devtmpfs                              1.9G     0  1.9G   0% /dev
tmpfs                                 2.0G   92K  2.0G   1% /dev/shm
tmpfs                                 780M  1.7M  778M   1% /run
none                                  1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
none                                  1.0M     0  1.0M   0% /run/credentials/systemd-resolved.service
/dev/vda3                              18G  6.9G   11G  40% /home
tmpfs                                 2.0G   52K  2.0G   1% /tmp
/dev/vda2                             2.0G  536M  1.3G  30% /boot
tmpfs                                 390M  140K  390M   1% /run/user/1000
tmpfs                                 390M   36K  390M   1% /run/user/0
server-ip:/usman-Blue-Sky        17G  4.1G   13G  25% /usman-mounting-nfs
blue@remote-ip:/usman-Blue-Sky   17G  4.1G   13G  25% /home/blue/usman-sshfs
``` 

Both NFS and SSHFS are mounted simultaneously and pointing to the same remote directory. Confirm they show identical contents:

```bash 
$ ls /usman-mounting-nfs/
poop
``` 
 
```bash
$ ls /home/blue/usman-sshfs/
poop
``` 
Both mount points reflect the same backing directory on the server — changes made through either mount point are immediately visible through the other.


## 7. SSHFS Persistence via fstab

### 7.1 Edit /etc/fstab

```bash
$ sudo vi /etc/fstab
```
Add the following line:

``` 
blue@remote-ip:/usman-Blue-Sky   /home/blue/usman-sshfs fuse.sshfs defaults 0 0
```
### 7.2 Test the fstab entry
```bash 
$ sudo umount /home/blue/usman-sshfs ; sudo mount -a 
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
The authenticity of host 'remote-ip (remote-ip)' can't be established.
ED25519 key fingerprint is: SHA256:
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? 
``` 

SSHFS prompts for SSH host key verification on first connection and will prompt for a password unless SSH keys are in place. For production or exam use, pre-accept the host key and configure `~/.ssh/authorized_keys` on the server to enable passwordless mounting.

**Note:** SSHFS is not an official RHCSA exam objective, but it demonstrates an important real-world pattern — mounting remote filesystems over encrypted SSH without a dedicated NFS server. It is included here as supplementary knowledge.


## 8. AutoFS — Concepts and Configuration

### 8.1 AutoFS configuration file

The primary configuration file is `/etc/autofs.conf`. AutoFS reads it at service startup. Key directives and their defaults:

```
master_map_name = auto.master
timeout         = 300
negative_timeout = 60
mount_nfs_default_protocol = 4
logging         = none
```

| Directive | Description |
|---|---|
| `master_map_name` | Defines the name of the master map. Default is `auto.master` in `/etc`. |
| `timeout` | Seconds of idle time before an unused mount is automatically unmounted. Default is 300 (5 minutes). |
| `negative_timeout` | How long to cache a failed mount attempt before retrying. Default 60 seconds. |
| `mount_nfs_default_protocol` | NFS version to use by default. Default is 4 (NFSv4). |
| `logging` | Controls AutoFS logging verbosity. `none` disables logging. |

```
System Boot ---> Read Map ---> User Access ---> Auto Mounts happens 
```

### 8.2 AutoFS map types

AutoFS stores mount information in text files called *maps*. There are three map types:

| Map Type | Description |
|---|---|
| **Master map** | The top-level map. Default file is `/etc/auto.master`. Lists all other maps and their mount points. It is recommended to store user-defined map files under `/etc/auto.master.d/` — AutoFS parses this directory automatically at startup. |
| **Direct map** | Mounts shares on any number of unrelated, absolute mount points. Referenced via `/-` in the master map. Direct-mounted shares are **always visible** to users. Local files and direct-mounted shares can coexist under the same parent directory. Each entry creates a separate record in `/proc/self/mounts`. |
| **Indirect map** | Mounts all shares under a single common parent directory. Referenced via the parent path in the master map. Indirect-mounted shares become visible **only after the corresponding subdirectory is accessed** — a trigger is required. Local files and indirect-mounted shares **cannot coexist** under the same parent directory. |

### 8.3 Install autofs

```bash
$ sudo dnf install autofs 
[sudo] password for blue: 
Last metadata expiration check: 0:42:47 ago on Mon 11 May 2026 12:35:15 AM +03.
Dependencies resolved.
=====================================================================================================
 Package                   Architecture       Version                       Repository          Size
=====================================================================================================
Installing:
 autofs                    x86_64             1:5.1.9-11.el10               BaseOs             395 k
Installing dependencies:
 libsss_autofs             x86_64             2.10.2-3.el10                 BaseOs              37 k

Transaction Summary
=====================================================================================================
Install  2 Packages

Total size: 432 k
Installed size: 1.1 M
Is this ok [y/N]: y
Downloading Packages:
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                             1/1 
  Installing       : libsss_autofs-2.10.2-3.el10.x86_64                                          1/2 
  Installing       : autofs-1:5.1.9-11.el10.x86_64                                               2/2 
  Running scriptlet: autofs-1:5.1.9-11.el10.x86_64                                               2/2 
Installed products updated.

Installed:
  autofs-1:5.1.9-11.el10.x86_64                  libsss_autofs-2.10.2-3.el10.x86_64                 

Complete!
```
The `autofs` package must be installed on the **client** machine. The NFS server does not need it.

## 9. AutoFS Direct Method — Server

### 9.1 Create the shared directory on the server
```bash
$ mkdir usman-autonfs1-server
```
### 9.2 Edit /etc/exports
```bash
$ sudo vi  /etc/exports
```
Add: 
```
/home/blue/usman-autonfs1-server client-ip(rw,sync) 
```
### 9.3 Export the share
```bash 
$ sudo exportfs  -av
exporting nfsclient-ip:/home/blue/usman-autonfs1-server
``` 
## 10. AutoFS Direct Method — Client

### 10.1 Install autofs
```bash 
$ sudo dnf install autofs 
[sudo] password for blue:
Last metadata expiration check: 0:42:47 ago on Mon 11 May 2026 12:35:15 AM +03.
Dependencies resolved.
=====================================================================================================
 Package                   Architecture       Version                       Repository          Size
=====================================================================================================
Installing:
 autofs                    x86_64             1:5.1.9-11.el10               BaseOs             395 k
Installing dependencies:
 libsss_autofs             x86_64             2.10.2-3.el10                 BaseOs              37 k

Transaction Summary
===============================================================================================# truncated output 
```  
### 10.2 Create the mount point
```bash
$ sudo mkdir /usman-autofs1-client 
[sudo] password for blue: 
``` 
### 10.3 Edit the master map 
```bash
$ sudo vi /etc/auto.master
```
Add: 
 ```
/- /etc/auto.master.d/auto.usmandirect
```
The `/-` key tells AutoFS this is a **direct** map — mount points are absolute paths defined in the referenced map file, not relative subdirectories under a parent.

### 10.4 Create the direct map file

```bash 
$ sudo vi /etc/auto.master.d/auto.usmandirect
```
Add: 
``` 
/usman-autofs1-client -fstype=rw   autofs-server-ip:/home/blue/usman-autofs1-server
```
Format: `mount-point  [options]  server:exported-path`

### 10.5 Enable, start, and verify autofs
```bash
$ sudo systemctl enable autofs ; sudo systemctl start autofs; sudo systemctl status autofs ;
Created symlink '/etc/systemd/system/multi-user.target.wants/autofs.service' → '/usr/lib/systemd/system/autofs.service'.
● autofs.service - Automounts filesystems on demand
     Loaded: loaded (/usr/lib/systemd/system/autofs.service; enabled; preset: disabled)
    Drop-In: /usr/lib/systemd/system/service.d
             └─10-timeout-abort.conf
     Active: active (running) since Sat 2026-05-09 21:44:18 PDT; 45ms ago
 Invocation: aff2317f2af74595a1221f77c0b55a3b
   Main PID: 31233 (automount)
      Tasks: 6 (limit: 4611)
     Memory: 3.8M (peak: 4.8M)
        CPU: 16ms
     CGroup: /system.slice/autofs.service
             └─31233 /usr/bin/automount --systemd-service --dont-check-daemon

May 09 21:44:18 Sky.usman.O.Olanrewaju systemd[1]: Starting autofs.service - Automounts filesystems on demand...
May 09 21:44:18 Sky.usman.O.Olanrewaju (automount)[31233]: autofs.service: Referenced but unset environment variable e>
May 09 21:44:18 Sky.usman.O.Olanrewaju systemd[1]: Started autofs.service - Automounts filesystems on demand.
```

The daemon binary is `/usr/bin/automount`. Unlike `nfs-server.service`, `autofs.service` shows `active (running)` because the `automount` daemon remains resident, listening for filesystem access events.

### 10.6 Trigger the direct mount

With direct maps, simply accessing the mount point triggers the mount:

```bash
$ ls /usman-autofs1-client/
```
### 10.7 Verify the mount
```bash 
$ df -h
Filesystem                             Size  Used Avail Use% Mounted on
/dev/vda3                               18G  6.9G   11G  40% /
devtmpfs                               1.9G     0  1.9G   0% /dev
tmpfs                                  2.0G   92K  2.0G   1% /dev/shm
tmpfs                                  780M  1.6M  778M   1% /run
none                                   1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
none                                   1.0M     0  1.0M   0% /run/credentials/systemd-resolved.service
/dev/vda3                               18G  6.9G   11G  40% /home
tmpfs                                  2.0G  8.0K  2.0G   1% /tmp
/dev/vda2                              2.0G  536M  1.3G  30% /boot
first-server-ip:/usman-Blue-Sky         17G  4.1G   13G  25% /usman-mounting-nfs
tmpfs                                  390M  116K  390M   1% /run/user/1000
autofs-server-ip:/usman-autofs1-server   17G  4.1G   13G  25% /usman-autofs1-client
```
### 10.8 Verify file synchronization

On the server:
 
```bash
$ sudo vi usman-autofs1-server/usman.touch 
```
On the client: 
```bash
$ ls /usman-autofs1-client/
usman.touch
```
Files created on either side are immediately visible on the other — NFS operates synchronously with the `sync` export option.

### 10.9 Adding a second direct map entry


Multiple mount points can be added to the same direct map file.

#### 10.9.1 Create the second mount point on the client
```bash 
$  mkdir usman-autofs2-client 
```

#### 10.9.2 Add the second entry to the direct map file

```bash
$ sudo vi /etc/auto.master.d/auto.usmandirect
``` 
Add:
```
/home/blue/usman-autofs2-client   autofs-serverip:/home/blue/usman-autofs1-server
```


Both entries in `auto.usmandirect` now point to the same server export — demonstrating that a single NFS export can be mounted at multiple client paths simultaneously.

#### 10.9.3 Reload the automount daemon
```bash 
$ sudo systemctl reload autofs 
```
`reload` signals the running daemon to re-read its configuration without restarting it. Use `restart` when making structural changes; `reload` is sufficient for map file updates.

#### 10.9.4 Verify the second mount
```bash 
$ ls usman-autofs2-client/
usman.touch
``` 


## 11. AutoFS Indirect Method — Server

### 11.1 Create the shared directory
```bash 
$ sudo mkdir  /usman-autofs-server-indirect
```
### 11.2 Set permissions
```bash
$ sudo chmod 777 /usman-autofs-server-indirect/
```

### 11.3 Edit /etc/exports
```bash 
$ sudo vi /etc/exports
```
Adding:
```
/usman-autofs-server-indirect  client-ip(rw)
``` 

### 11.4 Export the share
```bash 
$ sudo exportfs -av
exporting client-ip:/home/blue/usman-autofs1-server
exporting client-ip:/usman-autofs-server-indirect
exporting client-ip:/usman-Blue-Sky
``` 
All previously defined exports are re-exported on each `exportfs -av` run. The firewall rules for NFS (nfs, rpc-bind, mountd) carry over from the earlier configuration — no additional firewall changes needed.
 

## 12. AutoFS Indirect Method — Client (with pre-created mount point)
  
In the indirect method, the master map references a **parent directory**. AutoFS creates subdirectories under that parent on demand. Local files and indirect mounts cannot share the same parent directory.

**Key behavioral difference from direct:** With a direct map, accessing the mount point itself triggers the mount. With an indirect map, you must access a **subdirectory** under the parent — the parent directory alone does not trigger any mount.

### 12.1 Create the parent mount point

```bash
$ sudo mkdir /usman-autofs-client-indirect 
```

### 12.2 Edit the master map
```bash 
$ sudo vi /etc/auto.master
``` 
Add: 
```
/usman-autofs-client-indirect  /etc/auto.usman.indirect
```

### 12.3 Create the indirect map file
```bash 
$ sudo vi /etc/auto.usman.indirect  
```
Add:
``` 
usman-sitting-on-black-chair  autofs-server-ip:/usman-autofs-server-indirect
``` 
Format: `subdirectory-name  [options]  server:exported-path`

The key (`usman-sitting-on-black-chair`) becomes the subdirectory name under the parent (`/usman-autofs-client-indirect`). The full resulting mount path is `/usman-autofs-client-indirect/usman-sitting-on-black-chair`.

### 12.4 Reload the daemon
```bash 
$ sudo systemctl reload autofs 
```

### 12.5 Trigger the indirect mount

```bash 
$ ls /usman-autofs-client-indirect/usman-sitting-on-black-chair 
testing # i touched this file after creating on server 
``` 
The subdirectory name `usman-sitting-on-black-chair` must be explicitly used in the `ls` or `cd` command. Listing the parent directory alone (`ls /usman-autofs-client-indirect/`) does not trigger the mount.


### 12.6 Verify with df
```bash 

$ df -h | grep indirect 
server-ip:/usman-autofs-server-indirect   17G  4.1G   13G  25% /usman-autofs-client-indirect/usman-sitting-on-black-chair
```
### 12.7 Verify with mount 
```bash 
$ mount | grep indirect 
/etc/auto.usman.indirect on /usman-autofs-client-indirect type autofs (rw,relatime,fd=6,pgrp=11499,timeout=300,minproto=5,maxproto=5,indirect,pipe_ino=75460)
/etc/auto.misc on /misc type autofs (rw,relatime,fd=11,pgrp=11499,timeout=300,minproto=5,maxproto=5,indirect,pipe_ino=85103)
-hosts on /net type autofs (rw,relatime,fd=14,pgrp=11499,timeout=300,minproto=5,maxproto=5,indirect,pipe_ino=87130)
:/usman-autofs-server-indirect on /usman-autofs-client-indirect/usman-sitting-on-black-chair type nfs4 (rw,relatime,vers=4.2,rsize=524288,wsize=524288,namlen=255,hard,fatal_neterrors=none,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=52321...,local_lock=none,addr=52321...)
```
The `type autofs` entries are the AutoFS control entries — they represent the daemon's watch points, not actual NFS mounts. The `type nfs4` entry is the live NFS mount that appeared when the subdirectory was accessed.

  

## 13. AutoFS Indirect — Wildcard Mounting

Wildcard mounting allows any subdirectory name under the parent to trigger a mount to the same NFS export. This is particularly useful when the exact subdirectory name is not known in advance — for example, mounting user home directories by username.

### 13.1 Edit the indirect map file  
```bash 
$ sudo vi /etc/auto.usman.indirect
``` 

Replace the named entry with a wildcard:

```
*  server-ip:/usman-autofs-server-indirect
 ```

### 13.2 Reload the daemon
```bash 
$ sudo systemctl restart  autofs 
```

### 13.3 Trigger using any subdirectory name
```bash 
$ ls /usman-autofs-client-indirect/data
testing
```
```bash
$ ls /usman-autofs-client-indirect/just-me
testing
```
 
Any name used as the subdirectory maps to the same NFS export. The mount point path takes the name you provide — `data`, `just-me`, or anything else. All of them mount the same remote directory.

### 13.4 Verify

```bash 
$ df -h | grep indirect 
192.168.122.206:/usman-autofs-server-indirect     17G  4.1G   13G  25% /usman-autofs-client-indirect/data
```
The mount point in the output matches the subdirectory name used to trigger it.


## 14. AutoFS Indirect — Client (without pre-creating mount point)

AutoFS can manage indirect mounts without manually creating the parent directory first — the daemon creates it automatically from the master map entry.

### 14.1 Edit the master map

```bash
$ sudo vi /etc/auto.master
```
Add: 
 
```
/usman-autofs-indirect-withoutdir  /etc/auto.usman.indirect-withoutdir

``` 


The parent directory `/usman-autofs-indirect-withoutdir` does not need to exist before this step.

### 14.2 Create the map file

```bash
$ sudo vi /etc/auto.usman.indirect-withoutdir
```
Add: 
```
tea  server-ip:/usman-autofs-server-indirect
```

### 14.3 Reload the daemon
```bash
$ sudo systemctl restart autofs
```

### 14.4 Trigger the mount
```bash
$ ls /usman-autofs-indirect-withoutdir/tea 
testing
``` 
### 14.5 Verify with df
```bash
$ df -h | grep usman-autofs-indirect-withoutdir/
server-ip:/usman-autofs-server-indirect     17G  4.1G   13G  25% /usman-autofs-indirect-withoutdir/tea
``` 

### 14.6 Confirm the parent directory exists
```bash 
$ ls / |  grep without
usman-autofs-indirect-withoutdir
```
AutoFS created the parent directory automatically. It will disappear approximately 5 minutes after the last access, consistent with the `timeout = 300` setting in `/etc/autofs.conf`.

### 14.7 Wildcard variant

```bash 
$ sudo vi /etc/auto.usman.indirect-withoutdir 
[sudo] password for blue: 
``` 
Change `tea` to `*`:

```
*  autofs-server-ip:/usman-autofs-server-indirect
```


### 14.8 reload daemon
```bash 
$ sudo systemctl restart autofs
```

### 14.9 Trigger with any name  
```bash
$ ls /usman-autofs-indirect-withoutdir/usman-jump
testing
``` 
### 14.10 verify with df 

```bash 
$ df |  grep without
192.168.122.206:/usman-autofs-server-indirect    17756160 4295168  13460992  25% /usman-autofs-indirect-withoutdir/usman-jump
```

## 15. AutoFS via /misc (built-in indirect map)

AutoFS ships with a default indirect map for `/misc` pre-configured in `/etc/auto.master`. No master map editing is needed.

### 15.1 Confirm /misc is in the master map 
```bash
$ cat /etc/auto.master | grep misc
/misc	/etc/auto.misc
  
```
### 15.2 Edit the /misc map file 
```bash 
$ sudo vi /etc/auto.misc
[sudo] password for blue: 
```
Add a wildcard entry:

 
```
* autofs-server-ip:/usman-autofs-server-indirect
```

### 15.3 Reload the daemon
```bash
$ sudo systemctl restart autofs 
``` 

### 15.4 Trigger and verify
```bash
$ ls /misc/usman-hakan
testing
``` 
The `/misc` map works identically to a user-defined indirect map — any name under `/misc` triggers a mount to the specified NFS export.


## 16. AutoFS Home Directory Automounting — Server

Automounting user home directories is a canonical AutoFS use case. Users log into any client machine and their home directory — stored centrally on the NFS server — is mounted automatically under a consistent path.

### 16.1 Create the user on the server

The user's UID must match on both server and client. A consistent UID is the mechanism that maps file ownership correctly across the NFS boundary.

```bash 
$ sudo useradd -u 2010 usman-newuser 
[sudo] password for blue:
```
### 16.2 Set the user password 
```bash
$ sudo passwd usman-newuser 
New password: 
BAD PASSWORD: The password is shorter than 8 characters
Retype new password: 
passwd: password updated successfully
```

### 16.3 Export /home
```bash 
$ sudo vi /etc/exports
```

Add: 
```
/home            client-ip(rw,sync)
``` 
### 16.4 Apply the export
```bash
$ sudo exportfs -av
exporting 192.168.122.241:/home/blue/usman-autofs1-server
exporting 192.168.122.241:/usman-autofs-server-indirect
exporting 192.168.122.241:/home                              #  this is the home we want share
exporting 192.168.122.241:/usman-Blue-Sky
```

## 17. AutoFS Home Directory Automounting — Client

### 17.1 Create the matching user on the client

The user must have the same UID (`2010`) as on the server. The `-b` flag sets the base directory (the umbrella path where the home will be mounted). The `-M` flag skips local home directory creation — the home will come from the NFS server via AutoFS. 
```bash
 $ sudo useradd -u 2010 -b /nfshome-usman  -M usman-newuser 
[sudo] password for blue:
```

### 17.2 Set the password
```bash
$ echo user123 | sudo passwd --stdin usman-newuser 
BAD PASSWORD: The password is shorter than 8 characters
``` 

### 17.3 Create the umbrella mount point
 
```bash
$ sudo mkdir /nfshome-usman
``` 
This is the parent directory under which AutoFS will mount each user's home as a subdirectory.

### 17.4 Edit the master map
```bash
$ sudo vi /etc/auto.master
```
Add: 
```
/nfshome-usman /etc/auto.home
```

### 17.5 Create the home map file
```bash 
$ sudo vi /etc/auto.home 
```

Add:
```
* -rw  server-ip:/home/usman-newuser
``` 
The wildcard (`*`) matches any username. When user `usman-newuser` logs in, AutoFS mounts `server:/home/usman-newuser` at `/nfshome-usman/usman-newuser` automatically.

### 17.6 Reload the daemon
```bash
$ sudo systemctl restart autofs
```
### 17.7 Test by switching to the user

```bash 
$ su - usman-newuser 
Password: 
```
After Switching: 

```bash
$ pwd 
/nfshome-usman/usman-newuser
```
The shell lands in the correct home directory, which is now an NFS-backed AutoFS mount.

### 17.8 Verify with df
 ```bash 
 $ df -h  | grep newuser
server-ip:/home/usman-newuser   17G  4.1G   13G  25% /nfshome-usman/usman-newuser
```



### 17.9 Verify file synchronization

Create a file from the client:
```bash
$ touch usman-now
``` 

```bash
$ ls
usman-now
``` 
Verify on the server side — the file `usman-now` appears under `/home/usman-newuser/` on the server immediately.

---

## Key Takeaways

**NFS:**
- `nfs-utils` is installed on both server and client.
- `nfs-server.service` showing `active (exited)` is normal — kernel threads remain active after the launcher exits.
- `/etc/exports` controls what is shared and to whom. Apply changes with `exportfs -av` without restarting the service.
- Three firewall services are required: `nfs`, `rpc-bind`, `mountd`.
- Use `_netdev` in fstab for NFS entries so the mount waits for the network at boot.

**SSHFS:**
- Requires only `fuse-sshfs` on the client. No server-side packages needed beyond `sshd`.
- Filesystem type in fstab is `fuse.sshfs`.
- Passwordless mounting at boot requires SSH key authentication.

**AutoFS:**
- Install `autofs` on the client only.
- The daemon binary is `automount`. The config is at `/etc/autofs.conf`.
- Three map types: master, direct (`/-`), indirect (`/parent-path`).
- Direct maps: mount point is always visible; accessing it triggers the mount.
- Indirect maps: subdirectory must be explicitly accessed to trigger the mount.
- Wildcard (`*`) in a map file allows any subdirectory name to map to the same export.
- AutoFS mounts must not appear in `/etc/fstab` and must not be manually mounted/unmounted.
- Default idle timeout before auto-unmount is 300 seconds (configurable in `/etc/autofs.conf`).
- Home directory automounting requires matching UIDs on server and client. Use `useradd -M` on the client to skip local home creation.


