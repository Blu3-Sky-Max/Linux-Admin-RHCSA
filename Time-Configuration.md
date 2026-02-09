# 🕰️ Linux Time: Zone & NTP Configuration

**Author:** Usman O. Olanrewaju (Blu3 Sky)
**Date:** 2026/02/9 
**Environment:** Fedora (Host), RHEL (Lab), Framework 13

---

### 📝 Lab Overview

This lab documents the process of managing system time by configuring the time zone and verifying Network Time Protocol (NTP) synchronization. Accurate timekeeping is a fundamental skill for system administration, impacting everything from log analysis to security protocols.

### 1. Initial State Assessment

Before applying any changes, I first inspected the current time configuration to establish a baseline. The `timedatectl` command provides a comprehensive overview of the system's time settings.

```bash
$ timedatectl
               Local time: Mon 2026-02-09 14:02:37 PST
           Universal time: Mon 2026-02-09 22:02:37 UTC
                 RTC time: Mon 2026-02-09 22:02:37
                Time zone: America/Los_Angeles (PST, -0800)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no


$ date 

Mon Feb  9 02:12:13 PM PST 2026
```
### 2.  Time Zone Configuration

The primary objective was to change the system time zone from PST to US central Time (America/North_Dakota/New_salem).

First, I identified the exact time zone name required by the system by listing all available zones and filtering the results.

```bash
 
$ timedatectl list-timezones | grep New 
America/New_York
America/North_Dakota/New_Salem
Canada/Newfoundland
```
With the correct identifier, I executed the set-timezone command with sudo privileges to update the system configuration.

```bash 

$ sudo timedatectl set-timezone America/North_Dakota/New_Salem
```

### 3. Verification

To ensure the configuration was successful and immediately active, I re-ran the timedatectl command. This verification step is crucial to confirm that changes have taken effect without requiring a system reboot.

```bash 

$ timedatectl 
               Local time: Mon 2026-02-09 16:26:36 CST
           Universal time: Mon 2026-02-09 22:26:36 UTC
                 RTC time: Mon 2026-02-09 22:26:36
                Time zone: America/North_Dakota/New_Salem (CST, -0600)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no

$ date 
Mon Feb  9 04:30:01 PM CST 2026

``` 

