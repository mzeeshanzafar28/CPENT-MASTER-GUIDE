# 17 - TIPS AND EXAM STRATEGY

**Credits**  
Zeeshan  
GitHub: https://github.com/mzeeshanzafar28  
LinkedIn: https://www.linkedin.com/in/mzeeshanzafar28  

CPENT is an endurance test as much as a technical assessment. Reaching the 90% threshold for LPT Master requires strict discipline, impeccable time management, and a structured methodology. This module distills everything successful candidates wish they knew before starting.

---

## 1. EXAM FORMAT AND SCORING

| Aspect | Detail |
|--------|--------|
| **Total lab time** | 24 hours |
| **Format options** | Single 24-hour block OR two 12-hour sessions |
| **CPENT pass** | ~70% (approximately 10 of 14 flags) |
| **LPT Master** | ~90% (approximately 13 of 14 flags) |
| **Report deadline** | 7 days after lab time ends |
| **Environment** | Multi-segment corporate network: External, DMZ, Internal, OT, IoT |
| **AI allowed** | Yes — CPENT AI format allows AI assistance |

**12+12 Split (RECOMMENDED):** Most high-scoring candidates prefer two 12-hour sessions booked days apart within the allowed window. Questions generally stay the same between slots.

---

## 2. DAY 1 (First 12 Hours): THE FOOTHOLD AND WIDE NET

### Hours 1-3: External Reconnaissance

- Start automated scans FIRST (they run in background):
  ```bash
  nmap -sS -Pn -p- -oA full_scan TARGET &
  nmap -sU --top-ports 200 -oA udp_scan TARGET &
  ```
- While scans run, manually inspect web applications:
  - Browse every URL, check source code (`Ctrl+U`)
  - Launch Burp Suite, intercept traffic, map the application
  - Run `gobuster dir -u http://TARGET -w /usr/share/wordlists/dirb/common.txt`
  - Check for subdomains: `ffuf -u http://FUZZ.TARGET -w subdomains.txt`
- Identify ALL targets — there may be multiple IP ranges provided

### Hours 4-7: Web Exploitation and Initial Access

- **Priority 1:** Any file upload → reverse shell
- **Priority 2:** SQL Injection → data extraction or OS command execution
- **Priority 3:** Default credentials, weak passwords
- **Priority 4:** Known CMS vulnerabilities (WordPress plugins, etc.)

**On every shell you get:**
1. Run `ip a` / `ipconfig /all` → LOOK FOR SECOND INTERFACES
2. Run LinPEAS or WinPEAS
3. Escalate to root/SYSTEM if possible
4. Set up persistence (SSH key, Meterpreter, scheduled task)
5. START THE PIVOT immediately if you see additional subnets

### Hours 8-10: Active Directory and IoT/OT

- If you reached the internal network, run BloodHound immediately:
  ```bash
  proxychains bloodhound-python -c All -u USER -p PASS -d DOMAIN -dc DC_IP -ns DC_IP
  ```
- Hunt for GPP credentials (Groups.xml with cpassword)
- Start IoT firmware analysis if you have firmware files
- Map SCADA segments with passive discovery (Wireshark/tcpdump)
- Do NOT run aggressive scans on OT ranges

### Hours 11-12: Documentation and Reset

- **DO NOT END THE DAY HACKING.**
- Organize all screenshots by target/host
- Write rough attack path notes
- List: what you have, what you still need, what you'll attempt first tomorrow
- Formulate a strict plan for Day 2
- Save all terminal scrollback

---

## 3. DAY 2 (Second 12 Hours): DEEP PIVOTING AND HARD TARGETS

### Hours 1-4: Lateral Movement and AD Compromise

- Execute attacks based on Day 1 enumeration:
  - Kerberoasting (`impacket-GetUserSPNs`)
  - AS-REP Roasting (`impacket-GetNPUsers`)
  - Password spraying (if you have user list)
  - Pass-the-Hash with discovered NTLM hashes
- Target Domain Controller: DCSync, Golden Ticket
- Clean up any incomplete privilege escalations

### Hours 5-8: Binary Analysis and Deep Pivots

- Tackle buffer overflow / reverse engineering challenges
- These require extreme focus — best done with a clear head
- If stuck for >45 min on one binary, switch to another and return

### Hours 9-11: Flag Hunting

- Revisit every compromised host — did you grab ALL flags?
- Check for flags in non-standard locations:
  ```bash
  # Linux
  find / -name "flag*" -o -name "*.txt" 2>/dev/null | grep -v proc
  find /home /root /opt /var -type f 2>/dev/null | xargs grep -l "flag{" 2>/dev/null
  
  # Windows
  dir /s /b C:\*flag* C:\Users\*\Desktop\*flag*
  ```
