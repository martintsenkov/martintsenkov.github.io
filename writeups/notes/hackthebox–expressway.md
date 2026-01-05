# Hack The Box – Expressway

**Platform:** Hack The Box
**Category:** Network / VPN → IKE / IPsec → Credential Abuse → Local Enumeration → Sudo Misconfiguration
**Host:** Kali Linux

---
 
## Overview

Expressway is a network-centric machine that focuses on VPN infrastructure weaknesses rather than exposed web services.
The compromise path abuses a misconfigured IPsec/IKE setup using Aggressive Mode with PSK authentication, followed by credential reuse and a hostname-based sudo misconfiguration to achieve root access.

This machine is a good example of how “secure” services become dangerous when configured incorrectly.

---

## Reconnaissance

Initial TCP scanning showed a very limited attack surface:

```bash
nmap expressway.htb -A
```


Only SSH (22/tcp) was exposed.
Since no web services were available, attention shifted to UDP enumeration, which is often overlooked.

---

## UDP Enumeration & VPN Discovery

Scanning common UDP ports revealed several interesting services:

```bash
nmap expressway.htb -sU --top-ports 100
```


Notable findings:

500/udp – ISAKMP

4500/udp – NAT-T (IPsec)

This strongly suggested the presence of an IPsec VPN.

---

## IKE Service Enumeration

The IKE service was confirmed using ike-scan:

```bash
ike-scan -M expressway.htb
```


The response confirmed:

IKE Main Mode supported

Authentication: PSK

Weak crypto suite (3DES, SHA1, DH Group 2)

XAUTH enabled

This already indicates a potentially weak VPN configuration.

---

## Aggressive Mode Abuse & Identity Disclosure

To extract more information, Aggressive Mode was used:

```bash
ike-scan -P -M -A -n fakeID expressway.htb
```


This revealed a critical detail:

```bash
ID(Type=ID_USER_FQDN, Value=ike@expressway.htb)
```


At this point:

the VPN identity username was disclosed

the handshake material required to crack the PSK was obtained

Aggressive Mode should never be exposed publicly for this exact reason.

---

## PSK Cracking & Initial Access

The extracted IKE handshake data was used to crack the pre-shared key offline.

Once the PSK was recovered, it allowed successful authentication and SSH access as the user ike:

```bash
ssh ike@expressway.htb
```


This provided the initial foothold on the system.

---

## Local Enumeration

Checking group membership revealed something interesting:

```bash
id
```


The user ike belonged to the proxy group.

Enumerating directories owned by that group:


```bash
find / -group proxy -type d 2>/dev/null
```


This exposed Squid proxy directories, including log files.

---

## Internal Hostname Discovery

Inspecting the proxy logs revealed internal traffic:

```bash
 cat /var/log/squid/access.log.1 | grep htb
```


This disclosed an internal hostname:

```bash
offramp.expressway.htb
```


This hostname became extremely important in the next stage.

---

## Sudo Misconfiguration

Checking the sudo version showed a relatively recent release:

```bash
sudo -V
```


Testing sudo rules against the internal hostname revealed a severe misconfiguration:

```bash
sudo -h offramp.expressway.htb -l
```


Result:

```bash
User ike may run the following commands on offramp:
    (root) NOPASSWD: ALL
```


This means:

sudo rules were hostname-specific

when evaluated against offramp.expressway.htb, the user had full root access

---

## Root Compromise

With no password required, escalating to root was trivial:

```bash
sudo -h offramp.expressway.htb /bin/bash
```


Root access was obtained immediately, and the root flag was retrieved.

![](/writeups/notes/hackthebox–expressway/img/01.png)
