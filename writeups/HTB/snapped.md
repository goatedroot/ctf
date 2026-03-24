# Snapped — HackTheBox

**Category:** Web / Linux  
**Difficulty:** Hard  

---

## nmap

```
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
```

---

## enumeration

subdomain fuzzing:

```
admin                   [Status: 200, Size: 1407, Words: 164, Lines: 50, Duration: 294ms]
```

this subdomain was running Nginx UI. i couldnt see any versions but i still searched for its vulnerabilities. i also tried default creds but didnt work

there was a vulnerability in versions < 2.3.3 of Nginx UI where any user without even any authentication can access the `/api/backup` endpoint and download the backup file — so i tried it and it worked!

---

## foothold — CVE (CVE-2026-27944)

unzipping the backup file we got two more `.zip` files and a `hash_info.txt` which was not in plain text

tried using hashcat on `hash_info.txt` but it couldnt recognize the hash format. tried unzipping the other two `.zip` files but both showed errors — running `file nginx.zip` and `file nginx-ui.zip` both showed `data` which means no recognizable magic bytes, meaning they are encrypted

my fault cuz i didnt read the full CVE — the encryption key is also sent in the header. there was also a POC script provided so i used that:

```bash
python3 poc.py --target http://admin.snapped.htb --decrypt
```

POC: https://github.com/advisories/GHSA-g9w5-qffc-6762

got the whole decrypted backup. in `nginx-ui` there was a `database.db` file — opened it using `sqlite3` and from the users table got hashes for two users `admin` and `jonathan`

only jonathan's hash could be cracked:

```
jonathan : linkinpark
```

these creds worked for ssh aswell:

```bash
ssh jonathan@snapped.htb
```

---

## internal recon

there were no other users in the machine and this is rated as hard machine so I checked for internal networks via `ifconfig`, `ip a`, `/etc/resolv.conf` and found nothing — single machine

checked listening ports:

```bash
ss -tulnp
```

found two new ports running internally:

```
127.0.0.1:9000
127.0.0.1:631
```

- port `9000` — another internal Nginx UI instance (same CVE we just exploited)
- port `631` — running CUPS 2.4.7 (vulnerable to CVE-2024-47177 RCE but requires UDP port 631 to be up, which it isnt)

also noticed `snap` is installed in the user's home directory which felt like a hint:

```bash
snap --version
# 2.63.1
```

vulnerable to CVE-2026-3888. checking the OS:

```bash
cat /etc/os-release
# Ubuntu 24.04.4 — also vulnerable
```

the problem is this attack normally requires at least 30 days because thats when `/tmp` gets cleaned. but:

```bash
cat /usr/lib/tmpfiles.d/tmp.conf
# cleanup set to every 4 minutes!
```

---

## privilege escalation — CVE-2026-3888 (snap-confine + systemd-tmpfiles LPE)

i started studying the CVE because there must be a reason why this is a hard box. its hard to explain in short but i'll try:

**two components involved:**

```
1. snap-confine: creates temporary directories in /tmp for snap sandboxes.
   has root SUID. during sandbox setup, creates /tmp/.snap to build
   "mimics" — writable copies of read-only directories.

2. systemd-tmpfiles: automatically cleans up old files in /tmp.
   runs as root with configured time thresholds.
```

**how the exploit works:**

```
a) start a snap (like Firefox) which creates /tmp/.snap inside the sandbox (root-owned)

b) inside the sandbox, start a keepalive process that writes to /tmp (but NOT to .snap).
   wait for systemd-tmpfiles to delete /tmp/.snap (old and untouched).
   /tmp remains because its kept active by the keepalive writes.
   destroy the sandbox (keeping /tmp).

c) from outside the sandbox, navigate to the sandbox's /tmp via /proc/[pid]/cwd
   and create our own .snap directory (now owned by us).
   inside it, create /tmp/.snap/usr/lib/x86_64-linux-gnu.exchange,
   copy legitimate libraries into it, but replace ld-linux-x86-64.so.2
   (the dynamic loader) with malicious shellcode.

d) force snap-confine to set up the sandbox again (start Firefox again).
   during mimic creation of /usr/lib/x86_64-linux-gnu, snap-confine performs 3 steps:
     Step 1: bind-mount the original directory to /tmp/.snap/usr/lib/x86_64-linux-gnu
     Step 2: mount a new tmpfs over /usr/lib/x86_64-linux-gnu (empty but writable)
     Step 3: bind-mount everything back from .snap to /usr/lib/x86_64-linux-gnu

e) between Step 1 and Step 3, quickly replace /tmp/.snap/usr/lib/x86_64-linux-gnu
   with our malicious exchange directory. when Step 3 executes, snap-confine
   bind-mounts our malicious files (including the hijacked dynamic loader)
   into /usr/lib/x86_64-linux-gnu.

f) now we control the dynamic loader inside the sandbox. when we execute any
   dynamically-linked SUID-root binary (like snap-confine itself), the dynamic loader
   runs first with root privileges. our malicious shellcode executes,
   giving us a root shell inside the sandbox.

g) from inside this root shell, create a SUID bash binary in /var/snap/firefox/common/,
   exit the sandbox, and execute it — full root access on the host system.
```

exploited using a POC from: https://github.com/TheCyberGeek/CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE

and got ROOT!!

---

## proof

![root](../../proof/snapped.png)
