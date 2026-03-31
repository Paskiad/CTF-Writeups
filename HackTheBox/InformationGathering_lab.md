# HTB Academy — Information Gathering: Web Edition
## Skills Assessment Writeup

**Platform:** Hack The Box Academy
**Module:** Information Gathering - Web Edition
**Difficulty:** Easy
**Date:** 2026-03-31
**Author:** Paki

---

## Objective

Apply all web reconnaissance techniques learned throughout the module to perform a comprehensive assessment against the target system, answering five specific questions by chaining multiple recon methods.

---

## Environment Setup

**Target:** `inlanefreight.htb` (HTB Academy Spawned Instance)

Before starting, added all required virtual hosts to `/etc/hosts`:
```bash
echo "<TARGET_IP>  inlanefreight.htb web1337.inlanefreight.htb" | sudo tee -a /etc/hosts
```

---

## Q1 — What is the IANA ID of the registrar of the inlanefreight.com domain?

**Technique:** WHOIS Lookup
**Type:** Passive Reconnaissance

### Methodology

Performed a standard WHOIS lookup against the public domain `inlanefreight.com` to retrieve domain registration details.
```bash
whois inlanefreight.com
```

### Result

<!-- Screenshot: whoislookup.png -->
<img width="1902" height="792" alt="whoislookup" src="https://github.com/user-attachments/assets/5ac3ffac-04e0-4224-bd91-98c5d3449e08" />

The output clearly shows:
```
Registrar: Amazon Registrar, Inc.
Registrar IANA ID: 468
```

**Answer:** `468`

---

## Q2 — What HTTP server software is powering the inlanefreight.htb site on the target system?

**Technique:** Banner Grabbing
**Type:** Active Reconnaissance

### Methodology

Performed banner grabbing using `curl -I` to retrieve only the HTTP response headers from the target, identifying the web server software and version.
```bash
curl -I http://inlanefreight.htb:<PORT>
```

### Result

<img width="1897" height="268" alt="softwaredet" src="https://github.com/user-attachments/assets/a744e6c5-7a16-4bee-9da9-b038c69891e9" />

![Banner Grabbing](./screenshots/softwaredet.png)

The `Server` header in the response reveals:
```
Server: nginx/1.26.1
```

> **Note:** Despite `inlanefreight.com` running Apache, the `.htb` target runs nginx — a reminder that fingerprinting must always be performed directly against the target, not assumed from public records.

**Answer:** `nginx`

---

## Q3 — What is the API key in the hidden admin directory?

**Technique:** VHost Fuzzing → Directory Enumeration → robots.txt Analysis
**Type:** Active Reconnaissance

### Methodology

#### Step 1 — VHost Fuzzing

Used `gobuster` in `vhost` mode to discover virtual hosts running on the target IP. Used SecLists subdomain wordlist with `--append-domain` to properly construct full hostnames.
```bash
gobuster vhost -u http://inlanefreight.htb:<PORT> \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  --append-domain -t 50
```
<img width="1903" height="607" alt="gobuster" src="https://github.com/user-attachments/assets/d720f8de-5d63-4d9f-a243-2e1f314dea0d" />


![VHost Fuzzing](./screenshots/gobuster.png)

Discovered VHost: **`web1337.inlanefreight.htb`** (`Status: 200`)

#### Step 2 — Directory Enumeration on Discovered VHost

Ran `gobuster dir` against the newly discovered VHost to enumerate accessible paths.
```bash
gobuster dir -u http://web1337.inlanefreight.htb:<PORT> \
  -w /usr/share/wordlists/dirb/common.txt
```

<img width="1900" height="447" alt="gobuster discoverhost" src="https://github.com/user-attachments/assets/87008262-df4d-4a58-9685-4af66bc88ba2" />

![Directory Enumeration](./screenshots/gobuster_discoverhost.png)

Found:
```
/index.html   (Status: 200)
/robots.txt   (Status: 200)
```

#### Step 3 — robots.txt Analysis

