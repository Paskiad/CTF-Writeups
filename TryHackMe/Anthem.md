# THM — Anthem

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **OS:** Windows  
> **Tags:** `#thm` `#windows` `#umbraco` `#rdp` `#osint` `#credential-reuse` `#acl-abuse` `#privesc`

---

## Kill Chain Summary

```
Nmap Scan → Gobuster → robots.txt + /RSS + /umbraco
→ Passive OSINT (email pattern + admin name + password clue)
→ Umbraco Login → RDP (SG) → User Flag
→ Hidden Backup Folder → ACL Abuse (icacls) → Administrator Credentials → Root Flag
```

---

## 1. Reconnaissance — Nmap Scan

**Command:**

```bash
nmap -sV -sC -p- --min-rate 5000 <TARGET_IP>
```

**Key results:**

| Port | State | Service | Notes |
|------|-------|---------|-------|
| 80/tcp | open | HTTP | Web application |
| 3389/tcp | open | RDP | Microsoft Terminal Services |

The combination of HTTP and RDP defines the attack path: enumerate the web application first, extract credentials, then pivot to the desktop via RDP.

---

## 2. Web Enumeration

### Directory Busting — Gobuster

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

**Notable findings:**

| Path | Content |
|------|---------|
| `/robots.txt` | Contains a candidate password string |
| `/RSS` | Blog feed — reveals administrator name |
| `/umbraco` | CMS admin login panel |

### Passive OSINT — Credential Reconstruction

Manually reviewing the discovered resources surfaces three pieces of intelligence that, combined, reconstruct valid credentials without any active exploitation:

**1 — Password candidate from `/robots.txt`:**

```
UmbracoIsTheBest!
```

An unusual string for a robots.txt disallow directive — it reads as a passphrase.

**2 — Administrator name from `/RSS`:**

The blog feed reveals the administrator's full name: **Solomon Gurdy**.

**3 — Email pattern from the website:**

Analysing the site content reveals the company email convention:

```
FirstnameInitialLastname@ANTHEM.COM  →  e.g. JD@ANTHEM.COM
```

**Reconstructed credential:**

| Field | Value | Source |
|-------|-------|--------|
| Username | `SG@ANTHEM.COM` | Name (Solomon Gurdy) + email pattern |
| Password | `UmbracoIsTheBest!` | `/robots.txt` string |

> ⚠️ **Finding:** Three separate information sources, each individually benign, combine to produce a complete set of valid credentials. This is a classic example of **passive OSINT aggregation** — no vulnerability was exploited, only publicly accessible content was read.

