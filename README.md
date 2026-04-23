# 🔧 Linux Administration Labs

Documentation and logs of system-level configurations, troubleshooting, and maintenance work toward **Red Hat Certified System Administrator (RHCSA)** certification.

**Environment:** Fedora (Host) · Rocky Linux (Lab) · RHEL (Lab) · Framework 13

---

## 📁 Labs

### Lab 01: [🖥️ Storage Management & Partitioning](./Storage-Management.md)
- **Focus:** Partitioning disks with `parted`, file system creation, and persistent mounting via `/etc/fstab`.
- **Status:** Completed.

### Lab 02: [🕐 Time Zone & NTP Configuration](./Time-Configuration.md)
- **Focus:** Configuring the system time zone, verifying settings with `timedatectl`, and understanding NTP synchronization.
- **Status:** Completed.

### Lab 03: [🔐 Password Policy Management](./Password-Policy-Management.md)
- **Focus:** Configuring password aging via `chage`, auditing settings in `/etc/shadow`, and managing account locking/unlocking (`passwd -l`).
- **Status:** Completed.

### Lab 04: [👥 User & Access Control Management](./User-Mgmt-Labs.md)
- **Focus:** User lifecycle management (`useradd`, `usermod`, `userdel`), group operations (`groupadd`, `groupmod`), and granular file permission manipulation (`chown`, `chmod` octal/symbolic).
- **Status:** Completed.

### Lab 05: [⏰ Job Scheduling Management](./job-scheduling.md)
- **Focus:** Implementing and managing one-time jobs (`at`) and recurring tasks (`cron`), including user access control (`/etc/at.allow`) and log locations.
- **Status:** Completed.

### Lab 06: [📦 Package Management & Integrity with RPM](./RPM-Package-Management.md)
- **Focus:** Managing RPMs with `rpm`, dependency resolution, repository checks, and verifying package signatures.
- **Status:** Completed.

### Lab 07: [📦 Package Management & Integrity with DNF](./DNF-Package-Management.md)
- **Focus:** Managing packages with `dnf`, repository configuration (ISO mounting), and package integrity verification.
- **Status:** Completed.

### Lab 08: [🗃️ Application Sandboxing & Management with Flatpak](./Flatpak-Packagement-Management.md)
- **Focus:** Deploying containerized applications using `flatpak`, managing remotes (Flathub), executing isolated processes, and enforcing granular sandbox security permissions.
- **Status:** Completed.

### Lab 09: [⚙️ System Initialization & Management](./System-Initialization-Management.md)
- **Focus:** systemd architecture, unit file hierarchy (`/usr/lib/systemd/system/` → `/etc/systemd/system/`), managing service lifecycles (`enable`, `disable`, `mask`, `unmask`), boot target switching, and safe configuration overrides via `systemctl edit`.
- **Status:** Completed.
### Lab 10: [📋 System Logging Management](./System-Logging-Management.md)
- **Focus:** rsyslog architecture and config validation, log file inspection (`/var/log/`), custom message injection with `logger`, log rotation via `logrotate`, and systemd journal querying with `journalctl`.
- **Status:** Completed.

### Lab 11: [⚡ Performance Tuning Management](./Performance-Tuning-Management.md)
- **Focus:** System tuning with `tuned`, profile management via `tuned-adm` (list, switch, recommend, off), and workload-optimized profile selection for bare metal, VM, and power-saving environments.
- **Status:** Completed.
### Lab 12: [💾 LVM Storage Management](./Lvm-Storage-Management.md)
- **Focus:** Partition tables (MBR/GPT), Physical Volumes, Volume Groups, Logical Volume lifecycle (create, extend, resize, rename, teardown), filesystem formatting and persistent mounts.
- - **Status:** Completed.
