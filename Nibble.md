# Nibble – Web Application Exploitation Leading to Root Access

## Internal Linux Server Security Assessment

**Target:** Linux Server

**Service:** PostgreSQL Database

**Assessment Type:** Internal Penetration Test

**Author:** Tanvir Ahmed (OSCP+, CEH, AZ-500, SC-200)

**Date:** 2026

---

# Executive Summary

A penetration test was conducted against a Linux server hosting a PostgreSQL database service to evaluate potential security weaknesses.

The assessment identified several vulnerabilities that allowed an attacker to escalate privileges from an unauthenticated network position to **root-level access** on the system.

The compromise was achieved through the following attack chain:

* Network reconnaissance and service discovery
* Authentication using default PostgreSQL credentials
* Remote command execution through PostgreSQL
* Reverse shell access to the system
* Local privilege escalation via misconfigured SUID binary

These weaknesses ultimately allowed complete compromise of the Linux server.

### Business Impact

If exploited in a production environment, attackers could:

* access sensitive database information
* execute arbitrary commands on the server
* escalate privileges to root
* gain persistent control over the system

This represents a **critical risk to the security of the infrastructure**.

---

# Scope

| Target         | Description           |
| -------------- | --------------------- |
| 192.168.128.47 | Linux Database Server |

Service identified:

```text
PostgreSQL
```

Testing assumptions:

* Internal network access
* No initial credentials
* Limited knowledge of system configuration

---

# Methodology

The assessment followed a structured penetration testing methodology aligned with common industry standards.

### Testing Phases

1. Network reconnaissance
2. Service enumeration
3. Credential discovery
4. Remote exploitation
5. Local enumeration
6. Privilege escalation

### Tools Used

* Nmap
* PostgreSQL client (psql)
* Netcat
* Python
* LinPEAS

---

# Attack Path Overview

The following attack chain was successfully executed during the assessment.

```
Network Scan
      ↓
PostgreSQL Service Discovery
      ↓
Default Credential Authentication
      ↓
Remote Command Execution
      ↓
Reverse Shell
      ↓
Local Enumeration
      ↓
SUID Binary Discovery
      ↓
Privilege Escalation via find
      ↓
Root Access
```

---

# Findings

---

# Finding 1 – PostgreSQL Service Exposure

**Severity:** Medium

### Description

Network scanning identified an exposed PostgreSQL database service running on port **5437**. 

### Evidence

```bash
nmap -sV -sC -p- 192.168.128.47
```
<img width="1079" height="476" alt="image" src="https://github.com/user-attachments/assets/d27a3da8-5346-4141-ab9d-60d658d8200d" />

### Impact

Attackers could directly interact with the database service from the network.

### Recommendation

* Restrict database access to trusted hosts
* Implement firewall rules

---

# Finding 2 – Default Database Credentials

**Severity:** High

### Description

The PostgreSQL service accepted default credentials for the `postgres` user.

### Evidence

```bash
psql -h 192.168.128.47 -p 5437 -U postgres
```
<img width="1055" height="271" alt="image" src="https://github.com/user-attachments/assets/1c58cfcc-5ed9-486e-a614-6ff8162cd851" />

Authentication was successful using the default password. 

### Impact

Attackers could gain administrative access to the database.

### Recommendation

* Remove default credentials
* Implement strong password policies

---

# Finding 3 – PostgreSQL Remote Command Execution

**Severity:** Critical

### Description

A publicly available exploit was used to execute commands on the server through PostgreSQL.

### Evidence

```bash
python3 postgresql_rce.py
```
<img width="1041" height="788" alt="image" src="https://github.com/user-attachments/assets/6e8f123a-e15c-4769-a3d4-76f2cb2fecfc" />

This script established a reverse shell connection to the attacker system. 

### Impact

Attackers obtained remote shell access to the server.

### Recommendation

* Restrict database privileges
* Monitor database activity for suspicious commands

---

# Finding 4 – Local System Access

**Severity:** High

### Description

After exploitation, a shell was obtained under the `postgres` user account.

### Evidence

```bash
whoami
postgres
```

The attacker was able to access files belonging to other users. 

### Impact

Attackers could enumerate system files and search for privilege escalation opportunities.

### Recommendation

* Limit service account privileges
* Implement system monitoring

---

# Finding 5 – SUID Privilege Escalation

**Severity:** Critical

### Description

Local enumeration identified a SUID binary that could be abused to execute commands as root.

### Evidence

```bash
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```
<img width="1126" height="270" alt="image" src="https://github.com/user-attachments/assets/79d681d1-d78d-41eb-8e2b-215b9af8a856" />

The binary `/usr/bin/find` was discovered with SUID permissions. 

Privilege escalation was achieved using:

```bash
/usr/bin/find . -exec /bin/sh -p \; -quit
```
<img width="859" height="178" alt="image" src="https://github.com/user-attachments/assets/b6faa7a6-03db-4278-8ba7-e1301bda90bd" />

This resulted in root access. 

### Impact

Attackers could obtain full control of the system.

### Recommendation

* Remove unnecessary SUID permissions
* Audit privileged binaries regularly

---

# Recommendations Summary

| Priority | Recommendation                           |
| -------- | ---------------------------------------- |
| Critical | Remove default database credentials      |
| Critical | Restrict SUID binaries                   |
| High     | Monitor database authentication attempts |
| High     | Limit service account privileges         |
| Medium   | Restrict database network exposure       |

---

# Conclusion

The Linux server contained several vulnerabilities that allowed an attacker to escalate privileges from an unauthenticated network position to **root access**.

The attack chain involved:

* exploitation of default database credentials
* remote command execution via PostgreSQL
* reverse shell access
* privilege escalation through a SUID binary

Immediate remediation is recommended to prevent unauthorized access to the system.