- Ensure you have evidence for every flag and exploitation step

### Hour 12: Final Screenshot Verification

- Walk through your report outline
- Every flag needs a timestamped screenshot
- Every critical step needs a screenshot
- Save everything to a location you can access after the lab closes

---

## 4. SINGLE 24-HOUR SESSION STRATEGY

If you take the continuous 24-hour format:

| Block | Hours | Focus |
|-------|-------|-------|
| Discovery | 0-3 | Broad discovery, easiest footholds, first solid pivot |
| Expansion | 3-12 | Internal enumeration, AD work, high-value targets |
| Depth | 12-20 | Remaining ranges, double pivots, escalation, binary |
| Wrap-up | 20-24 | Fill gaps, collect evidence, prepare report notes |

Take a 30-minute break at hours 6, 12, and 18. Eat, hydrate, move.

---

## 5. CRITICAL SUCCESS HABITS

### The First-Command Reflex
On every shell, immediately run:
- **Linux:** `ip a; ip route; arp -a; cat /etc/hosts; id; sudo -l`
- **Windows:** `ipconfig /all; route print; arp -a; whoami /priv; whoami /groups`

### Pivoting Discipline
- Treat every dual-homed host as a pivot opportunity
- Prefer reverse SOCKS (Chisel) + proxychains over direct routing
- **Always test the tunnel** with a simple command before heavy scanning:
  ```bash
  proxychains nc -nv INTERNAL_IP 445
  ```
- Through proxychains: `-sT` only. Never `-sS`.
- If a proxychains scan hangs or fails, your tunnel is dead — restart it

### Documentation as You Go
- Screenshot every important command and its output
- Use Flameshot or Greenshot for fast annotated screenshots
- Keep one notes file per host with the template from Module 16
- Update your live network map every time you see a new interface

### Shell Stability
- Upgrade basic reverse shells immediately:
  ```bash
  python3 -c 'import pty;pty.spawn("/bin/bash")'
  CTRL+Z
  stty raw -echo; fg
  export TERM=xterm
  ```
- Add SSH keys for persistent, stable access
- Upgrade to Meterpreter when possible

---

## 6. COMMON PITFALLS THAT COST POINTS

| Pitfall | Fix |
|---------|-----|
| **Forgetting to check interfaces** | Make `ip a` a reflex — run it before `id` |
| **SYN scans through proxychains** | Always use `-sT` with proxychains |
| **Wrong LHOST in payloads** | If target can't reach your external IP, use the pivot host's internal IP and a bind shell |
| **Spending too long on one binary** | 45-minute limit. Switch ranges and return |
| **Poor or missing screenshots** | Screenshot every flag with timestamp. You CANNOT go back |
| **Noisy scans on OT/SCADA** | Use passive discovery, `-T2`, no `-A` or `-sV` |
| **Dead tunnels unnoticed** | Ping through the tunnel periodically |
| **Ignoring BloodHound paths** | Run it the moment you have ANY domain credentials |
| **Not looting credentials before leaving** | Run `find`/`dir` for config files, history files, SSH keys |
| **Using one pivot method with no backup** | Have Chisel AND SSH AND Meterpreter ready |

---

## 7. ATTACK PRIORITIZATION

When you have multiple possible attack vectors, use this priority order:

1. **Web app exploits** (file upload, SQLi, command injection) — fastest foothold
2. **Default/weak credentials** — check SSH, FTP, SMB, RDP, web logins
3. **Known CVEs on exposed services** — EternalBlue, BlueKeep, Shellshock
4. **SMB enumeration** — null sessions, open shares, GPP passwords
5. **Kerberoasting / AS-REP roasting** — needs domain user, high value
6. **Password spraying** — needs valid user list, slow but reliable
7. **Binary exploitation** — time-consuming, save for later
8. **Wireless attacks** — separate range, may need dedicated time

---

## 8. EVIDENCE AND SCREENSHOT DISCIPLINE

### What to Screenshot (minimum):
- Every flag with the command that revealed it
- Every shell obtained (showing `whoami` + `ip a`)
- Every pivot established (showing tunnel verification)
- Every privilege escalation (showing before/after `whoami`)
- Every credential discovery
- BloodHound paths to Domain Admin
- SCADA register reads

### Screenshot Best Practices:
- Include the terminal prompt showing the target IP/hostname
- Use a timestamp (manually or automated)
- Annotate with Flameshot arrows/text for clarity
- Name files consistently: `01_initial_scan.png`, `02_web_shell.png`, etc.

