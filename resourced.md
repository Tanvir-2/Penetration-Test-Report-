# Resourced – Domain Credential Abuse and Privilege Escalation

## Internal Active Directory Security Assessment

**Target:** Resourced Domain Controller

**Domain:** resourced.local

**Assessment Type:** Internal Penetration Test

**Author:** Tanvir Ahmed (OSCP+, CEH, AZ-500, SC-200)

**Date:** 2026

---

# Executive Summary

An internal penetration test was conducted against the **Resourced Active Directory environment** to evaluate its resilience against internal attackers.

The assessment identified several weaknesses that allowed an attacker to escalate privileges from an unauthenticated network position to **Domain Administrator**.

The compromise was achieved through the following attack chain:

* Domain user enumeration via RPC
* Credential discovery from SMB shares
* Extraction of Active Directory database files
* Offline password hash cracking
* Pass-the-hash authentication
* Abuse of Active Directory permissions
* Resource-Based Constrained Delegation (RBCD)
* Remote command execution as Domain Administrator

This chain ultimately resulted in **complete compromise of the domain controller**.

### Business Impact

If exploited in a real enterprise network, an attacker could:

* Extract credentials for all domain users
* Access sensitive organizational data
* Execute arbitrary commands on domain systems
* Maintain persistent domain administrator access

This represents a **critical security risk** to the enterprise infrastructure.

---

# Scope

| Target          | Description       |
| --------------- | ----------------- |
| 192.168.153.175 | Domain Controller |

Domain:

```
resourced.local
```
<img width="1065" height="235" alt="image" src="https://github.com/user-attachments/assets/00559084-c174-4355-85ca-660cb86f34f0" />

Testing assumptions:

* Internal network access
* No initial credentials
* Limited knowledge of environment

---

# Methodology

The assessment followed a structured penetration testing methodology aligned with industry frameworks including:

* OWASP Testing Guide
* NIST penetration testing methodology

### Testing Phases

1. Network reconnaissance
2. Service enumeration
3. User enumeration
4. Credential discovery
5. Privilege escalation
6. Domain compromise validation

### Tools Used

* Nmap
* RPCClient
* Enum4Linux
* SMBClient
* Impacket
* Evil-WinRM
* BloodHound
* PowerView

---

# Attack Path Overview

The following attack chain was executed during the assessment.

```
Network Scan
      ↓
RPC Enumeration
      ↓
Domain User Discovery
      ↓
Credential Discovery
      ↓
SMB Share Access
      ↓
Active Directory Database Extraction
      ↓
Hash Cracking
      ↓
Pass-the-Hash Authentication
      ↓
Privilege Escalation via RBCD
      ↓
Domain Administrator Access
```

---

# Findings

---

# Finding 1 – Excessive Information Disclosure via RPC

**Severity:** Medium

### Description

Anonymous RPC access allowed enumeration of domain users.

### Evidence

```
rpcclient -U '' -N 192.168.153.175
```
<img width="757" height="579" alt="image" src="https://github.com/user-attachments/assets/0da5fbdf-0e1d-4bfa-9aa6-689bf5500864" />

Multiple domain users were discovered during enumeration. 

### Impact

Attackers can collect valid usernames to perform password attacks.

### Recommendation

* Disable anonymous RPC enumeration
* Restrict network access to RPC services

---

# Finding 2 – Credential Exposure in SMB Environment

**Severity:** High

### Description

Credential information was discovered during SMB enumeration.

Example credential:

```
V.Ventz : HotelCalifornia194!
```
<img width="1280" height="643" alt="image" src="https://github.com/user-attachments/assets/1dbe4edc-955b-4e75-963c-4f18a5806bbb" />

The credentials were discovered during enumeration activities. 

### Impact

Attackers could authenticate to domain services.

### Recommendation

* Implement strong password policies
* Monitor authentication attempts

---

# Finding 3 – Sensitive Files Exposed in SMB Share

