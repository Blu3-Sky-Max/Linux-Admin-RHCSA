# Performance Tuning Management

**Author:** Usman O. Olanlanrewaju (Blu3 Sky)

**Date:** 2026/03/31

**Focus:** System tuning with `tuned`, profile management via `tuned-adm`, and applying recommended profiles for workload optimization

---


## 1. tuned — Dynamic System Tuning Daemon

`tuned` is a system tuning service that monitors storage, networking, processors, audio, video, and other connected devices, then adjusts their kernel parameters for better performance or power saving based on a chosen profile.

The management CLI is `tuned-adm`.

| Component | Location |
|-----------|---------|
| Vendor profiles | `/usr/lib/tuned/profiles/` |
| Custom profiles | `/etc/tuned/` |
| Service unit | `tuned.service` |
| Man pages | `man:tuned(8)`, `man:tuned.conf(5)`, `man:tuned-adm(8)` |

---
## 2. Installation & Service Management

### 2.1 Install tuned
```bash
$ sudo dnf install -y tuned
Updating Subscription Management repositories.
Last metadata expiration check: 1 day, 2:01:55 ago on Mon 30 Mar 2026 02:58:45 PM EDT.
Package tuned-2.25.1-1.el10.noarch is already installed.
Dependencies resolved.
Nothing to do.
Complete!
```
### 2.2 Enable and start the service

```bash
$ sudo systemctl enable --now tuned
```
### 2.3 Verify service status
```bash
$ systemctl status tuned
● tuned.service - Dynamic System Tuning Daemon
     Loaded: loaded (/usr/lib/systemd/system/tuned.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-03-30 09:45:06 EDT; 1 day 7h ago
 Invocation: 7de21ef939624e90a35ecfde93c21d74
       Docs: man:tuned(8)
             man:tuned.conf(5)
             man:tuned-adm(8)
   Main PID: 1046 (tuned)
      Tasks: 4 (limit: 23129)
     Memory: 18.6M (peak: 23M)
        CPU: 6.483s
     CGroup: /system.slice/tuned.service
             └─1046 /usr/bin/python3 -Es /usr/sbin/tuned -l -P

```

---
## 3. Profile Management

### 3.1 List all available profiles
```bash
$ sudo tuned-adm list
Available profiles:
- accelerator-performance     - Throughput performance based tuning with disabled higher latency STOP states
- aws                         - Optimize for aws ec2 instances
- balanced                    - General non-specialized tuned profile
- balanced-battery            - Balanced profile biased towards power savings changes for battery
- desktop                     - Optimize for the desktop use-case
- hpc-compute                 - Optimize for HPC compute workloads
- intel-sst                   - Configure for Intel Speed Select Base Frequency
- latency-performance         - Optimize for deterministic performance at the cost of increased power consumption
- network-latency             - Optimize for deterministic performance at the cost of increased power consumption, focused on low latency network performance
- network-throughput          - Optimize for streaming network throughput, generally only necessary on older CPUs or 40G+ networks
- optimize-serial-console     - Optimize for serial console use.
- powersave                   - Optimize for low power consumption
- throughput-performance      - Broadly applicable tuning that provides excellent performance across a variety of common server workloads
- virtual-guest               - Optimize for running inside a virtual guest
- virtual-host                - Optimize for running KVM guests
Current active profile: virtual-guest
```
### 3.2 Profile files are stored at `/usr/lib/tuned/profiles/`: 
```bash 
```bash
$ ll /usr/lib/tuned/profiles/
total 0
drwxr-xr-x. 2 root root 24 Jan 27 12:06 accelerator-performance
drwxr-xr-x. 2 root root 24 Jan 27 12:06 aws
drwxr-xr-x. 2 root root 24 Jan 27 12:06 balanced
drwxr-xr-x. 2 root root 24 Jan 27 12:06 balanced-battery
drwxr-xr-x. 2 root root 24 Jan 27 12:06 desktop
drwxr-xr-x. 2 root root 24 Jan 27 12:06 hpc-compute
drwxr-xr-x. 2 root root 24 Jan 27 12:06 intel-sst
drwxr-xr-x. 2 root root 24 Jan 27 12:06 latency-performance
drwxr-xr-x. 2 root root 24 Jan 27 12:06 network-latency
drwxr-xr-x. 2 root root 24 Jan 27 12:06 network-throughput
drwxr-xr-x. 2 root root 24 Jan 27 12:06 optimize-serial-console
drwxr-xr-x. 2 root root 41 Jan 27 12:06 powersave
drwxr-xr-x. 2 root root 24 Jan 27 12:06 throughput-performance
drwxr-xr-x. 2 root root 24 Jan 27 12:06 virtual-guest
drwxr-xr-x. 2 root root 24 Jan 27 12:06 virtual-host
```
### 3.3 Check the currently active profile
```bash
$ tuned-adm active
Current active profile: virtual-guest
```
### 3.4 Get a profile recommendation for the system

`tuned-adm recommend` detects the environment (bare metal, VM, container) and suggests the most appropriate profile

```bash
$ tuned-adm recommend
virtual-guest
```
### 3.5 Switch to a different profile
```bash
$ tuned-adm profile balanced-battery
```
Profile changes take effect immediately — no reboot required.

### 3.6 Disable tuning entirely, then restore recommended profile
```bash
$ sudo tuned-adm off ;tuned-adm active;  sudo tuned-adm profile virtual-guest ; sudo tuned-adm active
No current active profile.
Current active profile: virtual-guest
```
`tuned-adm off` deactivates all tuning. Kernel parameters revert to their defaults. Always restore a profile before leaving a system unattended.

