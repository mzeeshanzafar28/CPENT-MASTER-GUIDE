# 13 - OT & SCADA PENTESTING

**Author:** Zeeshan  
**GitHub:** https://github.com/mzeeshanzafar28  
**LinkedIn:** https://www.linkedin.com/in/mzeeshanzafar28

> **SAFETY WARNING:** OT environments prioritize availability and safety. Port scanning can disrupt fragile industrial controllers and cause physical consequences. Never apply aggressive scanning or disruptive exploitation against production control systems without explicit authorization and an agreed safety procedure.

---

## Table of Contents

1. [What Are OT, ICS, and SCADA?](#1-what-are-ot-ics-and-scada)
2. [Why OT/SCADA Pentesting Is Different](#2-why-otscada-pentesting-is-different)
3. [Control Plane vs Data Plane](#3-control-plane-vs-data-plane)
4. [Key Components (PLC, HMI, Engineering Workstations)](#4-key-components)
5. [SCADA Network Architecture](#5-scada-network-architecture)
6. [Modbus Protocol (Port 502)](#6-modbus-protocol)
7. [Modbus Data Model (Coils & Registers)](#7-modbus-data-model)
8. [Modbus Function Codes](#8-modbus-function-codes)
9. [BACnet (Port 47808)](#9-bacnet)
10. [Other OT Protocols](#10-other-ot-protocols)
11. [Safe Scanning Practices](#11-safe-scanning-practices)
12. [Conservative Host Discovery](#12-conservative-host-discovery)
13. [Why Aggressive Scans Are Dangerous](#13-why-aggressive-scans-are-dangerous)
14. [Conservative TCP Scanning](#14-conservative-tcp-scanning)
15. [Modbus NSE Scripts](#15-modbus-nse-scripts)
16. [Wireshark SCADA Analysis](#16-wireshark-scada-analysis)
17. [GrassMarlin (Passive Network Mapping)](#17-grassmarlin)
18. [Metasploit SCADA Modules](#18-metasploit-scada-modules)
19. [Shodan & OSINT for SCADA](#19-shodan--osint-for-scada)
20. [Device Separation & Data Diodes](#20-device-separation--data-diodes)
21. [Authentication Testing](#21-authentication-testing)
22. [Attack Categories in OT](#22-attack-categories-in-ot)
23. [Lab Setup (QModMaster + Modbus-Pal)](#23-lab-setup)
24. [Step-by-Step Lab Workflow](#24-step-by-step-lab-workflow)
25. [Practical Exam Workflow](#25-practical-exam-workflow)
26. [Tools & Techniques Mapping](#26-tools--techniques-mapping)

---

## 1. What Are OT, ICS, and SCADA?

### OT (Operational Technology)
Technology used to monitor or control physical processes. Examples:
```
Industrial machinery, Power systems, Water treatment,
Manufacturing, Building automation, Transportation, Process control
```
The defining characteristic: **OT interacts with the physical world.**

### ICS (Industrial Control System)
Broad term covering systems used to control industrial processes. Components:
```
Sensors, Actuators, PLCs, RTUs, HMIs, Engineering workstations,
Historian systems, SCADA servers, Network infrastructure
```

### SCADA (Supervisory Control and Data Acquisition)
Centralized monitoring and supervisory control of distributed processes.

```
                    SCADA Server
                         |
                        HMI
                         |
              +----------+----------+
              |                     |
             PLC                   RTU
              |                     |
           Sensors              Remote Site
              |
          Actuators
```

---

## 2. Why OT/SCADA Pentesting Is Different

| Priority | IT | OT |
|---|---|---|
| #1 | Confidentiality | **Safety** |
| #2 | Integrity | **Availability** |
| #3 | Availability | Process Integrity |

A scan that is harmless against a web server can potentially **disrupt an industrial controller** with physical consequences.

```
IT:  CIA (Confidentiality, Integrity, Availability)
OT:  Safety + Availability come first
```

---

## 3. Control Plane vs Data Plane

### Control Plane
Functions involved in managing and controlling the environment:
```
Controller configuration, Routing/control decisions,
SCADA supervisory functions, Management systems, Engineering operations
```

### Data Plane
Operational traffic and process information:
```
Sensor readings, Controller communications,
Process values, Telemetry, Control messages
```

The control plane changes process state (dangerous). The data plane provides monitoring and read-only information (safer to interact with).

---

## 4. Key Components

### PLC (Programmable Logic Controller)
Industrial computer designed for reliable real-time control.

```
Sensors → PLC → Control Logic → Actuators
```

Contains: **Logic, Configuration, Firmware.** May expose industrial protocols (Modbus/TCP, EtherNet/IP, S7comm).

### RTU (Remote Terminal Unit)
Similar to PLC but designed for remote locations; often used in distributed SCADA systems.

### HMI (Human-Machine Interface)
Operator interface to the industrial process:
```
Operator → HMI → SCADA / PLC → Industrial Process
```
Displays: temperature, pressure, tank level, motor state, alarm status, valve status. May provide authorized control functions.

### Engineering Workstation
Used to program and configure PLCs. Often runs Windows with vendor-specific engineering software. **Critical target** — compromise may give full control over industrial logic.

### SCADA Server / Historian
Collects and stores process data. The historian retains historical operational data.

### Data Diode / Unidirectional Gateway
Hardware device enforcing one-way data flow for security:
```
Network A → (allowed) → Network B
Network B → (BLOCKED) → Network A
```
Ensures OT can send logs out but no commands can come in.

---

## 5. SCADA Network Architecture

```
                 Enterprise Network (IT)
                         |
                    Firewall
                         |
                 OT Perimeter / DMZ
                         |
         +---------------+---------------+
         |                               |
    SCADA Server                       HMI
         |                               |
         +---------------+---------------+
                         |
                    Control Network
                         |
                 +-------+-------+
                 |               |
                PLC             RTU
                 |               |
              Sensors         Remote Site
              Actuators
```

---

## 6. Modbus Protocol

The most common and critical protocol in OT assessments.

| Property | Value |
|---|---|
| **Port** | **502/TCP** |
| **Architecture** | Master/Slave (Client/Server in modern terminology) |
| **Capacity** | 1 Master, up to 247 Slaves |
| **Unit ID** | Identifies target logical device (0-247) — multiple logical devices can exist behind one IP |
| **Data** | Coils (Boolean), Registers (numeric) |

```
SCADA / Master
      |
      | TCP/502
      v
PLC / Slave (Unit ID 1)
```

### Why Modbus Is Security-Sensitive
- Traditional design prioritized reliability, not security
- Weak or no built-in authentication
- No encryption/confidentiality
- Command manipulation possible
- Unauthorized reads and writes
- Replay opportunities

---

## 7. Modbus Data Model

### Coils (Boolean Values)
```
0 = OFF / FALSE
1 = ON / TRUE
```
Represent states: motor enabled/disabled, valve open/closed, relay state.

### Registers (Numerical Values)
```
Input Registers — Read-only process data (sensor readings)
Holding Registers — Read-write values (setpoints, thresholds)
```

Example holding register values: temperature, pressure setpoint, counter, speed parameter.

---

## 8. Modbus Function Codes

| Function Code | Name | Type |
|---|---|---|
| **01** | Read Coils | Read | Boolean |
| **02** | Read Discrete Inputs | Read | Boolean |
| **03** | Read Holding Registers | Read | Numeric |
| **04** | Read Input Registers | Read | Numeric |
| **05** | Write Single Coil | Write | Boolean |
| **06** | Write Single Register | Write | Numeric |
| **15** | Write Multiple Coils | Write | Boolean |
| **16** | Write Multiple Registers | Write | Numeric |

> **Start with read-only operations.** Only perform write testing when explicitly approved and the affected process is controlled.

---

## 9. BACnet

Building Automation and Control Networks protocol.

| Property | Value |
|---|---|
| **Port** | **47808/UDP** (also TCP) |
| **Use** | HVAC, lighting, building access, energy management |

Assessment should identify: devices, objects, services, network exposure, authentication controls.

---

## 10. Other OT Protocols

| Protocol | Default Port | Usage |
|---|---|---|
| **Modbus/TCP** | **502/TCP** | General industrial automation |
| **BACnet** | **47808/UDP** | Building automation |
| **DNP3** | **20000/TCP** | Electric/water utilities |
| **EtherNet/IP** | **44818/TCP+UDP** | Rockwell/Allen-Bradley |
| **S7comm** | **102/TCP** | Siemens S7 PLCs |
| **OPC UA** | **4840/TCP** | Modern platform-independent |

---

## 11. Safe Scanning Practices

**Availability is the absolute priority.** Follow these rules:

```
✅ Use slow timing: -T2 or slower
✅ Use --scan-delay to spread packets
✅ Prefer TCP Connect scans (-sT) over SYN
✅ Limit to known OT ports first
✅ Start with the smallest scope necessary
✅ Document every interaction

❌ NO -O (OS detection)
❌ NO -A (aggressive scan)
❌ NO -sV (version detection) on initial scans
❌ NO aggressive write operations
❌ NO denial-of-service testing
```

### Safe scan templates:

```bash
# Very polite ARP host discovery (same subnet)
sudo nmap -n -PR -sn 192.168.10.0/24

# ICMP host discovery only (no port scan)
sudo nmap -n -sn 192.168.10.0/24

# Top ports only, slow timing
sudo nmap -n -sT -T2 --top-ports 100 --scan-delay 0.1s 192.168.10.50

# Single Modbus port check
sudo nmap -n -sT -T2 -p 502 --scan-delay 0.3s 192.168.10.50
```

---

## 12. Conservative Host Discovery

### ARP Discovery (same Layer 2 network)
```bash
sudo nmap -n -PR -sn 192.168.10.0/24
```

### netdiscover (alternative)
```bash
sudo netdiscover -r 192.168.10.0/24
```

### Ping Sweep (no port scan)
```bash
sudo nmap -n -sn 192.168.10.0/24
```

---

## 13. Why Aggressive Scans Are Dangerous

Avoid these flags in OT environments:
```
-O     OS detection → probes stack in unpredictable ways
-A     Aggressive → OS + version + scripts + traceroute all at once
-sV    Version detection → sends probe packets that may crash legacy stacks
```

Risks include:
- Unexpected service interaction
- Controller instability / crash
- Legacy TCP/IP stack fragility
- Resource-constrained devices overwhelmed
- Process disruption and safety consequences

**Use `-T2` timing template** instead of default `-T3`.

---

## 14. Conservative TCP Scanning

Only when explicitly authorized and necessary:

```bash
# Connect scan, slow, common ports only
sudo nmap -n -sT -T2 --top-ports 100 192.168.10.0/24

# With scan delay for extra safety
sudo nmap -n -sT -T2 --scan-delay 0.1s --top-ports 100 192.168.10.50

# Specific Modbus port
sudo nmap -n -sT -T2 -p 502 192.168.10.50
```

**Start with the smallest scope necessary.** A positive TCP/502 result means the port is reachable, not automatically that it's a Modbus device.

---

## 15. Modbus NSE Scripts

### Locate Modbus scripts
```bash
ls /usr/share/nmap/scripts/ | grep -i modbus
```

### modbus-discover.nse
```bash
sudo nmap --script modbus-discover -p 502 192.168.10.50
```

### Check script help
```bash
nmap --script-help modbus-discover
```

The script reports: Unit IDs, device identification information, and supported function codes.

---

## 16. Wireshark SCADA Analysis

### Capture traffic
```bash
sudo tcpdump -i any -w scada.pcap host 192.168.10.50
# OR
sudo tcpdump -i any -w modbus.pcap port 502
```

### Open in Wireshark
```bash
wireshark scada.pcap
```

Wireshark natively decodes Modbus packets.

### Useful Display Filters

| Filter | Purpose |
|---|---|
| `modbus` | All Modbus traffic |
| `tcp.port == 502` | Modbus TCP port |
| `modbus.func_code == 3` | Read Holding Registers |
| `modbus.func_code == 6` | Write Single Register |
| `modbus.func_code == 1` | Read Coils |
| `modbus.func_code == 16` | Write Multiple Registers |
| `frame contains "write"` | Find write commands (text search) |
| `tcp.flags.syn == 1 and tcp.flags.ack == 1` | Established connections |
| `tcp.flags.push == 1` | Data being pushed |

### Analyze
- **Statistics → Conversations** — map master/slave relationships, identify IPs
- Look for write operations (function codes 5, 6, 15, 16) — these modify process state
- Baseline normal traffic: who talks to whom, how often, which registers

---

## 17. GrassMarlin

Passive network mapper designed specifically for ICS/SCADA networks. Builds a network topology picture **without active scanning**.

**Purpose:**
- OT network visibility
- Asset relationship mapping
- Protocol analysis
- Traffic baselining

Ideal for initial reconnaissance before any active scanning.

---

## 18. Metasploit SCADA Modules

### Discover Modbus devices
```bash
msfconsole

# Search for Modbus modules
search modbus
search scada

# Modbus detection
use auxiliary/scanner/scada/modbusdetect
set RHOSTS 192.168.10.50
set THREADS 1
run

# Find Unit IDs
use auxiliary/scanner/scada/modbus_findunitid
set RHOSTS 192.168.10.50
run

# Read data once Unit ID is known
use auxiliary/scanner/scada/modbusclient
set RHOSTS 192.168.10.50
set UNIT_NUMBER 1
set DATA_ADDRESS 0
set NUMBER 10
set FUNCTION read_holding_registers    # or read_coils, read_input_registers
run
```

### Shodan search in Metasploit
```bash
use auxiliary/gather/shodan_search
set SHODAN_APIKEY your_key_here
set QUERY modbus
run
```

**Be extremely careful with write functions.** Default to read-only operations.

---

## 19. Shodan & OSINT for SCADA

### Shodan search concepts
```
port:502                        # Modbus
port:47808                      # BACnet
port:102                        # S7comm
port:44818                      # EtherNet/IP
"Schneider Electric"
"Rockwell Automation"
"Siemens S7"
```

### Google Hacking Database
Search for SCADA-specific dorks — exposed web-based HMIs, default pages, vendor-specific interfaces.

### Wayback Machine
Discover historical SCADA administration pages, old product versions, deprecated endpoints.

> Shodan results are reconnaissance, NOT proof of exploitability. Validate before reporting.

---

## 20. Device Separation & Data Diodes

### Assess segmentation
```
IT VLAN ───X─── OT VLAN ───X─── SCADA VLAN ───X─── PLC VLAN
```
Test only explicitly authorized paths.

### Questions to answer
- Can IT reach TCP/502 (Modbus)?
- Can an engineering workstation reach management interfaces?
- Can a user VLAN reach an HMI?
- Can one PLC network reach another?
- Does the data diode genuinely enforce one-way communication?
- Are there alternate paths bypassing the diode?

### Data Diode Assessment
```
Network A → (allowed) → Network B
Network B → (BLOCKED) → Network A
```
Check: Is traffic genuinely one-way? What crosses the boundary? Are management exceptions present?

---

## 21. Authentication Testing

Assess these layers:
```
SCADA application authentication
HMI authentication
Engineering workstation authentication
Remote access (VPN)
Web management interfaces
Industrial gateway authentication
Protocol-level authentication (where supported)
```

Common issues:
- Lack of authentication on Modbus
- Default credentials on HMIs and engineering stations
- Unpatched Windows hosts as HMIs/historians
- Weak or missing network segmentation

---

## 22. Attack Categories in OT

| Attack | Concern | Approach |
|---|---|---|
| **Live Computer Jacks** | Unauthorized physical access to control network | Check port security, NAC, VLAN assignment |
| **Active Port Attacks** | Exposed network interfaces | Identify service, protocol, authentication first |
| **Filtering Bypass** | Bypassing firewall/ACL/data diode | Test allowed vs denied paths, alternate protocols |
| **MITM** | Intercepting/modifying process traffic | Requires strict safety controls |
| **DoS/DDoS** | Loss of visibility/control | Replace with configuration review unless explicitly authorized |

**Do not perform denial-of-service or uncontrolled writes** unless the exam/lab explicitly requires and allows it.

---

## 23. Lab Setup

### Components
```
                 Kali (Attacker)
                   |
              Lab Network (192.168.56.0/24)
                   |
          +--------+--------+
          |                 |
     QModMaster         Modbus-Pal
     (Master)           (Slave)
          |                 |
          +---- TCP/502 ----+
```

### Tools
- **QModMaster** — GUI Modbus master simulator (free)
- **Modbus-Pal** or **diagslave** — Modbus slave emulator (free)
- **Python pymodbus** — Alternative for scripting

### Verify both are running
```bash
sudo netstat -lntp | grep ':502'
# OR
sudo ss -lntp | grep ':502'
```

Expected output:
```
LISTEN ... :502
```

---

## 24. Step-by-Step Lab Workflow

### Step 1: Discover Modbus service
```bash
sudo nmap -sT -T2 -p 502 --open 192.168.56.0/24
```

### Step 2: Run Modbus NSE script
```bash
sudo nmap --script modbus-discover -p 502 192.168.56.20
```

### Step 3: Find Unit ID via Metasploit
```bash
msfconsole
use auxiliary/scanner/scada/modbus_findunitid
set RHOSTS 192.168.56.20
set THREADS 1
run
```

### Step 4: Detect Modbus with found Unit ID
```bash
use auxiliary/scanner/scada/modbusdetect
set RHOSTS 192.168.56.20
set UNIT_ID 1
run
```

### Step 5: Read data (Holding Registers)
```bash
use auxiliary/scanner/scada/modbusclient
set RHOSTS 192.168.56.20
set UNIT_NUMBER 1
set DATA_ADDRESS 0
set NUMBER 10
set FUNCTION read_holding_registers
run
```

### Step 6: Capture and analyze traffic
```bash
sudo tcpdump -i any -w modbus_lab.pcap port 502
# Run steps 1-5, then stop capture (Ctrl+C)
wireshark modbus_lab.pcap
# In Wireshark: filter "modbus", inspect conversations, function codes
```

### Step 7: Read coils
```bash
use auxiliary/scanner/scada/modbusclient
set RHOSTS 192.168.56.20
set UNIT_NUMBER 1
set DATA_ADDRESS 0
set NUMBER 10
set FUNCTION read_coils
run
```

### Step 8: Document everything
Screenshots of: discovery, Unit ID enumeration, register reads, Wireshark analysis.

---

## 25. Practical Exam Workflow

```
1. Passive recon first (GrassMarlin, existing PCAP, network taps)
        |
2. Polite host discovery (-sn, -PR, netdiscover)
        |
3. Identify hosts with port 502 (Modbus) open
        |
4. Conservative TCP scan on common OT ports (-T2, --scan-delay)
        |
5. Run modbus-discover.nse OR Metasploit unit ID finder
        |
6. Read data carefully (coils → holding registers)
        |
7. Capture traffic and analyze in Wireshark
        |
8. Look for associated IT assets (Windows HMIs, engineering workstations)
        |
9. Pivot to normal Windows/AD techniques if HMIs are found
        |
10. Document every interaction (screenshots of reads are critical)
```

---

## 26. Tools & Techniques Mapping

| Technique | Tool | Command |
|---|---|---|
| ARP Discovery | nmap | `nmap -n -PR -sn TARGET` |
| Host Discovery | netdiscover | `sudo netdiscover -r SUBNET` |
| Ping Sweep | nmap | `nmap -n -sn TARGET` |
| Safe TCP Scan | nmap | `nmap -n -sT -T2 --top-ports 100 IP` |
| Modbus Discovery | nmap NSE | `nmap --script modbus-discover -p 502 IP` |
| Modbus Detection | Metasploit | `auxiliary/scanner/scada/modbusdetect` |
| Unit ID Discovery | Metasploit | `auxiliary/scanner/scada/modbus_findunitid` |
| Modbus Read/Write | Metasploit | `auxiliary/scanner/scada/modbusclient` |
| Traffic Capture | tcpdump | `tcpdump -i any -w file.pcap port 502` |
| Protocol Analysis | Wireshark | `wireshark file.pcap` (filter: `modbus`) |
| Passive Network Map | GrassMarlin | GUI-based ICS network mapping |
| Config Auditing | Nipper Studio | Network device security analysis |
| Lab Master | QModMaster | GUI Modbus master emulator |
| Lab Slave | Modbus-Pal | GUI Modbus slave emulator |
| OSINT | Shodan | `port:502`, `port:102`, vendor searches |