**Severity:** Critical

### Description

An SMB share named **Password Audit** contained sensitive Active Directory files including:

* `ntds.dit`
* `SYSTEM`
* `SECURITY`

These files were downloadable using SMB client access. 

### Evidence

```
smbclient \\192.168.153.175\Password Audit -U resourced\V.Ventz
```
<img width="1187" height="466" alt="image" src="https://github.com/user-attachments/assets/8ae7c60c-ff01-45a4-afdc-81bbc16beb84" />

### Impact

Attackers could extract domain password hashes offline.

### Recommendation

* Restrict access to sensitive file shares
* Audit SMB permissions regularly

---

# Finding 4 – Active Directory Credential Extraction

**Severity:** Critical

### Description

Using the extracted AD database files, domain password hashes were recovered.

### Evidence

```
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```
<img width="1242" height="619" alt="image" src="https://github.com/user-attachments/assets/73487ca7-ff85-4398-b4cf-761c78d98bf6" />

Recovered credential example:

```
Administrator : ItachiUchiha888
```
<img width="1074" height="471" alt="image" src="https://github.com/user-attachments/assets/87ffbf36-f624-4810-9769-fee5aff8a9ce" />

### Impact

Full domain credential compromise.

### Recommendation

* Restrict access to Active Directory database files
* Monitor suspicious file access activity

---

# Finding 5 – Pass-the-Hash Authentication

**Severity:** Critical

### Description

Recovered NTLM hashes were used to authenticate remotely without needing plaintext passwords.

### Evidence

```
evil-winrm -i 192.168.153.175 \
-u resourced.local\L.Livingstone \
-H 19a3a7550ce8c505c2d46b5e39d6f808
```
<img width="1234" height="522" alt="image" src="https://github.com/user-attachments/assets/e84cfcd8-cb68-4cff-a514-d75fda4b2ce7" />

This allowed remote shell access to the system. 

### Impact

Attackers could execute commands on the compromised system.

### Recommendation

* Implement Credential Guard
* Reduce NTLM authentication usage

---

# Finding 6 – Active Directory Privilege Escalation via RBCD

**Severity:** Critical

### Description

The compromised account possessed **GenericAll permissions** over the **Account Operators group**.

This permission enabled **Resource-Based Constrained Delegation (RBCD)** exploitation.

### Evidence

A new machine account was created:

```
impacket-addcomputer resourced.local/l.livingstone
```
<img width="1239" height="299" alt="image" src="https://github.com/user-attachments/assets/f584933d-230c-4db6-861c-c79243396830" />

Delegation privileges were then configured:

```
rbcd.py
```
<img width="1266" height="462" alt="image" src="https://github.com/user-attachments/assets/e9df54e8-1d1d-4e19-a86c-ca37e4e991e4" />

Administrator privileges were obtained through Kerberos ticket impersonation.

### Impact

Attackers could impersonate domain administrator accounts.

### Recommendation

* Audit Active Directory permissions regularly
* Restrict GenericAll privileges
* Monitor delegation configuration changes

---

# Recommendations Summary

| Priority | Recommendation                                     |
| -------- | -------------------------------------------------- |
| Critical | Restrict access to Active Directory database files |
| Critical | Remove excessive Active Directory privileges       |
| High     | Enforce strong password policies                   |
| High     | Monitor Kerberos authentication activity           |
| Medium   | Disable anonymous RPC enumeration                  |
| Medium   | Audit SMB shares regularly                         |

---

# Conclusion

The Resourced Active Directory environment contained multiple weaknesses that allowed full domain compromise.

Starting from an unauthenticated network position, an attacker was able to:

* Enumerate domain users
* Discover valid credentials
* Extract Active Directory database files
* Recover domain password hashes
* Authenticate remotely via pass-the-hash
* Abuse Active Directory delegation permissions
* Obtain Domain Administrator privileges

Immediate remediation of the identified vulnerabilities is recommended to secure the environment.
