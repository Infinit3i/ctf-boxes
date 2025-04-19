```shell
feroxbuster -u http://10.10.11.11 -x php
```

```bash
ffuf -u http://10.10.11.11 -H "Host: FUZZ.board.htb" -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -mc all
```
