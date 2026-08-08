# CPENT / LPT Master Guide — Exam-Ready Reference

> **Zero to CPENT: The world's most authentic, AI-ready penetration testing reference.**
>
> Built for the CPENT AI exam — open any file, copy a command, replace the target, and go.
>
> **Author:** Zeeshan  
> **GitHub:** [github.com/mzeeshanzafar28](https://github.com/mzeeshanzafar28)  
> **LinkedIn:** [linkedin.com/in/mzeeshanzafar28](https://www.linkedin.com/in/mzeeshanzafar28)  
> **Companion Repo:** [CEH Practical CheatSheet](https://github.com/mzeeshanzafar28/CEH-Practical-CheatSheet)

---

## What This Is

A comprehensive, modular CPENT reference built from personal notes, official EC-Council material, community cp resources, and real exam experiences. Consolidated from multiple LLM generations into the single most complete guide available.

### Design Principles

1. **Concept-first** — explains the technology before attacking it
2. **Copy-paste ready** — every command uses real syntax with placeholders (`TARGET_IP`, `DOMAIN`, `USERNAME`)
3. **Industry-standard tools first** — Nmap before alternatives, Impacket before custom scripts
4. **Port numbers everywhere** — every protocol and service includes its port
5. **Modular files** — open only the module you're working on during the exam
6. **Exam-optimized** — trimmed of fluff; keeps what actually matters on the ranges

---

## Repository Structure

| # | File | Module |
|---|------|--------|
| 00 | [About CPENT / LPT Master](00-ABOUT-CPENT.md) | Exam format, scoring, ranges, AI policy, prep strategy |
| 01 | [The Basics](01-THE-BASICS.md) | Networking, protocols, PowerShell, methodology |
| 02 | [Tools & Setup](02-TOOLS-AND-SETUP.md) | Tool installation, wordlists, aliases, exam machine prep |
| 03 | [OSINT & Reconnaissance](03-OSINT-AND-RECON.md) | Domain/subdomain, email, personnel, DNS, automation |
| 04 | [Network Pentesting](04-NETWORK-PENTESTING.md) | Nmap mastery, service enumeration, Wireshark verification |
| 05 | [Perimeter Defense Evasion](05-PERIMETER-DEFENSE-EVASION.md) | Firewall bypass, ProxyChains, tunneling, IDS evasion |
| 06 | [Pivoting & Lateral Movement](06-PIVOTING-AND-LATERAL-MOVEMENT.md) | Single/double pivot, port forwarding, proxychains, multi-hop |
| 07 | [Active Directory](07-ACTIVE-DIRECTORY.md) | AD architecture, Kerberos, enumeration, Kerberoasting, DCSync, full attack chains |
| 08 | [Privilege Escalation](08-PRIVILEGE-ESCALATION.md) | Windows + Linux privesc checklists and exploitation |
| 09 | [Web Application & API](09-WEB-APPLICATION-AND-API.md) | Burp Suite, SQLi, XSS, file upload, WordPress, JWT, WAF |
| 10 | [Binary Exploitation](10-BINARY-EXPLOITATION.md) | x86/IA-32, stack overflow, ROP, shellcode, GDB, labs |
| 11 | [IoT Pentesting](11-IOT-PENTESTING.md) | Firmware extraction, Binwalk, QEMU, Firmadyne, known exploits |
| 12 | [Wireless Pentesting](12-WIRELESS.md) | Aircrack-ng, Wifite, Airgeddon, handshake/PMKID attacks |
| 13 | [OT / SCADA Pentesting](13-OT-SCADA.md) | Modbus, BACnet, safe scanning, NSE scripts, GrassMarlin |
| 14 | [Cloud Pentesting](14-CLOUD-PENTESTING.md) | AWS/Azure enumeration, IAM, metadata, S3, misconfigurations |
| 15 | [Social Engineering](15-SOCIAL-ENGINEERING.md) | Phishing, vishing, SET, GoPhish, authorization, pretexting |
| 16 | [CTF & Exam Scenarios](16-CTF-AND-EXAM-SCENARIOS.md) | Attack chains, decision trees, objective-driven workflows |
| 17 | [Tips & Exam Strategy](17-TIPS-AND-EXAM-STRATEGY.md) | Time management, evidence, common pitfalls, survival guide |
| 18 | [Tools & Techniques Map](18-TOOLS-AND-TECHNIQUES-MAP.md) | Quick lookup: technique → primary tool → alternatives |

---

## How to Use During the Exam

1. **Keep the whole folder open** (or cloned to your exam machine)
2. **Jump to the range** you are working on
3. **`Ctrl+F`** for the service/port/technique you need
4. **Copy the command block**, replace placeholders with your targets
5. **Document as you go** — screenshots, notes, host discovery maps
6. **Use 17-TIPS** when stuck or to verify you're not in a rabbit hole

---

## Recommended Learning Order

```
00-ABOUT CPENT          ← Start here — understand the exam
       │
01-THE BASICS           ← Foundation — read fully before attack modules
       │
02-TOOLS & SETUP        ← Install and verify everything
       │
03-OSINT & RECON        ← Learn to find targets
       │
04-NETWORK PENTESTING   ← Master Nmap and service enumeration
       │
05-PERIMETER EVASION    ← Bypass defenses to reach targets
       │
06-PIVOTING             ← Critical CPENT skill — practice until blind
       │
07-ACTIVE DIRECTORY     ← HIGHEST VALUE — largest time investment
       │
08-PRIVILEGE ESCALATION ← Win/Linux escalation after foothold
       │
09-WEB APPLICATION      ← Entry point for many ranges
       │
10-BINARY EXPLOITATION  ← CPENT-specific (32-bit focus)
       │
11-IOT                  ← Firmware analysis + known exploits
       │
12-WIRELESS             ← Quick reference, not a primary range
       │
13-OT/SCADA             ← Modbus + safe scanning methodology
       │
14-CLOUD                ← AWS/Azure enumeration
       │
15-SOCIAL ENGINEERING   ← Human attack surface (SET, GoPhish)
       │
16-CTF SCENARIOS        ← Full attack chain walkthroughs
       │
17-TIPS & STRATEGY      ← Exam day survival
       │
18-TOOLS MAP            ← Quick lookup reference
```

---

## Quick Reference: Key Ports

| Port | Service | Module |
|------|---------|--------|
| 21 | FTP | [04-Network](04-NETWORK-PENTESTING.md) |
| 22 | SSH | [04-Network](04-NETWORK-PENTESTING.md), [06-Pivoting](06-PIVOTING-AND-LATERAL-MOVEMENT.md) |
| 23 | Telnet | [01-Basics](01-THE-BASICS.md), [04-Network](04-NETWORK-PENTESTING.md) |
| 25 | SMTP | [03-OSINT](03-OSINT-AND-RECON.md) |
| 53 | DNS | [01-Basics](01-THE-BASICS.md), [03-OSINT](03-OSINT-AND-RECON.md) |
| 79 | Finger | [01-Basics](01-THE-BASICS.md), [04-Network](04-NETWORK-PENTESTING.md) |
| 80/443 | HTTP/HTTPS | [09-Web](09-WEB-APPLICATION-AND-API.md) |
| 88 | Kerberos | [07-AD](07-ACTIVE-DIRECTORY.md) |
| 111 | RPC | [01-Basics](01-THE-BASICS.md), [04-Network](04-NETWORK-PENTESTING.md) |
| 135 | MSRPC | [07-AD](07-ACTIVE-DIRECTORY.md) |
| 139/445 | SMB | [04-Network](04-NETWORK-PENTESTING.md), [07-AD](07-ACTIVE-DIRECTORY.md) |
| 161 | SNMP | [04-Network](04-NETWORK-PENTESTING.md) |
| 389/636 | LDAP/LDAPS | [07-AD](07-ACTIVE-DIRECTORY.md) |
| 502 | Modbus | [13-OT/SCADA](13-OT-SCADA.md) |
| 2049 | NFS | [01-Basics](01-THE-BASICS.md), [04-Network](04-NETWORK-PENTESTING.md) |
| 3268/3269 | Global Catalog | [07-AD](07-ACTIVE-DIRECTORY.md) |
| 3389 | RDP | [01-Basics](01-THE-BASICS.md), [04-Network](04-NETWORK-PENTESTING.md) |
| 5985/5986 | WinRM | [07-AD](07-ACTIVE-DIRECTORY.md), [08-PrivEsc](08-PRIVILEGE-ESCALATION.md) |

---

**Legal:** All techniques are for authorized testing only (your own lab, CPENT exam ranges, or written permission). Unauthorized use is illegal.
