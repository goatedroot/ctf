# MonitorsFour - Hack The Box Writeup
**Category:** Windows  
**Difficulty:** Easy 

## Enumeration

### Nmap

Initial scan revealed two accessible services:

```text
80/tcp   open  http    nginx
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (WinRM)
```

---

# Web Enumeration

## Gobuster

Running Gobuster against the website:

```bash
gobuster dir -u http://monitorsfour.htb \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Results:

```text
/contact              (Status: 200)
/login                (Status: 200)
/user                 (Status: 200)
/static               (Status: 301)
/views                (Status: 301)
/forgot-password      (Status: 200)
/controllers          (Status: 301)
```

---

## Information Disclosure

### `/contact`

Visiting `/contact` revealed a PHP warning:

```text
Warning: include(/var/www/app/views/contact.php):
Failed to open stream: No such file or directory

Warning: include():
Failed opening '/var/www/app/views/contact.php'
for inclusion
```

This immediately disclosed:

```text
/var/www/app/Router.php
```

and suggested that the application dynamically loads files using PHP's `include()` function.

The warning also revealed:

```text
include_path='.:/usr/local/lib/php'
```

which indicates PHP searches for files in:

```text
.
(current directory)

/usr/local/lib/php
```

---

### `/views`

Browsing `/views` showed raw HTML templates.

This strongly suggested that requests were routed through something similar to:

```php
include('/var/www/app/views/' . $page . '.php');
```

Meaning requests such as:

```text
/contact
```

were likely translated into:

```text
/var/www/app/views/contact.php
```

by the router.

At this point I suspected there might be an LFI vulnerability hidden somewhere in the routing logic.

---

## API Enumeration

The application made extensive use of API endpoints.

Enumerating `/api/v1/` revealed:

```text
/user
/users
/logout
/auth
```

Results:

```text
/api/v1/user
/api/v1/users
/api/v1/logout
/api/v1/auth
```

---

# Authentication Bypass

Visiting:

```text
/api/v1/user
```

returned:

```json
{
  "error": "Missing token parameter"
}
```

This suggested that access control relied on a token value.

---

## Type Juggling Vulnerability

The token validation appeared vulnerable to PHP type juggling.

Supplying:

```text
token=0
```

successfully bypassed the check.

The likely cause was code similar to:

```php
if ($_GET['token'] == $stored_hash)
```

instead of:

```php
if ($_GET['token'] === $stored_hash)
```

Several user tokens began with:

```text
0e...
```

which PHP interprets as scientific notation:

```text
0e12345 = 0
```

As a result:

```php
"0" == "0e123456789"
```

evaluates to:

```php
true
```

allowing authentication bypass.

---

# User Enumeration

Using the vulnerability, I extracted user information.

Interesting entries:

```json
{
  "username": "admin",
  "email": "admin@monitorsfour.htb",
  "password": "56b32eb43e6f15395f6c46c1c9e1cd36",
  "role": "super user",
  "token": "dbe13a95bed2af875f"
}
```

```json
{
  "username": "mwatson",
  "token": "0e543210987654321"
}
```

```json
{
  "username": "janderson",
  "token": "0e999999999999999"
}
```

```json
{
  "username": "dthompson",
  "token": "0e111111111111111"
}
```

---

## Password Cracking

The passwords were MD5 hashes.

Admin hash:

```text
56b32eb43e6f15395f6c46c1c9e1cd36
```

Cracked to:

```text
wonderful1
```

Using the credentials:

```text
admin : wonderful1
```

I logged into the application.

---

## WinRM Attempt

Since port 5985 was exposed, I attempted:

```bash
evil-winrm -i monitorsfour.htb -u admin -p wonderful1
```

However, authentication failed.

The credentials were valid only for the web application.

---

# Internal Information Disclosure

While exploring the admin panel, I found:

```text
/admin/changelog
```

One entry stood out:

```text
To enhance our product delivery, we have migrated to Windows and ported websites to Docker via Docker Desktop 4.44.2
```

This revealed several important facts:

1. The host was Windows. (we already know)
2. Docker Desktop was installed.
3. The version was:

```text
Docker Desktop 4.44.2
```

---

## Researching Docker Desktop

Research showed that Docker Desktop 4.44.2 was affected by:

```text
CVE-2025-9074
```

a critical vulnerability with a high CVSS score.

Before attempting exploitation, I needed access to Docker.

---

# Docker Enumeration

Checking the Docker API port:

```bash
nmap -p 2375 monitorsfour.htb
```

Result:

```text
2375/tcp filtered docker
```

The Docker daemon appeared protected by a firewall.

At this point I assumed there might be an internal route or API functionality that could eventually expose it.

---

## API Key Generation

The application allowed generation of API keys.

Generated key:

```text
e893dab37187e48ca3
```

Although interesting, it ultimately wasn't required for the final compromise.

---

# Subdomain Enumeration

After vhost enumeration, I discovered:

```text
cacti.monitorsfour.htb
```

After adding it to `/etc/hosts`, I accessed the application.

---

# Cacti Exploitation

Using the previously recovered credentials:

```text
Marcus / wonderful1
```

I successfully logged into Cacti.

Version enumeration revealed a vulnerable release affected by:

```text
CVE-2025-22604
CVE-2025-24367
```

I chose to exploit:

```text
CVE-2025-24367
```

which resulted in remote code execution.

A shell was obtained on the target.

---

# Docker Access

Once inside the system, I tested connectivity to the Docker API:

```bash
curl http://192.168.65.7:2375/info
```

Success.

The Docker daemon was accessible internally.

---

# Exploiting Docker Desktop

The original plan was to transfer and execute the Docker Desktop exploit directly on the target.

However, Python was not installed.

Instead of attempting to port the exploit manually, I routed traffic through the compromised host using Ligolo.

Start Ligolo on the pivot:

```bash
ligolo-ng
```

After establishing the tunnel, I executed the exploit directly from my attacking machine against the internal Docker daemon.

This worked successfully.

---

# Final Compromise

Using the Docker Desktop vulnerability:

```text
CVE-2025-9074
```

I achieved code execution on the underlying Windows host.

From there I launched a reverse shell and obtained full access.

User and root flags were successfully captured.

---

# Lessons Learned

- Error messages often disclose valuable filesystem paths.
- PHP type juggling vulnerabilities remain surprisingly common.
- API endpoints frequently expose sensitive functionality.
- Internal changelogs can reveal critical infrastructure details.
- Docker environments should always be investigated when exposed.
- Virtual host enumeration can uncover entirely new attack surfaces.
- Pivoting tools such as Ligolo can dramatically simplify exploitation of internal services.
- Internal-only services may become accessible after gaining an initial foothold.

---

**Personal Opinion:** The most interesting part of this machine was the chain of discoveries. The initial foothold came from a classic PHP type-juggling issue, but the real challenge was recognizing the significance of the Docker Desktop version disclosure and eventually pivoting through Cacti to reach the internal Docker API. The box rewarded careful enumeration far more than brute-force exploitation.
