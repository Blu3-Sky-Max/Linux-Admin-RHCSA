# RHEL Package Management: Flatpak Repository Auditing

**Author:** Usman O. Olanrewaju (Blu3 Sky)
**Date:** 2026/03/22 
**Focus:** Managing containerized desktop applications using `flatpak`, including remote repository configuration (Flathub), application lifecycle management, and granular sandbox permission control (`override`).



Flatpak is an application sandboxing and distribution framework designed to provide secure, portable, and version-flexible software deployment across different Linux distributions. It allows users to install applications in isolated environments, separating them from the host system for enhanced security and stability.

## 1. Initial Setup and Configuration

### 1.1 Installing the Flatpak Framework
```bash 
$ sudo dnf install flatpak -y 
``` 
### 1.2  Adding a System-Wide Remote `Flathub`:
Applications will be available to all users on the system. 
```bash 
$ flatpak remote-add flathub https://flathub.org/repo/flathub.flatpakrepo
```
### 1.3 Adding a User-Only Remote  `fedora`:
Applications will be  available only to the current user.
```bash 
$ flatpak remote-add --user fedora  oci+https://registry.fedoraproject.org 
```
### 1.4 Verifying Configured Remotes: 
```bash 
$ flatpak remotes -d 
Name    Title                    URL                                    Collection ID     Subset Filter Priority Options                  … … Homepage                                               Icon
flathub Flathub                  https://dl.flathub.org/repo/           -                 -      -      1        system                   … … https://flathub.org/                                   https://dl.flathub.org/repo/logo.svg
rhel    Red Hat Enterprise Linux oci+https://flatpaks.redhat.io/rhel/   com.redhat.Stable -      -      1        system,oci,no-gpg-verify … … https://catalog.redhat.com/software/containers/explore https://www.redhat.com/misc/favicon.ico
fedora  -                        oci+https://registry.fedoraproject.org -                 -      -      1        user,oci                 … … -                                                      -
``` 
## 2. Application Discovery and Management 

### 2.1 Numbers of Application in both Remotes: 
```bash 
$ flatpak remote-ls --app -u |wc -l; flatpak remote-ls --app flathub | wc -l
518
3288
```
### 2.2 Number of Runtime in both Remotes: 
```bash 
$ flatpak remote-ls --runtime -u  | wc -l ; flatpak remote-ls --runtime | wc -l
90
2398
```

### 2.3 Searching for an Application: 
```bash 
$ flatpak search gedit
Name         Description                Application ID                  Version Branch Remotes
gedit        Text editor                org.gnome.gedit                 49.0    stable flathub,fedora
Text Editor  Edit text files            org.gnome.TextEditor            49.1    stable fedora
Apostrophe   Edit Markdown in style     ….gnome.gitlab.somas.Apostrophe 2.6.3   stable fedora
ThemeGenera… Generate Styles with Styl… ….github.thiefmd.themegenerator 0.1.3   stable flathub
```

### 2.4 Viewing Application Metadata:
viewing metadata for calculator 
```bash 
$ flatpak remote-info flathub org.gnome.Calculator 

Calculator - Perform arithmetic, scientific or financial calculations

        ID: org.gnome.Calculator
       Ref: app/org.gnome.Calculator/x86_64/stable
      Arch: x86_64
    Branch: stable
   Version: 50.0
   License: GPL-3.0-or-later
Collection: org.flathub.Stable
  Download: 1.8 MB
 Installed: 4.7 MB
   Runtime: org.gnome.Platform/x86_64/50
       Sdk: org.gnome.Sdk/x86_64/50

    Commit: 83ef900d1ed3f3ff2edfb6e894d4a2120da21848f1bbf16c57d9b1b299c59f5b
    Parent: 5e09c43a2b65723fd726685d08ec45f72e89669756ea9c07b440f385a027acc6
   Subject: Merge pull request #52 from flathub/update-master-19cd934 (ee57e22e7a51)
      Date: 2026-03-18 19:39:58 +0000
```

### 2.5 installing an Application : 
```bash 
$ flatpak install flathub  org.gnome.gedit  -y 
```

### 2.6 Listing Installed Applications: 
Using column for listing
```bash
$ flatpak list --columns=size,name,app 
Installed size     Name                                      Application ID
444.9 MB       Mesa                                      org.freedesktop.Platform.GL.default
444.9 MB       Mesa (Extra)                              org.freedesktop.Platform.GL.default
 43.6 MB       Codecs Extra Extension                    org.freedesktop.Platform.codecs-extra
  1.1 GB       GNOME Application Platform version 50     org.gnome.Platform
  9.4 MB       gedit                                     org.gnome.gedit
```
### 2.7 Running a Flatpak Application: 
```bash
$ flatpak run org.gnome.gedit  &
[1] 75837

# process status 
$ flatpak ps
Instance   PID   Application     Runtime
2288385663 75837 org.gnome.gedit org.gnome.Platform

```

## 3. Sandbox Permission Management: 

### 3.1 Granting Network Access: 
```bash 
$ sudo flatpak override  org.gnome.gedit --share=network 
```

### 3.2 Verifying Current Permissions: 
```bash 
$ sudo flatpak override  org.gnome.gedit --show
[Context]
shared=network;
```
### 3.3 Revoke network access: 
```bash 
$ sudo flatpak override  org.gnome.gedit --unshare=network 
```
### 3.4 Confirmation of Revocation: 
```bash
$ sudo flatpak override  org.gnome.gedit --show
[Context]
shared=!network;
```
### 3.5 Resetting All Permissions:  
Resetting Gedit 
```bash 
$ flatpak --reset override  org.gnome.gedit 
```
## 4. Cleanup and Maintenance: 

### 4.1 Uninstalling an Application: 
```bash 
$ sudo flatpak uninstall org.gnome.gedit -y 
```
### 4.2 Removing Unused Runtime: 
```bash
$ sudo flatpak  uninstall --unused -y 
```
### 4.3 Checking Disk Usage: 
```bash
# Check disk usage for system-wide Flatpak installations
$ du -sh /var/lib/flatpak/ 
11G	/var/lib/flatpak/

# Check disk usage for the current user's Flatpak installations
$ du -sh .local/share/flatpak/   # user usage 
20K	.local/share/flatpak/
``` 
## 5. Forensic Inspection (Key File Locations)

Understanding the Flatpak file system structure is essential for troubleshooting, auditing, and manual intervention.

-    System-Wide Installations:

        Location: /var/lib/flatpak/

        Content: This is the root for all system-wide Flatpak data.

        app/: Contains the actual installed application files.

        runtime/: Contains the runtime and SDKs shared by applications.

        repo/: The OSTree repository that manages all the data. Modifications here should be done with extreme care.

-     User-Specific Installations:

        Location: ~/.local/share/flatpak/

        Content: Same structure as the system-wide location, but for applications installed with the --user flag.

-     Permission Overrides:

        System-Wide: /etc/flatpak/overrides/

        User-Specific: ~/.local/share/flatpak/overrides/

        Content: When you use flatpak override, it creates files in these locations. A file named org.gnome.gedit (for example) would contain the specific permissions you've granted or revoked.

-    Application Data (User-Specific):

        Location: ~/.var/app/

        Content: This is where sandboxed applications store their user-specific configuration files, cache, and data. Each application gets its own directory (e.g., ~/.var/app/org.gnome.gedit/). This is a critical location for backing up application settings.
