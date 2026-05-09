# 📦 RHEL Package Management: DNF Repository Auditing

**Author:** Usman O. Olanrewaju (Blu3 Sky)  

**Date:** 2026/03/03 
 
**Focus:** Advanced CLI package management on RHEL/Fedora, including repository configuration, package integrity verification, and DNF group management.

**modify:** 2026/04/09 

Danified yellowdog update modified (DNF) package management:
A DNF repository (yum repository or simply a repo) is a digital library for storing software packages and metadata.

## 1. Repository Configuration and State (`/etc/yum.repos.d`)

Using the local ISO image to configure repository sources:

### 1.1 Custom Repository Snippet (BaseOS) & (AppStream):
```bash

[BaseOS]      #there should not be a space in the id or else it will says 'bad id' 
name=Base OS Software 
baseurl=//mnt/BaseOS 
enabled=1 # enable
gpgcheck=0  # disable pretty good privacy

[AppStream]
name=Application Software 
baseurl=//mnt/AppStream
enabled=1 
gpgcheck=0  
```
### 1.2 Checking Repository Status: 
```bash 
$ dnf repolist 
Not root, Subscription Management repositories not updated
repo id                                           repo name
AppStream                                         Application Software
BaseOS                                            Base OS Software by Blu3-Sky
```
## 2. Package Maintenance (List,Install, Reinstall, Remove,upgrade): 

### 2.1 List all packages available for installation:
```bash 
$ dnf repoquery | wc -l 
Last metadata expiration check: 20:27:27 ago on Sat 21 Mar 2026 12:52:37 AM +03.
5371
``` 
### 2.2 List package in a single repo: 
```bash 
$ dnf repoquery --repo "BaseOS" | wc -l 
Last metadata expiration check: 20:32:00 ago on Sat 21 Mar 2026 12:52:37 AM +03.
946 # package in our BaseOS
```
### 2.3 List all installed packages: 
```bash 
$ dnf list installed  | head -5 
Installed Packages
ModemManager.x86_64                                  1.22.0-7.el10                  @anaconda 
ModemManager-glib.x86_64                             1.22.0-7.el10                  @anaconda 
NetworkManager.x86_64                                1:1.52.0-1.el10_0              @anaconda 
```
### 2.4 Installing a package: 
```bash 
$ sudo dnf install ant -y 
```
### 2.5 Removing a package: 
```bash 
$ sudo dnf remove ant -y 
```
### 2.6 Check for update: 
```bash 
$ dnf check-update
Not root, Subscription Management repositories not updated
Last metadata expiration check: 0:26:00 ago on Sat 21 Mar 2026 09:40:27 PM +03.
```
### 2.7 Updating:
```bash  
$ sudo dnf update 
Updating Subscription Management repositories.
Last metadata expiration check: 2:56:14 ago on Sat 21 Mar 2026 09:01:28 PM +03.
Dependencies resolved.
Nothing to do.
Complete!
```
## 3.Package Querying and Integrity Verification

### 3.1 Inspecting Installed & Source Packages:  
 ```bash 
$ dnf provides /etc/passwd 
Not root, Subscription Management repositories not updated
Last metadata expiration check: 0:24:19 ago on Sat 21 Mar 2026 09:40:27 PM +03.
setup-2.14.5-4.el10.noarch : A set of system configuration and setup files
Repo        : @System
Matched from:
Filename    : /etc/passwd

setup-2.14.5-4.el10.noarch : A set of system configuration and setup files
Repo        : BaseOS
Matched from:
Filename    : /etc/passwd
```
### 3.2 Metadata check: 
```bash 
$ dnf info setup 
Not root, Subscription Management repositories not updated
Last metadata expiration check: 0:28:40 ago on Sat 21 Mar 2026 09:40:27 PM +03.
Installed Packages
Name         : setup
Version      : 2.14.5
Release      : 4.el10
Architecture : noarch
Size         : 720 k
Source       : setup-2.14.5-4.el10.src.rpm
Repository   : @System
From repo    : anaconda
Summary      : A set of system configuration and setup files
URL          : https://pagure.io/setup/
License      : LicenseRef-Fedora-Public-Domain
Description  : The setup package contains a set of important system configuration and
             : setup files, such as passwd, group, and profile.
```
### 3.3  Dependency Checking:  
```bash  
$ dnf repoquery --whatdepends setup 
Not root, Subscription Management repositories not updated
Last metadata expiration check: 0:31:51 ago on Sat 21 Mar 2026 09:40:27 PM +03.
basesystem-0:11-22.el10.noarch
console-login-helper-messages-issuegen-0:0.21.3-10.el10.noarch
console-login-helper-messages-motdgen-0:0.21.3-10.el10.noarch
console-login-helper-messages-profile-0:0.21.3-10.el10.noarch
cups-1:2.4.10-11.el10.x86_64
cups-filesystem-1:2.4.10-11.el10.noarch
cyrus-imapd-0:3.8.3-7.el10.x86_64
filesystem-0:3.18-16.el10.x86_64
hplip-0:3.23.12-8.el10.x86_64
initscripts-0:10.26-2.el10.x86_64
lockdev-0:1.0.4-0.46.20111007git.el10.x86_64
pam-0:1.6.1-7.el10.x86_64
procmail-0:3.24-8.el10.x86_64
restore-1:0.4-0.59.b47.el10.x86_64
rpcbind-0:1.2.7-3.el10.x86_64
shadow-utils-2:4.15.0-5.el10.x86_64
util-linux-0:2.40.2-10.el10.x86_64
```
## 4 Advanced Management: Groups and File Location: 

### 4.1 Listing installed and available groups: 
```bash 
$ dnf grouplist --installed ; dnf grouplist --available 
Not root, Subscription Management repositories not updated
Last metadata expiration check: 0:59:10 ago on Sat 21 Mar 2026 09:40:27 PM +03.
Installed Environment Groups:
   Server with GUI
Installed Groups:
   Container Management
   Headless Management
   Security Tools
Not root, Subscription Management repositories not updated
Last metadata expiration check: 0:59:10 ago on Sat 21 Mar 2026 09:40:27 PM +03.
Available Environment Groups:
   Server
   Minimal Install
   Workstation
   Custom Operating System
   Virtualization Host
Available Groups:
   Legacy UNIX Compatibility
   Smart Card Support
   Console Internet Tools
   Development Tools
   .NET Development
   Graphical Administration Tools
   Network Servers
   RPM Development Tools
   Scientific Support
   System Tools
```

### 4.2 Installing groups `security tools` and `scientific support`: 
```bash 
$ sudo dnf groupinstall "Security Tools" "Scientific Support" -y 
```
### 4.3 Checking the metadata for `"Scientific Support"`: 
```bash 
$ dnf groupinfo "Scientific Support"
Not root, Subscription Management repositories not updated
Last metadata expiration check: 1:01:37 ago on Sat 21 Mar 2026 09:40:27 PM +03.
Group: Scientific Support
 Description: Tools for mathematical and scientific computations, and parallel computing.
 Optional Packages:
   fftw
   fftw-devel
   fftw-static
   lapack
   mpich-devel
   openmpi
   openmpi-devel
   python3-numpy
   python3-scipy
   units
```
### 4.4 Removing `"Scientific Support"`: 
```bash 
$ sudo dnf groupremove "Scientific Support" -y
```
## 5. Forensic Inspection (File Location)
Understanding where the system stores package metadata and logs is essential for security auditing.
- Package Log: All DNF/RPM activities are logged at `/var/log/dnf.log.`
- Repository Location: Any custom repository file created should end with .repo and typically resides in `/etc/yum.repos.d/.`
