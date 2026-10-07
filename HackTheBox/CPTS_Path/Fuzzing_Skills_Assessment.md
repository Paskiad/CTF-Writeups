# Attacking Web Applications with Ffuf — Skills Assessment

## Overview

For this skills assessment we were given only an IP address with no further explanation.
**Target:** `154.57.164.70:31160`

The methodology below follows a single principle: **each step feeds the next**. We move from the outermost layer (virtual hosts) down to the innermost (parameter values), never skipping a stage.

---

## Step 1 — Subdomain vs VHost Fuzzing

The first task asks us to enumerate the subdomains of the target. We start with classic subdomain fuzzing:

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
     -u http://FUZZ.academy.htb/
```

This returns **no results**. That's expected: `academy.htb` is not a public domain, so it has no public DNS records. Subdomain fuzzing relies on the public DNS to resolve each candidate — since the DNS knows nothing about these hosts, every request fails.

The solution is **VHost fuzzing**. Instead of relying on DNS resolution, we send every request straight to the target IP and fuzz the `Host` header. This reaches virtual hosts served on the same IP, whether or not they have public DNS records.

First, we run the scan without a filter to learn the size of the "false positive" responses (every request returns `200 OK` because we're always hitting the same IP):

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
     -u http://academy.htb:31160/ \
     -H 'Host: FUZZ.academy.htb'
```

The common response size is **985**. We re-run with `-fs 985` to filter out the noise:

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
     -u http://academy.htb:31160/ \
     -H 'Host: FUZZ.academy.htb' \
     -fs 985
```

This reveals **three virtual hosts**: `archive`, `test`, and `faculty`.

<img width="1517" height="530" alt="vhost_fuzzing_result" src="https://github.com/user-attachments/assets/7ccb71de-c46f-4f4c-8fb9-dd487dbe9069" />


We immediately add all three to `/etc/hosts` so we can reach them by name:

```bash
sudo nano /etc/hosts
```

<img width="713" height="248" alt="aggiunta_vhost" src="https://github.com/user-attachments/assets/71884d37-63fc-43b9-9fa3-e7b12337afeb" />


---

## Step 2 — Extension Fuzzing

With the three vhosts mapped, we need to know what file extensions the applications use before we can fuzz for pages. We fuzz the extension against `index.*` — a file that almost always exists — on each vhost:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ \
     -u http://faculty.academy.htb:31160/indexFUZZ \
     -ic

ffuf -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ \
     -u http://test.academy.htb:31160/indexFUZZ \
     -ic

ffuf -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ \
     -u http://archive.academy.htb:31160/indexFUZZ \
     -ic
```

> **Note:** the `web-extensions.txt` wordlist already contains the leading dot, so we write `indexFUZZ`, not `index.FUZZ`.

We find three interesting extensions in play: `.php`, `.php7`, and `.phps`.

<img width="1252" height="482" alt="vhost_extension_fuzzing" src="https://github.com/user-attachments/assets/c4650ccf-5bbc-4593-af10-4b9fcef9ac84" />


---

## Step 3 — Recursive Fuzzing

Now we fuzz for directories and pages across each vhost, feeding in the extensions we just discovered. Recursive fuzzing automates the process: whenever ffuf finds a directory, it queues a new scan inside it.

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/big.txt:FUZZ \
     -u http://faculty.academy.htb:31160/FUZZ \
     -recursion \
     -recursion-depth 1 \
     -e .php,.php7,.phps \
     -v \
     -ic
```

This uncovers an interesting page: **`linux-security.php7`** under the `/courses/` directory.

---

## Step 4 — Parameter Fuzzing (GET)

The page appears to expect some kind of identifier to grant access. We fuzz for a valid GET parameter name, placing `FUZZ` where the parameter key would go.

First without a filter to measure the default response size:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
     -u http://faculty.academy.htb:31160/courses/linux-security.php7?FUZZ=key
```

Then we filter the default size (**774**):

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
     -u http://faculty.academy.htb:31160/courses/linux-security.php7?FUZZ=key \
     -fs 774
```

Two parameters come back: **`user`** and **`username`**.

<img width="1237" height="471" alt="fuzzing_parameters" src="https://github.com/user-attachments/assets/0de8dddb-b94e-4631-b178-e1762af4ed12" />


Testing them in the browser tells us which one is live. Visiting `?user=key` returns **"This method is no longer used"** — the `user` parameter is deprecated:

<img width="1842" height="485" alt="deprecated_user" src="https://github.com/user-attachments/assets/11ac1fbf-6c35-4d2c-a404-5f58e06ccee2" />


Visiting `?username=key` instead returns **"You don't have access!"** — this confirms `username` is the valid, active parameter. It's accepted, but the value is wrong:

<img width="1801" height="452" alt="valid" src="https://github.com/user-attachments/assets/6f9069e7-5555-4e84-bfe6-278203ded18e" />


---

## Step 5 — Value Fuzzing

We know the parameter is `username`; now we need the correct value. Since usernames are the expected input, we use a username wordlist and fuzz the value via POST.

First without a filter to read the default size:

```bash
ffuf -w /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt:FUZZ \
     -u http://faculty.academy.htb:31160/courses/linux-security.php7 \
     -X POST \
     -d 'username=FUZZ' \
     -H 'Content-Type: application/x-www-form-urlencoded'
```

Then we filter the default size (**781**):

```bash
ffuf -w /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt:FUZZ \
     -u http://faculty.academy.htb:31160/courses/linux-security.php7 \
     -X POST \
     -d 'username=FUZZ' \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -fs 781
```

> **Note:** the `Content-Type: application/x-www-form-urlencoded` header is mandatory for PHP to parse the POST body correctly.

The scan returns a valid username: **`harry`**.

<img width="1517" height="483" alt="fuzzed user harry" src="https://github.com/user-attachments/assets/639ae0d6-1e98-41f2-920f-dcf04bf9e57d" />


---

## Step 6 — Capturing the Flag

With the correct username in hand, we send a final POST request with `curl`:

```bash
curl http://faculty.academy.htb:31160/courses/linux-security.php7 \
     -X POST \
     -d 'username=harry' \
     -H 'Content-Type: application/x-www-form-urlencoded'
```

The server returns the flag:

<img width="1600" height="671" alt="flag" src="https://github.com/user-attachments/assets/6b082c0e-2d07-43b6-8ffe-d5c25721d932" />

A few lessons worth internalising:

- **Subdomain fuzzing fails on non-public domains** — reach for VHost fuzzing and target the IP directly.
- **Always measure the default response size first**, then filter it with `-fs` to cut through the noise.
- **A "deprecated" parameter is still a signal** — it confirms the app accepts parameters and narrows the search to the active one.
- **Match the wordlist to the expected value type** — usernames call for a username list, numeric IDs call for a generated sequence.
