# System Job Scheduling: AT vs. CRON Management 

**Author:** Usman Opeyemi Olanrewaju (Blu3 Sky)  
**Date:** 2026/02/24  
**Focus:** Implementing one-time jobs (`at`) and recurring scheduled tasks (`cron`) while managing system access controls.

---

## 1. Job Scheduling Overview

*   **`at`:** Used to schedule a **one-time job** to run at a specific future time.
*   **`cron`:** Used to schedule **recurring jobs** based on minute, hour, day, month, and weekday settings.

---

## 2. Configuring Access Permissions for `at`

The `at` service requires explicit permission for users. This is controlled via configuration files, which may vary by distribution.

###  2.1: Enabling User Access

If the system does not have the necessary files, they must be created.
1.  **Allow List Check:** Verify or create `/etc/at.allow`. This file lists users **permitted** to use the `at` command (one user per line).
    ```bash
    # Check if the file exists, or create it if necessary
    vi /etc/at.allow
    ```
    *Add your username (e.g., `blu3sky`) to this file.*

2.  **Deny List (Optional):** If `/etc/at.allow` is empty or does not exist, the system checks `/etc/at.deny`. For strict security, `/etc/at.deny` should be present, listing users who are **explicitly forbidden**.

###  2.2: Scheduling a One-Time Job
Use `crontab -e` to open the user's crontab file. For single jobs, use the `at` command interface.

**Example: Scheduling a simple output to a specific terminal session (pts/1):**

```bash 
$ at now + 5min
warning: commands will be executed using /bin/sh
at Wed Feb 25 08:55:00 2026
at> echo "usman is still here" > /dev/pts/1 
at> <EOT> # note: you press ctrl+d 
job 15 at Wed Feb 25 08:55:00 2026
 ```
#### Checking:
```bash  
$ at -l 
15	Wed Feb 25 08:55:00 2026 a blu3sky
```
#### Output in (`pts\1`) 
```bash
$ tty
/dev/pts/1
blu3sky@localhost:~$ usman is still here
``` 

#### Deleting Job: 
```bash 
$ at -d (jobid...e.g 15 is our jobid) 
``` 


## 3. Managing Recurring Jobs with cron
 
### 3.1 Listing and Deleting Jobs

Use crontab to manage user-specific scheduled jobs.
#### Creating a spool using `cron` 

**Example: Scheduling a simple output to a home dir:**

#### Creating a job to be excuted at 10:05 

```bash 
$ crontab -e 
```
#### List all scheduled jobs for the current user
```bash
$ crontab -l
03  10  *  * * echo "Hello World is super crazy " > /~/usman.txts 
```
#### Adding to jobs 
Scheduling  to  execute at every fifth minute past the hour between 10:00 a.m. and 11:00 a.m. on the fifth and twentieth of every month 
```bash
 
$ crontab -e 
crontab: installing new crontab
Backup of vm3's previous crontab saved to /home/vm3/.cache/crontab/crontab.bak

$ crontab -l 
05  10  *  * * echo "Hello World is super crazy " > /home/vm3/usman.txts 
*/5 10-12 5,20 * * echo " Usman is always in the library " >> /home/vm3/usman.txts 
```

#### Delete all scheduled jobs for the current user
```bash
$ crontab -r
```

## 4. Auditing and System Context

Understanding where the system stores job information is key for security auditing.

Key Locations to Verify Job Status/Logs:

* Job Spool: /var/spool/cron (Actual job definitions) also for `at` /var/spool/at the dir for job files

* System Logs: /var/log/cron (RHEL/CentOS) or /var/log/syslog (Debian/Ubuntu) for execution details.

* Backup: On some systems, backups of the crontab may exist in ~/cache/cron/cron.bak.

### Security Consideration:
Recursion, as noted in our previous attempt, is a risk here. Scripts scheduled via cron that re-schedule themselves must be carefully monitored to prevent infinite loops that can exhaust system resources.
