## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)](Easy)

## 🔍 Recon

```bash
nmap -p- -T4 10.10.11.X  # Full port scan to discover open services
```

> Only port [80](HTTP.md) was relevant and led to the vulnerable web page.

---

## 🧪 XSS Testing

```http
GET /?input=%0A
```

> `"%0A"` is a **newline character** that bypassed the input filter.

---

## 🐍 SSTI Testing (Ruby-based SSTI Payloads)

Use resources like:

- https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection
- https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection

Example payload (Ruby SSTI — URL-encoded key characters):

```http
{{7*7}}
```

> When URL-encoded, key characters bypassed filters and enabled enumeration.

---

## 🧭 Directory Enumeration (Burp Repeater)

Used Burp Suite to:
- Enumerate directories and files
- Manually modify SSTI payloads
- Identify accessible paths and sensitive files

---

## 🔑 Password Hash Discovery

While browsing exposed files, discovered a file containing:

- Username
- SHA256 hash
- Password format hint:  
  `"susan_nasus_<random_number between 1–1,000,000,000>"`

---

## 💥 Cracking Password with Hashcat

Brute-force the password using **mask attack**:

### Initial (no mode):

```bash
hashcat -a 3 hash.txt ?d?d?d?d?d?d?d?d?d
```

> `?d` = digit; 9-digit numeric brute-force

### Refined (SHA-256 mode 1400):

```bash
hashcat -m 1400 -a 3 hash.txt susan_nasus_?d?d?d?d?d?d?d?d?d
```

> Fast crack confirmed by matching the correct pattern.

---

## 🔐 SSH Access

```bash
ssh susan@10.10.11.X
```

> Logged in using the cracked password.