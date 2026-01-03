# YouCTFOne – Forgotten Key CTF Writeup

**Host:** Kali Linux
**Target:** YouCTFOne – Forgotten Key
**Environment:** Local VM (VMware)
**Category:** Web → SSH → Privilege Escalation

---

## Overview

This write-up documents the compromise of the Forgotten Key CTF machine from YouCTFOne.
The challenge focuses on basic network enumeration, web directory discovery, credential exposure through backups, and privilege escalation via poor secret management.

---

## Network Enumeration

The first step was to identify active services on the target machine within the local network.

![](/writeups/notes/youctfone-forgottenkey/img/01.png)

```bash
sudo nmap -sS --top-ports 100 <target_ip>
```

The scan revealed that HTTP (port 80) was open, indicating a potential web attack surface.

---

## Web Enumeration

Given the exposed HTTP service, the next step was to enumerate the web application.

![](/writeups/notes/youctfone-forgottenkey/img/02.png)

Directory discovery was performed using Burp Suite Intruder with:

A long directory wordlist

A / suffix added to payloads to improve detection accuracy

![](/writeups/notes/youctfone-forgottenkey/img/03.png)

During enumeration, the directory /backup/ returned a 200 OK response, immediately indicating exposed content.

---

## Sensitive File Discovery

Accessing the /backup/ directory revealed multiple files of interest.

![](/writeups/notes/youctfone-forgottenkey/img/04.png)

After inspection, it became clear that:

passwords.txt contained outdated credentials

backup-2025-may.tar.gz was a compressed archive worth investigating

![](/writeups/notes/youctfone-forgottenkey/img/05.png)

The archive was downloaded and extracted, revealing two files:

authorized_keys

ctf_key

---

## Credential Exposure
authorized_keys

The authorized_keys file contained an SSH public key along with a username reference:

```bash
ctf@forgotten
```

ctf_key

The ctf_key file was identified as an OpenSSH private key, which immediately suggested key-based SSH access.

This indicated a critical security misconfiguration: private SSH keys stored in a publicly accessible backup directory.

---

## Initial Access (SSH)

Using the exposed private key, SSH access to the target machine was obtained:

![](/writeups/notes/youctfone-forgottenkey/img/06.png)

```bash
ssh -i ctf_key ctf@<target_ip>
```

Authentication succeeded without requiring a password.

---

## User Flag

Once logged in as the ctf user, the user flag was found in the same directory, confirming successful initial compromise.

![](/writeups/notes/youctfone-forgottenkey/img/07.png)

---

## Privilege Escalation

After gaining user-level access, basic system exploration was performed.

A note containing the root password was discovered on the system, indicating another serious operational security failure.

![](/writeups/notes/youctfone-forgottenkey/img/08.png)

Using the recovered password, privilege escalation was achieved:

```bash
su root
```

---

## Root Flag

With root access obtained, the root flag was located and retrieved successfully.

![](/writeups/notes/youctfone-forgottenkey/img/09.png)

---

## Result

✔️ Web directory enumeration successful

✔️ Sensitive backup files exposed

✔️ SSH private key leaked

✔️ User access obtained via key-based authentication

✔️ Root access achieved through exposed credentials
