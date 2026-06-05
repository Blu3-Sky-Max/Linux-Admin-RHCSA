# System Job Scheduling: AT vs. CRON Management 

**Author:** Usman Opeyemi Olanrewaju (Blu3 Sky)  

**Date:** 2026/02/24  

**Focus:** Implementing one-time jobs (`at`) and recurring scheduled tasks (`cron`) while managing system access controls.

**modify:** 2026/07/05

---
## Table of Contents

1. [Job Scheduling Overview](#1-job-scheduling-overview)
2. [Configuring Access Permissions for `at`](#2-configuring-access-permissions-for-at)
3. [Managing Recurring Jobs with `cron`](#3-managing-recurring-jobs-with-cron)
4. [Managing Periodic Jobs with `anacron`](#4-managing-periodic-jobs-with-anacron)
5. [Access Control for `cron`](#5-access-control-for-cron)
6. [Auditing and System Context](#6-auditing-and-system-context)
7. [Key Takeaways](#7-key-takeaways)

---

## 1. Job Scheduling Overview

Job scheduling lets you execute commands at a specified time in the future — either one time or periodically based on a pre-determined schedule. A one-time execution is useful for activities that need to happen at a time of low system usage (e.g. running a lengthy shell program). Recurring jobs are used for tasks like creating compressed archives, trimming log files, monitoring the system, or removing unwanted files.

Job scheduling and execution is handled by two service daemons: `atd` and `crond`. While `atd` manages jobs scheduled to run one time in the future, `crond` is responsible for running jobs repetitively at pre-specified times. At startup, `crond` reads the schedules in files located in `/var/spool/cron` and `/etc/cron.d`, loads them into memory for on-time execution, scans them at short intervals, and updates the in-memory schedules to reflect any modifications.

| Tool | Purpose | Recurrence | Requires System Running? |
|------|---------|------------|--------------------------|
| `at` | Schedule a **one-time** job at a specific future time | No | Yes |
| `cron` | Schedule **recurring** jobs by minute/hour/day/month/weekday | Yes | Yes |
| `anacron` | Schedule **periodic** jobs that survive system downtime | Yes (daily/weekly/monthly) | No — catches up on boot |

> **Key distinction:** `cron` misses a job entirely if the system is off at the scheduled time. `anacron` records the last execution time and runs missed jobs on the next boot — making it ideal for non-24/7 systems.


## 2. Configuring Access Permissions for `at`

The `at` service controls user access through two files: `/etc/at.allow` and `/etc/at.deny`. The same logic applies to `cron` via `/etc/cron.allow` and `/etc/cron.deny`. The full decision matrix is:

| `at.allow` / `cron.allow` | `at.deny` / `cron.deny` | Impact |
|---------------------------|-------------------------|--------|
| Exists, and contains user entries | Existence does not matter | All users listed in allow files are permitted |
| Exists, but is empty | Existence does not matter | No users are permitted |
| Does not exist | Exists, and contains user entries | All users, other than those listed in deny files, are permitted |
| Does not exist | Exists, but is empty | All users are permitted |
| Does not exist | Does not exist | No users are permitted (only `root`) |



### 2.1 Enabling User Access

```bash
# Verify or create the allow file (one username per line)
$ vi /etc/at.allow
```

Add your username (e.g., `blu3sky`) to this file. Once saved, that user can schedule `at` jobs.


###  2.2 Scheduling a One-Time Job

All submitted `at` jobs are spooled in `/var/spool/at` and executed by the `atd` daemon at their scheduled times. Each submitted job gets a file containing the shell environment settings needed for successful execution, plus the command to run. You do **not** need to restart `atd` or `crond` after submitting a job.


**`at` time format variants:**

| Expression | Meaning |
|------------|---------|
| `at 1:15am` | Next 1:15 a.m. |
| `at noon` | 12:00 p.m. |
| `at 23:45` | 11:45 p.m. |
| `at midnight` | 12:00 a.m. |
| `at now + 5min` | 5 minutes from now |
| `at 10:30am 07/05/2026` | Specific time on a specific date |
| `at now + 2 days` | Same time, 2 days from now |
| `at now + 3 weeks` | Same time, 3 weeks from now |


**Example: Scheduling a simple output to a specific terminal session (pts/1):**

```bash 
$ at now + 5min
warning: commands will be executed using /bin/sh
at Wed Feb 25 08:55:00 2026
at> echo "usman is still here" > /dev/pts/1 
at> <EOT> # note: you press ctrl+d 
job 15 at Wed Feb 25 08:55:00 2026
 ```
#### 2.2.1 Checking:
```bash  
$ at -l 
15	Wed Feb 25 08:55:00 2026 a blu3sky
```
#### 2.2.2  Output in (`pts\1`) 
```bash
$ tty
/dev/pts/1
blu3sky@localhost:~$ usman is still here
``` 

#### 2.2.3  Deleting Job: 
```bash 
$ at -d (jobid...e.g 15 is our jobid) 
``` 
### 2.3 Add more job

#### 2.3.1 Adding a job at 10:30am with date 
 ```bash
$ at 10:30am 07/05/2026
warning: commands will be executed using /bin/sh
at Sun Jul  5 10:30:00 2026
at> echo "Memory info at $date \n" &>> /home/blue/memory.info
at> free -h | grep Mem &>> /home/blue/memory.info 
at> <EOT>
job 8 at Sun Jul  5 10:30:00 2026
```
#### 2.3.2 Verify 
```bash
$ at -l 
6	Sun Jul  5 10:30:00 2026 a blue
7	Sun Jul  5 10:30:00 2026 a blue
8	Sun Jul  5 10:30:00 2026 a blue
```  
### 2.4  Deleting job
```bash
$ at -d 6
```

#### 2.4.1 Verify
```bash
$ at -l 
7	Sun Jul  5 10:30:00 2026 a blue
8	Sun Jul  5 10:30:00 2026 a blue
```

#### 2.4.2 Verify from a log file
```bash
$ sudo ls -l   /var/spool/at
[sudo] password for blue: 
total 12
----------. 1 blue blue 4096 Jun  5 17:41 a0000701c58022
-rwx------. 1 blue blue 4651 Jun  5 17:43 a0000801c58022
drwx------. 2 root root    6 Jun  5 17:22 spool
``` 

## 3. Managing Recurring Jobs with cron
 
### 3.1 Cron Syntax Reference

Every line in a crontab follows this exact field order:

```
┌---------------> minute        (0 - 59)
|  ┌------------> hour          (0 - 23)
|  |  ┌---------> day of month  (1 - 31)
|  |  |  ┌------> month         (1 - 12)
|  │  |  │  ┌---> day of week   (0 - 7)  [0 and 7 are both Sunday]
|  |  |  |  |
*  *  *  *  *  command to execute
```

| Field | Allowed Values | Special Characters |
|-------|---------------|--------------------|
| Minute | 0–59 | `*` `,` `-` `/` |
| Hour | 0–23 | `*` `,` `-` `/` |
| Day of Month | 1–31 | `*` `,` `-` `/` |
| Month | 1–12 (or `jan`–`dec`) | `*` `,` `-` `/` |
| Day of Week | 0–7 (or `sun`–`sat`; 0 and 7 = Sunday) | `*` `,` `-` `/` |

**Special character meanings:**

| Symbol | Meaning | Example |
|--------|---------|---------|
| `*` | Every possible value for this field | `* * * * *` — every minute |
| `,` | List of discrete values | `5,20` — at the 5th and 20th |
| `-` | Range of values | `10-12` — hours 10, 11, and 12 |
| `/` | Step / interval | `*/5` — every 5 units |

**Practical examples:**

| Crontab Expression | Meaning |
|--------------------|---------|
| `0 2 * * *` | Every day at 02:00 |
| `*/15 * * * *` | Every 15 minutes |
| `0 9-17 * * 1-5` | Every hour from 09:00–17:00, Monday through Friday |
| `30 4 1,15 * *` | At 04:30 on the 1st and 15th of every month |
| `*/5 10-12 5,20 * *` | Every 5 min during hours 10–12, only on the 5th and 20th of the month |

> **tip:** When both "day of month" and "day of week" are set (not `*`), cron runs the job if **either** condition is true — not both simultaneously.
### 3.2 Special Cron Strings

Some cron implementations (and RHEL's `cronie`) support shorthand strings as a replacement for the five fields:

| String | Equivalent | Meaning |
|--------|-----------|---------|
| `@reboot` | — | Once at system startup |
| `@hourly` | `0 * * * *` | Once per hour, at the start |
| `@daily` | `0 0 * * *` | Once per day at midnight |
| `@weekly` | `0 0 * * 0` | Once per week, Sunday midnight |
| `@monthly` | `0 0 1 * *` | Once per month, 1st at midnight |
| `@yearly` | `0 0 1 1 *` | Once per year, Jan 1st midnight |


#### 3.2.1 Run a script once at every system boot 
```bash
$ crontab -l 
@reboot /home/blue/usman.sh >> /home/blue/testing.usman.sh.com  
```

### 3.3 Listing, Adding, and Deleting Jobs

**Open the current user's crontab for editing:**


```bash 
$ crontab -e 
```
**Add:**  
```
05  10  *  *  *   echo "Hello World is super crazy" > /home/vm3/usman.txts
```

#### 3.3.1 Verify the installed crontab: 
```bash
$ crontab -l
05  10  *  * * echo "Hello World is super crazy " > /~/usman.txts 
```
### 3.4  Add more jobs 
Scheduling  to  execute at every fifth minute past the hour between 10:00 a.m. and 11:00 a.m. on the fifth and twentieth of every month 

```bash 
$ crontab -e 
crontab: installing new crontab
Backup of vm3's previous crontab saved to /home/vm3/.cache/crontab/crontab.bak

$ crontab -l 
05  10  *  * * echo "Hello World is super crazy " > /home/vm3/usman.txts 
*/5 10-12 5,20 * * echo " Usman is always in the library " >> /home/vm3/usman.txts 
```
**Breaking down `*/5 10-12 5,20 * *`:**
- `*/5` → every 5 minutes (0, 5, 10, 15 … 55)
- `10-12` → only during hours 10, 11, and 12
- `5,20` → only on the 5th and 20th day of the month
- `*` → any month
- `*` → any day of the week
### 3.4 Add job for other users 
```bash 
$ sudo crontab -e -u user20 
[sudo] password for blue: 
no crontab for user20 - using an empty one
crontab: installing new crontab
```

#### 3.4.2 Verify
```bash
$ sudo crontab -l -u user20 
*/5 * * *  * echo "usman is tired" > usman.output.txt 
```

### 3.4 Log file cron

All activities for the `atd` and `crond` services are logged to `/var/log/cron`. The log captures the time of activity, hostname, process name and PID, owner, and a message for each invocation. It also keeps track of other `crond` events such as service start time and any delays.

```bash
$ sudo cat /var/log/cron  | tail
Jun  5 15:19:17 server30 crontab[4887]: (blue) END EDIT (blue)
Jun  5 15:19:32 server30 crond[1710]: (CRON) STARTUP (1.7.0)
Jun  5 15:19:32 server30 crond[1710]: (CRON) INFO (Syslog will be used instead of sendmail.)
Jun  5 15:19:32 server30 crond[1710]: (CRON) INFO (RANDOM_DELAY will be scaled with factor 4% if used.)
Jun  5 15:19:32 server30 crond[1710]: (CRON) INFO (running with inotify support)
Jun  5 15:19:32 server30 CROND[2045]: (blue) CMD (/home/blue/usman.sh >> /home/blue/testing.usman.sh.com  )
Jun  5 15:19:32 server30 CROND[1727]: (blue) CMDEND (/home/blue/usman.sh >> /home/blue/testing.usman.sh.com  )
Jun  5 15:20:22 server30 crontab[4641]: (blue) BEGIN EDIT (blue)
Jun  5 15:20:28 server30 crontab[4641]: (blue) END EDIT (blue)
Jun  5 15:20:39 server30 crontab[4695]: (blue) LIST (blue)
``` 
### 3.5 Log file at
```bash
$ sudo ls -l   /var/spool/at
total 16
-rwx------. 1 blue blue 4599 Jun  5 17:17 a0000401c4d8fe
-rwx------. 1 blue blue 4599 Jun  5 17:18 a0000501c4d8fc
drwx------. 2 root root    6 Jun  5 17:17 spool
```
### 3.6  Teardown 
```bash
$ crontab -r
```



## 4. Managing Periodic Jobs with `anacron`

`anacron` is designed for machines that are not running 24/7. It reads `/etc/anacrontab` and ensures periodic jobs run even if the system was off when they were due.

**View the anacrontab:**

```bash
$ cat /etc/anacrontab
```

**`/etc/anacrontab` field format:**

```
period    delay    job-id    command
```

| Field | Meaning |
|-------|---------|
| `period` | Frequency in days (e.g., `1` = daily, `7` = weekly) |
| `delay` | Minutes to wait after boot before running (prevents load spikes) |
| `job-id` | Unique identifier string for logging |
| `command` | The command or script to execute |

**Typical default entries:**

```bash
# /etc/anacrontab
SHELL=/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root

1       5       cron.daily      run-parts /etc/cron.daily
7       25      cron.weekly     run-parts /etc/cron.weekly
@monthly 45     cron.monthly    run-parts /etc/cron.monthly
```

**Last execution timestamps are stored in:**

```bash
$ ls /var/spool/anacron/
cron.daily  cron.monthly  cron.weekly
```

> `anacron` reads these timestamps on boot to determine if a job was missed and needs to run.

## 5. Access Control for `cron`

Same two-file logic as `at`, applied to `cron`:

| File | Effect |
|------|--------|
| `/etc/cron.allow` | If this file exists, **only** listed users may use `crontab` |
| `/etc/cron.deny` | If `cron.allow` doesn't exist, users listed here are **blocked** |

- If both files are absent → implementation-defined (on RHEL, all users are permitted by default).
```bash
# Permit only specific users
$ echo "blu3sky" >> /etc/cron.allow

# Block a specific user (when cron.allow does not exist)
$ echo "baduser" >> /etc/cron.deny
```
## 6. Auditing and System Context

Understanding where the system stores job information is essential for security auditing and troubleshooting.

| Location | Purpose |
|----------|---------|
| `/var/spool/cron/` | User crontab definitions (one file per user, named by username) |
| `/var/spool/at/` | Queued `at` job files |
| `/var/spool/anacron/` | Anacron last-run timestamps |
| `/var/log/cron` | Execution log for cron and at jobs (RHEL/CentOS) |
| `/etc/cron.allow`, `/etc/cron.deny` | Access control for `cron` |
| `/etc/at.allow`, `/etc/at.deny` | Access control for `at` |

**Check cron execution logs in real time:**

```bash
$ tail -f /var/log/cron
```

**Verify a user's spool file directly (as root):**

```bash
$ cat /var/spool/cron/vm3
05   10      *     *  *  echo "Hello World is super crazy" > /home/vm3/usman.txts
*/5  10-12   5,20  *  *  echo "Usman is always in the library" >> /home/vm3/usman.txts
```

## 7. Key Takeaways

- **`atd`** handles one-time jobs; **`crond`** handles recurring jobs; **`anacron`** handles periodic jobs that must survive reboots.
- At startup, `crond` reads `/var/spool/cron` and `/etc/cron.d`, loads schedules into memory, and scans for changes at short intervals — no daemon restart needed after editing a crontab.
- All `at` jobs are spooled in `/var/spool/at`; you do not need to restart `atd` after submitting a job.
- `at` accepts many time formats: `at noon`, `at midnight`, `at 23:45`, `at now + 5min`, `at 10:30am 07/05/2026`.
- The **allow file always wins**: if `at.allow` or `cron.allow` exists (even empty), the deny file is completely ignored. Only when the allow file is absent does the deny file apply.
- If **both files are absent**, no users are permitted (only `root`). An **empty deny file** permits all users.
- Cron's five fields: `minute hour day-of-month month day-of-week`. Memorize the order.
- Special characters: `*` (all), `,` (list), `-` (range), `/` (step). They combine — e.g., `*/5 10-12 5,20 * *`.
- When both day-of-month and day-of-week are specified (neither is `*`), cron fires if **either** condition matches — not both.
- `@reboot`, `@daily`, `@weekly`, etc. are valid shorthand replacements for the five time fields.
- Both `atd` and `crond` log all activity to `/var/log/cron` — including job launch (`CMD`), completion (`CMDEND`), daemon restarts, and crontab edits (`BEGIN EDIT`/`END EDIT`). Root privilege required to read.
- `anacron` checks `/var/spool/anacron/` timestamps on boot and runs any overdue jobs after a configurable delay (`delay` field in `/etc/anacrontab`).
                                                               
