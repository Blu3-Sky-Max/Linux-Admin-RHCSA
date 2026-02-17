# 👤 User, Group, and Access Control Management 

**Author:** Usman O. Olanrewaju (Blu3 Sky)  
**Date:** 2026/02/16  
**Focus:** Mastering User Lifecycle Management, Group Membership, and Mandatory File Permissions (ACLs).

---

## 1. User Account Lifecycle Management

This section documents the core commands for managing user existence and shell access.

### 1.1 Creating a User
Creating a user with explicit UID, Home Directory, and Shell assignment.

#### Create user with UID 1010, custom home, and /bin/sh shell
```bash
$ sudo useradd -u 1010 -d /usr/testuser -s /bin/sh testuser
```
Verification: 
```bash
$ tail -1 /etc/passwd
testuser:x:1010:1010::/usr/testuser:/bin/sh
 ```

### 1.2 Setting Initial Password 
Setting an initial password for the user account instantly.
#### Setting Password: 
```bash 
$ sudo passwd testuser
new passwd: 
retype password: 
```

### 1.3 Modifying and Deleting Users

Using `usermod` for configuration changes and `userdel -r` for removal  
#### Rename the user account:
```bash
sudo usermod -l user1 testuser
$ tail -1 /etc/passwd
user1:x:1010:1010::/usr/testuser:/bin/sh
```
#### Add user to a new group:  
```bash 
$ sudo usermod -aG newlife testuser
$ tail  -5 /etc/group
user200:x:1002:
dba:x:6000:blu3sky
manuser:x:1005: 
newlife:x:7000:testuser
testuser:x:1005:
```
#### Delete user with recursive removal of home directory: 

```bash
sudo userdel -r user1
$ tail -1 /etc/passwd
user200:x:1002:1002::/home/user200:/bin/bash
```

## 2. Group Management Operations

Managing group identities, membership, and primary GID assignment.

### 2.1 Creating and Modifying Groups

#### Create a new group with a specific GID:
```bash
$ sudo groupadd -g 7000 newexist
$ tail -1 /etc/group
newexist:x:7000:
```
#### Change user's primary GID to the new group (7000):
```bash
$ sudo groupmod -og 7000 testuser
$ tail -2 /etc/group
newexist:x:7000:
testuser:x:7000:
```

###  2.2 Changing User Group Membership
Adjusting secondary group memberships using usermod.

#### Creating a  group with id: 
```bash
$ sudo groupadd -g 7000 newexist
$ tail -1 /etc/group
newexist:x:7000:
```
#### Adding a new group to same gid: 
```bash
$ sudo groupmod -og 7000 testuser
$ tail -2 /etc/group
newexist:x:7000:
testuser:x:7000:
```
#### Changing group membership dynamically:
```bash
$ sudo groupmod -n newlife newexist  
$ tail -2 /etc/group
manuser:x:1005:
newlife:x:7000:
```

### Deleting group:  
```bash
$ sudo groupdel newlife
$ tail -5 /etc/group
blu3sky:x:1000:
user100:x:1001:
user200:x:1002:
dba:x:6000:blu3sky
manuser:x:1005:
```

sometime the need to use `-f` with the groupdel espically when linked to different user

## 3. File Permissions Management (Security Context)

Understanding permissions is crucial for access control audits. This section documents managing ownership (`chown`/`chgrp`) and access modes (`chmod`).

### 3.1 Initial State Check
Before modification, checking the default permissions on the target file/directory.

```bash
$ ls -l testuser
-rwxr-xr-x. 1 user100 blu3sky 0 Feb 13 22:09 testuser
```
### 3.2 Changing Ownership and Group (chown / chgrp)
We demonstrate changing the owner, the group, and setting both recursively (-R).

#### Changing Owner (File):
```bash
$  sudo  chown newexist testuser
$ ls -l  testuser
-rw-r--r--. 1 newexist blu3sky 0 Feb 13 22:09 testuser
```
#### Changing Group (File):
```bash
$ sudo chgrp user200 testuser
ls -l  testuser
-rw-r--r--. 1 newexist user200 0 Feb 13 22:09 testuser
```
#### Changing Owner and Group Recursively (Directory):
```bash 
$ sudo chown -R user100:blu3sky playground
$ ls -ld playground
drwxr-xr-x. 2 user100 blu3sky 32 Feb 5 11:26 playground
```
#### Changing Owner and Group Recursively (File):
```bash
$ sudo chown  user100:blu3sky testuser
$ ls -l  testuser
-rwxr-xr-x. 1 user100 blu3sky 0 Feb 13 22:09 testuser
```
###  3.3 Modifying Permissions (chmod)
Demonstrate setting explicit permissions using both octal and symbolic modes

#### Symbolic Method (Adding Permissions):

```bash 
$ sudo chmod +x testuser
$ ls -l  testuser
-rwxr-xr-x. 1 newexist user200 0 Feb 13 22:09 testuser
```

#### Octal Method: 
```bash
$  sudo chmod 777 testuser
$ ls -l  testuser
-rwxrwxrwx. 1 newexist user200 0 Feb 13 22:09 testuser
```
**Note:** Differences between `usermod -aG` `usermod -g` and  `groupmod -og`
* `usermod -aG`: Adds a user to an additional group for extra permissions; does not change their primary GID.

* `usermod -g`: Changes a user's primary group; this changes their GID and the default group for new files.

* `groupmod -g`: Modifies the group itself by changing its numerical GID system-wide; this is a rare and risky operation.
