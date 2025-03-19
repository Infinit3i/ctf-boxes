smbmap -H <remote_ip> -u <username> -p <password>

smbclient //10.10.10.100/<share_name> -U ""%""
    RECURSE ON
    PROMPT OFF
    mget *
