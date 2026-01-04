# Hack The Box – Conversor

**Platform:** Hack The Box
**Category:** Web → XXE/XSLT Injection → RCE → Privilege Escalation
**Host:** Kali Linux

---

## High-Level Summary

Conversor is a web application that converts XML and XSLT input into HTML.
The compromise path relies on unsafe XSLT processing, which enables file write through EXSLT, leading to remote code execution. A misconfigured cron job and a vulnerable system package are then abused to achieve full root compromise.

---

## Attack Surface Discovery

After adding the domain to /etc/hosts, the application presents a simple workflow:

user registration

XML + XSLT upload

server-side transformation to HTML

The About page exposes a download link to the full application source code — a critical mistake that immediately shifts the attack from black-box to white-box analysis.

---

## Source Code Review & Vulnerability Identification

The application is built using Flask and relies on lxml for XML/XSLT processing.

Key observation:

```bash
parser = etree.XMLParser(resolve_entities=False, no_network=True)
xml_tree = etree.parse(xml_path, parser)
xslt_tree = etree.parse(xslt_path)  # No parser restrictions
transform = etree.XSLT(xslt_tree)
```

## Core Issue

XML parsing is hardened

XSLT parsing is completely unrestricted

This opens the door for XSLT-based XXE / file write attacks using EXSLT functionality.

The vulnerability is not in XML — it is inside the XSLT engine itself.

---

## Exploitation Strategy

Instead of reading files, the goal was arbitrary file write.

From the documentation, a cron job is revealed:

```bash
* * * * * www-data for f in /var/www/conversor.htb/scripts/*.py; do python3 "$f"; done
```


This means:

any Python file written to /scripts/

will be executed every minute

as www-data

This immediately defines the attack chain.

---
 
## Remote Code Execution via XSLT Injection

Using Burp Suite, a malicious XSLT payload was uploaded that leveraged EXSLT to write a Python reverse shell into:

```bash
/var/www/conversor.htb/scripts/
```


Once written:

a listener was started

the cron job executed the payload

a reverse shell was received as www-data

At this point, web → system boundary was fully broken.

---

## User Privilege Escalation

With www-data access, attention shifted to local enumeration.

The application database revealed:

user credentials

stored using MD5 hashing (cryptographically broken)

After cracking the hashes, valid SSH credentials were recovered, allowing login as a real system user and retrieval of the user flag.

![](/writeups/notes/HackTheBox–Conversor/img/02.png)

---

## Root Privilege Escalation (CVE Abuse)

Enumeration revealed:

needrestart version 3.7

vulnerable to CVE-2024-48990

The exploit abuses:

Python import path hijacking

a malicious shared object

automatic execution during needrestart scans

A crafted .so payload was loaded, resulting in:

SUID shell creation

persistent sudo access

immediate root shell

The root flag was successfully obtained.

![](/writeups/notes/HackTheBox–Conversor/img/01.png)
