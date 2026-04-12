# Hutch – Windows Privilege Escalation via LAPS Misconfiguration

## Internal Active Directory Security Assessment

**Target:** Hutch Domain Controller

**Domain:** hutch.offsec

**Assessment Type:** Internal Penetration Test

**Author:** Tanvir Ahmed (OSCP+, CEH, AZ-500, SC-200)

**Date:** 2026

---

# Executive Summary

An internal penetration test was conducted against the **Hutch Active Directory environment** to evaluate potential weaknesses that could allow unauthorized privilege escalation.

During testing, several vulnerabilities were identified that allowed an attacker to escalate privileges from an unauthenticated network position to **Domain Administrator**.

The compromise was achieved through the following attack chain:

* Anonymous LDAP enumeration
* Discovery of credentials stored in user attributes
* Authentication using exposed credentials
* Active Directory enumeration using BloodHound
* Abuse of LAPS configuration
* Retrieval of Domain Administrator credentials
* Remote command execution on the domain controller

These weaknesses ultimately resulted in **complete compromise of the domain environment**.

### Business Impact

If exploited in a production network, attackers could:

* obtain administrative credentials
* gain unrestricted access to domain resources
* execute commands across enterprise systems
* maintain persistent control over the network

This represents a **critical security risk to the organization's infrastructure**.

---

# Scope

| Target          | Description       |
| --------------- | ----------------- |
| 192.168.230.122 | Domain Controller |

Domain:

```text
hutch.offsec
```
<img width="719" height="187" alt="image" src="https://github.com/user-attachments/assets/d1708dbe-ae81-4c3d-8bf0-e89f5e8d7edd" />

Testing assumptions:

* Internal network access
* No initial credentials
* Limited prior knowledge of the environment

---

# Methodology

The assessment followed a structured penetration testing methodology aligned with industry standards including:

* OWASP Testing Guide
* NIST Penetration Testing Framework

### Testing Phases

1. Network reconnaissance
2. Service enumeration
3. User enumeration
4. Credential discovery
5. Active Directory enumeration
6. Privilege escalation
7. Domain compromise validation

### Tools Used

* Nmap
* LDAPSearch
* Kerbrute
* NetExec (nxc)
* BloodHound
* Impacket

---

# Attack Path Overview

The following attack chain was executed during the engagement.

```
Network Scan
      ↓
LDAP Enumeration
      ↓
Domain User Discovery
      ↓
Credential Exposure
      ↓
Domain Authentication
      ↓
Active Directory Enumeration
      ↓
LAPS Misconfiguration
      ↓
Domain Administrator Password Retrieval
      ↓
Administrator Shell
```

---

# Findings

---

# Finding 1 – Anonymous LDAP Information Disclosure

**Severity:** Medium

### Description

The LDAP service allowed anonymous queries which exposed domain user information.

### Evidence

```
ldapsearch -x -H ldap://192.168.230.122 -b "dc=hutch,dc=offsec"
```
<img width="1264" height="412" alt="image" src="https://github.com/user-attachments/assets/a4520ae9-9a78-4329-9c77-145f03fcd4fa" />

This query returned multiple domain usernames. 

### Impact

Attackers can enumerate valid domain accounts for further attacks such as password spraying.

### Recommendation

* Disable anonymous LDAP queries
* Restrict directory access to authenticated users

---

# Finding 2 – Credential Exposure in LDAP Description Field

**Severity:** High

### Description

Sensitive information was discovered within the LDAP description attribute.

### Evidence

```
description: CrabSharkJellyfish192
```
<img width="1260" height="681" alt="image" src="https://github.com/user-attachments/assets/391da7ef-553d-4520-a0db-63b44c8b8ae1" />

This value corresponded to a valid domain password. 

### Impact

Attackers could obtain valid credentials and authenticate to domain services.

### Recommendation

* Avoid storing credentials in directory attributes
* Implement credential monitoring and rotation policies

---

# Finding 3 – Successful Domain Authentication

**Severity:** High

### Description

Using the discovered password, valid domain authentication was achieved.

### Evidence

```
nxc smb 192.168.230.122 -u hutch_usr.txt -p CrabSharkJellyfish192
```
<img width="1250" height="464" alt="image" src="https://github.com/user-attachments/assets/cbdffead-887e-405c-bd31-f3fa6977c0d7" />

This confirmed successful authentication for domain users. 

### Impact

Attackers gained a foothold in the Active Directory environment.

### Recommendation

* Implement strong password policies
* Enforce multi-factor authentication where possible

---

# Finding 4 – Active Directory Privilege Escalation via LAPS

**Severity:** Critical

### Description

Active Directory enumeration revealed that the compromised account had permission to read **LAPS passwords** for domain systems.

### Evidence

```
nxc ldap 192.168.230.122 -u fmcsorley -p CrabSharkJellyfish192 -M laps
```
<img width="1228" height="194" alt="image" src="https://github.com/user-attachments/assets/eea65693-ba49-4b7a-95b4-0c7ed1e51ae3" />

This command successfully retrieved the LAPS password for the domain controller. 

### Impact

Attackers could retrieve administrative passwords for domain computers.

### Recommendation

* Restrict access to LAPS password attributes
* Implement strict access control policies

---

# Finding 5 – Domain Administrator Access

**Severity:** Critical

### Description

Using the recovered LAPS password, the attacker authenticated as the Domain Administrator.

### Evidence

```
impacket-psexec Administrator@192.168.230.122
```
<img width="1182" height="669" alt="image" src="https://github.com/user-attachments/assets/d4a813fd-491e-4ad7-a481-ac76c6a89970" />

Administrative command execution was successfully obtained. 

### Impact

Full control of the domain controller and Active Directory environment.

### Recommendation

* Monitor administrative authentication attempts
* Rotate administrative credentials regularly
* Implement privileged access monitoring

---

# Recommendations Summary

| Priority | Recommendation                                    |
| -------- | ------------------------------------------------- |
| Critical | Restrict access to LAPS password attributes       |
| Critical | Remove credentials stored in directory attributes |
| High     | Implement strong password policies                |
| High     | Monitor Active Directory authentication activity  |
| Medium   | Disable anonymous LDAP enumeration                |

---

# Conclusion

The Hutch Active Directory environment contained multiple weaknesses that allowed an attacker to escalate privileges from an unauthenticated position to **Domain Administrator**.

Starting from anonymous access, it was possible to:

* enumerate domain users
* discover credentials in LDAP attributes
* authenticate to the domain
* enumerate Active Directory permissions
* retrieve LAPS passwords
* gain Domain Administrator access

Immediate remediation of these vulnerabilities is recommended to secure the domain infrastructure.
