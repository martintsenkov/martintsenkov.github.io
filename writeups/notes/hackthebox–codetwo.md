# Hack The Box – CodeTwo

**Platform:** Hack The Box
**Category:** Web → Source Code Disclosure → Sandbox Escape → RCE → Credential Abuse → Backup Tool Privilege Escalation
**Host:** Kali Linux

---

## Overview

CodeTwo is a web-centric machine built around a Python Flask application that allows users to store and execute JavaScript code snippets.
The core weakness lies in unsafe JavaScript sandboxing using js2py, which enables sandbox escape and remote command execution.

The attack chain continues through weak password storage and ends with a misconfigured backup utility that allows reading arbitrary root-owned files.

---

## Reconnaissance

Initial scanning revealed a minimal exposed surface:

```bash
nmap codetwo.htb -A
```


Open services:

22/tcp – OpenSSH

8000/tcp – Gunicorn (Flask application)

The web service on port 8000 became the primary focus.

---

## Web Application Analysis

Accessing the application revealed a simple platform where users can:

register accounts

save JavaScript code snippets

execute code via a “Run Code” feature

A /download endpoint was also present, allowing download of the full application source code.

---

## Source Code Disclosure

The /download route exposed app.zip, containing the entire Flask application.

Reviewing the source code revealed several critical issues:

Hardcoded Flask secret key

Passwords stored using MD5

A dangerous endpoint:

```bash
@app.route('/run_code', methods=['POST'])
def run_code():
    code = request.json.get('code')
    result = js2py.eval_js(code)
    return jsonify({'result': result})
```


User-supplied JavaScript is passed directly into js2py.eval_js().

Although js2py.disable_pyimport() is used, this does not prevent sandbox escapes.

---

## Sandbox Escape (CVE-2024-28397)

Research revealed CVE-2024-28397, a js2py sandbox escape vulnerability.

By abusing Python object inheritance and __subclasses__(), it is possible to reach subprocess.Popen and execute system commands.

A custom proof-of-concept was created to spawn a reverse shell.

---

## Remote Code Execution

Reverse shell payload was sent to the /run_code endpoint:

```bash
nc -lnvp 4444
```


PoC execution resulted in a shell as the app user.

Verification:

```bash
id
```


At this point, full remote code execution was achieved.

---

## Local Enumeration & Credential Recovery

Local enumeration revealed the SQLite database:

```bash
/home/app/app/instance/users.db
```


Extracting user hashes showed MD5 password storage.

The password for user marco was successfully cracked offline.

Using the recovered credentials:

```bash
su marco
```


User access was obtained and user.txt retrieved.

---

## Sudo Privileges & Backup Tool Analysis

Checking sudo permissions:

```bash
sudo -l
```


Revealed:

```bash
(ALL) NOPASSWD: /usr/local/bin/npbackup-cli
```


npbackup-cli is a backup utility that supports:

custom configuration files

arbitrary backup paths

restoring or dumping backed-up data

Critically, configuration files are user-controlled.

---

## Privilege Escalation via Backup Abuse

A custom npbackup.conf file was created with the backup path set to /root.

Because npbackup-cli runs as root, this allowed reading root-owned files.

Forcing a backup:

```bash
sudo npbackup-cli -c npbackup.conf -b -f
```


The backup operation successfully included /root, allowing direct access to:

```bash
/root/root.txt
```

Root Access

Although no interactive root shell was spawned, full root file access was achieved, which is sufficient to compromise the system.

The root flag was read successfully.
