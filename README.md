# CodeAlpha_NetworkSniffer

**CodeAlpha Cyber Security Internship – Task 1: Basic Network Sniffer**

A Python packet sniffer built with [Scapy](https://scapy.net/). It captures live traffic, decodes packet layers, and shows how data flows through a network.

## Features
- Live capture on any interface, with optional **BPF filters** (`tcp port 80`, `udp port 53`, ...)
- Decodes **Ethernet → IP (v4/v6) → TCP / UDP / ICMP / ARP**
- Shows source/destination IP, ports (with service names), TCP flags, packet length
- Application hints: **DNS queries**, **HTTP request lines**, **TLS/HTTPS** detection
- **Payload preview** (`-v`) in safe printable form
- End-of-capture **summary**: protocol breakdown and top talkers
- Export to **CSV** and **.pcap** (open in Wireshark)
- Offline analysis of existing `.pcap` files

## Setup
```bash
git clone https://github.com/<your-username>/CodeAlpha_NetworkSniffer.git
cd CodeAlpha_NetworkSniffer
pip install -r requirements.txt
```
- **Windows:** install [Npcap](https://npcap.com/) and run the terminal as Administrator.
- **Linux/macOS:** run with `sudo`.

## Usage
```bash
sudo python3 sniffer.py                          # sniff until Ctrl+C
sudo python3 sniffer.py -c 50 -v                 # 50 packets, show payloads
sudo python3 sniffer.py -f "tcp port 80" -v      # only HTTP traffic
sudo python3 sniffer.py -f "udp port 53"         # only DNS
sudo python3 sniffer.py -i wlan0 --csv out.csv --pcap out.pcap
python3 sniffer.py -r out.pcap                   # analyse a saved capture
python3 sniffer.py --list-ifaces
```

| Option | Description |
|---|---|
| `-i, --iface` | Interface to sniff on |
| `-c, --count` | Number of packets (0 = unlimited) |
| `-f, --filter` | BPF filter expression |
| `-v, --verbose` | Show payload preview |
| `--csv FILE` | Log packets to CSV |
| `--pcap FILE` | Save raw packets |
| `-r, --read FILE` | Read from a pcap instead of live capture |

## Sample output
```
[   1] 10:40:40.414  HTTP   192.168.1.5:51000 -> 93.184.216.34:80/HTTP  len=101  flags=PA  | GET /index.html HTTP/1.1
        payload: GET /index.html HTTP/1.1..Host: example.com....
[   2] 10:40:40.414  DNS    192.168.1.5:5353/mDNS -> 8.8.8.8:53/DNS  len=70  | Query: openai.com
[   3] 10:40:40.415  ICMP   192.168.1.5 -> 8.8.8.8  len=42  | Echo Request
[   4] 10:40:40.415  TCP    192.168.1.5:51001 -> 1.1.1.1:443/HTTPS  len=54  flags=S  | TLS/HTTPS (encrypted)
```

## Web dashboard (Docker) — http://localhost:5555

A live dashboard (Flask) on top of the same sniffer engine, packaged as one Docker project (`codealpha_networksniffer`). **Start the project = capture starts. Stop the project = capture stops.**

**Features:** live packet table (protocol filter, search, pause, clear, export CSV), packets-per-second graph, protocol distribution, top sources / destinations / ports, security alerts (port-scan heuristic, cleartext Telnet/FTP, credentials in plain HTTP), and manual Start/Stop, interface and filter controls.

### Windows / macOS (Docker Desktop) — real PC traffic
Docker containers run inside a VM and cannot see your PC's network cards, and a web page must never be able to launch programs on your PC. So real capture uses two small Windows-side pieces:
- **Watcher** (`agent.py --supervise`) – tiny, never captures. It launches **one fixed script** (the agent) only when you press **Start agent** in the dashboard.
- **Agent** (`agent.py`) – captures **real** packets and streams them to the dashboard. It exits completely on **Stop agent** or when Docker stops.

Prerequisites: Docker Desktop, [Npcap](https://npcap.com/), Python, and `pip install scapy`.

**One-time setup** (PowerShell as Administrator, in the project folder):
```powershell
powershell -ExecutionPolicy Bypass -File windows\install-agent.ps1
```
This starts the watcher at every login (remove with `-Remove`). Prefer no installation? Use `start.bat` / `stop.bat` instead (they start/stop Docker + watcher together).

**Daily use**
1. Press **Start** on the project in Docker Desktop (or `docker compose up -d`) → dashboard loads at http://localhost:5555. Nothing is captured yet.
2. Choose Source **My PC via agent.py** → press **▶ Start agent** (a fresh agent process launches).
3. Pick your Wi-Fi/Ethernet interface → press **▶ Start capture** → browse a site and watch real packets.
4. **■ Stop capture** pauses capture; **■ Stop agent** ends the agent. **Stop** in Docker Desktop stops the dashboard and the agent exits within ~10 s.

### Security design
- Dashboard bound to `127.0.0.1` only (Docker port published to localhost).
- Every state-changing request needs a custom `X-CodeAlpha` header, a local `Origin` and a local `Host`, so other websites open in your browser cannot start/stop capture (CSRF / DNS-rebinding protection).
- The dashboard can only *request* "start/stop agent"; it cannot choose what program runs or pass commands. The watcher only ever runs `agent.py` from the project folder.
- Captured data never leaves your computer.

### Linux — capture directly from Docker
```bash
docker compose --profile linux up --build   # capture starts immediately on the host network
docker compose --profile linux down         # stops it
```

### Without Docker
```powershell
pip install -r requirements.txt
python dashboard.py        # run as Administrator (Npcap installed)
```
Choose *This machine – live capture*, pick the interface, press Start.

### Options (environment variables in `docker-compose.yml`)
| Variable | Meaning | Default |
|---|---|---|
| `AUTOSTART` | `agent` / `live` = start capturing automatically | off (`live` in the Linux profile) |
| `AUTOSTART_IFACE` | interface for auto-start | first usable interface |
| `AUTOSTART_FILTER` | capture filter for auto-start | none |
| `PORT` / `HOST` | dashboard port / bind address | 5555 / 127.0.0.1 |

> **Security note:** the dashboard shows captured traffic, so it is published to `127.0.0.1` only. Don't expose port 5555 to other machines. Capture only networks you own or have permission to monitor.

## How it works
1. `scapy.sniff()` captures frames and passes each to a callback.
2. `parse_packet()` walks the layers (`pkt.haslayer(...)`) and extracts fields.
3. Raw payload bytes are inspected for HTTP/DNS patterns and shown as printable text.
4. Stats are collected and printed when capture ends.

## Protocol basics learned
- **TCP** – reliable, connection-based (flags: `S`=SYN, `A`=ACK, `F`=FIN, `P`=PSH, `R`=RST)
- **UDP** – fast, connectionless (DNS, streaming)
- **ICMP** – diagnostics (`ping`)
- **ARP** – maps IP addresses to MAC addresses on a LAN
- HTTP is readable in plain text; **HTTPS payloads are encrypted**

## Legal & ethical notice
Use only on networks you own or have written permission to monitor. Capturing others' traffic without consent may be illegal. For educational use.

## Author
Your Name – CodeAlpha Cyber Security Intern