Fetched the `robots.txt` file from the discovered VHost, looking for disallowed paths that might reveal hidden directories.
```bash
curl http://web1337.inlanefreight.htb:<PORT>/robots.txt
```
<img width="1918" height="422" alt="robotspage" src="https://github.com/user-attachments/assets/3306f30e-610a-478a-a4b0-7061a9900ece" />


![robots.txt](./screenshots/robotspage.png
```
User-agent: *
Allow: /index.html
Allow: /index-2.html
Allow: /index-3.html
Disallow: /admin_h1dd3n
```

#### Step 4 — Access Hidden Admin Directory

Navigated to the disallowed path revealed by `robots.txt`:
```bash
curl http://web1337.inlanefreight.htb:<PORT>/admin_h1dd3n
```

Found the API key in plain text within the page content.

**Answer:** `e963d863ee0e82ba7080fbf558ca0d3f`

---

## Q4 — What is the email address found after crawling inlanefreight.htb?

**Technique:** Web Crawling with ReconSpider
**Type:** Active Reconnaissance

### Methodology

Used ReconSpider (Scrapy-based custom crawler) to crawl the `dev.web1337.inlanefreight.htb` subdomain discovered during previous enumeration steps. The crawler automatically extracts emails, links, files, JS references, and HTML comments.
```bash
# Activate virtual environment
source ~/scrapy-env/bin/activate

# Run crawler against the target subdomain
python3 ReconSpider.py http://dev.web1337.inlanefreight.htb:<PORT>

# Inspect results
cat results.json
```

### Result

The `emails` field in `results.json` contained:
```json
{
  "emails": [
    "1337testing@inlanefreight.htb"
  ]
}
```

**Answer:** `1337testing@inlanefreight.htb`

---

## Q5 — What is the API key the developers will be changing to?

**Technique:** Web Crawling — Source Code Analysis
**Type:** Active Reconnaissance

### Methodology

The same `results.json` generated by ReconSpider in the previous step contained additional findings. Inspecting the `comments` field in the JSON output revealed a new API key embedded in an HTML comment in the page source, indicating a planned key rotation.
```bash
cat results.json | jq '.comments'
```

### Result

Found within page comments or JavaScript:
```json
{
  "comments": ["ba988b835be4aa97d068941dc852ff33"]
}
```

**Answer:** `ba988b835be4aa97d068941dc852ff33`

---

## Attack Chain Summary
```
WHOIS (inlanefreight.com)
    → IANA ID: 468

curl -I (inlanefreight.htb)
    → Server: nginx/1.26.1

gobuster vhost (inlanefreight.htb)
    → web1337.inlanefreight.htb

gobuster dir (web1337.inlanefreight.htb)
    → /robots.txt

robots.txt analysis
    → Disallow: /admin_h1dd3n
    → API key: e963d863ee0e82ba7080fbf558ca0d3f

ReconSpider (dev.web1337.inlanefreight.htb)
    → Email: 1337testing@inlanefreight.htb
    → New API key: ba988b835be4aa97d068941dc852ff33
```

---

## Key Takeaways

- **Passive recon first:** WHOIS revealed registrar info without touching the target
- **Never assume:** `inlanefreight.com` runs Apache, but the `.htb` target runs nginx — always fingerprint directly
- **robots.txt is a map:** `Disallow` entries explicitly reveal what the owner wants to hide
- **VHost fuzzing unlocks hidden surfaces:** `web1337.inlanefreight.htb` had no public DNS record but was fully accessible
- **Crawlers find what humans miss:** ReconSpider extracted both the email and the new API key from page comments in a single run
- **Chain your findings:** every step fed the next — VHost → directory enum → robots.txt → admin panel → crawler → credentials

---

## Tools Used

| Tool | Purpose |
|---|---|
| `whois` | Domain registration info |
| `curl -I` | HTTP banner grabbing |
| `gobuster vhost` | Virtual host discovery |
| `gobuster dir` | Directory enumeration |
| `ReconSpider` | Web crawling and data extraction |
