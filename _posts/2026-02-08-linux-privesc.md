---
title: Linux Privilege Escalation
author: 0vld
date: 2026-02-08
category: linux
layout: post
---

This guide covers essential enumeration techniques for Linux privilege escalation.

> ##### TIP
>
>Enumeration is the key to privilege escalation. Understand what pieces of information to look for.
{: .block-tip }

## Initial Orientation

Typically, we want to run a few basic commands to orient ourselves:

+ `whoami` - what user are we running as
+ `id` - what groups does our user belong to?
+ `hostname` - what is the server named, can we gather anything from the naming convention?
+ `ifconfig` or `ip a` - what subnet did we land in, does the host have additional NICs in other subnets?
+ `sudo -l` - can our user run anything with sudo (as another user as root) without needing a password?

In some cases, `sudo -l` can lead to an immediate privilege escalation (for example, using `sudo su`).

Including screenshots of this information is helpful in client reports to demonstrate successful Remote Code Execution (RCE).

## Key Enumeration Targets

When you gain initial shell access to the host, check the following details:

+ OS Version
+ Kernel Version
+ Running Services
+ Installed Packages and Versions
+ Logged in Users
+ User Home Directories
+ .bash_history
+ Sudo Privileges
+ Configuration Files
+ Readable Shadow File
+ Password Hashes in /etc/passwd
+ Cron Jobs
+ Unmounted File Systems and Additional Drives
+ SETUID and SETGID Permissions
+ Writable Directories
+ Writable Files

##### Operating System Information
```bash
cat /etc/os-release
```

##### Environment Variables
```bash
echo $PATH
```
```bash
env
```

##### Group Information
```bash
groups
```
```bash
cat /etc/group
```

##### Kernel Information
```bash
uname -a
```
```bash
cat /proc/version
```
```bash
lscpu
```

##### Login Shells
```bash
cat /etc/shells
```

##### File Systems
```bash
cat /etc/fstab
```
```bash
cat /etc/fstab | grep -v "#" | column -t
```
```bash
lsblk
```

##### Network Routes
```bash
route
```

Or:
```bash
ip route
```

##### Hidden Files
```bash
find / -type f -name ".*" -exec ls -l {} \; 2>/dev/null
```

##### Hidden Directories
```bash
find / -type d -name ".*" -ls 2>/dev/null
```

##### Temporary Directories
```bash
ls -l /tmp /var/tmp /dev/shm
```

##### Running Processes
```bash
ps aux | grep root
```
```bash
ps au
```

##### Writable Directories
```bash
find / -path /proc -prune -o -type d -perm -o+w 2>/dev/null
```

##### Writable Files
```bash
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
```
+ [Exec Shield](https://en.wikipedia.org/wiki/Exec_Shield)
+ [iptables](https://linux.die.net/man/8/iptables)
+ [AppArmor](https://apparmor.net/)
+ [SELinux](https://www.redhat.com/en/topics/linux/what-is-selinux)
+ [Fail2ban](https://github.com/fail2ban/fail2ban)
+ [Snort](https://www.snort.org/faq/what-is-snort)
+ [Uncomplicated Firewall (ufw)](https://wiki.ubuntu.com/UncomplicatedFirewall)

---