# HTB Academy — Network Foundations

> Module notes + skills assessment writeup. Foundational networking concepts for infosec, ending with a guided lab on the HTB Pwnbox and a target machine (FTP/HTTP enumeration).

**Tools used:** `ifconfig` · `netstat` · `ip route` · `ping` · `nmap` · `nc` (netcat) · `curl`

---

## 1. Network Components

| Component | Examples | Notes |
|---|---|---|
| **End Devices (hosts)** | PCs, phones, IoT | Generate/consume data; user's interface to the network |
| **Intermediary Devices** | Routers, switches, modems, APs | Forward packets, manage traffic, enforce security |
| **Network Media & Software** | Cables, Wi-Fi, protocols, firewalls | Physical pathways + rules for data transmission |
| **Servers** | Web, file, mail, DB | Provide services to clients (client-server model) |

**Intermediary devices by OSI layer:**

| Device | OSI Layer | Role |
|---|---|---|
| **Router** | Layer 3 (Network) | Forwards packets *between networks* by IP; uses routing tables/protocols (OSPF, BGP) |
| **Switch** | Layer 2 (Data Link) | Forwards frames *within* a LAN by MAC address |
| **Hub** | Layer 1 (Physical) | Legacy; broadcasts to all ports (inefficient, collisions) |
| **NIC** | Layer 2 | Hardware interface; holds the device's unique MAC address |

**Key concepts:** Fiber-optic = long distance, minimal loss. Ethernet = high-speed LAN. Software/host-based firewall protects a single device (e.g. Linux `iptables`).

---

## 2. Network Communication & Addressing

Three pillars of communication: **MAC addresses**, **IP addresses**, **ports**.

| Identifier | OSI Layer | Format / Range | Scope |
|---|---|---|---|
| **MAC** | Layer 2 | 48-bit hex, `00:1A:2B:3C:4D:5E` (first 24 bits = OUI/vendor) | Local segment |
| **IPv4** | Layer 3 | 32-bit, `192.168.1.1` | Routable across networks |
| **IPv6** | Layer 3 | 128-bit, `2001:0db8:...:7334` | Created to fix IPv4 exhaustion |
| **Port** | Layer 4 (Transport) | `0`–`65535` (TCP/UDP) | Directs traffic to the right app |

**ARP** = maps a known IP → its MAC address within a LAN (bridges Layer 3 logical and Layer 2 physical addressing).

**Port ranges:**

| Range | Name | Use |
|---|---|---|
| 0–1023 | Well-known | Standard services (HTTP 80, HTTPS 443, FTP 20/21) |
| 1024–49151 | Registered | Vendor-assigned (e.g. MSSQL 1433) |
| 49152–65535 | Dynamic/ephemeral | Temporary client-side session ports |

**Web request order:** DNS Lookup → Encapsulation (HTTP→TCP→IP→Ethernet) → Transmission → Server processing → Response.

---

## 3. DHCP — Automatic IP Assignment

Automates IP config (IP, subnet mask, gateway, DNS) so admins don't assign manually. Process = **DORA**:

| Step | Message | What happens |
|---|---|---|
| **D**iscover | client → broadcast | New device looks for DHCP servers |
| **O**ffer | server → client | Server proposes an IP lease |
| **R**equest | client → server | Client accepts the offered IP |
| **A**cknowledge | server → client | Server confirms; IP now usable |

IPs are leased (time-limited) and must be renewed before expiry.

---

## 4. NAT — Network Address Translation

Lets many private IPs share one public IP (conserves IPv4 + adds a security layer by hiding the internal network).

**Private ranges (RFC 1918):** `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` — not routable on the internet.

| NAT Type | Mapping |
|---|---|
| **Static NAT** | One-to-one (one private ↔ one public) |
| **Dynamic NAT** | Public IP pulled from a pool as needed |
| **PAT (NAT Overload)** | Many private IPs share one public IP, separated by unique port numbers — most common in homes |

NAT is typically performed by the **router** (LAN interface = private, WAN interface = public).

---

## 5. DNS — The Internet's Phonebook

Resolves human-friendly domain names → IP addresses.

**Hierarchy:** Root → TLD (`.com`, `.org`) → Second-level (`example`) → Subdomain/host (`www`).

**Resolution steps:** type domain → check local **DNS cache** → query **recursive DNS server** (ISP/Google DNS) → recursive hits **root server** → pointed to **TLD name server** → pointed to **authoritative name server** → IP returned.

---

## 6. Internet Architectures

| Architecture | Centralized? | Use Cases |
|---|---|---|
| **P2P** | Decentralized | File-sharing (BitTorrent), blockchain — every node is client *and* server |
| **Client-Server** | Centralized | Websites, email |
| **Hybrid** | Partial | Messaging/video conferencing — server handles auth/coordination, data flows peer-to-peer |
| **Cloud** | Provider infra | SaaS (Google Drive, Dropbox), PaaS |
| **SDN** | Centralized control plane | Datacenters, large enterprises |

**Client-Server tiers:** Single-tier (all on one machine) → Two-tier (client + server) → Three-tier (+ application server) → N-tier (multiple app-server layers).

**SDN key idea:** separates the **control plane** (decides where traffic goes) from the **data plane** (forwards traffic), centralizing control in a software controller. *Answer format note: "Software-Defined Networking."*

---

## 7. Wireless Technologies

- **Wireless router** = routing + Wi-Fi access point in one (WAN port to ISP, LAN ports for wired devices, antennae).
- **Mobile hotspot** = shares cellular data via Wi-Fi.
- **Cell tower** = creates a cellular "cell"; managed by **Base Station Controllers (BSC)**, linked to core network via backhaul (fiber/microwave).

