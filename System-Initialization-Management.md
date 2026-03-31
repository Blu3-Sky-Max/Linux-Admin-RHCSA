# System Initialization & Management

**Author:** Usman O. Olanlanrewaju (Blu3 Sky)
**Date:** 2026/03/31
**Focus:** systemd architecture, unit management, service lifecycle, boot targets, and configuration file hierarchy
## 1. systemd Architecture
systemd is the init system used in RHEL 7+. The primary management tool is `systemctl`.

### 1.1 Inter-Process Communication (IPC) Methods

**Sockets** — A communication endpoint that allows two processes on the same or different systems to exchange data. systemd can socket-activate services: the socket is created first, and the service only starts when a connection arrives.

**D-Bus** — A message bus that allows multiple services running on the same system to communicate with one another. Services that use D-Bus register a bus name (e.g. `org.bluez`) and others can call methods on them by name.

### 1.2 Unit Files
Unit files replace the SysV init scripts (`/etc/rc.d/init.d/`) used in RHEL 6 and earlier. They define how systemd manages a resource.

unit service and target is located at `/usr/lib/systemd/system/`

Every unit file has three possible sections:

| Section | Purpose |
|---------|---------|
| `[Unit]` | Description, documentation, ordering, and conditions |
| `[Service]` / `[Socket]` / `[Target]` | Type-specific configuration |
| `[Install]` | Defines `WantedBy` and `Alias` for `enable`/`disable` |


**Example — `bluetooth.service`:**
#####  for  bluetooth.service: 
```bash 
$ cat /usr/lib/systemd/system/bluetooth.service 
[Unit]
Description=Bluetooth service
Documentation=man:bluetoothd(8)
ConditionPathIsDirectory=/sys/class/bluetooth

[Service]
Type=dbus
BusName=org.bluez
ExecStart=/usr/libexec/bluetooth/bluetoothd
NotifyAccess=main
#WatchdogSec=10
#Restart=on-failure
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
LimitNPROC=1

# Filesystem lockdown
ProtectHome=true
ProtectSystem=strict
PrivateTmp=true
ProtectKernelTunables=true
ProtectControlGroups=true
StateDirectory=bluetooth
StateDirectoryMode=0700
ConfigurationDirectory=bluetooth
ConfigurationDirectoryMode=0555

# Execute Mappings
MemoryDenyWriteExecute=true

# Privilege escalation
NoNewPrivileges=true

# Real-time
RestrictRealtime=true

[Install]
WantedBy=bluetooth.target
Alias=dbus-org.bluez.service #(Bluez  is an Api) usman have this written 
```
**Note:** `Alias=dbus-org.bluez.service` means BlueZ registers itself on D-Bus under `org.bluez` — it is an API, not a separate service.

**Example — `bluetooth.target`:**
```bash 
$ cat /usr/lib/systemd/system/bluetooth.target 
#  SPDX-License-Identifier: LGPL-2.1-or-later
#
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.

[Unit]
Description=Bluetooth Support
Documentation=man:systemd.special(7)
StopWhenUnneeded=yes
```
--- 

## 2. Managing systemd with systemctl: 

### 2.1 Reload all unit files from disk

After editing any unit file, notify systemd:
```bash 
$ systemctl daemon-reload 
```
### 2.2 Inspect the dependency tree
```bash 
$ systemctl list-dependencies 
default.target
● ├─accounts-daemon.service
● ├─gdm.service
○ ├─nvmefc-boot-connections.service
● ├─rtkit-daemon.service
● ├─switcheroo-control.service
○ ├─systemd-update-utmp-runlevel.service
● ├─tuned-ppd.service
● ├─udisks2.service
● ├─upower.service
● └─multi-user.target
●   ├─atd.service
○   ├─audit-rules.service

#truncated  output 
```
### 2.3 List loaded units and installed unit files
 ```bash 
$ systemctl list-units; systemctl list-unit-files 
```
### 2.4 List machines and active jobs: 
```bash
$ systemctl list-machines ; systemctl list-jobs 
  NAME                         STATE    FAILED JOBS
● localhost.localdomain (host) degraded 2      0

1 machines listed.
No jobs running.
```
---

## 3. Service Unit Lifecycle

All examples use `crond.service` (the cron daemon / command scheduler).