![umbraco](https://github.com/user-attachments/assets/71ccf545-af0c-4761-b652-2ac99016917a)
---

## 3. Initial Access — Umbraco Admin Panel

Navigating to `http://<TARGET_IP>/umbraco` and logging in with the reconstructed credentials:

```
Username: SG@ANTHEM.COM
Password: UmbracoIsTheBest!
```

✅ Login successful — administrator access to the Umbraco CMS panel granted.

The panel does not expose critical data directly, but confirms the credentials are valid. The more important implication is **credential reuse** — the same password may work on other services.

---

## 4. Lateral Movement — RDP Access

Testing the same credentials against the open RDP service:

```bash
xfreerdp /v:<TARGET_IP> /u:SG /p:'UmbracoIsTheBest!'
```

✅ RDP session established as user `SG`.

**User Flag:**

```
C:\Users\SG\Desktop\user.txt
```

---

## 5. Privilege Escalation — ACL Abuse on Backup File

Exploring the filesystem, a hidden folder named `Backup` is found at the root of `C:\`. Inside it: `restore.txt`.

Attempting to read the file:

```cmd
type C:\Backup\restore.txt
# Access is denied.
```

The current user (`SG`) does not have read permissions. However, checking the ACL reveals that `SG` has **modify** (write/change permissions) rights on the file — meaning we can grant ourselves read access.

**Abuse — Grant read permission:**

```cmd
icacls C:\Backup\restore.txt /grant SG:R
```

**Read the file:**

```cmd
type C:\Backup\restore.txt
```

✅ The file contains the **Administrator account password**.

> ⚠️ **Finding:** A sensitive credential file was protected by ACL restrictions, but those restrictions were applied incorrectly — the target user was denied read access but retained the ability to modify permissions. The ability to change an ACL is equivalent to owning the file for practical purposes.

---

## 6. Root Flag — Administrator RDP Session

Using the recovered Administrator credentials, open a new RDP session:

```bash
xfreerdp /v:<TARGET_IP> /u:Administrator /p:'<recovered_password>'
```

**Root Flag:**

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

✅ Root flag captured.

---

## 7. Bonus — Web Flags via findstr

Additional flags embedded in the web application can be recovered from the Administrator shell by searching for the flag format across relevant config files:

```cmd
findstr /i "THM{" C:\inetpub\wwwroot\Web\App_Data\umbraco.config

findstr /i "THM{" "C:\inetpub\wwwroot\Web\App_Plugins\Articulate\Themes\VAPOR\Views\Partials\SearchBox.cshtml"
```

> 💡 `findstr /i` performs a case-insensitive string search across files — a fast way to locate flag patterns without manually opening each file.

---

## 8. Vulnerability Summary

| # | Finding | Severity | Detail |
|---|---------|----------|--------|
| 1 | Password exposed in `/robots.txt` | 🔴 High | Sensitive string stored in a publicly indexed, unauthenticated file |
| 2 | Administrator name leaked via RSS feed | 🟡 Medium | Personal information combined with email pattern enables credential construction |
| 3 | Credential reuse across web and RDP | 🔴 High | Single password valid for both Umbraco CMS and Windows RDP |
| 4 | Incorrect ACL on credential backup file | 🔴 Critical | User denied read access but retains permission modification rights — equivalent to full access |
| 5 | Plaintext credentials in backup file | 🔴 Critical | Administrator password stored unencrypted in a file accessible on the local filesystem |

---

## 9. Key Takeaways

- **Passive OSINT before active exploitation:** Every piece of content on a web application is potential intelligence. `robots.txt`, RSS feeds, blog posts, email footers, and page metadata can collectively reveal usernames, passwords, and internal naming conventions — before a single exploit is fired.
- **Credential reuse is systemic:** A password found on a web portal should be immediately tested against every other service exposed on the target (SSH, RDP, SMB, WinRM). The same password working on Umbraco and RDP is not a coincidence — it is a pattern.
- **ACL misconfiguration is a privileged escalation class of its own:** On Windows, the right to modify permissions (`WRITE_DAC`) on a sensitive file is functionally equivalent to reading it. Any enumeration of sensitive files should include `icacls` output, not just content accessibility.
- **`findstr` as a post-exploitation search tool:** `findstr /s /i "THM{" C:\` recursively searches for flag patterns across the entire drive — a fast, built-in alternative to `grep` that works in any Windows CMD session.

---

## Quick Reference — Key Commands

```bash
# Recon
nmap -sV -sC -p- --min-rate 5000 <TARGET_IP>

# Web enumeration
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt

# Manual OSINT checks
curl http://<TARGET_IP>/robots.txt
curl http://<TARGET_IP>/RSS

# RDP access
xfreerdp /v:<TARGET_IP> /u:SG /p:'UmbracoIsTheBest!'
xfreerdp /v:<TARGET_IP> /u:Administrator /p:'<recovered_password>'

# ACL abuse (Windows CMD)
icacls C:\Backup\restore.txt /grant SG:R
type C:\Backup\restore.txt

# Flags
type C:\Users\SG\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt

# Web flag search
findstr /i "THM{" C:\inetpub\wwwroot\Web\App_Data\umbraco.config
findstr /s /i "THM{" C:\inetpub\wwwroot\
```

---

*Writeup completed on 30/03/2026 — TryHackMe / Anthem*