---

## 9. NOTE-TAKING DURING THE EXAM

### Host-per-File System
Create one markdown file per compromised host:
```
exam-notes/
├── 01-external-recon.md
├── 02-web-server-pivot1.md
├── 03-dc-internal.md
├── 04-iot-device.md
├── 05-scada-ot.md
└── network-map.md
```

### Network Map (keep live)
```
[Kali] 10.10.14.5
    │
    └── [Web Server] 10.10.10.20 (eth0)
                     172.16.20.20 (eth1 → Internal Segment A)
                         │
                         ├── [DC] 172.16.20.5
                         ├── [File Server] 172.16.20.30
                         └── [Pivot-2] 172.16.20.40 (eth0)
                                       10.10.30.40 (eth1 → Segment B)
```

---

## 10. RABBIT-HOLE AVOIDANCE

### Signs You're in a Rabbit Hole:
- You've spent 45+ minutes on one attack vector with zero progress
- You're trying increasingly exotic techniques when simpler ones exist
- You're working on a service that isn't on the exam syllabus
- You've restarted the same exploit 5+ times with only minor tweaks

### What to Do:
1. **Take a 5-minute break.** Walk away from the keyboard.
2. **Re-enumerate.** Run LinPEAS/WinPEAS again. You may have missed something.
3. **Switch ranges.** Move to a completely different target or domain.
4. **Check fundamentals.** Default credentials? Simple misconfiguration? Checked ALL interfaces?
5. **Return with fresh eyes.** Many candidates find the solution immediately after a range change.

---

## 11. RANGE-SPECIFIC STRATEGIES

### Web / IoT
- Usually the intended entry points into the network
- Get the shell, check interfaces, pivot FAST
- Don't overthink; the web app vuln is usually straightforward

### Active Directory
- BloodHound + Impacket + CrackMapExec are your trifecta
- Goal is DCSync or Domain Admin shell
- If BloodHound shows no path, re-enumerate — you missed something

### Binary Exploitation
- Classic methodology only: crash → EIP offset → JMP ESP → shellcode
- 32-bit binaries are the CPENT focus
- Reset the machine if your exploit crashes it twice

### OT / SCADA
- **SLOW AND POLITE.** Availability is the priority
- Passive capture first, then minimal active probes
- Modbus (port 502) is the most common protocol
- Document all captures for evidence
- Safe scan: `nmap -n -sT -T2 -p 502 --open SUBNET`

### Wireless
- Separate physical/virtual range
- Capture WPA handshake → crack with aircrack-ng or hashcat
- Tools: airgeddon, wifite, aircrack-ng suite

### Pivoting
- **THE REAL BOSS OF THE EXAM**
- Practice web → pivot → AD until it's automatic
- Always have a backup pivot method
- Test each hop before trusting it

---

## 12. PHYSICAL AND MENTAL PREPARATION

### Before the Exam:
- [ ] All tools installed and verified (Kali fully updated and snapshotted)
- [ ] Proxychains configured and tested with a lab pivot
- [ ] Windows binaries ready to transfer: mimikatz.exe, Rubeus.exe, SharpHound.exe, winPEAS.exe, chisel.exe, nc.exe
- [ ] Wordlists unzipped and ready (`/usr/share/wordlists/rockyou.txt`)
- [ ] Note-taking system set up
- [ ] BloodHound + Impacket workflow comfortable
- [ ] Classic buffer overflow methodology timed and reliable
- [ ] Firmware analysis practiced (binwalk + firmadyne/QEMU)
- [ ] Backup pivot methods (Chisel AND SSH AND Meterpreter) all tested
- [ ] Meal prep done — food that doesn't require cooking
- [ ] Sleep schedule optimized in the days before
- [ ] Quiet workspace secured for 12+ hours

### During the Exam:
- **Take breaks.** 5 minutes every hour, 15 minutes every 3 hours
- **Eat real food.** Not just caffeine and sugar
- **Hydrate.** Water on the desk at all times
- **If tired, change tasks.** Physical movement helps reset focus
- **Sleep between Day 1 and Day 2** if using the split format

### Confidence:
- The actual exam often feels more straightforward than the practice range
- You've practiced the chains — trust your methodology
- Many candidates report that once pivoting was established, the rest fell into place
- Momentum matters more than perfection on any single target

---

## 13. USING AI DURING CPENT AI

The CPENT AI format allows AI assistance. Use it strategically:

**Good AI prompts:**
> "I have a Meterpreter session on Windows Server 2016. `whoami /priv` shows SeImpersonatePrivilege. Give me the exact syntax to exploit this using PrintSpoofer."