### 3.1 Check if a service is currently running:  
```bash 
$ systemctl is-active crond.service 
active
```
### 3.2 Inspect full service status
```bash 
$ systemctl status crond.service 
● crond.service - Command Scheduler
     Loaded: loaded (/usr/lib/systemd/system/crond.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-03-30 09:45:05 EDT; 1 day 2h ago
 Invocation: 8072ebac6a09475b8bd226e811f6ea86
   Main PID: 1055 (crond)
      Tasks: 1 (limit: 23129)
     Memory: 1.3M (peak: 4.6M)
        CPU: 97ms
     CGroup: /system.slice/crond.service
             └─1055 /usr/sbin/crond -n
```
### 3.3 Check if a service is enabled at boot

```bash
$ systemctl is-enabled crond.service 
enabled
```
### 3.4 Stop, inspect, and restart a service
 
``` bash 
$ sudo systemctl stop  crond.service; systemctl status crond.service;sudo systemctl restart crond.service ; systemctl status crond.service  
○ crond.service - Command Scheduler
     Loaded: loaded (/usr/lib/systemd/system/crond.service; enabled; preset: enabled)
     Active: inactive (dead) since Tue 2026-03-31 12:16:57 EDT; 1min 14s ago
   Duration: 1d 2h 31min 51.246s
 Invocation: 8072ebac6a09475b8bd226e811f6ea86
    Process: 1055 ExecStart=/usr/sbin/crond -n $CRONDARGS (code=exited, status=0/SUCCESS)
   Main PID: 1055 (code=exited, status=0/SUCCESS)
   Mem peak: 4.6M
        CPU: 98ms

● crond.service - Command Scheduler
     Loaded: loaded (/usr/lib/systemd/system/crond.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-03-31 12:18:11 EDT; 15ms ago
 Invocation: 1020b7e10295484a8d9b5ab8ba1b673f
   Main PID: 11494 (crond)
      Tasks: 1 (limit: 23129)
     Memory: 648K (peak: 1.1M)
        CPU: 6ms
     CGroup: /system.slice/crond.service
             └─11494 /usr/sbin/crond -n
#truncated output
```

### 3.5 Mask a service (hard-disable)
Masking symlinks the unit file to `/dev/null`, making it impossible to start — even as a dependency.
```bash
$ systemctl mask crond.service 
Created symlink '/etc/systemd/system/crond.service' → '/dev/null'.
```
### 3.6  Behavior while masked: 
```bash
$ systemctl disable crond.service; systemctl reload crond.service
Unit /etc/systemd/system/crond.service is masked, ignoring.
Failed to reload crond.service: Unit crond.service is masked.
``` 

### 3.7 Unmask a service: 
```bash 
$ systemctl unmask crond.service 
Removed '/etc/systemd/system/crond.service'.
```
--- 
## 4. Boot Targets

Targets are the systemd equivalent of SysV runlevels. They group units that should be active together.

| Target | Equivalent Runlevel | Description |
|--------|-------------------|-------------|
| `multi-user.target` | 3 | Multi-user, network up, no GUI |
| `graphical.target` | 5 | Multi-user with display manager |
| `rescue.target` | 1 | Single-user maintenance mode |
| `emergency.target` | — | Minimal emergency shell |

many more such as: 
initrd-fs.target           initrd-switch-root.target  integritysetup-pre.target
initrd-root-device.target  initrd.target              integritysetup.target
initrd-root-fs.target      initrd-usr-fs.target      
 
### 4.1  Get the current default boot target: 
```bash
$ systemctl get-default 
graphical.target
```
### 4.2 Change the default boot target (persistent): 
```bash
$ systemctl set-default multi-user.target 

# the terminal will change  to full terminal base i.e The system will boot into a text-only environment after the next reboot.

```
### 4.3 Switch targets on the running system (non-persistent):  
```bash 
$ systemctl isolate graphical.target
```
`isolate` starts the target and stops everything not required by it. Only targets with `AllowIsolate=yes` in their unit file can be used here.

---
## 5. Configuration File Hierarchy

systemd reads unit files from three locations, searched in this priority order (highest to lowest):

| Path | Owner | Purpose |
|------|-------|---------|
| `/etc/systemd/system/` | System administrator | Local overrides and customizations; takes precedence |
| `/run/systemd/system/` | systemd / runtime | Created at runtime, destroyed on reboot; used by transient units |
| `/usr/lib/systemd/system/` | Package manager (RPM) | Vendor-supplied defaults; never edit these directly |

**Drop-in overrides** — Instead of editing `/usr/lib/systemd/system/<unit>.service` directly (which would be overwritten on package update), create a drop-in:

```bash
$ systemctl edit crond.service
Creates /etc/systemd/system/crond.service.d/override.conf
```
This merges your changes on top of the vendor file and survives package upgrades.
