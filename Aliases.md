
```shell
htbscan() {
    if [ "$#" -ne 2 ]; then
        echo "Usage: htbscan <IP> <BOX_NAME>"
        return 1
    fi

    IP="$1"
    BOX="$2"
    CURRENT_USER=$(whoami)
    NMAP_DIR="/home/${CURRENT_USER}/HTB/${BOX}/nmap"
    mkdir -p "$NMAP_DIR"

    echo "[+] Running full TCP port scan on $IP..."
    nmap -sT -p- --min-rate 10000 -oA "$NMAP_DIR/${BOX}_tcp" "$IP"

    echo "[+] Extracting open ports..."
    PORTS=$(grep -oP '^[0-9]+(?=/tcp\s+open)' "$NMAP_DIR/${BOX}_tcp.nmap" | paste -sd, -)

    if [ -z "$PORTS" ]; then
        echo "[-] No open ports found."
        return 1
    fi

    echo "[+] Running service scan on detected ports: $PORTS"
    nmap -sCV -p "$PORTS" -oA "$NMAP_DIR/${BOX}_svc" "$IP"

    echo "[+] Checking for Vhost"
    VHOST=$(grep -oP '\|_http-title: Did not follow redirect to http://\K[^/]*\.htb' "$NMAP_DIR/${BOX}_svc.nmap" | head -n 1)
    if [ -n "$VHOST" ]; then
        echo "[+] Found vhost: $VHOST — Appending to /etc/hosts..."
        echo "$IP    $VHOST" | sudo tee -a /etc/hosts > /dev/null
    else
        echo "[-] No .htb vhost detected."
    fi
}
```
