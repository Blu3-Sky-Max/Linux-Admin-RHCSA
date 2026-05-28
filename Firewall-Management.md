# Firewall management 


**Author:** Usman O. Olanrewaju (Blu3 Sky)

**Created:** 2026/05/27

**Focus:** firewalld zone and service management, port rules, rich rules, custom zone creation, and port forwarding via `firewall-cmd`

---
## Table of Contents

1. [Concepts](#1-concepts)
2. [Firewall Zones](#2-firewall-zones)
   - 2.1 [Predefined Zone Descriptions](#21-predefined-zone-descriptions)
   - 2.2 [System Zone Files](#22-system-zone-files)
   - 2.3 [User Zone Files](#23-user-zone-files)
3. [Service Configuration](#3-service-configuration)
   - 3.1 [System Service Files](#31-system-service-files)
   - 3.2 [SSH Service XML](#32-ssh-service-xml)
   - 3.3 [DHCPv6 Service XML](#33-dhcpv6-service-xml)
4. [Firewall Management with firewall-cmd](#4-firewall-management-with-firewall-cmd)
   - 4.1 [Checking firewalld State](#41-checking-firewalld-state)
   - 4.2 [Verifying with systemctl](#42-verifying-with-systemctl)
5. [Adding Services](#5-adding-services)
   - 5.1 [Getting the Default Zone](#51-getting-the-default-zone)
   - 5.2 [Initial Services Allowed](#52-initial-services-allowed)
   - 5.3 [Adding HTTPS at Runtime](#53-adding-https-at-runtime)
   - 5.4 [Making a Service Persistent](#54-making-a-service-persistent)
   - 5.5 [Verifying via Zone XML](#55-verifying-via-zone-xml)
6. [Adding Ports and Managing Zones](#6-adding-ports-and-managing-zones)
   - 6.1 [Initial Port State](#61-initial-port-state)
   - 6.2 [Adding a Runtime TCP Port](#62-adding-a-runtime-tcp-port)
   - 6.3 [Adding a Permanent TCP Port Range to a Different Zone](#63-adding-a-permanent-tcp-port-range-to-a-different-zone)
   - 6.4 [Switching the Default Zone](#64-switching-the-default-zone)
   - 6.5 [Adding a Permanent UDP Port Range](#65-adding-a-permanent-udp-port-range)
7. [Removing Ports and Services](#7-removing-ports-and-services)
   - 7.1 [Removing a Port from the Internal Zone](#71-removing-a-port-from-the-internal-zone)
   - 7.2 [Removing a Service from the Public Zone](#72-removing-a-service-from-the-public-zone)
8. [Live Firewall Effect Demonstration](#8-live-firewall-effect-demonstration)
   - 8.1 [Server Setup and Removing SSH at Runtime](#81-server-setup-and-removing-ssh-at-runtime)
   - 8.2 [SSH Connection Blocked from Client](#82-ssh-connection-blocked-from-client)
   - 8.3 [Restoring SSH Access](#83-restoring-ssh-access)
9. [Rich Rules](#9-rich-rules)
   - 9.1 [Getting the Client IP](#91-getting-the-client-ip)
   - 9.2 [Adding a MySQL Rich Rule](#92-adding-a-mysql-rich-rule)
   - 9.3 [Tearing Down the Rich Rule](#93-tearing-down-the-rich-rule)
10. [Creating a Custom Firewall Zone](#10-creating-a-custom-firewall-zone)
    - 10.1 [Listing Initial Zones](#101-listing-initial-zones)
    - 10.2 [Reviewing the Zone Template](#102-reviewing-the-zone-template)
    - 10.3 [Creating the Custom Zone File](#103-creating-the-custom-zone-file)
    - 10.4 [Reloading and Verifying the New Zone](#104-reloading-and-verifying-the-new-zone)
    - 10.5 [Managing Services and Ports in the Custom Zone](#105-managing-services-and-ports-in-the-custom-zone)
    - 10.6 [Adding a Rich Rule to the Custom Zone](#106-adding-a-rich-rule-to-the-custom-zone)
    - 10.7 [Binding an Interface to the Custom Zone](#107-binding-an-interface-to-the-custom-zone)
    - 10.8 [Changing the Zone Target via XML](#108-changing-the-zone-target-via-xml)
    - 10.9 [Tearing Down the Custom Zone](#109-tearing-down-the-custom-zone)
11. [Port Forwarding](#11-port-forwarding)
    - 11.1 [Forwarding Port 8888 to 443](#111-forwarding-port-8888-to-443)
12. [Key Takeaways](#12-key-takeaways)





--- 


## 1. Concepts

A firewall is a security system — implemented in hardware, software, or both — that monitors and controls network traffic based on predefined rules. There are two main categories: **network firewalls**, which protect entire networks and are deployed at the perimeter, and **host-based firewalls**, which run directly on individual servers to protect local traffic.

Port numbers are standardized identifiers used by network services to communicate. In RHEL, common port assignments are listed in `/etc/services`. Key examples:

| Service | Protocol | Port |
|---------|----------|------|
| FTP     | TCP      | 21   |
| SSH     | TCP      | 22   |
| Postfix | TCP      | 25   |
| HTTP    | TCP      | 80   |
| NTP     | UDP      | 123  |

RHEL's host-based firewall is built on the **netfilter** framework inside the Linux kernel, which provides packet filtering, Network Address Translation (NAT), and packet classification. The modern user-space interface is **nftables**, which replaces the legacy `iptables`. Together, netfilter and nftables inspect, modify, drop, forward, and enforce packets based on defined rule sets.

RHEL manages all of this through the **firewalld** service, which dynamically applies nftables rules via the `firewall-cmd` command — no service restart required for rule changes.

---

## 2. Firewall Zones

`firewalld` uses the concept of *zones* for transparent traffic management. Each zone defines policies based on the trust level of a network connection and its source IP address. A network connection belongs to only one zone at a time, but a zone can hold multiple connections. Zone configuration covers services, ports, protocols, masquerading, port forwarding, NAT, ICMP filters, and rich language rules.

When `firewalld` receives an incoming packet, it checks for a zone with a matching source IP. If none matches, it falls back to the zone holding the packet's network interface. If that also fails, it enforces the rules of the **default zone**. By default, outgoing traffic is allowed for all predefined zones unless restricted by a specific rule.

### 2.1 Predefined Zone Descriptions

`firewalld` ships with the following predefined zones, ordered from most to least trusted:

| Zone     | Description |
|----------|-------------|
| trusted  | Allow all incoming traffic |
| internal | Reject all except allowed. Intended for internal networks |
| home     | Reject all except allowed. Intended for home networks |
| work     | Reject all except allowed. Intended for workplaces |
| dmz      | Reject all except allowed. Intended for publicly accessible demilitarized zones |
| external | Reject all except allowed. Outgoing IPv4 traffic is masqueraded to appear as the interface's IP. Intended for external networks |
| public   | Reject all except allowed. Default zone for newly added interfaces. Intended for public places |
| block    | Reject all incoming traffic; an `icmp-host-prohibited` message is returned to the sender |
| drop     | Drop all incoming traffic silently — no ICMP errors returned |

### 2.2 System Zone Files

Zone rules are stored as XML at two locations. System-defined templates live under `/usr/lib/firewalld/zones/` and serve as read-only references. When modified through a management tool, they are automatically copied to `/etc/firewalld/zones/`.

```bash
$ sudo ls -l /usr/lib/firewalld/zones
total 40
-rw-r--r--. 1 root root 312 Nov  6  2025 block.xml
-rw-r--r--. 1 root root 306 Nov  6  2025 dmz.xml
-rw-r--r--. 1 root root 304 Nov  6  2025 drop.xml
-rw-r--r--. 1 root root 317 Nov  6  2025 external.xml
-rw-r--r--. 1 root root 410 Nov  6  2025 home.xml
-rw-r--r--. 1 root root 425 Nov  6  2025 internal.xml
-rw-r--r--. 1 root root 729 Feb 12 17:14 nm-shared.xml
-rw-r--r--. 1 root root 356 Nov  6  2025 public.xml
-rw-r--r--. 1 root root 175 Nov  6  2025 trusted.xml
-rw-r--r--. 1 root root 352 Nov  6  2025 work.xml
```


### 2.3 Public/User Zone Files

User-defined or modified zone rules are written to `/etc/firewalld/zones/`. `firewalld` reads from this directory at runtime and applies the rules defined here.

```bash
$ sudo ls -l  /etc/firewalld/zones/
total 16
-rw-r--r--. 1 root root 425 May 27 05:56 internal.xml
-rw-r--r--. 1 root root 467 May 27 05:46 internal.xml.old
-rw-r--r--. 1 root root 380 May 27 05:53 public.xml
-rw-r--r--. 1 root root 405 May 27 05:32 public.xml.old
```

> `.old` files are backups automatically created by `firewalld` before modifying a zone. They are not loaded as active rules.




## 3. Service Configuration

`firewalld` uses *services* to group firewall rules for well-known applications. A service definition bundles the required port number, protocol, and optional kernel helper modules into a single named unit. Services are stored as XML at two locations — system-defined templates in `/usr/lib/firewalld/services/` and user-defined overrides in `/etc/firewalld/services/`.

By default, `firewalld` blocks all incoming traffic unless a service or port is explicitly opened in the active zone.

### 3.1  System Service file 
```bash
$ sudo ls -l  /usr/lib/firewalld/services/
[sudo] password for blue: 
total 1060
-rw-r--r--. 1 root root 433 Nov  7  2025 0-AD.xml
-rw-r--r--. 1 root root 352 Nov  7  2025 afp.xml
-rw-r--r--. 1 root root 381 Nov  7  2025 alvr.xml
-rw-r--r--. 1 root root 399 Nov  7  2025 amanda-client.xml
-rw-r--r--. 1 root root 427 Nov  7  2025 amanda-k5-client.xml
-rw-r--r--. 1 root root 283 Nov  7  2025 amqps.xml
-rw-r--r--. 1 root root 273 Nov  7  2025 amqp.xml
-rw-r--r--. 1 root root 465 Nov  7  2025 anno-1602.xml
-rw-r--r--. 1 root root 431 Nov  7  2025 anno-1800.xml
-rw-r--r--. 1 root root 285 Nov  7  2025 apcupsd.xml
-rw-r--r--. 1 root root 210 Nov  7  2025 aseqnet.xml
-rw-r--r--. 1 root root 301 Nov  7  2025 audit.xml
-rw-r--r--. 1 root root 436 Nov  7  2025 ausweisapp2.xml
-rw-r--r--. 1 root root 320 Nov  7  2025 bacula-client.xml
-rw-r--r--. 1 root root 346 Nov  7  2025 bacula.xml
-rw-r--r--. 1 root root 390 Nov  7  2025 bareos-director.xml
-rw-r--r--. 1 root root 255 Nov  7  2025 bareos-filedaemon.xml
-rw-r--r--. 1 root root 316 Nov  7  2025 bareos-storage.xml
-rw-r--r--. 1 root root 429 Nov  7  2025 bb.xml
-rw-r--r--. 1 root root 339 Nov  7  2025 bgp.xml
-rw-r--r--. 1 root root 275 Nov  7  2025 bitcoin-rpc.xml
-rw-r--r--. 1 root root 307 Nov  7  2025 bitcoin-testnet-rpc.xml
-rw-r--r--. 1 root root 281 Nov  7  2025 bitcoin-testnet.xml
-rw-r--r--. 1 root root 244 Nov  7  2025 bitcoin.xml
-rw-r--r--. 1 root root 410 Nov  7  2025 bittorrent-lsd.xml
-rw-r--r--. 1 root root 222 Nov  7  2025 ceph-exporter.xml
-rw-r--r--. 1 root root 294 Nov  7  2025 ceph-mon.xml
-rw-r--r--. 1 root root 329 Nov  7  2025 ceph.xml
-rw-r--r--. 1 root root 168 Nov  7  2025 cfengine.xml
-rw-r--r--. 1 root root 234 Nov  7  2025 checkmk-agent.xml
-rw-r--r--. 1 root root 755 Nov  7  2025 civilization-iv.xml
-rw-r--r--. 1 root root 904 Nov  7  2025 civilization-v.xml
-rw-r--r--. 1 root root 211 Nov  7  2025 cockpit.xml
-rw-r--r--. 1 root root 296 Nov  7  2025 collectd.xml
#trucated 

```

### 3.2 SSH Service XML
```bash 
$ sudo cat /usr/lib/firewalld/services/ssh.xml 
[sudo] password for blue: 
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>SSH</short>
  <description>Secure Shell (SSH) is a protocol for logging into and executing commands on remote machines. It provides secure encrypted communications. If you plan on accessing your machine remotely via SSH over a firewalled interface, enable this option. You need the openssh-server package installed for this option to be useful.</description>
  <port protocol="tcp" port="22"/>
</service>
```

### 3.3 DHCPv6 Service XML
```bash
$ sudo cat /usr/lib/firewalld/services/dhcpv6.xml 
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>DHCPv6</short>
  <description>This allows a DHCPv6 server to accept messages from DHCPv6 clients and relay agents.</description>
  <port protocol="udp" port="547"/>
</service>
```


## 4. Firewall Management with firewall-cmd

There are three ways to manage `firewalld` in RHEL: the `firewall-cmd` CLI, a web interface (Cockpit), or manual editing of zone and service XML files. `firewall-cmd` is the primary operational tool.

Key behavioral rules:

- Changes applied **without** `--permanent` take effect immediately at runtime but are **lost** on daemon reload or system reboot.
- Changes applied **with** `--permanent` are written to the zone XML file on disk but do **not** take effect until the daemon is reloaded with `--reload`.


### 4.1 Checking the state of firewalld 
```bash 
$ sudo firewall-cmd --state
running
```

### 4.1.1 Verfing with systemctl 
```bash 
$ sudo systemctl status firewalld
[sudo] password for blu3sky: 
● firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-05-26 09:08:25 EDT; 1 day 3h ago
 Invocation: d0780963b5f4415a92ab311460ceda5d
       Docs: man:firewalld(1)
   Main PID: 1465 (firewalld)
      Tasks: 4 (limit: 22950)
     Memory: 52.4M (peak: 53.6M)
        CPU: 1.395s
     CGroup: /system.slice/firewalld.service
             └─1465 /usr/bin/python3 -sP /usr/sbin/firewalld --nofork --nopid

May 26 09:08:24 server40.usman.com systemd[1]: Starting firewalld.service - firewalld - dynamic fire>
May 26 09:08:25 server40.usman.com systemd[1]: Started firewalld.service - firewalld - dynamic firew>
``` 

## 5. Adding Service 

### 5.1 Getting the default zone
```bash 
$ sudo firewall-cmd --get-default-zone 
public
``` 

### 5.2 Initial Services Allowed
```bash 
 $ sudo firewall-cmd --list-services 
cockpit dhcpv6-client nfs ssh
```

### 5.3 Adding HTTPS at Runtime

```bash 
$ sudo firewall-cmd --add-service=https 
success
``` 
#### 5.3.1 Verify 
```bash 
$ sudo firewall-cmd --list-services 
cockpit dhcpv6-client https nfs ssh
```
#### 5.3.2 Reload the Daemon
```bash 
$ sudo firewall-cmd --reload
success
``` 
#### 5.3.3 Verify After Reload

```bash 
$ sudo firewall-cmd --list-services 
cockpit dhcpv6-client nfs ssh
``` 
> `https` is gone. Runtime-only rules are wiped when the daemon reloads or the system reboots. To survive reloads, use `--permanent`.


### 5.4 Making a Service Persistent

```bash 
$ sudo firewall-cmd --permanent --add-service=https 
[sudo] password for blu3sky: 
success
```
> Any change made with `--permanent` is written to the zone XML file on disk, but does not take effect until `--reload` is run. Conversely, changes made without `--permanent` apply instantly but do not survive a reload

#### 5.4.1 Reload the Daemon


```bash 
$ sudo firewall-cmd --reload
success
```

#### 5.4.2 Verify Persistence

```bash 
$ sudo firewall-cmd --list-services 
cockpit dhcpv6-client https nfs ssh
```
### 5.5 Verifying via Zone XML

The persistent change is confirmed by inspecting the zone file directly:

```bash
$ sudo cat /etc/firewalld/zones/public.xml 
[sudo] password for blu3sky: 
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Public</short>
  <description>For use in public areas. You do not trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.</description>
  <service name="ssh"/>
  <service name="dhcpv6-client"/>
  <service name="cockpit"/>
  <service name="nfs"/>
  <service name="https"/>  # --> here we go 
  <forward/>
</zone>
``` 

## 6 Adding ports and managing zones 

### 6.1 intial status of ports
```bash
$ sudo firewall-cmd --list-ports 

```
### 6.2 Adding a Runtime TCP Port
```bash
$ sudo firewall-cmd --add-port=443/tcp 
success
``` 
#### 6.2.1 Verify
```bash
$ sudo firewall-cmd --list-ports 
443/tcp
```

### 6.3 Adding a Permanent TCP Port Range to a Different Zone

```bash
$ sudo firewall-cmd --add-port=5901-5910/tcp --permanent --zone=internal 
success
``` 
#### 6.3.1 Reload the Daemon
```bash
$ sudo firewall-cmd --reload
success
```

#### 6.3.2 Verify via firewall-cmd
```bash 
$ sudo firewall-cmd --list-ports --zone=internal 
5901-5910/tcp
```

#### 6.3.3 Verify via Zone XML
```bash
$ sudo cat /etc/firewalld/zones/internal.xml
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Internal</short>
  <description>For use on internal networks. You mostly trust the other computers on the networks to not harm your computer. Only selected incoming connections are accepted.</description>
  <service name="ssh"/>
  <service name="mdns"/>
  <service name="samba-client"/>
  <service name="dhcpv6-client"/>
  <service name="cockpit"/>
  <port port="5901-5910" protocol="tcp"/>  # --> here we go
  <forward/>
</zone>
```

### 6.4 Switching the Default Zone


```bash
$ sudo firewall-cmd --set-default-zone=internal 
success
```
#### 6.4.1 Verify Port Visibility After Zone Switch

```bash
$ sudo firewall-cmd --list-ports 
5901-5910/tcp
```
> After switching the default zone to `internal`, `--list-ports` now reflects the internal zone's ports.


### 6.5 Adding a Permanent UDP Port Range
```bash
$ sudo firewall-cmd --add-port=8000-8005/udp --permanent --zone=trusted
success
```
#### 6.5.1 Reload the Daemon

```bash
$ sudo firewall-cmd --reload
success
```
#### 6.5.2 Verify

```bash
$ sudo firewall-cmd --list-ports  --zone=trusted
8000-8005/udp
```

#### 6.5.3 Cleanup

```bash 
$ sudo firewall-cmd --remove-port=8000-8005/udp --permanent  --zone=trusted
success
```


## 7. Removing Ports and Services

### 7.1 Removing a Port from the Internal Zone

```bash
$ sudo firewall-cmd --permanent --remove-port=5901-5910/tcp 
[sudo] password for blu3sky: 
success
``` 

#### 7.1.1 Reload the Daemon
```bash
$ sudo firewall-cmd --reload
success
```

#### 7.1.2 Verify
```bash 
$ sudo firewall-cmd --list-ports 

```
### 7.2 Removing a Service from the Public Zone

#### 7.2.1 Switch Back to Public
```bash
$ sudo firewall-cmd --set-default-zone=public
success
```

#### 7.2.2 Remove the Service
```bash
$ sudo firewall-cmd --permanent --remove-service=https 
success
```
#### 7.2.3 Reload the Daemon

```bash
$ sudo firewall-cmd --reload 
success
```

### 7.2.4 Verify 
```bash
$ sudo firewall-cmd --list-service
cockpit dhcpv6-client nfs ssh
```

## 8. Live Firewall Effect Demonstration

This section demonstrates the real-world impact of firewall rules using a two-machine setup: `server40.usman.com` as the server and a second VM as the SSH client.

### 8.1 Server Setup and Removing SSH at Runtime

Confirm the server's identity and active network path:

```bash
$ hostname; ip route show ; whoami 
server40.usman.com
default via 192.168.122.1 dev enp1s0 proto dhcp src 192.168.122.88 metric 100 
192.168.122.0/24 dev enp1s0 proto kernel scope link src 192.168.122.88 metric 100 
blu3sky
```
****Remove SSH from the firewall at runtime only — no `--permanent`, so this applies only until the next reload:**** 

```bash
$ sudo firewall-cmd --remove-service=ssh 
[sudo] password for blu3sky: 
success
``` 
> Not reloading intentionally. This is a runtime-only change to demonstrate live firewall behavior on an active system.

#### 8.1.1 Verify SSH Is No Longer Allowed

```bash
$ sudo firewall-cmd --list-service
cockpit dhcpv6-client nfs
```

#### 8.1.2 Confirm the SSH Daemon Is Still Running

The firewall rule removal has no effect on the `sshd` daemon itself — it is still listening on port 22. The firewall drops packets before they ever reach the daemon.

```bash
$ sudo systemctl status sshd
● sshd.service - OpenSSH server daemon
     Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-05-26 09:08:25 EDT; 1 day 4h ago
 Invocation: 8c5f3f743f7c43fbb4fcb0bce3e2ce0e
       Docs: man:sshd(8)
             man:sshd_config(5)
   Main PID: 1530 (sshd)
      Tasks: 1 (limit: 22950)
     Memory: 2.3M (peak: 2.6M)
        CPU: 12ms
     CGroup: /system.slice/sshd.service
             └─1530 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"

May 26 09:08:25 server40.usman.com systemd[1]: Starting sshd.service - OpenSSH server daemon...
May 26 09:08:25 server40.usman.com sshd[1530]: Server listening on 0.0.0.0 port 22.
May 26 09:08:25 server40.usman.com sshd[1530]: Server listening on :: port 22.
``` 
### 8.2 SSH Connection Blocked from Client
Attempting to connect from the client VM (`192.168.122.206`):

```bash
$ ssh blu3sky@192.168.122.88  
ssh: connect to host 192.168.122.88 port 22: No route to host
```
> The firewall dropped the packet silently — no TCP RST, no ICMP error back to the client. The connection attempt hits the firewall and goes nowhere. The daemon is still running and untouched.

### 8.3 Restoring SSH Access

#### 8.3.1 Reload the Daemon — Server Side

Reloading discards the runtime-only removal and restores the permanent configuration from disk:

```bash
$ sudo firewall-cmd --reload 
[sudo] password for blu3sky: 
success
```
#### 8.3.2 Verify SSH Is Restored
```bash
$ sudo firewall-cmd --list-service
cockpit dhcpv6-client nfs ssh
``` 

#### 8.3.3 Reconnect from the Client
```bash
$ ssh blu3sky@192.168.122.88  
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
Last login: Tue May 26 09:09:09 2026
blu3sky@server40:~$ 
```

## 9. Rich Rules

Rich rules allow fine-grained access control beyond simple service or port rules. They support matching on source IP, destination IP, protocol, port, and service name — all in a single rule expression. This makes them the right tool for per-source access control.

### 9.1 Getting the Client IP
```bash
$ ip route show
default via 192.168.122.1 dev enp1s0 proto dhcp src 192.168.122.206 metric 100 
192.168.122.0/24 dev enp1s0 proto kernel scope link src 192.168.122.206 metric 100 
```
Client IP: `192.168.122.206`

 
### 9.2 Adding a MySQL Rich Rule

Allow MySQL traffic from the client IP only. Two equivalent methods:

**Method 1 — by service name:**
```bash
$ sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" service name="mysql" source address="192.168.122.206"  accept ' 
success
```

**Method 2 — by port number (equivalent):**

```bash
$ sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" port port="3306" protocol="tcp" source address="192.168.122.206" accept '
```
> Use one method, not both — `firewalld` treats them as equivalent for the same service. MySQL's port mapping can be confirmed from `/etc/services`:

```bash 
$ cat /etc/services | grep mysql
mysql           3306/tcp                        # MySQL  # --> you can seee here the port 
mysql           3306/udp                        # MySQL    # --> here is udp so i choose tcp 
mysql-cluster   1186/tcp                # MySQL Cluster Manager
mysql-cluster   1186/udp                # MySQL Cluster Manager
mysql-cm-agent  1862/tcp                # MySQL Cluster Manager Agent
mysql-cm-agent  1862/udp                # MySQL Cluster Manager Agent
mysql-im        2273/tcp                # MySQL Instance Manager
mysql-im        2273/udp                # MySQL Instance Manager
mysql-proxy     6446/tcp                # MySQL Proxy
mysql-proxy     6446/udp                # MySQL Proxy
mysqlx          33060/tcp               # MySQL Database Extended Interface
``` 
> Since MySQL uses TCP for client connections, `protocol="tcp"` is the correct choice in the port-based variant.

#### 9.2.1 Reload daemon
```bash
$ sudo firewall-cmd --reload 
success
``` 
#### 9.2.2 Verify
```bash 
$ sudo firewall-cmd --list-rich-rules 
rule family="ipv4" source address="192.168.122.206" service name="mysql" accept
```


### 9.3 Tearing Down the Rich Rule

#### 9.3.1 Remove the Rule
```bash 
$ sudo firewall-cmd --permanent --remove-rich-rule='rule family="ipv4" service name="mysql" source address="192.168.122.206"  accept ' 
[sudo] password for blu3sky: 
success
``` 
#### 9.3.2 Reload the Daemon

```bash
$ sudo firewall-cmd --reload 
success
```
#### 9.3.3 Verify 
```bash
$ sudo firewall-cmd --list-rich-rules 

```

## 10. Creating a Custom Firewall Zone

`firewalld` allows creation of custom zones by copying a system-defined XML template into `/etc/firewalld/zones/` and modifying it. Once placed there and reloaded, the custom zone is available alongside the predefined ones.

### 10.1 Listing Initial Zones

```bash
$ sudo firewall-cmd --list-all-zones 
block
  target: %%REJECT%%
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: 
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

dmz
  target: default
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

drop
  target: DROP
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: 
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

external
  target: default
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: yes
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

home
  target: default
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: cockpit dhcpv6-client mdns samba-client ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

internal
  target: default
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: cockpit dhcpv6-client mdns samba-client ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

nm-shared
  target: ACCEPT
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: dhcp dns ssh
  ports: 
  protocols: icmp ipv6-icmp
  forward: no
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
	rule priority="32767" reject

public (default, active)
  target: default
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: enp1s0
  sources: 
  services: cockpit dhcpv6-client nfs ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

trusted
  target: ACCEPT
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: 
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

work
  target: default
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```
### 10.2 Reviewing the Zone Template

The `public.xml` system template is used as a starting point: 

```bash
$ sudo cat  /usr/lib/firewalld/zones/public.xml 
[sudo] password for blu3sky: 
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Public</short>
  <description>For use in public areas. You do not trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.</description>
  <service name="ssh"/>
  <service name="dhcpv6-client"/>
  <service name="cockpit"/>
  <forward/>
</zone>
``` 
### 10.3 Creating the Custom Zone File

Redirect the template content into a new file locally, then copy it into the `firewalld` zone directory:
```bash
$ sudo cat  /usr/lib/firewalld/zones/public.xml > usman-library.xml
``` 
### 10.3.1  cp to `/etc/firewalld/zones/` 
```bash
$ sudo cp usman-library.xml  /etc/firewalld/zones/usman-library.xml 
```
> **Important:** you cannot redirect output directly into `/etc/firewalld/zones/` in one step — permission and path restrictions prevent it. Always write the file locally first, then `cp` it into place. Attempting `mv` will also fail. The supported workflow is always `cp`.

   
### 10.4 Reloading and Verifying the New Zone
```bash
$ sudo firewall-cmd --reload 
success
``` 
####  10.4.1 Verify the new zone 
```bash
block
  target: %%REJECT%%
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: 
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

dmz
  target: default
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

drop
  target: DROP
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: 
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

external
  target: default
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: yes
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

home
  target: default
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: cockpit dhcpv6-client mdns samba-client ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

internal
  target: default
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: cockpit dhcpv6-client mdns samba-client ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

nm-shared
  target: ACCEPT
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: dhcp dns ssh
  ports: 
  protocols: icmp ipv6-icmp
  forward: no
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
	rule priority="32767" reject

public (default, active)
  target: default
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: enp1s0
  sources: 
  services: cockpit dhcpv6-client nfs ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

trusted
  target: ACCEPT
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: 
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

usman-library                                   # --> here we are. custom zone is now live 
  target: default
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
# truncated output
```
### 10.5 Managing Services and Ports in the Custom Zone

#### 10.5.1 Verify Current Services

```bash 
$ sudo firewall-cmd --list-services --zone=usman-library
cockpit dhcpv6-client ssh
```
#### 10.5.2 Remove a Service

```bash
$ sudo firewall-cmd  --permanent --remove-service=cockpit --zone=usman-library 
success
```
#### 10.5.3 Add a Service

```bash
$ sudo firewall-cmd  --permanent --add-service=dns --zone=usman-library 
success
```
#### 10.5.4 Switch Default Zone to the Custom Zone

```bash
$ sudo firewall-cmd --set-default-zone=usman-library 
success
```
#### 10.5.5 Reload the Daemon

```bash 
$ sudo firewall-cmd --reload 
success
```
#### 10.5.6 Verify Active Services

```bash
$ sudo firewall-cmd --list-services 
dhcpv6-client dns ssh
``` 
#### 10.5.7 Add More Services

```bash
$ sudo firewall-cmd --permanent --add-service=https 
success
```
```bash
$ sudo firewall-cmd --reload 
success
```
#### 10.5.8 Verify
```bash
$ sudo firewall-cmd --list-services 
dhcpv6-client dns https ssh
```
### 10.6 Adding a Rich Rule to the Custom Zone

Confirm the PostgreSQL port from `/etc/services`:
```bash
$ cat /etc/services | grep postgresql 
postgres        5432/tcp        postgresql      # POSTGRES
postgres        5432/udp        postgresql      # POSTGRES
```
####  10.6.1 Add a rich rule allowing PostgreSQL access from the client IP only:

```bash
$ sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" port port="5432" protocol="tcp" source address="192.168.122.206" accept' 
[sudo] password for blu3sky: 
success
``` 
#### 10.6.2 Reload the Daemon

```bash
$ sudo firewall-cmd --reload 
success
```
#### 10.6.3 Verify
```bash
$ sudo firewall-cmd --list-rich-rules 
rule family="ipv4" source address="192.168.122.206" port port="5432" protocol="tcp" accept
```

### 10.7 Binding an Interface to the Custom Zone

#### 10.7.1 Initial Zone State

```bash
$ sudo firewall-cmd --list-all
usman-library (default, active)
  target: default
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: enp1s0
  sources: 
  services: dhcpv6-client dns https ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
	rule family="ipv4" source address="192.168.122.206" port port="5432" protocol="tcp" accept
```
#### 10.7.2 Get Available Network Interfaces

```bash
$ nmcli connection show
NAME    UUID                                  TYPE      DEVICE 
enp1s0  5130fcd1-6982-357e-af13-f82b376f2dbc  ethernet  enp1s0 
lo      786247cb-5ee5-4b38-a033-0c6e23047620  loopback  lo     
```
#### 10.7.3 Add Interface to the Custom Zone

```bash
$ sudo firewall-cmd --add-interface="enp1s0" 
Warning: ZONE_ALREADY_SET: 'enp1s0' already bound to ''
success
```
> The warning indicates `enp1s0` was previously assigned to the `public` zone. Binding it to `usman-library` automatically removes it from `public` — an interface can only belong to one zone at a time.

#### 10.7.4 Verify the Public Zone Lost the Interface

```bash
$ sudo firewall-cmd --list-all --zone=public
public
  target: default
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces:                       # --> now it empty the interface is with usman-library now 
  sources: 
  services: cockpit dhcpv6-client nfs ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```



### 10.8 Changing the Zone Target via XML

The zone target controls what happens to traffic that does not match any explicit allow rule. Editing the XML directly is the supported method for changing it.

#### 10.8.1 Edit the Zone File

```bash 
$ sudo vi /etc/firewalld/zones/usman-library.xml 
```
Change the opening `<zone>` tag to:
```bash
<zone target="%%REJECT%%">
```

#### 10.8.2 Reload the Daemon

```bash
$ sudo firewall-cmd --reload 
success
``` 


#### 10.8.3 Verify the Target Change

```bash
$ sudo firewall-cmd --list-all
usman-library (default, active)
  target: %%REJECT%%
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: enp1s0
  sources: 
  services: dhcpv6-client dns https ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
	rule family="ipv4" source address="192.168.122.206" port port="5432" protocol="tcp" accept
``` 

> With `%%REJECT%%` as the target, all unmatched traffic is rejected with an ICMP error sent back to the sender — unlike `DROP`, which discards packets silently.

### 10.9 Tearing Down the Custom Zone

#### 10.9.1 Switch Default Back to Public
```bash
$ sudo firewall-cmd --set-default-zone=public 
success
``` 

#### 10.9.2 Delete the Custom Zone File

```bash
$ sudo rm  /etc/firewalld/zones/usman-library.xml 
```

#### 10.9.3 Re-bind the Interface to the Default Zone

```bash
$ sudo firewall-cmd --permanent --add-interface=enp1s0 
The interface is under control of NetworkManager and already bound to the default zone
The interface is under control of NetworkManager, setting zone to default.
success
```
> When `usman-library.xml` was deleted, `enp1s0` lost its zone binding. This command reassigns it to the current default zone (`public`).


#### 10.9.4 verify
```bash
$ sudo firewall-cmd --list-all
public (default, active)
  target: default
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: enp1s0
  sources: 
  services: cockpit dhcpv6-client nfs ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```

## 11. Port Forwarding

Port forwarding redirects traffic arriving on one port to a different port — on the same host or a remote one. This is useful for transparently proxying traffic without changing client configuration.

### 11.1 Forwarding Port 8888 to 443

Redirect all TCP traffic hitting port `8888` to port `443` on the same host, in the `public` zone:

```bash
$ sudo firewall-cmd --permanent --add-forward-port=port=8888:proto=tcp:toport=443
[sudo] password for blu3sky: 
success
```

#### 11.1.1 Reload the Daemon
```bash
$ sudo firewall-cmd --reload 
success
```
#### 11.1.2 Verify

```bash
$ sudo firewall-cmd --list-forward-ports 
port=8888:proto=tcp:toport=443:toaddr=
``` 
> The empty `toaddr=` field means traffic is redirected locally on the same machine. To forward to a different host, specify `toaddr=<IP>` in the rule.

---

## 12. Key Takeaways

- **Runtime vs. Permanent:** Changes without `--permanent` apply instantly but are lost on daemon reload or reboot. Changes with `--permanent` persist to disk but require `--reload` to become active in the running firewall.
- **Zone fallback order:** `firewalld` matches incoming packets against zones in this order — source IP match → interface match → default zone. The first match wins.
- **Zone XML locations:** System templates in `/usr/lib/firewalld/zones/` are read-only references. Active rules come from `/etc/firewalld/zones/`.
- **Custom zone creation:** Must be done by copying a system template and placing it in `/etc/firewalld/zones/`. Redirecting directly to that path fails — always use `cp`. `mv` also fails.
- **Rich rules:** Allow source-IP-scoped access control for services or ports. Use either `service name=` or `port port=` — not both for the same service, as they are equivalent to `firewalld`.
- **Zone targets:** `default` drops unmatched traffic silently; `%%REJECT%%` sends an ICMP error back to the sender; `ACCEPT` allows everything; `DROP` silently discards.
- **Interface binding:** An interface can only belong to one zone. Assigning it to a new zone automatically removes it from the previous one — confirmed by the `ZONE_ALREADY_SET` warning.
- **Port forwarding:** `toaddr=` empty means local redirect on the same host. Populate it with an IP to forward to a remote machine.

