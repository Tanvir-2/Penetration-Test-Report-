
# Internal Active Directory Security Assessment

## Forest Environment

**Assessment Type:** Internal Penetration Test
**Target System:** Active Directory Domain Controller
**Domain:** htb.local
**Prepared by:** Tanvir Ahmed
**Date:** 2026

---

# 1. Executive Summary

An internal penetration test was conducted against the Active Directory environment to evaluate the impact of potential misconfigurations and authentication weaknesses.

During testing, multiple security issues were identified that allowed an attacker to escalate privileges from an unauthenticated position to **Domain Administrator**.

The compromise was achieved through a chain of vulnerabilities including:

* Kerberos misconfiguration enabling **AS-REP Roasting**
* Weak password usage on service accounts
* Excessive privileges assigned to user accounts
* Improper Active Directory access control configuration

By exploiting these weaknesses, an attacker was able to extract domain credentials and gain full administrative control over the environment.

### Business Impact

If exploited in a real enterprise network, these issues could allow attackers to:

* Access sensitive organizational data
* Modify or delete domain resources
* Deploy malware across the network
* Maintain persistent administrative access

This represents a **critical security risk** requiring immediate remediation.

---

# 2. Scope

| Target         | Description                        |
| -------------- | ---------------------------------- |
| 10.129.154.248 | Active Directory Domain Controller |

Testing assumptions:

* Internal network access
* No valid credentials at start
* No prior knowledge of environment

---

# 3. Methodology

The assessment followed a structured penetration-testing approach aligned with industry frameworks such as
OWASP and
NIST.

### Testing Phases

1. Network reconnaissance
2. Service enumeration
3. User enumeration
4. Credential access
5. Privilege escalation
6. Active Directory abuse
7. Domain compromise validation

Tools used included:

* Nmap
* Kerbrute
* BloodHound
* Impacket
* Evil-WinRM

---

# 4. Attack Path Overview

The following attack chain was successfully executed.

```
User Enumeration
      ↓
AS-REP Roasting
      ↓
Credential Cracking
      ↓
WinRM Access
      ↓
Active Directory Enumeration
      ↓
ACL Abuse
      ↓
DCSync Attack
      ↓
Domain Administrator
```

This chain demonstrates how a low-privileged attacker could fully compromise the domain.

---

# 5. Findings

---

# Finding 1 – Kerberos Pre-Authentication Disabled

**Severity:** High

### Description

Certain user accounts in the domain were configured without Kerberos pre-authentication. This allowed attackers to request encrypted authentication data from the domain controller without valid credentials.

### Impact

Attackers can perform **AS-REP Roasting**, retrieving Kerberos authentication material that can be cracked offline to recover user passwords.

### Evidence

```bash
impacket-GetNPUsers htb.local/ -dc-ip 10.129.154.248 -usersfile users.txt
```

The retrieved hashes were cracked offline, revealing valid credentials.

### Recommendation

* Enforce Kerberos pre-authentication for all accounts
* Audit domain accounts with this setting disabled
* Implement strong password policies

---

# Finding 2 – Weak Password on Service Account

**Severity:** High

### Description

The cracked Kerberos authentication data revealed valid credentials for the service account **svc-alfresco**.

### Impact

Attackers could authenticate to domain services and gain remote system access.

### Evidence

Password cracking was performed using an offline dictionary attack.

```bash
hashcat -m 18200 hashes.txt rockyou.txt
```

### Recommendation

* Enforce strong password policies
* Use long randomly generated service account credentials
* Implement account lockout controls

---

# Finding 3 – Remote Command Execution via WinRM

**Severity:** Medium

### Description

The compromised credentials allowed remote login via the Windows Remote Management (WinRM) service.

### Impact

Authenticated attackers could execute commands remotely on the system.

### Evidence

```bash
evil-winrm -i 10.129.154.248 -u svc-alfresco -p <password>
```

### Recommendation

* Restrict WinRM access to administrative hosts
* Monitor remote login activity
* Implement multi-factor authentication where possible

---

# Finding 4 – Excessive Privileges Assigned to User Accounts

**Severity:** Critical

### Description

Active Directory enumeration revealed that the compromised account had excessive privileges allowing modification of privileged groups.

### Impact

Improper privilege delegation allowed escalation of privileges within the domain.

### Evidence

Privilege escalation was achieved by abusing group membership permissions.

```powershell
Add-ADGroupMember -Identity "Exchange Windows Permissions" -Members attacker
```

### Recommendation

* Apply the principle of least privilege
* Audit membership of privileged groups
* Regularly review delegated permissions

---

# Finding 5 – Active Directory ACL Misconfiguration (DCSync)

**Severity:** Critical

### Description

Misconfigured access control lists allowed a non-privileged account to obtain **replication privileges** within the domain.

### Impact

This enabled a **DCSync attack**, allowing extraction of password hashes for all domain accounts.

### Evidence

```bash
impacket-secretsdump htb.local/user@10.129.154.248
```

### Recommendation

* Restrict replication permissions
* Audit domain ACL configurations
* Monitor for abnormal replication requests

---

# Finding 6 – NTLM Hash Authentication Abuse

**Severity:** Critical

### Description

Extracted NTLM hashes were used to authenticate as the domain administrator without needing the plaintext password.

### Impact

This resulted in full administrative access to the domain controller.

### Evidence

```bash
impacket-psexec administrator@10.129.154.248 -hashes <NTLM_HASH>
```

### Recommendation

* Reduce reliance on NTLM authentication
* Implement Credential Guard
* Monitor lateral movement activity

---

# 6. Recommendations Summary

| Priority | Recommendation                                  |
| -------- | ----------------------------------------------- |
| Critical | Enable Kerberos pre-authentication              |
| Critical | Review Active Directory ACL permissions         |
| High     | Reset weak service account passwords            |
| High     | Limit privileged group membership               |
| Medium   | Restrict WinRM remote management                |
| Medium   | Monitor authentication and replication activity |

---

# 7. Conclusion

The Active Directory environment contained multiple weaknesses that allowed a complete domain compromise.

Starting from an unauthenticated position, it was possible to:

* Enumerate domain users
* Extract authentication data
* Recover valid credentials
* Escalate privileges
* Extract domain password hashes
* Obtain Domain Administrator access

These findings highlight the importance of secure authentication configuration, least-privilege access controls, and continuous monitoring of domain activity.

Immediate remediation of the identified vulnerabilities is recommended to prevent similar attacks in production environments.
