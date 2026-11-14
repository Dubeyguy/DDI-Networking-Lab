# Linux DDI (DNS, DHCP, IPAM) & Packet-Level Troubleshooting Lab

A hands-on enterprise networking lab built on **Arch Linux** to simulate core **DDI (DNS, DHCP, IP Address Management)** infrastructure, Layer 3 NAT routing, and packet-level root-cause analysis (RCA) using `dnsmasq`, `iptables`, `tcpdump`, and Wireshark.

---

## Architecture & Topology

In this setup, a secondary consumer router is stripped of its Layer 3 routing and DHCP functions and converted into a pure **Layer 2 Bridge / Wireless Access Point**. An Arch Linux gateway (`10.10.10.1/24`) acts as the centralized DDI server and stateful NAT router for all downstream clients.

```text
[Client Device: 10.10.10.63 / 10.10.10.140]
       │  (802.11 Wi-Fi / Ethernet)
       ▼
[Secondary Router: Pure Layer 2 Bridge / AP]  (Built-in DHCP disabled)
       │  (Ethernet Link)
       ▼
[Arch Linux Gateway (enp63s0): 10.10.10.1/24]
       ├── dnsmasq (UDP 67): DHCP IPAM Pool (10.10.10.50–150) + Options 3 & 6
       ├── dnsmasq (UDP 53): Authoritative Local Zones (*.lab.local) + Upstream Forwarding
       └── Linux Kernel / iptables: IPv4 Forwarding + Stateful NAT Masquerade ──► WAN (Wi-Fi) ──► Internet
