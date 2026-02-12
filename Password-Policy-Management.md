# 🔒 Auditing & Enforcing Password Aging Policies (RHCSA Focus)

**Author:** Usman O. Olanrewaju (Blu3 Sky)  
**Date:** 2026/02/12  
**Environment:** RHEL/Fedora System Administration

---

## 1. Overview: The Security Imperative

Password aging policies are foundational to system hardening. They mandate timely rotation of credentials, significantly mitigating the risk posed by compromised or stale passwords. This lab focuses on configuring and verifying these settings using standard Linux utilities.

---

## 2. Inspecting Current Password State (`chage` & `/etc/shadow`)

Before enforcing new rules, the current policy state must be read. We use `chage -l` to extract expiration data from the encrypted shadow file.

### Baseline State Check
```bash
$ sudo chage -l testuser
Last password change                                    : Feb 12, 2026
Password expires                                        : never
Password inactive                                       : never
Account expires                                         : never
Minimum number of days between password change          : 0
Maximum number of days between password change          : 99999
Number of days of warning before password expires       : 7
```

**Reading Encrypted Data** 
To confirm the system's actual configuration (as stored in /etc/shadow), look at the last field of the hash line.

```bash 
$ sudo tail -1 /etc/shadow
[sudo] password for blu3sky:
testuser:$y$j9T$fKAxQ8oSWqLLM3gcUtKRC1$IxaJpPO8cWZP99WmcAIamjxJCn.JWDpzwVocgScnkhA:20496:0:99999:7:::
```
found the username and the password is encrypted with yescrypt and the last two numeric fields (`7::`) control the warning period and inactive period before expiration.

## 3. Enforcing New Password Policy (chage)

implement the following required policy for testuser:

* Max Age: 90 days

*  Min Age: 0 days (can change it immediately)

*  Warning: 10 days before expiration

*  Inactivity: 14 days after password expires

*  last change days: 0 to change password 

**Command** 
```bash 
#Syntax: chage -E(expiry date) -I(inactive) -m(min days) -M(max days) -W(warning) -d(last change date)

$ sudo chage -E 2026-05-14 -m 0 -M 90 -W 10 -d 0 -I 0 testuser
```

**Verification of New Policy** 
```bash 
$ sudo chage -l testuser
Last password change                                    : password must be changed   <---(New requirement for using `-d 0` )
Password expires                                        : password must be changed
Password inactive                                       : password must be changed
Account expires                                         : May 14, 2026               <---(Account expiration is set) 
Minimum number of days between password change          : 0
Maximum number of days between password change          : 90                         <---(90days set)  
Number of days of warning before password expires       : 10                         <---(Warning 10 times before password exp) 


## 4.Advanced Account & Policy Management(`passwd`)  

sudo passwd -n (min days) -x(max days) -w (warning) -l (lock) -e (expire) -u(unlock) -d ( remove passwwd without deleting the account) -i(inactive)  username

sudo passwd -S(status)  username
```bash
$ sudo passwd -S testuser 
testuser P 2026-02-12 0 90 10 0
```

### Account Locking / Unlocking with `passwd`
These commands directly manipulate the password hash field in `/etc/shadow` to lock or unlock an account without changing the password.

*   **Lock Account:**
    ```bash
   $ sudo passwd -l testuser
    passwd: password changed.
    ```
*   **Unlock Account:**
    ```bash
    $ sudo passwd -u testuser
    passwd: password changed.
    ```


## 5. Advanced Command Review
The following commands are available for related password management tasks:

**Command Function:**
* `passwd -S [username]`   ->	Show password status (hash, status, dates).
* `sudo passwd [username]` ->	Interactively change a user's password.
* `chage -d 0 [username]`  -> 	Force a password change on next login.

   **Note:** All default policies are defined in /etc/login.defs
