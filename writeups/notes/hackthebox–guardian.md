 # Hack The Box – Guardian

**Platform:** Hack The Box
**Category:** Web → IDOR → XSS → CSRF → LFI → RCE → Privilege Escalation
**Host:** Kali Linux

---

# Overview

Guardian is a complex web-focused machine simulating a university environment.
The compromise path demonstrates how weak defaults, poor access control, insecure file handling and flawed privilege separation can be chained into a full root compromise.

The machine is realistic, noisy, and intentionally layered – exactly the type of environment where attackers win through persistence rather than a single exploit.

---

# Reconnaissance

Initial scanning revealed a classic web + SSH setup:

```bash
nmap guardian.htb -A
```


Services discovered:

22/tcp – OpenSSH

80/tcp – Apache (Guardian University website)

Reviewing the homepage source revealed a hidden subdomain:

```bash
portal.guardian.htb
```


This immediately became the primary attack surface.

---

## Portal Access & Default Credentials

The student portal login page included a Help / Guide section explicitly stating:

![](/writeups/notes/HackTheBox–Guardian/img/01.png)

```bash
Default password: GU1234
```

Email addresses on the main site followed a predictable naming scheme, which aligned with backend usernames.

Using:

```bash
username: GU0142023
password: GU1234
```


Successful login was achieved.

➡️ Security failure: default credentials + no forced password rotation.

## IDOR in Student Chat

While reviewing the student chat feature, the URL structure revealed:

```bash
chat_users[0]=X&chat_users[1]=Y
```


This strongly suggested IDOR.

Using ffuf with a clusterbomb attack (while keeping the session cookie):

```bash
ffuf -u 'http://portal.guardian.htb/student/chat.php?chat_users[0]=FUZZ1&chat_users[1]=FUZZ2' \
-w nums.txt:FUZZ1 -w nums.txt:FUZZ2 \
-mode clusterbomb \
-H 'Cookie: PHPSESSID=...' \
-fl 178,164
```


This exposed private chat messages, one of which contained credentials for Gitea.

---

## Gitea Access & Source Code Review

The subdomain gitea.guardian.htb was accessible.

Credentials recovered:

```bash
jamil.enockson@guardian.htb
DHsNnk3V503
```


Login succeeded.

![](/writeups/notes/HackTheBox–Guardian/img/02.png)

While no public Gitea vulnerability existed, source code access revealed:

database credentials

authentication logic

CSRF implementation

report handling logic

At this point the attack fully transitioned to white-box.

---

## Stored XSS via PhpSpreadsheet

The portal allowed document uploads (.xlsx, .docx) and internally used PhpSpreadsheet to render them as HTML.

Reviewing dependencies revealed a known XSS issue:

worksheet titles are rendered without htmlspecialchars()

By crafting an XLSX file with a malicious worksheet name and uploading it, a stored XSS was triggered when staff viewed the document.

![](/writeups/notes/HackTheBox–Guardian/img/03.png)

This allowed theft of a lecturer session token.

## CSRF → Admin User Creation

With lecturer access, the admin panel source code revealed a broken CSRF implementation:

CSRF tokens are never invalidated

any previously issued token remains valid indefinitely

A malicious HTML page was created to silently submit a POST request to:


```bash
/admin/createuser.php
```


This resulted in creation of a new admin user:

```bash
username: attacker
password: P@ssw0rd123
```


➡️ Full admin access achieved.

---

## LFI via PHP Filter Chains

Admin reports functionality accepted a report parameter with partial filtering.

Although ../ traversal was blocked, the logic allowed:

specific filenames

suffix-based validation

By abusing php://filter chains, arbitrary file inclusion was achieved:

```bash
php://filter/convert.base64-decode/resource=...
```


This led to remote code execution and a reverse shell as www-data.

![](/writeups/notes/HackTheBox–Guardian/img/05.png)

---

## Database Access & Password Cracking

Local enumeration showed MySQL listening on localhost.

Using database credentials from config files:

```bash
SELECT username, password_hash FROM users;
```


Passwords were stored as:

```bash
SHA256(password + salt)
```


The salt was hardcoded in config.php.

A custom Python brute-force script recovered valid credentials:

```bash
admin: fakebake000
jamil.enockson: copperhouse56
```

Switching users:

su jamil


User flag obtained.

![](/writeups/notes/HackTheBox–Guardian/img/06.png)

---

## Privilege Escalation: jamil → mark

Checking sudo privileges:

```bash
sudo -l
```


Result:

```bash
(mark) NOPASSWD: /opt/scripts/utilities/utilities.py
```


The script imports multiple modules.
Inspection showed that utils/status.py was writable by jamil.

By inserting a reverse shell into status.py and running:

```bash
sudo -u mark utilities.py system-status
```


A shell as mark was obtained.

---

## Privilege Escalation: mark → root

The user mark had the following sudo rule:

```bash
(ALL) NOPASSWD: /usr/local/bin/safeapache2ctl
```


Binary analysis revealed:

it validates Apache config paths

but allows LoadModule from /home/mark/confs/

A malicious .so shared object was compiled with a constructor that sets SUID on /bin/bash.

Using:

```bash
sudo safeapache2ctl -f /home/mark/confs/exploit.conf
```


The malicious module executed as root.

![](/writeups/notes/HackTheBox–Guardian/img/08.png)

Root shell obtained.