| Band | Trait |
|---|---|
| **2.4 GHz** | Better wall penetration, more interference (older 802.11b/g/n) |
| **5 GHz** | Faster, shorter range (802.11a/n/ac/ax) |
| **Cellular** | 700 MHz → 28+ GHz (4G/5G) |

Rule of thumb: lower frequency = farther range, less data; higher = more data, shorter range.

---

## 8. Network Security

**Goal = CIA triad:** Confidentiality, Integrity, Availability.

**Firewall types:**

| Type | OSI Layer | What it inspects |
|---|---|---|
| **Packet Filtering** | L3/L4 | Source/dest IP, port, protocol |
| **Stateful Inspection** | L3/L4 | Tracks connection state (whole conversation) |
| **Application Layer (Proxy)** | L7 | Actual content (e.g. malicious HTTP requests) |
| **Next-Gen (NGFW)** | L3–L7 | Stateful + **deep packet inspection** + IDS/IPS + app control |

**IDS vs IPS:**

| System | Action |
|---|---|
| **IDS** | Detects + **alerts** (does not block) |
| **IPS** | Detects + **prevents/blocks** in real time |

**Detection methods:** Signature-based (matches known-exploit DB) · Anomaly-based (flags unusual behavior).

**Best practices:** least-privilege policies, regular updates, monitor/log, defense in depth (layered), periodic pentesting.

---

## 9. Skills Assessment — Lab Walkthrough

### Investigating Pwnbox interfaces

```bash
ifconfig -a
```
Shows all network interfaces (`-a` includes inactive ones). Three appear: `ens3`, `lo`, `tun0`.

| Interface | Meaning |
|---|---|
| `lo` | Loopback (`127.0.0.1`) — host talks to itself; used for testing & hiding internal services (e.g. a DB only the server can reach) |
| `ens3` | Public IP — how the Pwnbox is reached over the internet |
| `tun0` | VPN tunnel — connects Pwnbox to the private lab network |

```bash
netstat -tulnp4
```
Lists listening TCP/UDP IPv4 ports as `IP:PORT` (`-t` TCP, `-u` UDP, `-l` listening, `-n` numeric, `-p` program, `4` IPv4). Drop `-n` to resolve names — loopback shows as `localhost`; `0.0.0.0` = listening on all interfaces. (VNC service `Xtigervnc` listens on `localhost:5901`; port forwarding bridges browser HTTP → VNC.)

### Confirming the route to target

```bash
ip route get <target-ip>
```
Shows the path taken — confirms target traffic routes via `tun0` (the VPN).

```bash
ping -c 4 <target-ip>
```
Tests reachability (`-c 4` = 4 pings). `ping` is Layer 3 (no ports). Watch **ttl** (hop allowance) and **time** (latency).

### Port enumeration

```bash
nmap <target-ip>
```
Lists open TCP ports. On Windows hosts, expect `135/139/445` (SMB/RPC), `3389` (RDP), `5357` (WSDAPI).

```bash
nmap -p21,80 -sC -sV <target-ip>
```
Scans only ports 21 & 80. `-sV` = service version detection, `-sC` = default scripts. Revealed **anonymous FTP allowed** on 21; port 80 returned little → hints at **request filtering**.

> ⚠️ Only scan hosts you own or have explicit permission to test.

### FTP enumeration via raw netcat

```bash
nc <target-ip> 21
```
Raw TCP connect to FTP control channel. Then authenticate (FTP needs `\r\n`, so `Ctrl+V` then `Enter` `Enter`):

```
USER anonymous
PASS anything
PASV
```

**FTP uses two channels:** Control (port 21 — commands like `USER`, `PASS`, `LIST`, `RETR`) and Data (dynamic port — file transfers).

**Passive-mode port math:** `PASV` returns 6 numbers; last two are `p1,p2`. Real data port = `p1 * 256 + p2`.
Example: `194 * 256 + 40 = 49704`.

```bash
nc -v <target-ip> <calculated-port>   # data channel (second terminal)
LIST                                   # in control channel → lists files
```
Found `Note-From-IT.txt`. Re-run `PASV` (new port), then:

```bash
RETR Note-From-IT.txt
```
Note reveals: IIS only serves requests carrying `User-Agent: Server Administrator`.

### Bypassing the HTTP request filter

```bash
nc -v <target-ip> 80
```
Then send the request manually with the magic header:

```
GET / HTTP/1.1
Host: <target-ip>
User-Agent: Server Administrator

```
(Blank line ends the request.) The server responds, and the flag is hidden in an **HTML comment** — visible in raw output but *not* in a browser.

**Faster one-liner:**
```bash
curl -s http://<target-ip>/ -H "User-Agent: Server Administrator"
```
`-H` sets the custom header; `-s` keeps output clean.

**Flag:** `HTB{REDACTED}`

---

## Commands Quick Reference

| Command | Purpose |
|---|---|
| `ifconfig -a` | Show all network interfaces |
| `netstat -tulnp4` | List listening TCP/UDP IPv4 ports + programs |
| `ip route get <ip>` | Show route taken to reach a host |
| `ping -c 4 <ip>` | Test reachability (4 packets) |
| `nmap <ip>` | Enumerate open TCP ports |
| `nmap -p21,80 -sC -sV <ip>` | Version + default-script scan on specific ports |
| `nc <ip> 21` | Raw connect to FTP control channel |
| `nc -v <ip> <port>` | Raw connect to data channel |
| `curl -s <url> -H "Header: value"` | Send HTTP request with custom header |

## Key Concepts

- **Loopback `127.0.0.1`** — host-to-self traffic; testing and hiding internal services.
- **Passive-mode FTP port** = `p1*256 + p2`.
- **User-Agent filtering** is a weak access control — trivially bypassed by spoofing the header.
- **Flags in HTML comments** only show in raw responses (netcat/curl), never in a rendered browser view.
