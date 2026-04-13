# Extplorer – Web Application Exploitation and Linux Privilege Escalation

## Internal Web Application Security Assessment

**Target:** Linux Web Server

**Application:** eXtplorer File Manager

**Assessment Type:** Internal Penetration Test

**Author:** Tanvir Ahmed (OSCP+, CEH, AZ-500, SC-200)

**Date:** 2026

---

# Executive Summary

A penetration test was conducted against a Linux web server hosting a WordPress website and an exposed **eXtplorer file management application**.

The assessment identified multiple vulnerabilities that allowed an attacker to escalate privileges from an unauthenticated web user to **root-level access** on the server.

The compromise was achieved through the following attack chain:

* Web directory enumeration
* Authentication using weak default credentials
* File upload vulnerability leading to remote command execution
* Reverse shell access
* Credential discovery in configuration files
* Privilege escalation through disk group permissions
* Extraction and cracking of the root password hash

These vulnerabilities ultimately resulted in **complete compromise of the Linux system**.

### Business Impact

If exploited in a production environment, attackers could:

* upload malicious files to the server
* execute arbitrary commands
* obtain sensitive credentials
* escalate privileges to root
* maintain persistent access to the system

This represents a **critical security risk to the server infrastructure**.

---

# Scope

| Target         | Description      |
| -------------- | ---------------- |
| 192.168.143.16 | Linux Web Server |

Application identified:

```text
eXtplorer File Manager
```
<img width="1136" height="582" alt="image" src="https://github.com/user-attachments/assets/06a45193-c9c1-4862-a627-eaa3f1aecdbd" />

Testing assumptions:

* Internal network access
* No initial credentials
* Web services exposed via HTTP

---

# Methodology

The assessment followed a structured penetration testing methodology aligned with industry frameworks.

### Testing Phases

1. Network reconnaissance
2. Web application enumeration
3. Vulnerability identification
4. Exploitation
5. Post-exploitation
6. Privilege escalation

### Tools Used

* Nmap
* Feroxbuster
* Netcat
* Hashcat
* LinPEAS
* John the Ripper

---

# Attack Path Overview

The following attack chain was executed during the engagement.

```
Network Scan
      ↓
Web Enumeration
      ↓
eXtplorer Discovery
      ↓
Weak Credential Authentication
      ↓
File Upload Exploit
      ↓
Web Shell Access
      ↓
Reverse Shell
      ↓
Credential Discovery
      ↓
Privilege Escalation via Disk Group
      ↓
Root Access
```

---

# Findings

---

# Finding 1 – Weak Web Application Credentials

**Severity:** High

### Description

The eXtplorer web application allowed authentication using weak credentials.

### Evidence

```
Username: admin
Password: admin
```

Successful login to `/filemanager`. 
<img width="1094" height="639" alt="image" src="https://github.com/user-attachments/assets/ed6dacb1-6f08-4d22-b95e-7f8cc7148196" />

### Impact

Attackers could gain administrative access to the web application.

### Recommendation

* Enforce strong password policies
* Implement account lockout controls
* Use multi-factor authentication

---

# Finding 2 – Arbitrary File Upload

**Severity:** Critical

### Description

The file manager allowed uploading executable PHP files.

### Evidence

Example web shell:

```php
<?php system($_REQUEST["cmd"]); ?>
```
<img width="628" height="169" alt="image" src="https://github.com/user-attachments/assets/21b662f3-db3a-49c2-8db4-30e51257289a" />

Command execution confirmed via browser request. 

### Impact

Attackers could execute arbitrary commands on the server.

### Recommendation

* Restrict file upload types
* Validate file content server-side
* Disable execution in upload directories

---

# Finding 3 – Remote Command Execution

**Severity:** Critical

### Description

The uploaded PHP shell allowed remote command execution.

### Evidence

Example command execution:

```
http://target-ip/simple_cmd.php?cmd=id
```
<img width="1118" height="185" alt="image" src="https://github.com/user-attachments/assets/352be5d2-ae06-439a-b845-0207f9c02e61" />

This confirmed execution as the `www-data` user. 

### Impact

Attackers could run system commands remotely.

### Recommendation

* Implement strict input validation
* Remove unnecessary file upload functionality

---

# Finding 4 – Reverse Shell Access

**Severity:** Critical

### Description

A reverse shell was established using the pentestmonkey PHP reverse shell.

### Evidence

```
nc -lvnp 8888
```
<img width="986" height="231" alt="image" src="https://github.com/user-attachments/assets/315a3a9f-e08d-47e1-ac7a-76daa6e11b71" />

Reverse connection successfully obtained. 

### Impact

Attackers gained interactive shell access to the server.

### Recommendation

* Monitor suspicious outbound network connections
* Implement egress filtering

---

# Finding 5 – Credential Exposure in Configuration Files

**Severity:** High

### Description

Sensitive credential hashes were discovered in application configuration files.

### Evidence

Hash extracted from the eXtplorer configuration file. 

Password cracking result:

```
dora : doraemon
```
<img width="1061" height="370" alt="image" src="https://github.com/user-attachments/assets/86f13ae0-e087-4b35-9e6a-5f48ed7c7a36" />

### Impact

Attackers could authenticate as a system user.

### Recommendation

* Store credentials securely
* Use strong password hashing algorithms
* Restrict access to configuration files

---

# Finding 6 – Privilege Escalation via Disk Group

**Severity:** Critical

### Description

The compromised user belonged to the **disk group**, allowing direct access to raw disk devices.

### Evidence

Using `debugfs`, attackers were able to read sensitive files including `/etc/shadow`. 

<img width="1117" height="312" alt="image" src="https://github.com/user-attachments/assets/9494cee9-cad7-4921-946a-5ab6578d1f7b" />

Example command:

```
debugfs -w /dev/mapper/ubuntu--vg-ubuntu--lv
```
<img width="1124" height="615" alt="image" src="https://github.com/user-attachments/assets/3789d3d8-bf3c-4851-9005-2ebff3fcf4d1" />

The extracted root password hash was cracked offline. 

### Impact

Attackers could obtain root credentials and fully compromise the system.

### Recommendation

* Remove unnecessary disk group memberships
* Restrict access to block devices
* Implement least privilege principles

---

# Recommendations Summary

| Priority | Recommendation                             |
| -------- | ------------------------------------------ |
| Critical | Remove arbitrary file upload vulnerability |
| Critical | Restrict disk group privileges             |
| High     | Remove weak credentials                    |
| High     | Protect configuration files                |
| Medium   | Monitor web application activity           |

---

# Conclusion

The Extplorer server contained several vulnerabilities that allowed attackers to escalate privileges from an unauthenticated web user to **root access**.

The attack chain involved:

* exploitation of weak credentials
* file upload leading to remote code execution
* reverse shell access
* credential discovery
* privilege escalation through disk group access

Immediate remediation of these vulnerabilities is recommended to prevent unauthorized system compromise.
