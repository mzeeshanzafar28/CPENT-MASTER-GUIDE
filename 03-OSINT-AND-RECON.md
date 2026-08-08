# 03 — OSINT & RECONNAISSANCE

> **Author:** Zeeshan  
> **GitHub:** https://github.com/mzeeshanzafar28  
> **LinkedIn:** https://www.linkedin.com/in/mzeeshanzafar28

OSINT (Open-Source Intelligence) is the first real phase of every engagement. In CPENT you use it to build an accurate picture of the target — domains, subdomains, IP ranges, email addresses, personnel, technologies, and exposed infrastructure — before you send a single aggressive packet.

---

## Table of Contents

1. [OSINT Workflow & Mindset](#1-osint-workflow--mindset)
2. [Passive vs Active Reconnaissance](#2-passive-vs-active-reconnaissance)
3. [Domain & Subdomain Enumeration](#3-domain--subdomain-enumeration)
4. [Similar / Typosquat Domains](#4-similar--typosquat-domains)
5. [Google Dorks & Google Hacking Database (GHDB)](#5-google-dorks--google-hacking-database-ghdb)
6. [Shodan, Censys & Internet-Wide Search](#6-shodan-censys--internet-wide-search)
7. [WHOIS, RDAP & IP Block Identification](#7-whois-rdap--ip-block-identification)
8. [DNS Records & Zone Transfers](#8-dns-records--zone-transfers)
9. [Reverse DNS Lookups](#9-reverse-dns-lookups)
10. [Email Harvesting & Document Metadata](#10-email-harvesting--document-metadata)
11. [Personnel Discovery & Job Post Analysis](#11-personnel-discovery--job-post-analysis)
12. [Technology & Network Device Identification](#12-technology--network-device-identification)
13. [Email Header Analysis](#13-email-header-analysis)
14. [Network Diagramming & Traceroute](#14-network-diagramming--traceroute)
15. [Automation & Frameworks](#15-automation--frameworks)
16. [Practical OSINT Workflow (Copy-Paste Ready)](#16-practical-osint-workflow-copy-paste-ready)
17. [Tools & Techniques Mapping](#17-tools--techniques-mapping)
18. [Practice & Labs](#18-practice--labs)

---

## 1. OSINT Workflow & Mindset

OSINT should answer:

```
What domains belong to the organization?
What subdomains exist?
Which IP ranges belong to it?
Who provides its DNS and where is infrastructure hosted?
What technologies are exposed?
Which email addresses are publicly associated?
What historical assets existed?
What personnel, departments, and vendors can be inferred?
What external attack surface should be tested next?
```

**Intelligence lifecycle:**
```
Direction → Collection → Processing → Analysis → Validation → Dissemination
```

**Golden rule:**
```
In scope? → Current? → Owned by target? → Actually resolving? → Relevant?
```
Never assume that because something is publicly associated with a company, it is automatically authorized for testing.

---

## 2. Passive vs Active Reconnaissance

### Passive OSINT
Uses information that already exists publicly. The target's infrastructure is NOT directly queried:

```
Search engines, Certificate Transparency (crt.sh), WHOIS/RDAP
Public DNS information, Public company documents, Job postings
Public repositories, Internet archives (Wayback Machine), Breach notifications
Shodan/Censys indexes, Social networks, News articles, Public cloud metadata
```

### Active Reconnaissance
Directly interacts with target infrastructure — can be logged and may be prohibited outside scope:

```
DNS brute forcing, DNS queries against target nameservers
Zone-transfer attempts, Port scanning, Service enumeration
Web crawling, Directory enumeration
```

---

## 3. Domain & Subdomain Enumeration

Start with the primary domain from scope, then expand. Use multiple sources — never rely on a single tool.

### 3.1 Passive Discovery (No Target Interaction)

**Certificate Transparency (crt.sh)**
```bash
curl -s "https://crt.sh/?q=%25.example.com&output=json" | jq -r '.[].name_value' | sort -u > crtsh-subs.txt
```

**Subfinder (passive, fast)**
```bash
subfinder -d example.com -all -o subs-passive.txt
subfinder -d example.com -silent | httpx -silent -o live-subs.txt
```

**Amass (passive then active)**
```bash
# Passive only (no direct target queries)
amass enum -passive -d example.com -o amass-passive.txt

# Active (only when authorized)
amass enum -active -d example.com -o amass-active.txt
```

**Assetfinder**
```bash
assetfinder --subs-only example.com | tee assetfinder-subs.txt
```

### 3.2 Active DNS Enumeration

**Nmap DNS Brute (always first for scanning)**
```bash
nmap --script dns-brute --script-args dns-brute.domain=example.com
```

**Gobuster DNS Mode**
```bash
gobuster dns -d example.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -o gobuster-dns.txt
```

**dnsenum** — combines brute force + zone transfer attempts
```bash
dnsenum example.com
dnsenum -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt example.com
```

**dnsrecon** — standard + brute force + zone transfer
```bash
dnsrecon -d example.com -t std
dnsrecon -d example.com -t brt -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

**Fierce**
```bash
fierce --domain example.com
```

**dnsmap**
```bash
dnsmap example.com
```

**Subbrute**
```bash
./subbrute.py example.com
```

**Sublist3r**
```bash
python sublister.py -d example.com -p 80 -e Bing
```

**Combine and deduplicate all results:**
```bash
cat *.txt | sort -u > all-subs.txt
```

### 3.3 Validate Live Hosts

```bash
# Check which subdomains resolve and respond
cat all-subs.txt | httpx -silent -o live-subs.txt

# Quick check for web servers
cat all-subs.txt | httpx -silent -status-code -title -tech-detect -o subs-detailed.txt
```

### 3.4 Wildcard DNS Detection

Wildcard DNS creates false positives. Always test:
```bash
dig random-not-real-123456.example.com
```
If it resolves, the domain has wildcard DNS. Validate brute-force results carefully.

---

## 4. Similar / Typosquat Domains

Attackers register lookalike domains for phishing. As a pentester, check if your target has already been typosquatted.

**urlcrazy**
```bash
urlcrazy -p example.com
```

**dnstwist** (modern alternative)
```bash
dnstwist example.com
dnstwist --format csv example.com > typosquat.csv
```

Classification of findings:
```
Typo, Character substitution, Hyphenation, Transposition,
TLD variation (.com → .net, .org), Homoglyph (exαmple.com)
```

Also manually check:
- Regional TLDs (.co, .net, .io, .org variants)
- Acquired brand domains
- Are these owned by the client? Malicious? Parked? Hosting content?

---

## 5. Google Dorks & Google Hacking Database (GHDB)

### 5.1 Core Search Operators

| Operator | Purpose | Example |
|---|---|---|
| `site:` | Limit to domain | `site:example.com` |
| `filetype:` / `ext:` | Specific file types | `site:example.com filetype:pdf` |
| `inurl:` | Keyword in URL | `site:example.com inurl:admin` |
| `intitle:` | Keyword in page title | `intitle:"index of" site:example.com` |
| `intext:` | Keyword in page body | `site:example.com intext:"password"` |
| `-` | Exclude keyword | `site:example.com -www` |

### 5.2 High-Value Dork Examples (Exam-Ready)

```
# Exposed documents
site:example.com filetype:pdf
site:example.com filetype:docx
site:example.com filetype:xlsx
site:example.com ext:log
site:example.com ext:bak
site:example.com ext:sql

# Login portals & admin interfaces
site:example.com inurl:login
site:example.com inurl:admin
site:example.com intitle:"admin panel"
site:example.com intitle:"login"

# Sensitive keywords
site:example.com "password"
site:example.com "confidential"
site:example.com "internal use only"

# Directory listings
site:example.com intitle:"index of"
site:example.com intitle:"index of" "parent directory"

# Configuration files
site:example.com inurl:wp-config
site:example.com ext:env
site:example.com inurl:.git

# Personnel / email
site:example.com "@example.com" filetype:pdf
"example.com" "CV" OR "resume" filetype:pdf

# GitHub leaks
site:github.com "example.com" password
site:github.com "example.com" "BEGIN PRIVATE KEY"
```

### 5.3 Google Hacking Database (GHDB)

Available at exploit-db.com/google-hacking-database. Categories:
```
Exposed files, Login portals, Configuration files, Cameras/devices
Error messages, Backup files, Credentials, Network infrastructure
```

**SiteDigger / Diggity** — GUI tools that automate GHDB queries against a target domain.

---

## 6. Shodan, Censys & Internet-Wide Search

### 6.1 Shodan CLI

```bash
# Initialize (requires API key from shodan.io)
shodan init YOUR_API_KEY

# Search by organization
shodan search 'org:"Example Corp"' --fields ip_str,port,org,hostnames

# Search by domain / SSL subject
shodan search 'ssl:"example.com"'

# Search exposed SMB
shodan search 'port:445 org:"Example Corp"'

# Download results
shodan download results.json.gz 'org:"Example Corp"'
shodan parse results.json.gz --fields ip_str,port,hostnames
```

### 6.2 Shodan Web Queries

```
org:"Example Company"
ssl:"example.com"
hostname:example.com
port:3389 org:"Example"
port:22 city:"New York"
product:"Apache httpd"
```

### 6.3 Censys

Censys provides host, certificate, and domain intelligence. Correlate with DNS, CT logs, WHOIS, and Shodan.

### 6.4 OSINT-to-Exploitation Path

```
vpn.example.com → 203.0.113.10 → Shodan → Censys → DNS → TLS cert → Reverse DNS
                                                  ↓
                                         Strong intelligence picture
```

---

## 7. WHOIS, RDAP & IP Block Identification

### 7.1 WHOIS Lookups

```bash
# Domain WHOIS
whois example.com

# IP WHOIS
whois 203.0.113.10

# Nmap WHOIS scripts
nmap --script whois-domain example.com
nmap --script whois-ip,whois-domain -iL targets.txt
```

Look for: Registrar, Creation/Expiry dates, Nameservers, Registrant organization, Admin/Tech contacts (increasingly redacted by GDPR).

### 7.2 Regional Internet Registries (RIRs)

| Registry | Region |
|---|---|
| **ARIN** (arin.net) | North America |
| **RIPE NCC** (ripe.net) | Europe, Middle East, Central Asia |
| **APNIC** (apnic.net) | Asia-Pacific |
| **LACNIC** | Latin America |
| **AFRINIC** | Africa |

```bash
# Query ARIN for netblock
whois -h whois.arin.net "n 192.0.2.0"
```

### 7.3 Finding Nameservers

```bash
dig NS example.com
```
Investigate: Who operates them? Cloud-managed? What other domains share them?

---

## 8. DNS Records & Zone Transfers

### 8.1 Record Types

| Record | Intelligence |
|---|---|
| **A** | IPv4 address |
| **AAAA** | IPv6 address |
| **CNAME** | Alias/dependency |
| **MX** | Mail infrastructure |
| **NS** | DNS infrastructure |
| **TXT** | SPF, DKIM, verification, policy |
| **SOA** | Zone authority, primary nameserver |
| **SRV** | Service discovery |
| **PTR** | Reverse DNS |

### 8.2 dig — Industry Standard

```bash
# All record types
dig example.com ANY

# Specific records
dig A example.com
dig AAAA example.com
dig MX example.com
dig NS example.com
dig TXT example.com
dig CNAME www.example.com
dig SOA example.com

# Query specific nameserver
dig @ns1.example.com example.com ANY

# SPF / DMARC / DKIM
dig TXT example.com                          # SPF
dig TXT _dmarc.example.com                   # DMARC
dig TXT selector1._domainkey.example.com     # DKIM
```

### 8.3 dnsrecon

```bash
# Standard enumeration
dnsrecon -d example.com

# All record types + zone transfer attempt
dnsrecon -d example.com -a

# Brute force with wordlist
dnsrecon -d example.com -t brt -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

### 8.4 Zone Transfers (AXFR)

A successful zone transfer dumps the ENTIRE DNS zone — hostnames, internal IPs, mail servers, potentially an outright network map. **Always test for it.**

```bash
# dig AXFR
dig axfr @ns1.example.com example.com

# dnsrecon zone transfer
dnsrecon -d example.com -t axfr

# dnsenum (includes zone transfer attempt by default)
dnsenum example.com
```

**Practice:** Test against `zonetransfer.me` — a domain deliberately configured to allow AXFR for training.

---

## 9. Reverse DNS Lookups

Reverse DNS resolves an IP address to a hostname (the opposite of normal DNS).

```bash
# Standard PTR lookup
dig -x 203.0.113.10

# host command
host 203.0.113.10

# Nmap reverse DNS
nmap -R 203.0.113.10

# Nmap with custom DNS server
nmap -R --dns-servers 1.1.1.1 203.0.113.0/24
```

**Reverse DNS at scale** (for authorized IP ranges):
```bash
for ip in $(seq 1 254); do
  host 203.0.113.$ip 2>/dev/null | grep -v "not found"
done
```

**yougetsignal.com** — web-based reverse IP lookup (find other domains on the same IP).

---

## 10. Email Harvesting & Document Metadata

### 10.1 theHarvester (primary tool)

```bash
theHarvester -d example.com -b all -l 500 -f harvester-results
theHarvester -d example.com -b google,bing,linkedin,dnsdumpster
```

Output includes: Emails, Hosts, Subdomains, Names, URLs.

**Email naming conventions:** Harvested emails reveal the organization's format (first.last@, flast@, etc.), enabling username list generation for password spraying.

### 10.2 Metagoofil — Metadata Extraction

```bash
./metagoofil.py -d example.com -t doc,pdf -l 100 -n 10 -o results -f results.html
```

Extracts from public documents:
```
Author, Username, Software version, Creation/Modification dates
Company, File path, Printer information, Email addresses
```

Why it matters: A PDF with `Author: jsmith` and `Path: C:\Users\jsmith\Documents\` reveals Windows environment and username convention.

### 10.3 Have I Been Pwned

Check if harvested emails appear in known breaches:
- haveibeenpwned.com (web/API)
- DeHashed, Intelligence X
- Use for exposure intelligence, NOT as a credential source

### 10.4 h8mail (breach checking)
```bash
h8mail -t emails.txt -c h8mail_config.ini
```

---

## 11. Personnel Discovery & Job Post Analysis

### 11.1 Personal Information Sources

```
LinkedIn, Facebook, Twitter/X, Instagram (job titles, org charts, tech stack)
dice.com, Monster (job boards — IT postings reveal vendor technology)
pipl, Spokeo, PeopleFinder (people-search aggregators)
Wayback Machine / archive.org (historical team pages, removed staff listings)
Website-Watcher (change monitoring over time)
```

### 11.2 Job Posts as Technical Intelligence

Job posts are one of the HIGHEST-VALUE and most overlooked sources. A single posting reveals what's guarding the perimeter:

> "Network Engineer — must have Cisco ASA, Palo Alto, Fortinet experience"
> "Azure Administrator with experience in Entra ID, Intune, Defender, Terraform"

Extract from job descriptions:
```
AWS/Azure/GCP, Kubernetes, Docker, Terraform, Jenkins
Active Directory, Cisco, Fortinet, Palo Alto
Linux, Windows Server, Specific databases
Programming languages, Security products (SIEM, EDR)
```

Also check competitor organizations — similar industry orgs often run similar stacks.

---

## 12. Technology & Network Device Identification

### 12.1 WhatWeb

```bash
whatweb https://example.com
whatweb -v https://example.com                   # Verbose
whatweb -a 3 https://example.com                 # Aggression level 3
whatweb -i live-subs.txt                         # Batch scan
```

Identifies: Web server, Framework, CMS, JS libraries, Headers, Cookies.

### 12.2 Manual Technology Discovery

```bash
# HTTP headers
curl -I https://example.com

# View page source
curl -s https://example.com/ | grep -E 'src=|href='

# JavaScript recon
curl -s https://example.com/app.js | grep -Eio 'https?://[^"]+|[A-Za-z0-9._/-]+\.(json|xml|yaml|yml)'
```

Look for: API endpoints, Subdomains, Cloud storage URLs, Framework names, Repository links, Admin/staging/dev paths.

### 12.3 BuiltWith / Wappalyzer

Browser extensions for instant technology fingerprinting — use on the target website's homepage.

---

## 13. Email Header Analysis

When authorized to analyze a message:

```bash
# Inspect full headers — look at the Received: chain
cat email_headers.txt

# Check SPF/DKIM/DMARC
dig TXT example.com
dig TXT _dmarc.example.com
dig TXT selector1._domainkey.example.com
```

Key fields:
```
From, To, Date, Message-ID, Received, Return-Path, Reply-To,
Authentication-Results, DKIM-Signature
```

The `Received:` chain may reveal mail-routing infrastructure, internal hostnames, and relay hops.

**eMailTrackerPro** — GUI tool that traces sender IP/location from headers automatically.

---

## 14. Network Diagramming & Traceroute

Build a mental (and literal) map of the target's network topology as you go.

### 14.1 Traceroute

```bash
# Linux
traceroute example.com
traceroute -n example.com          # No DNS resolution

# Alternative
tracepath example.com

# Nmap (combine with geolocation)
nmap --traceroute example.com
nmap --traceroute --script traceroute-geolocation example.com

# Continuous path monitoring
mtr example.com
```

Windows: `tracert example.com`

Traceroute reveals:
```
Network path, Routers/gateways, Transit providers,
Approximate geography, Segmentation clues
```

Note: Results may be affected by filtering, asymmetric routing, load balancing, and MPLS.

### 14.2 Geolocation

```bash
nmap --script ip-geolocation-geoplugin 203.0.113.10
```

Treat geolocation as approximate — IP geolocation does NOT prove physical server/office/employee location.

### 14.3 Build an Initial Network Diagram

Combine WHOIS netblocks, DNS records, Shodan results, and traceroute to sketch:
```
Public ranges → Mail servers → Web servers → VPN/remote access endpoints → Possible cloud assets
```
Annotate each entry with the source of evidence.

---

## 15. Automation & Frameworks

### 15.1 Maltego

Graph-based OSINT automation. Transform entities (Domain → IP → Email → Person). The closest thing to an "OSINT command center." Worth learning properly for CPENT.

### 15.2 Recon-ng

```bash
recon-ng
[recon-ng] > marketplace install all
[recon-ng] > workspaces create example
[recon-ng] > add domains example.com
[recon-ng] > modules search
```

### 15.3 SpiderFoot

Automated OSINT scanner — web GUI or CLI.

### 15.4 FOCA

Automated metadata harvesting from discovered documents. Identifies usernames, software, and internal naming from metadata.

### 15.5 OSINT Framework

https://osintframework.com — categorized directory of virtually every OSINT resource/tool that exists. Use as your reference map when stuck.

### 15.6 Quick Bash Automation

```bash
#!/bin/bash
DOMAIN=$1
echo "[*] Passive subdomain enumeration for $DOMAIN..."

subfinder -d $DOMAIN -silent > subs.txt
assetfinder --subs-only $DOMAIN >> subs.txt
curl -s "https://crt.sh/?q=%25.$DOMAIN&output=json" | jq -r '.[].name_value' 2>/dev/null >> subs.txt

cat subs.txt | sort -u > all-subs.txt
echo "[+] Found $(wc -l < all-subs.txt) unique subdomains"

cat all-subs.txt | httpx -silent -o live-subs.txt
echo "[+] $(wc -l < live-subs.txt) live hosts"

# Optional: technology fingerprinting
whatweb -i live-subs.txt --no-errors > tech-fingerprints.txt
```

---

## 16. Practical OSINT Workflow (Copy-Paste Ready)

Replace `example.com` with your target domain.

```bash
TARGET="example.com"
OUTDIR="osint-$TARGET"
mkdir -p $OUTDIR && cd $OUTDIR

# Phase 1: WHOIS & DNS basics
whois $TARGET > whois-domain.txt
dig $TARGET ANY > dns-any.txt
dig $TARGET NS > dns-ns.txt
dig $TARGET MX > dns-mx.txt
dig $TARGET TXT > dns-txt.txt
dig $TARGET SOA > dns-soa.txt

# Phase 2: Passive subdomain enumeration
subfinder -d $TARGET -all -silent > subs-subfinder.txt
assetfinder --subs-only $TARGET > subs-assetfinder.txt
curl -s "https://crt.sh/?q=%25.$TARGET&output=json" | jq -r '.[].name_value' 2>/dev/null | sort -u > subs-crtsh.txt

# Phase 3: Active DNS enumeration (only when authorized)
nmap --script dns-brute --script-args dns-brute.domain=$TARGET -oN nmap-dns-brute.txt
gobuster dns -d $TARGET -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -o gobuster-dns.txt
dnsrecon -d $TARGET -t std -a > dnsrecon.txt
fierce --domain $TARGET > fierce.txt

# Phase 4: Combine & validate
cat subs-*.txt | sort -u > all-subs.txt
cat all-subs.txt | httpx -silent -status-code -title -o subs-httpx.txt

# Phase 5: Zone transfer test
dnsrecon -d $TARGET -t axfr

# Phase 6: Email harvesting
theHarvester -d $TARGET -b all -l 500 -f harvester-results

# Phase 7: Typosquatting
urlcrazy -p $TARGET > urlcrazy.txt 2>&1
dnstwist $TARGET > dnstwist.txt

# Phase 8: Technology fingerprinting
whatweb -i live-subs.txt --no-errors > whatweb-results.txt 2>/dev/null

# Phase 9: Shodan (requires API key)
shodan search "org:\"$(whois $TARGET | grep -i 'org' | head -1 | cut -d: -f2 | xargs)\"" --fields ip_str,port,org,hostnames 2>/dev/null

# Phase 10: Reverse DNS on discovered IPs
for ip in $(grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' dns-any.txt | sort -u); do
  echo "=== $ip ===" >> rdns.txt
  dig -x $ip +short >> rdns.txt
done

echo "[+] OSINT complete. Review $OUTDIR/"
```

---

## 17. Tools & Techniques Mapping

| Technique / Goal | Primary Tool | Secondary Tool |
|---|---|---|
| **Domain registration** | whois / RDAP | Web registries |
| **DNS records** | dig | nslookup |
| **DNS enumeration** | dnsrecon | dnsenum |
| **Subdomain discovery (passive)** | subfinder / amass | crt.sh, assetfinder |
| **Subdomain discovery (active)** | gobuster dns | nmap dns-brute, dnsmap, fierce |
| **Typosquatting** | urlcrazy / dnstwist | Google Dorks |
| **Email harvesting** | theHarvester | Hunter.io |
| **Metadata extraction** | Metagoofil | FOCA |
| **Web fingerprinting** | WhatWeb | Wappalyzer, BuiltWith |
| **Internet exposure** | Shodan | Censys, ZoomEye |
| **Historical sites** | Wayback Machine | archive.org |
| **Relationship mapping** | Maltego | recon-ng |
| **Network path** | traceroute | nmap --traceroute |
| **Google dorks** | Manual construction | GHDB, SiteDigger |
| **Automation** | Custom bash script | OSINT Framework |

---

## 18. Practice & Labs

- **TryHackMe:** "OhSINT", "Google Dorking", "Searchlight Inc", "OSINT" rooms
- **HackTheBox:** Recon-focused challenges in Starting Point
- **Zone Transfer Practice:** `zonetransfer.me` — deliberately allows AXFR
- **Local practice:** Point tools at your own domain to learn syntax risk-free
- **Public bug bounty:** Programs on HackerOne that allow wide-scope OSINT (read rules carefully)
- **Self-assessment:** Can you produce a clean subdomain list, email list, and initial network sketch within 15 minutes?

---

> **Next Module:** [04 — Network Pentesting (External)](04-NETWORK-PENTESTING.md)