> "This exploit failed with error: [paste exact error]. Here's the command I used: [paste command]. What should I change?"

**Bad AI prompts:**
> "How do I hack a Windows machine?"

Be specific. Provide context. Paste error output. The AI is more useful as a targeted debugger than a generic guide.

---

## 14. REPORTING STRATEGY

You have 7 days after the lab to submit a professional penetration test report.

### Report Structure for LPT Master:
1. **Executive Summary** — For management. High-level findings, risk ratings, overall security posture.
2. **Methodology** — Your approach. Tools used, phases followed.
3. **Attack Narrative** — The story of your attack chain. How you moved from foothold to Domain Admin. **This is what separates CPENT from LPT Master.**
4. **Detailed Findings** — Per-vulnerability breakdown with:
   - Vulnerability description
   - Affected hosts/IPs
   - Step-by-step reproduction
   - Screenshot evidence with timestamps
   - CVSS score and risk rating
   - Actionable remediation steps
5. **Remediation Roadmap** — Prioritized fix list

### Writing During the Exam:
- Keep notes that already look like report material
- Write one sentence per finding as you discover it
- Screenshot everything — you can't go back

---

## 15. KEYBOARD SHORTCUTS AND EFFICIENCY TIPS

### Terminal Efficiency
| Shortcut | Action |
|----------|--------|
| `Ctrl+R` | Reverse search command history |
| `Ctrl+A` | Go to beginning of line |
| `Ctrl+E` | Go to end of line |
| `Ctrl+U` | Delete from cursor to start |
| `Ctrl+K` | Delete from cursor to end |
| `Ctrl+W` | Delete previous word |
| `!!` | Repeat last command |
| `!$` | Last argument of previous command |
| `Ctrl+L` | Clear screen |
| `Ctrl+Shift+C/V` | Copy/paste in terminal |

### Workflow Speedups
- **Use tmux or screen.** Multiple panes = enumeration, exploitation, and notes simultaneously.
- **Pre-build payloads.** Have generic reverse shells for PHP, Python, bash, PowerShell saved to a cheat sheet.
- **Alias common commands:**
  ```bash
  alias pscan='proxychains nmap -sT -Pn'
  alias psmb='proxychains crackmapexec smb'
  alias pevil='proxychains evil-winrm'
  ```
- **Keep a second terminal open** for file transfers (Python HTTP server).
- **Pre-configure listener terminals.** Have `nc -lvnp 4444`, `nc -lvnp 5555` ready.

---

## 16. WHAT TO SKIP VS WHAT TO FOCUS ON

| FOCUS ON | SKIP OR DEPRIORITIZE |
|----------|---------------------|
| Web → pivot → AD chain | Exotic kernel exploits |
| Chisel/SSH pivoting | Custom Metasploit module development |
| BloodHound + Impacket | 64-bit ROP chains (CPENT focuses on 32-bit) |
| Classic buffer overflow (32-bit) | Advanced heap exploitation |
| Modbus/SCADA passive recon | Aggressive OT scanning |
| SMB enumeration (CrackMapExec) | Exotic protocol fuzzing |
| Password spraying common lists | Brute-forcing without user lists |
| LinPEAS/WinPEAS enumeration | Manual enumeration of every possible vector |
| File upload + reverse shells | Complex multi-step web attack chains without foothold |
| GPP/ Groups.xml credential theft | Custom malware development |

---

## 17. FINAL PRE-EXAM CHECKLIST

- [ ] All tools installed and tested
- [ ] Proxychains and Chisel/SSH pivoting verified in a lab
- [ ] BloodHound + Impacket workflow comfortable
- [ ] Classic buffer overflow methodology timed (<30 min)
- [ ] binwalk firmware extraction practiced
- [ ] Note-taking and screenshot system ready
- [ ] Kali snapshot taken
- [ ] Windows binaries folder ready for transfer
- [ ] Food, water, sleep sorted
- [ ] Second monitor / tmux workflow tested
- [ ] VPN/proctor software installed and tested

---

## 18. CLOSING ADVICE

CPENT rewards methodical, well-documented, chained exploitation. The candidate who gets stuck for hours on a single binary while ignoring an exposed web shell on another range will fail. The candidate who moves fluidly between ranges, documents everything, and treats pivoting as their primary skill will pass at the LPT Master level.

**Master the full chain: Web/IoT foothold → Privilege Escalation → Pivot → Internal Enumeration → AD Compromise.** Everything else is a variation on this pattern.

---

**End of 17-TIPS-AND-EXAM-STRATEGY.md**
