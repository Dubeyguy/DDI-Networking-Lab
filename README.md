# Linux DDI (DNS, DHCP, IPAM) & Packet Analysis Lab

A local enterprise networking lab built on **Arch Linux** demonstrating core **DDI (DNS, DHCP, IP Address Management)** infrastructure, Layer 3 NAT routing, and packet-level root-cause analysis using `dnsmasq`, `iptables`, `tcpdump`, and Wireshark.

## Architecture

```text
[Client Device: 10.10.10.63 / 10.10.10.140]
       │  (Wi-Fi / Ethernet)
       ▼
[Secondary Router: Layer 2 Bridge / AP]  (DHCP disabled)
       │  (Ethernet)
       ▼
[Arch Linux Gateway (enp63s0): 10.10.10.1/24]
       ├── dnsmasq (UDP 67): DHCP Pool (10.10.10.50–150) + Options 3 & 6
       ├── dnsmasq (UDP 53): Local Zones (*.lab.local) + Upstream Forwarding (8.8.8.8, 1.1.1.1)
       └── Kernel / iptables: IPv4 Forwarding + NAT Masquerade ──► WAN (Wi-Fi) ──► Internet
```

## Repository Structure

```text
linux-ddi-network-lab/
├── README.md
├── configs/
│   ├── dnsmasq.conf              # Interface-isolated DHCP/DNS config
│   ├── iptables.rules            # NAT masquerade & stateful FORWARD rules
│   └── sample_leases.txt         # Sanitized DHCP lease table
├── captures/
│   ├── baseline_ddi_clean.pcap   # DHCP DORA handshake & normal DNS resolution
│   ├── broken_dns_clean.pcap     # Simulated UDP/53 firewall drop outage
│   └── dns_resolved_clean.pcap   # Post-remediation DNS recovery
└── screenshots/
    ├── dhcp_dora.png             # Wireshark DHCP Discover/Offer/Request/ACK
    ├── baseline_dns.png          # Recursive DNS query-response flow
    ├── broken_dns_timeout.png    # Unanswered DNS queries (3s retry deltas)
    └── dns_resolved.png          # Sub-60ms DNS recovery after rule fix
```

*(All `.pcap` files and screenshots are sanitized via `tcprewrite` to mask hardware MAC addresses.)*

## Core Setup

**1. Gateway Interface (`NetworkManager`)**
```bash
sudo nmcli connection modify "Wired connection 1" ipv4.method manual ipv4.addresses 10.10.10.1/24 ipv4.gateway "" ipv4.dns "" ipv4.never-default yes
sudo nmcli connection up "Wired connection 1"
```

**2. DDI Service (`configs/dnsmasq.conf`)**
* Locked strictly to `enp63s0` (`bind-interfaces`, `except-interface=lo`) to prevent loopback port `53` collisions.
* Assigns `10.10.10.50–150` with Option 3 (`Router: 10.10.10.1`) and Option 6 (`DNS: 10.10.10.1`), plus authoritative local records (`gateway.lab.local`, `app.lab.local`).

**3. Kernel Forwarding & Stateful NAT**
```bash
WAN_IFACE=$(ip route | awk '/default/ {print $5; exit}')
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o "$WAN_IFACE" -j MASQUERADE
sudo iptables -A FORWARD -i enp63s0 -o "$WAN_IFACE" -j ACCEPT
sudo iptables -A FORWARD -i "$WAN_IFACE" -o enp63s0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

## Root-Cause Troubleshooting Case Study

| Phase | Symptom | Packet Evidence (`Wireshark`) | Root Cause & Fix |
| :--- | :--- | :--- | :--- |
| **Baseline** (`baseline_ddi_clean.pcap`) | Client obtains `10.10.10.63` and resolves domains normally. | Complete 4-way **DHCP DORA** + `Standard query` / `response` pairs. | Healthy L2 bridge transport and `dnsmasq` Option 3/6 provisioning. |
| **Outage** (`broken_dns_clean.pcap`) | Client pings `10.10.10.1` fine, but all DNS lookups time out. | Outbound `A`/`AAAA` queries hit `10.10.10.1:53` with `3.0s` retries and **0 replies**. | Injected `iptables INPUT DROP` rule on `udp/53` blocked queries before `dnsmasq`. |
| **Recovery** (`dns_resolved_clean.pcap`) | Domain resolution and internet access restored immediately. | `Standard query response` packets resume in `~56ms`. | Removed blocking `INPUT` rule (`iptables -D INPUT ...`) and verified via `dig`. |
