# OpenSSH Service Management

**Author:** Usman O. Olanrewaju (Blu3 Sky)

**Date:** 2026-05-25

**Focus:** OpenSSH service configuration, key-based authentication, SFTP, and PermitRootLogin control

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Server Configuration File](#3-server-configuration-file)
3. [Client Configuration File](#4-client-configuration-file)
4. [OpenSSH Packages](#2-openssh-packages)
5. [Server Setup](#5-server-setup)
6. [Client Setup and Initial Connection](#6-client-setup-and-initial-connection)
7. [Multi-User Access](#7-multi-user-access)
8. [SFTP File Transfer](#8-sftp-file-transfer)
9. [SSH Key Management](#9-ssh-key-management)
10. [Troubleshooting — authorized_keys Permission Repair](#10-troubleshooting--authorized_keys-permission-repair)
11. [PermitRootLogin Configuration](#11-permitrootlogin-configuration)
12. [Key Takeaways](#12-key-takeaways)

---


## 1. Core Concepts

### 1.1 The OpenSSH Service

Secure Shell (SSH) is a protocol that delivers a secure mechanism for data transmission between source and destination systems over IP networks. It was designed to replace legacy remote login and file transfer programs — *telnet*, *rsh*, and *ftp* — which transmitted user credentials in clear text and did not encrypt data in transit.

| Term | Definition |
|---|---|
| **SSH Server** | The machine running `sshd`, accepting and handling incoming connections. |
| **SSH Client** | The machine initiating the connection using the `ssh` or `sftp` command. |
| **MACs** | Message Authentication Codes — used by SSH to verify data integrity throughout a session. |

### 1.2 Encryption Techniques

OpenSSH uses a combination of symmetric and asymmetric encryption, also referred to as *secret key encryption* and *public key encryption* respectively.

**Symmetric Technique**
Uses a single shared secret called the *session key* to encrypt and decrypt data. This key is generated dynamically during the initial key exchange when the SSH client and server establish a connection. Because symmetric encryption is computationally efficient, it handles the bulk of data transferred during an SSH session.

**Asymmetric Technique**
Uses a *public key* and a *private key* that are mathematically related. The public key is shared freely; the private key must remain confidential. In SSH, asymmetric encryption is used primarily during key exchange and user authentication. Data encrypted with a public key can only be decrypted using the corresponding private key, allowing secure identity verification without transmitting sensitive information such as passwords.

### 1.3 Authentication Methods

OpenSSH supports five authentication methods:

| Method | Description |
|---|---|
| **Password-Based** | Server checks the password against `/etc/shadow`. Enabled by default; least secure. |
| **Private/Public Key-Based** | Client holds a private key; server stores the matching public key. No password transmitted. Preferred method. |
| **Host-Based** | Trust established via `~/.shosts` or `/etc/ssh/shosts.equiv`. Disabled by default in RHEL due to security concerns. |
| **Challenge-Response** | User must answer authentication challenges; used with OTP or MFA systems. |
| **GSSAPI-Based** | Generic Security Services API, typically for Kerberos integration. Beyond RHCSA scope. |

The public/private key method is the preferred and recommended approach and is the focus of this lab.

---

## 2. Server Configuration File
The server configuration file is `/etc/ssh/sshd_config`. It controls how the `sshd` daemon behaves. Authentication events are logged to `/var/log/secure`.

Key directives:

| Directive | Default | Description |
|---|---|---|
| `Port` | 22 | Port number `sshd` listens on |
| `ListenAddress` | All local addresses | Restricts which local addresses `sshd` listens on |
| `SyslogFacility` | AUTH | Facility code used when logging to `/var/log/secure` |
| `LogLevel` | INFO | Criticality level for log messages |
| `PermitRootLogin` | prohibit-password | Controls whether root can log in directly |
| `PubKeyAuthentication` | yes | Enables or disables public key-based authentication |
| `AuthorizedKeysFile` | ~/.ssh/authorized_keys | Location of the authorized keys file |
| `PasswordAuthentication` | yes | Enables or disables local password authentication |

> Linux supports drop-in configuration files under `/etc/ssh/sshd_config.d/`. Files placed there are automatically included and take precedence over matching directives in the main `sshd_config`. Always check drop-ins when a directive does not behave as expected.

### 2.1 View the server configuration file

```bash 
$ sudo cat /etc/ssh/sshd_config
[sudo] password for blu3sky: 
#	$OpenBSD: sshd_config,v 1.104 2021/07/02 05:11:21 dtucker Exp $

# This is the sshd server system-wide configuration file.  See
# sshd_config(5) for more information.

# This sshd was compiled with PATH=/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin

# The strategy used for options in the default sshd_config shipped with
# OpenSSH is to specify options with their default value where
# possible, but leave them commented.  Uncommented options override the
# default value.

# To modify the system-wide sshd configuration, create a  *.conf  file under
#  /etc/ssh/sshd_config.d/  which will be automatically included below
Include /etc/ssh/sshd_config.d/*.conf

# If you want to change the port on a SELinux system, you have to tell
# SELinux about this change.
# semanage port -a -t ssh_port_t -p tcp #PORTNUMBER
#
#Port 22
#AddressFamily any
#ListenAddress 0.0.0.0
#ListenAddress ::

#HostKey /etc/ssh/ssh_host_rsa_key
#HostKey /etc/ssh/ssh_host_ecdsa_key
#HostKey /etc/ssh/ssh_host_ed25519_key

# Ciphers and keying
#RekeyLimit default none

# Logging
#SyslogFacility AUTH
#LogLevel INFO

# Authentication:

#LoginGraceTime 2m
PermitRootLogin  no 
#StrictModes yes
#MaxAuthTries 6
#MaxSessions 10

#PubkeyAuthentication yes

# The default is to check both .ssh/authorized_keys and .ssh/authorized_keys2
# but this is overridden so installations will only check .ssh/authorized_keys
AuthorizedKeysFile	.ssh/authorized_keys

#AuthorizedPrincipalsFile none

#AuthorizedKeysCommand none
#AuthorizedKeysCommandUser nobody

# For this to work you will also need host keys in /etc/ssh/ssh_known_hosts
#HostbasedAuthentication no
# Change to yes if you don't trust ~/.ssh/known_hosts for
# HostbasedAuthentication
#IgnoreUserKnownHosts no
# Don't read the user's ~/.rhosts and ~/.shosts files
#IgnoreRhosts yes

# To disable tunneled clear text passwords, change to no here!
#PasswordAuthentication yes
#PermitEmptyPasswords no

# Change to no to disable s/key passwords
#KbdInteractiveAuthentication yes

# Kerberos options
#KerberosAuthentication no
#KerberosOrLocalPasswd yes
#KerberosTicketCleanup yes
#KerberosGetAFSToken no
#KerberosUseKuserok yes

# GSSAPI options
#GSSAPIAuthentication no
#GSSAPICleanupCredentials yes
#GSSAPIStrictAcceptorCheck yes
#GSSAPIKeyExchange no
#GSSAPIEnablek5users no

# Set this to 'yes' to enable PAM authentication, account processing,
# and session processing. If this is enabled, PAM authentication will
# be allowed through the KbdInteractiveAuthentication and
# PasswordAuthentication.  Depending on your PAM configuration,
# PAM authentication via KbdInteractiveAuthentication may bypass
# the setting of "PermitRootLogin prohibit-password".
# If you just want the PAM account and session checks to run without
# PAM authentication, then enable this but set PasswordAuthentication
# and KbdInteractiveAuthentication to 'no'.
# WARNING: 'UsePAM no' is not supported in this build and may cause several
# problems.
#UsePAM no

#AllowAgentForwarding yes
#AllowTcpForwarding yes
#GatewayPorts no
#X11Forwarding no
#X11DisplayOffset 10
#X11UseLocalhost yes
#PermitTTY yes
#PrintMotd yes
#PrintLastLog yes
#TCPKeepAlive yes
#PermitUserEnvironment no
#Compression delayed
#ClientAliveInterval 0
#ClientAliveCountMax 3
#UseDNS no
#PidFile /var/run/sshd.pid
#MaxStartups 10:30:100
#PermitTunnel no
#ChrootDirectory none
#VersionAddendum none

# no default banner path
#Banner none

# override default of no subsystems
Subsystem	sftp	/usr/libexec/openssh/sftp-server

# Example of overriding settings on a per-user basis
#Match User anoncvs
#	X11Forwarding no
#	AllowTcpForwarding no
#	PermitTTY no
#	ForceCommand cvs server
``` 

## 3. Client Configuration File

The client configuration file is `/etc/ssh/ssh_config`. It controls outbound SSH connections. Users can also define per-user settings in `~/.ssh/config`.

Key directives:

| Directive | Default | Description |
|---|---|---|
| `Host` | * | Container for directives applying to a host or group of hosts. Default `*` applies globally. |
| `ForwardX11` | no | Enables or disables automatic X11 traffic redirection over SSH |
| `PasswordAuthentication` | yes | Enables or disables password-based login |
| `StrictHostKeyChecking` | ask | Controls whether new hosts are added to `known_hosts` and how key mismatches are handled |
| `IdentityFile` | ~/.ssh/id_ed25519 | Private key file used for authentication |
| `Port` | 22 | Port number to connect on |

The `~/.ssh` directory does not exist by default. It is created when the user first runs `ssh-keygen` or accepts a remote server's host key for the first time. On first connection, the server's host key is stored in `~/.ssh/known_hosts` and verified on all subsequent connections.

### 3.1 View the client configuration file
```bash
$ sudo cat /etc/ssh/ssh_config
#	$OpenBSD: ssh_config,v 1.36 2023/08/02 23:04:38 djm Exp $

# This is the ssh client system-wide configuration file.  See
# ssh_config(5) for more information.  This file provides defaults for
# users, and the values can be changed in per-user configuration files
# or on the command line.

# Configuration data is parsed as follows:
#  1. command line options
#  2. user-specific file
#  3. system-wide file
# Any configuration value is only changed the first time it is set.
# Thus, host-specific definitions should be at the beginning of the
# configuration file, and defaults at the end.

# Site-wide defaults for some commonly used options.  For a comprehensive
# list of available options, their meanings and defaults, please see the
# ssh_config(5) man page.

# Host *
#   ForwardAgent no
#   ForwardX11 no
#   PasswordAuthentication yes
#   HostbasedAuthentication no
#   GSSAPIAuthentication no
#   GSSAPIDelegateCredentials no
#   GSSAPIKeyExchange no
#   GSSAPITrustDNS no
#   BatchMode no
#   CheckHostIP no
#   AddressFamily any
#   ConnectTimeout 0
#   StrictHostKeyChecking ask
#   IdentityFile ~/.ssh/id_rsa
#   IdentityFile ~/.ssh/id_dsa
#   IdentityFile ~/.ssh/id_ecdsa
#   IdentityFile ~/.ssh/id_ed25519
#   Port 22
#   Ciphers aes128-ctr,aes192-ctr,aes256-ctr,aes128-cbc,3des-cbc
#   MACs hmac-md5,hmac-sha1,umac-64@openssh.com
#   EscapeChar ~
#   Tunnel no
#   TunnelDevice any:any
#   PermitLocalCommand no
#   VisualHostKey no
#   ProxyCommand ssh -q -W %h:%p gateway.example.com
#   RekeyLimit 1G 1h
#   UserKnownHostsFile ~/.ssh/known_hosts.d/%k
#
# This system is following system-wide crypto policy.
# To modify the crypto properties (Ciphers, MACs, ...), create a  *.conf
#  file under  /etc/ssh/ssh_config.d/  which will be automatically
# included below. For more information, see manual page for
#  update-crypto-policies(8)  and  ssh_config(5).
Include /etc/ssh/ssh_config.d/*.conf
```

### 4. OpenSSH Packages

### 4.1 Verify the OpenSSH package
```bash
$ sudo dnf install openssh
Updating Subscription Management repositories.
Last metadata expiration check: 20:24:21 ago on Sun 24 May 2026 09:53:22 AM EDT.
Package openssh-9.9p1-23.el10_2.x86_64 is already installed.
Dependencies resolved.
Nothing to do.
Complete!
```

> In RHEL, `openssh` is installed by default. The package includes both the server daemon (`sshd`) and the client utilities (`ssh`, `sftp`, `ssh-keygen`, `ssh-copy-id`).


## 5. Server Setup

All steps in this section are performed on the **SSH server** machine.

### 5.1 Enable and start the sshd daemon
```bash
$ sudo  systemctl enable sshd; sudo  systemctl start sshd
```
 
### 5.2 Verify the daemon is active
```bash
$ sudo systemctl status sshd
● sshd.service - OpenSSH server daemon
     Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-05-24 11:25:43 EDT; 19h ago
 Invocation: 3c1486a84f9b427aa92416ac50bd0652
       Docs: man:sshd(8)
             man:sshd_config(5)
   Main PID: 1531 (sshd)
      Tasks: 1 (limit: 22950)
     Memory: 3.3M (peak: 23.1M)
        CPU: 287ms
     CGroup: /system.slice/sshd.service
             └─1531 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"

May 24 11:51:37 server40.usman.com sshd-session[10152]: PAM 2 more authentication failures; logname= uid=0 euid=0 tty>
May 24 11:52:15 server40.usman.com systemd[1]: Reloading sshd.service - OpenSSH server daemon...
May 24 11:52:15 server40.usman.com sshd[1531]: Received SIGHUP; restarting.
May 24 11:52:15 server40.usman.com sshd[1531]: Server listening on 0.0.0.0 port 22.
May 24 11:52:15 server40.usman.com sshd[1531]: Server listening on :: port 22.
May 24 11:52:15 server40.usman.com systemd[1]: Reloaded sshd.service - OpenSSH server daemon.
May 24 11:52:34 server40.usman.com sshd-session[10396]: Accepted password for root from 192.168.122.241 port 60426 ss>
May 24 11:52:34 server40.usman.com sshd-session[10396]: pam_unix(sshd:session): session opened for user root(uid=0) b>
May 24 12:02:13 server40.usman.com sshd-session[11004]: Accepted password for blu3sky from 192.168.122.241 port 35738>
```
### 5.3 Get the server IP address

```bash
$ ip route show
default via 192.168.122.1 dev enp1s0 proto dhcp src 192.168.122.88 metric 100 
192.168.122.0/24 dev enp1s0 proto kernel scope link src 192.168.122.88 metric 100 
``` 
### 5.4 Check active user
```bash
$ whoami
blu3sky
```
### 5.5 Check active sessions
```bash
$ who
blu3sky  tty2         2026-05-25 05:37 (local)
``` 


## 6. Client Setup and Initial Connection

The SSH client is enabled by default on RHEL — no additional setup is required on the client side.

### 6.1 Verify network connectivity to the server
```bash 
$ ping 192.168.122.88 
PING 192.168.122.88 (192.168.122.88) 56(84) bytes of data.
64 bytes from 192.168.122.88: icmp_seq=1 ttl=64 time=0.408 ms
64 bytes from 192.168.122.88: icmp_seq=2 ttl=64 time=0.393 ms
64 bytes from 192.168.122.88: icmp_seq=3 ttl=64 time=0.482 ms
64 bytes from 192.168.122.88: icmp_seq=4 ttl=64 time=0.472 ms
^C
--- 192.168.122.88 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3053ms
rtt min/avg/max/mdev = 0.393/0.438/0.482/0.038 ms
``` 

### 6.2 Check active user
```bash 
$ whoami
blue
``` 
### 6.2 Connect from client to server
```bash 
$ ssh blu3sky@192.168.122.88  
The authenticity of host '192.168.122.88 (192.168.122.88)' can't be established.
ED25519 key fingerprint is SHA256:SteWT6i/EuAZLfl/oIhwtybyeBz5D91jK9sBYs3KPsA.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:1: server40
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.122.88' (ED25519) to the list of known hosts.
blu3sky@192.168.122.88's password: 
Web console: https://server40.usman.com:9090/ or https://192.168.122.88:9090/

Register this system with Red Hat Lightspeed: rhc connect

Example:
# rhc connect --activation-key <key> --organization <org>

The rhc client and Red Hat Lightspeed will enable analytics and additional
management capabilities on your system.
View your connected systems at https://console.redhat.com/insights

You can learn more about how to register your system 
using rhc at https://red.ht/registration
Last login: Mon May 25 05:37:41 2026
blu3sky@server40:~$   # --> you have the login prompt
```

### 6.3  Check active user after connection 
```bash 
$ whoami
blu3sky
```

### 6.4 Verify active sessions on the server
```bash 
$ who
blu3sky  tty2         2026-05-25 05:37 (local)
blu3sky  pts/2        2026-05-25 06:48 (192.168.122.206)
```

### 6.5 Log out
```bash 
$ exit
logout
Connection to 192.168.122.88 closed.
``` 

The roles can be reversed — the server can connect back to the client. SSH is bidirectional.




## 7. Multi-User Access

### 7.1 Add a test user on the server
 
```bash 
$ sudo useradd usman12

$ echo user123 | sudo  passwd --stdin usman12 
BAD PASSWORD: The password is shorter than 8 characters
```


### 7.2 Connect from the client as the new user
```bash 
$ ssh usman12@192.168.122.88  
usman12@192.168.122.88's password: 
Register this system with Red Hat Lightspeed: rhc connect

Example:
# rhc connect --activation-key <key> --organization <org>

The rhc client and Red Hat Lightspeed will enable analytics and additional
management capabilities on your system.
View your connected systems at https://console.redhat.com/insights

You can learn more about how to register your system 
using rhc at https://red.ht/registration
usman12@server40:~$ # --> access gained
```
### 7.3 Check active user after connection on client 
```bash 
$ whoami
usman12
``` 

### 7.4 Verify active sessions on the server

```bash 
$ who
blu3sky  tty2         2026-05-25 05:37 (local)
usman12  pts/2        2026-05-25 07:02 (192.168.122.206)
```

### 7.5  Verify via the security log
```bash

$ sudo tail -5 /var/log/secure
May 25 07:07:26 server40 sudo[26719]: pam_unix(sudo:session): session opened for user root(uid=0) by blu3sky(uid=1000)
May 25 07:07:26 server40 sudo[26719]: pam_unix(sudo:session): session closed for user root
May 25 07:07:46 server40 sshd-session[25285]: Received disconnect from 192.168.122.206 port 46718:11: disconnected by user
May 25 07:07:46 server40 sshd-session[25285]: Disconnected from user usman12 192.168.122.206 port 46718
May 25 07:07:46 server40 sshd-session[25260]: pam_unix(sshd:session): session closed for user usman12
```

### 7.5 Connect as root
```bash 
$ ssh root@192.168.122.88  
root@192.168.122.88's password: 
Web console: https://server40.usman.com:9090/ or https://192.168.122.88:9090/

Register this system with Red Hat Lightspeed: rhc connect

Example:
# rhc connect --activation-key <key> --organization <org>

The rhc client and Red Hat Lightspeed will enable analytics and additional
management capabilities on your system.
View your connected systems at https://console.redhat.com/insights

You can learn more about how to register your system 
using rhc at https://red.ht/registration
Last login: Sun May 24 11:52:34 2026 from 192.168.122.241
root@server40:~# 
``` 
### 7.6 Logout
```bash
# exit
logout
Connection to 192.168.122.88 closed.
```
### 7.7 Verify root session in the security log 
```bash
$ sudo tail -5 /var/log/secure
May 25 07:09:40 server40 (systemd)[27234]: pam_unix(systemd-user:session): session opened for user root(uid=0) by root(uid=0)
May 25 07:09:40 server40 sshd-session[27228]: pam_unix(sshd:session): session opened for user root(uid=0) by root(uid=0)
May 25 07:11:50 server40 sshd-session[27252]: Received disconnect from 192.168.122.206 port 45496:11: disconnected by user
May 25 07:11:50 server40 sshd-session[27252]: Disconnected from user root 192.168.122.206 port 45496
May 25 07:11:50 server40 sshd-session[27228]: pam_unix(sshd:session): session closed for user root
``` 

## 8. SFTP File Transfer

`sftp` uses the same authentication mechanism as SSH and provides a secure interactive file transfer interface. No separate package is needed — it is included in `openssh`.

### 8.1 Prepare test files - client

```bash
$ touch usmanclient
``` 
### 8.2  Prepare test files - server
```bash
$ touch usmanserver
``` 
### 8.3 Connect via SFTP from the client
```bash
$ sftp  blu3sky@192.168.122.88  
blu3sky@192.168.122.88's password: 
Connected to 192.168.122.88.
sftp> 
```
 
### 8.4 Upload a file from client to server
```bash
sftp> put usmanclient
Uploading usmanclient to /home/blu3sky/usmanclient
usmanclient                                                                         100%    0     0.0KB/s   00:00    
```
### 8.5  Download a file from server to client
```bash
sftp> get usmanserver 
Fetching /home/blu3sky/usmanserver to usmanserver
``` 

### 8.5 Upload a directory
```bash 
sftp> put -r Templates/
Uploading Templates/ to /home/blu3sky/Templates
Entering Templates/
```

Use `-r` for directories. Use `lcd` to navigate locally and `cd` for the remote directory. Run `?` inside the SFTP session for a full command reference.

 
## 9. SSH Key Management

All key generation steps are performed on the **client** machine.

### 9.1 Generate a key pair
```bash
$ ssh-keygen -N "" 
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/blue/.ssh/id_ed25519): 
Your identification has been saved in /home/blue/.ssh/id_ed25519
Your public key has been saved in /home/blue/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:QF7YchTHzcdgk9PHYXtuGwi5k76V1+vin3ZZvBk0fVk blue@server30.usman.com
The key's randomart image is:
+--[ED25519 256]--+
|      .++o.o+= +.|
|     oo.o...*.= E|
|      oo   o o o=|
|       .    + .=+|
|        S  + ..o=|
|          . . .o*|
|           . o oB|
|            o..=+|
|           ...==.|
+----[SHA256]-----+
```
> `-N ""` sets an empty passphrase, enabling fully automated passwordless login. In production, a passphrase should be set to protect the private key on disk.
 
### 9.2 Distribute the public key to the server
```bash
$ ssh-copy-id blu3sky@192.168.122.88
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: ssh-add -L
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
blu3sky@192.168.122.88's password:

Number of key(s) added: 1

Now try logging into the machine, with: "ssh 'blu3sky@192.168.122.88'"
and check to make sure that only the key(s) you wanted were added.
```
> `ssh-copy-id` appends the client's public key to the server user's `~/.ssh/authorized_keys` file. The password prompt here is a one-time requirement for the distribution step only.


### 9.3  Verify the generated public key
```bash
$ cat .ssh/id_ed25519.pub 
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJzLTvVYmcT53gD4+wFemVjsOD/U/+mGMbH+wTT95g2y blue@server30.usman.com
``` 
### 9.4 Verify the generated id_ed25519

```bash
$ cat .ssh/id_ed25519
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACCcy071WJnE+d4A+PsBXplY7Dg/1P/phjGx/sE0/eYNsgAAAKD6l3wU+pd8
FAAAAAtzc2gtZWQyNTUxOQAAACCcy071WJnE+d4A+PsBXplY7Dg/1P/phjGx/sE0/eYNsg
AAAEDvc5rUX2svls2TpUECH12b53aNEjWcMJVf9dAbgLQwD5zLTvVYmcT53gD4+wFemVjs
OD/U/+mGMbH+wTT95g2yAAAAF2JsdWVAc2VydmVyMzAudXNtYW4uY29tAQIDBAUG
-----END OPENSSH PRIVATE KEY-----
```
### 9.5 Verify the generated known_hosts 
```bash
$ cat .ssh/known_hosts
server40 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILdnkjS6DvLj8R4fcbeTj84469Fn6284a5BDOPN0IbHq
server40 ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCmI1RbR/MEqSixiPMORcAMTJBpFmdcUetNr/LhyAF4UlNstzkgVSp9sDVfb8/ULFLQBbHcOFMfs2rqaAYUGDx4w0A+keMaIwNMDZfsgWSw1nsLadqAX2kLZDZLdtMQ+xOOvWsRZj3y0BlLUy0vYNj00GNBuWz746EwJZULhjo0CwGPuiCNOuicS++G5mWCRP26zLkgz9uqh38Lz8dGagTPq8ic25RWn2XZSz/tododQoscZpt2DuxNW1QRewFHpth0+jIASzu/VB679yQal4hc5iZ8VGEbmZ1Rde4UTuXzXfqvqnseh3hS4lPWC3NNHWnCXa/vu55f2W23nzfsyxniT+SUlmsasw9iKMCiw0CJR+ZBnF3rzqxVkCquCzxbMUZUqxY1iTxbLSCjpErDJxCiMu9ZAuX++2VOFQ7VBizzC2S4Tf8zNprUmMr+ViFBy5E01yFVRqbaPmDZfAPcudrpen0uoffZa/JjENNjcy+iucaQalrZno92jjuUzxTnr30=
server40 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBKSk5pqSRz6jeJHz5xHx6fIJhKxhRug+IrAWOGKErSVk7UDAidOqsUZHKw+7gWMC/LqyNplloUrP3bJa3p4rUCg=
192.168.122.88 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILdnkjS6DvLj8R4fcbeTj84469Fn6284a5BDOPN0IbHq
```  

### 9.6 Verify passwordless login

```bash
$ ssh blu3sky@192.168.122.88  
Web console: https://server40.usman.com:9090/ or https://192.168.122.88:9090/

Register this system with Red Hat Lightspeed: rhc connect

Example:
# rhc connect --activation-key <key> --organization <org>

The rhc client and Red Hat Lightspeed will enable analytics and additional
management capabilities on your system.
View your connected systems at https://console.redhat.com/insights

You can learn more about how to register your system 
using rhc at https://red.ht/registration
Last login: Mon May 25 06:48:44 2026 from 192.168.122.206
blu3sky@server40:~$ 
``` 
### 9.7 Logout
```bash
$ exit
logout
Connection to 192.168.122.88 closed.
```

### 9.8 Log into other users - without distributing keys 
```bash
$ ssh usman12@192.168.122.88  
usman12@192.168.122.88's password: 

$ ssh root@192.168.122.88  
root@192.168.122.88's password: 
```

### 9.9 Distribute to additional users
```bash
$ ssh-copy-id usman12@192.168.122.88  
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: ssh-add -L
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
usman12@192.168.122.88's password: 

Number of key(s) added: 1

Now try logging into the machine, with: "ssh 'usman12@192.168.122.88'"
and check to make sure that only the key(s) you wanted were added.
```
### 9.10 Verify passwordless login
```bash
$ ssh usman12@192.168.122.88  
Register this system with Red Hat Lightspeed: rhc connect

Example:
# rhc connect --activation-key <key> --organization <org>

The rhc client and Red Hat Lightspeed will enable analytics and additional
management capabilities on your system.
View your connected systems at https://console.redhat.com/insights

You can learn more about how to register your system 
using rhc at https://red.ht/registration
Last login: Mon May 25 07:02:35 2026 from 192.168.122.206
usman12@server40:~$ 
```

### 9.11 Logout 
```bash
$ exit
logout
Connection to 192.168.122.88 closed.
``` 
### 9.12 Distribute to additional root

```bash 
~$ ssh-copy-id root@192.168.122.88  
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: ssh-add -L
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@192.168.122.88's password: 

Number of key(s) added: 1

Now try logging into the machine, with: "ssh 'root@192.168.122.88'"
and check to make sure that only the key(s) you wanted were added.
```
### 9.13 Verify passwordless login
```bash
$ ssh root@192.168.122.88  
Web console: https://server40.usman.com:9090/ or https://192.168.122.88:9090/

Register this system with Red Hat Lightspeed: rhc connect

Example:
# rhc connect --activation-key <key> --organization <org>

The rhc client and Red Hat Lightspeed will enable analytics and additional
management capabilities on your system.
View your connected systems at https://console.redhat.com/insights

You can learn more about how to register your system 
using rhc at https://red.ht/registration
Last login: Mon May 25 07:09:40 2026 from 192.168.122.206
root@server40:~# 

```
### 9.14 Logout
```bash
# exit
logout
Connection to 192.168.122.88 closed.
``` 

## 10. Troubleshooting — authorized_keys Permission Repair

### 10.1 Symptom

After key distribution, SSH still prompts for a password:
```bash
$ ssh blu3sky@192.168.122.88  
blu3sky@192.168.122.88's password: 
``` 

### 10.2 Diagnose — check permissions on the server
```bash
$ ll
total 12
-rwxrwxrwx. 1 blu3sky 2000  105 May 25 07:35 authorized_keys  # --> the authrozited key has been tempered
-rw-------. 1 blu3sky 2000 1665 May 25 07:53 known_hosts
-rw-------. 1 blu3sky 2000  919 May 25 07:52 known_hosts.old
```
> `authorized_keys` has `777` permissions — world-writable. SSH enforces strict permission checks and silently ignores the file if it is writable by group or others, falling back to password authentication without any error message.

### 10.3 Fix — run on the server

#### 10.3.1 Restrict the .ssh directory
```bash
$ chmod 500 .ssh/
```
#### 10.3.2 Restrict the authorized_keys file
```bash
$ chmod 500 authorized_keys 
```
#### 10.3.3 Restore the SELinux security context
```bash
$ restorecon -Rv ~/.ssh 
```
#### 10.3.3  Reload daemon
```bash
$ sudo systemctl reload sshd
[sudo] password for blu3sky: 
``` 

### 10.4 Verify from the client
```bash
$ ssh blu3sky@192.168.122.88  
Web console: https://server40.usman.com:9090/ or https://192.168.122.88:9090/

Register this system with Red Hat Lightspeed: rhc connect

Example:
# rhc connect --activation-key <key> --organization <org>

The rhc client and Red Hat Lightspeed will enable analytics and additional
management capabilities on your system.
View your connected systems at https://console.redhat.com/insights

You can learn more about how to register your system 
using rhc at https://red.ht/registration
Last login: Mon May 25 07:36:40 2026 from 192.168.122.206
blu3sky@server40:~$ 

```
> Passwordless login is restored
### 10.5  Logout

```bash
blu3sky@server40:~$ exit
logout
Connection to 192.168.122.88 closed.
blue@server30:~$ 

```
## 11. PermitRootLogin Configuration - server 

### 11.1 Locate the active directive

Use `grep -rin` to search across all SSH configuration files
```bash 
$ sudo grep -rin "Permit"  /etc/ssh/
[sudo] password for blu3sky: 
/etc/ssh/ssh_config:44:#   PermitLocalCommand no
/etc/ssh/sshd_config.d/01-permitrootlogin.conf:3:PermitRootLogin yes
/etc/ssh/sshd_config:40:PermitRootLogin  no 
/etc/ssh/sshd_config:66:#PermitEmptyPasswords no
/etc/ssh/sshd_config:90:# the setting of "PermitRootLogin prohibit-password".
/etc/ssh/sshd_config:104:#PermitTTY yes
/etc/ssh/sshd_config:108:#PermitUserEnvironment no
/etc/ssh/sshd_config:115:#PermitTunnel no
/etc/ssh/sshd_config:129:#	PermitTTY no
```
The drop-in `sshd_config.d/01-permitrootlogin.conf` sets `PermitRootLogin yes` and takes precedence over the `no` in `sshd_config`. This is why root login works even when the main config appears to disable it. Always audit drop-in files first when a directive behaves unexpectedly — use `grep -rin` to search across all files at once.

### 11.2 Disable root login

```bash
$ sudo vi /etc/ssh/sshd_config.d/01-permitrootlogin.conf 
```
change: `yes` to `no`
### 11.3 Reload daemon 
```bash
$ sudo systemctl reload sshd
```
### 11.4 Attempt root login from the client
```bash
$ ssh root@192.168.122.88  
root@192.168.122.88's password: 
Permission denied, please try again.
root@192.168.122.88's password: 
Permission denied, please try again.
root@192.168.122.88's password: 
root@192.168.122.88: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).
```
> Root login is blocked — even with the correct password and a previously distributed key.

### 11.5 Behavior with an invalid directive value
```bash
$ sudo vi /etc/ssh/sshd_config.d/01-permitrootlogin.conf 
```
leave the directive with no value:
```
PermitRootLogin
```

#### 11.5.1 Reload daemon 
```bash 
$ sudo systemctl reload sshd
```

#### 11.5.2 Attempting root user
```bash
$ ssh root@192.168.122.88  
ssh: connect to host 192.168.122.88 port 22: Connection refused
```

A `PermitRootLogin` directive with no value causes `sshd` to treat the configuration as invalid on reload, resulting in the daemon refusing all incoming connections — not just root ones. This is a host-wide outage affecting every user.


### 11.6 Reset root login
```bash
$ sudo vi /etc/ssh/sshd_config.d/01-permitrootlogin.conf 
```
Restore to:
```
PermitRootLogin yes
```
### 11.7 Reload the daemon
```bash
$ sudo systemctl reload sshd
```

### 11.8 Attempting Login
```bash
$ ssh root@192.168.122.88  
Web console: https://server40.usman.com:9090/ or https://192.168.122.88:9090/

Register this system with Red Hat Lightspeed: rhc connect

Example:
# rhc connect --activation-key <key> --organization <org>

The rhc client and Red Hat Lightspeed will enable analytics and additional
management capabilities on your system.
View your connected systems at https://console.redhat.com/insights

You can learn more about how to register your system 
using rhc at https://red.ht/registration
Last failed login: Mon May 25 08:15:17 EDT 2026 from 192.168.122.206 on ssh:notty
There were 3 failed login attempts since the last successful login.                # --> our previous 
Last login: Mon May 25 07:56:56 2026 from 192.168.122.206
root@server40:~# 

```
> Root login is restored. The login banner shows the failed attempts from testing above.


## 12. Key Takeaways

**OpenSSH:**
- `openssh` is installed by default on RHEL. No manual installation needed for basic SSH access.
- `sshd_config` controls the server; `ssh_config` controls the client. Both support drop-in files under `.d/` directories.
- Always use `grep -rin "directive" /etc/ssh/` to locate the active value — a directive may be overridden by a drop-in without any visible warning.
- Three firewall services are required for full SSH access in restrictive environments: `ssh`, and any additional ports if the default port is changed.

**Key-Based Authentication:**
- Generate with `ssh-keygen -N ""` for no passphrase (lab only). Always set a passphrase in production.
- Distribute with `ssh-copy-id user@host` — this appends the public key to `~/.ssh/authorized_keys` on the server.
- `ssh-copy-id` reads from the SSH agent via `ssh-add -L`. Verify with `ssh-add -L` before distributing.
- If passwordless login fails silently after `ssh-copy-id`, check `~/.ssh/` and `authorized_keys` permissions. Both must not be world-writable. Fix with `chmod 500` and `restorecon -Rv ~/.ssh`.

**PermitRootLogin:**
- RHEL ships with `/etc/ssh/sshd_config.d/01-permitrootlogin.conf` setting `PermitRootLogin yes`. This overrides any `no` in the main config.
- Setting `PermitRootLogin no` blocks root — even with key-based auth already distributed.
**SFTP:**
- `sftp` requires no additional packages. It uses the same authentication as SSH.
- Use `put` to upload, `get` to download, `put -r` for directories.
- Use `lcd` to navigate locally and `cd` for the remote directory inside the SFTP session.

**Hostname aliasing:**
- Add `server-ip hostname` to `/etc/hosts` on the client to use hostnames instead of IPs: `ssh user@server40` instead of `ssh user@192.168.122.88`.




