# 🛠️ Linux Administration Labs

This directory is dedicated to my journey toward becoming a **Red Hat Certified System Administrator (RHCSA)**. It contains documentation and logs of system-level configurations, troubleshooting, and maintenance.

## 📂 Current Labs

###  **Lab 01:** [🗄️ Storage Management & Partitioning](./Storage-Management.md)
- **Focus:** Partitioning disks with `parted`, file system creation, and persistent mounting via `/etc/fstab`.
- **Status:** Completed.
 
### **Lab 02:** [🕰️ Time Zone & NTP Configuration](Time-Configuration.md)
-   **Focus:** Configuring the system time zone, verifying settings with `timedatectl`, and understanding NTP synchronization.
-   **Status:** Completed.

### **Lab 03:** [🔐 Password Policy Management](Password-Policy-Management.md)
-   **Focus:** Configuring password aging via `chage`, auditing settings in `/etc/shadow`, and managing account locking/unlocking (`passwd -l`).
-   **Status:** Completed.

### **Lab 04:** [👥 User & Access Control Management](User-Mgmt-Labs.md)
-   **Focus:** Mastering user lifecycle (`useradd`, `usermod`, `userdel`), group operations (`groupadd`, `groupmod`), and granular file permission manipulation (`chown`, `chmod` octal/symbolic).
-   **Status:** Completed.
### **Lab 05:** [⏱️ Job Scheduling Management](job-scheduling.md)
-   **Focus:** Implementing and managing one-time jobs (`at`) and recurring tasks (`cron`), including understanding user access control (`/etc/at.allow`) and log locations.
-   **Status:** Completed.
### **Lab 06:** [📦 Package Management & Integrity With RPM](RPM-Package-Management.md)
-   **Focus:** Managing RPMs with `rpm`, dependency resolution repository checks, and verifying package signatures.
-   **Status:** Completed.
### **Lab 07:** [📦 Package Management & Integrity with DNF](DNF-Package-Management.md)
-   **Focus:** Managing packages with `dnf`, repository configuration (ISO mounting), and package integrity verification.
-   **Status:** Completed.
### **Lab 08:** [📦 Application Sandboxing & Management with Flatpak](Flatpak-Package-Management.md)
-   **Focus:** Deploying containerized applications using `flatpak`, managing remotes (Flathub), executing isolated processes, and enforcing granular sandbox security permissions.
-   **Status:** Completed.
---
**Environment:** Fedora (Host), Rocky Linux (Lab),RHEL (Lab),  Framework 13.
