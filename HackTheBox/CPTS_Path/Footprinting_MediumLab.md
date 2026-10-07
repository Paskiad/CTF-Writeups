# HTB Footprinting Lab — Medium

> **Target**: 10.129.202.41
> **Difficulty**: Medium
> **Category**: Footprinting

---

## Step 1 — Initial Nmap Scan

Started with a standard service version scan to identify open ports and running services.
```bash
nmap -sV -sS 10.129.202.41
```

<img width="1906" height="302" alt="ScanNmap1" src="https://github.com/user-attachments/assets/90dca2f7-79bb-4e9a-b3b7-cc195bf85787" />


The scan revealed several interesting open ports:

| Port | Service | Note |
|---|---|---|
| 111/tcp | rpcbind | RPC portmapper |
| 135/tcp | msrpc | Microsoft RPC |
| 139/445/tcp | netbios-ssn | SMB |
| 2049/tcp | nlockmgr | **NFS** |
| 3389/tcp | ms-wbt-server | RDP |
| 5985/tcp | http | WinRM |

> **Note**: No MSSQL port was visible — this turned out to be an internal service accessible only after gaining initial access via RDP.

---

## Step 2 — NFS Enumeration

Given port 2049 was open, NFS was the first service to investigate.
```bash
showmount -e 10.129.202.41
mkdir /tmp/nfs
sudo mount -t nfs 10.129.202.41:/TechSupport ./targetNFS -o nolock
sudo ls -lah targetNFS
sudo cat /tmp/nfs/TechSupport/ticket4238791283782.txt
```

<img width="1917" height="507" alt="foto cosa interessante trovata" src="https://github.com/user-attachments/assets/dfbdbe6b-0446-4a09-9311-88f8da9806c8" />


Inside the mounted share, a **support ticket conversation** was found between user `alex` and an operator. The ticket contained a **plaintext SMTP configuration file** that exposed valid credentials:
```
user="alex"
password="lol123!mD"
from="alex.g@web.dev.inlanefreight.htb"
```

---

## Step 3 — Credentials Found

From the NFS share we obtained the following credentials:

| Field | Value |
|---|---|
| **Username** | alex |
| **Password** | lol123!mD |

---

## Step 4 — SMB Enumeration

Using the credentials found in the NFS share, SMB shares were enumerated with CrackMapExec.
```bash
crackmapexec smb 10.129.202.41 -u 'alex' -p 'lol123!mD' --shares
```

<img width="1887" height="237" alt="enumsmb" src="https://github.com/user-attachments/assets/64943d26-7c14-4863-92d3-aacca836eb28" />


Several shares were identified on the **WINMEDIUM** host:

| Share | Permissions | Note |
|---|---|---|
| ADMIN$ | — | Remote Admin |
| C$ | — | Default share |
| **devshare** | **READ,WRITE** | Interesting! |
| IPC$ | READ | Remote IPC |
| Users | READ | — |

Connecting to the `devshare` share revealed a file named `important.txt` containing **MSSQL credentials**.
```bash
smbclient //10.129.202.41/devshare -U 'alex'
smb: \> get important.txt
```

<img width="1887" height="237" alt="enumsmb" src="https://github.com/user-attachments/assets/8d112798-f8f9-48ed-ab57-8bfb99d97c3b" />

---

## Step 5 — RDP Access

With the credentials `alex:lol123!mD`, an RDP session was established.
```bash
xfreerdp /u:alex /p:'lol123!mD' /v:10.129.202.41
```

<img width="1022" height="795" alt="desktop alex" src="https://github.com/user-attachments/assets/71c5fbcb-6ba1-43ef-acc0-0e4e255b2940" />


On alex's desktop, a shortcut to **Microsoft SQL Server Management Studio 18** was found. This explained why MSSQL was not visible in the initial Nmap scan — the service was running **internally** and not exposed on the network.

---

## Step 6 — MSSQL Enumeration via SSMS

Opened SSMS using the credentials found in `important.txt` and navigated to:
```
WINMEDIUM > Databases > accounts > Tables > dbo.devsacc
```

Scrolling through the records in the `dbo.devsacc` table revealed the **HTB flag** stored as a password entry for the `HTB` user.

<img width="1018" height="783" alt="SSQL database" src="https://github.com/user-attachments/assets/27454f38-5bc5-4a80-8016-cf35aaeea3dd" />


---

## Attack Chain
```
NFS share exposed (unauthenticated mount)
            |
Plaintext credentials in support ticket
            | alex:lol123!mD
SMB enumeration — devshare READ/WRITE
            |
important.txt — MSSQL credentials
            |
RDP access — SSMS shortcut on desktop
            |
MSSQL internal — dbo.devsacc table
            |
HTB{flag}
```

---

## Key Takeaways

- Always enumerate **NFS shares** — they are often misconfigured and accessible without authentication
- **Credential reuse** across services is extremely common — the same credentials worked for NFS, SMB and RDP
- Internal services like MSSQL may **not appear in Nmap scans** — gaining initial access through other vectors reveals hidden services
- **SMB shares with READ/WRITE permissions** often contain sensitive files left by developers or administrators
- Support tickets and chat logs in exposed shares can contain **plaintext credentials**
