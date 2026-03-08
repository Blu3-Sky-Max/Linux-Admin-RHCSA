# 📦 RHEL Package Management  and Integrity Checks

**Author:** Usman O. Olanlanrewaju (Blu3 Sky)  
**Date:** 2026/03/03  
**Focus:** Comprehensive management of RPM packages, repository configuration, and integrity verification on Red Hat-based systems.

---

## 1. Core Package Information Retrieval (RPM)

i use `rpm` to query database entries, verify installation integrity, and list package contents without relying on DNF.

### Checking uninstalled Package Details
Querying the details of an uninstalled package (`setup` package used as an example):
```bash
$ rpm -qip /mnt/BaseOS/Packages/setup-2.14.5-4.el10.noarch.rpm 
Name        : setup
Version     : 2.14.5
Release     : 4.el10
Architecture: noarch
Install Date: (not installed)
Group       : System Environment/Base
Size        : 737724
License     : LicenseRef-Fedora-Public-Domain
Signature   : RSA/SHA256, Mon 11 Nov 2024 06:12:09 AM CST, Key ID 199e2f91fd431d51
Source RPM  : setup-2.14.5-4.el10.src.rpm
Build Date  : Tue 29 Oct 2024 03:51:47 PM CDT
Build Host  : ppc-015.brew-001.prod.nsh.tph.redhat.com
Packager    : Red Hat, Inc. <http://bugzilla.redhat.com/bugzilla>
Vendor      : Red Hat, Inc.
URL         : https://pagure.io/setup/
Summary     : A set of system configuration and setup files
Description :
The setup package contains a set of important system configuration and
setup files, such as passwd, group, and profile.
```

### Verifying Package Signatures (Integrity Check): 
```bash
$ sudo rpmkeys --import /mnt/RPM-GPG-KEY-redhat-release 

$ sudo rpmkeys -K /mnt/BaseOS/Packages/setup-2.14.5-4.el10.noarch.rpm 
/mnt/BaseOS/Packages/setup-2.14.5-4.el10.noarch.rpm: digests signatures OK
``` 

## 2. Package Management Operations (RPM)
****Managing Repositories and Installation**** 
While DNF is the preferred front-end, understanding the underlying RPM operations is key.

### Installing a package:
```bash  
$ sudo rpm -ivh /mnt/BaseOS/Packages/setup-2.14.5-4.el10.noarch.rpm
Verifying...                          ################################# [100%]
Preparing...                          ################################# [100%]
    package setup-2.14.5-4.el10.noarch is already installed
```
### Looking for a pacakage path: 
```bash
$ find /mnt -name zsh* 
/mnt/BaseOS/Packages/zsh-5.9-15.el10.x86_64.rpm
```
### Reinstalling/Verifying a Package:
```bash
$ sudo  rpm --reinstall -vh /mnt/BaseOS/Packages/zsh*
warning: /mnt/BaseOS/Packages/zsh-5.9-15.el10.x86_64.rpm: Header V4 RSA/SHA256 Signature, key ID fd431d51: NOKEY
Verifying...                          ################################# [100%]
Preparing...                          ################################# [100%]
Updating / installing...
   1:zsh-5.9-15.el10                  ################################# [100%]
```
###  removing  the zsh package:
```bash
$ sudo rpm -e zsh-5.*
```

### what package provides a command:
```bash 
$ rpm -qf /bin/ls
coreutils-9.5-6.el10.x86_64  # the output shows the package owner
```
## 3. Advanced Operations: Extracting Package Contents 

### Extracting Files to Temporary Directory

This is useful for auditing configuration files before deployment.

```bash
# Extracting the /etc/passwd file from the setup package to the /tmp directory using rpm2cpio:
$ rpm2cpio /mnt/BaseOS/Packages/setup*.rpm | cpio -idv ./etc/passwd
./etc/passwd
1452 blocks
```

