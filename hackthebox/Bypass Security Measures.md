# Network Enumeration with Nmap — Firewall and IDS/IPS Evasion

**Platform:** Hack The Box Academy  
**Module:** Network Enumeration with Nmap  
**Topic:** Firewall and IDS/IPS Evasion  
**Date:** February 28, 2026  
**Difficulty:** Medium  

---

## 🎯 Objective

The lab simulates a real-world client engagement where:
1. We first needed to find the **DNS server version** of the target.
2. After the client's admin underwent IDS/IPS training and hardened the system, we had to **bypass the new defenses** and find the version of a hidden service (FTP) that was moved to a non-standard port.

---

## 🧠 Key Concepts Covered

- DNS version enumeration
- UDP vs TCP scanning
- Service version detection (`-sV`)
- IDS/IPS evasion using trusted source ports (`--source-port 53`)
- Scanning non-standard ports
- Banner grabbing with `ncat`
- Killing processes occupying a port

---

## 📝 Part 1: DNS Server Version Enumeration

### Goal
Find the DNS server version of the target.

### What I Tried

**Command:**
```bash
dig @<target_ip> version.bind CHAOS TXT
```
**What it does:** Sends a special DNS query asking the server to reveal its own version. DNS servers have a hidden "chaos class" that can expose this info if not configured to hide it.  
**Result:** Returned `version.bind` but no actual version string — the server had it hidden.

---

**Command:**
```bash
nmap -sV -sU -p 53 <target_ip>
```
**What it does:**
- `-sV` → Service version detection (probes the service and asks "what are you and what version?")
- `-sU -p 53` → Only scan UDP port 53 (DNS default port)

**Result:** Returned the DNS version string → submitted as the flag. ✅

---

## 📝 Part 2: Bypassing Hardened IDS/IPS to Find Hidden FTP Service

### Scenario
After the client's admin completed IDS/IPS training, they:
- Hardened the firewall rules
- Added an FTP service (for large data transfers) on a **non-standard port**
- Configured IDS to block suspicious scanning patterns

### Goal
Find the hidden FTP service and retrieve its version/banner.

---

### Step 1: Initial Scan (Blocked)

**Command:**
```bash
nmap -sV -p 21 <target_ip>
```
**What it does:** Scans the default FTP port (21) with version detection.  
**Result:** Port 21 was **closed** — FTP was moved to a non-standard port.

---

### Step 2: Full Port Scan with Source Port Evasion

**Command:**
```bash
nmap -sV -p 1-10000 -Pn --source-port 53 <target_ip>
```
**What it does:**
- `-p 1-10000` → Scan ports 1 through 10000
- `-Pn` → Skip host discovery ping (treat host as online regardless — useful when ICMP is blocked)
- `--source-port 53` → Disguise scan traffic as if it's coming from port 53 (DNS). Firewalls often **whitelist/trust** DNS traffic, so this helps sneak past IDS/IPS rules.

> 💡 **Analogy:** Like wearing a mail carrier uniform to get past a security checkpoint — you look "official" so you're let through.

**Result:** Found open ports 22 (SSH) and 80 (HTTP) but no FTP. Service versions:
- `22/tcp` → OpenSSH 7.6p1 Ubuntu 4ubuntu0.7
- `80/tcp` → Apache httpd 2.4.29 (Ubuntu)

FTP must be hiding above port 10000.

---

### Step 3: Scan Higher Port Range

**Command:**
```bash
nmap -sV -p 10000-50000 -Pn --source-port 53 <target_ip>
```
**What it does:** Same as above but scanning the upper range of ports (10,000–50,000).

**Result:** Found **port 50000/tcp open — tcpwrapped**

> 💡 **What is tcpwrapped?** It means the port is open and something is listening, but it's refusing to identify itself. The IDS is still blocking full version detection. We know something's there, but we can't see what it is yet through nmap alone.

---

### Step 4: Banner Grabbing with ncat

Since nmap couldn't identify the service, we connected directly using `ncat` — a tool for making raw network connections — while still disguising our traffic as DNS (source port 53).

**Command (failed — no root privileges):**
```bash
ncat --source-port 53 <target_ip> 50000
```
**Error:** `Bind to 0.0.0.0:53 failed: Permission denied (13)`  
**Why it failed:** Ports below 1024 are reserved and require **root/sudo** privileges to use.

---

**Command (failed — port in use):**
```bash
sudo ncat --source-port 53 <target_ip> 50000
```
**Error:** `Bind to 0.0.0.0:53 failed: Address already in use (98)`  
**Why it failed:** Something on the local machine was already using port 53.

---

### Step 5: Free Up Port 53

First, identified what was using port 53:

**Command:**
```bash
sudo lsof -i :53
```
**What it does:** Lists all processes currently using port 53.  
**Result:** `dnsmasq` (PID 1246) was occupying port 53.

> Note: Tried `sudo systemctl stop systemd-resolved` and `sudo systemctl stop dnsmasq` — both failed because dnsmasq wasn't loaded as a systemd service in this environment. Had to kill it directly by PID instead.

**Command:**
```bash
sudo kill 1246
```
**What it does:** Forcefully terminates the process with PID 1246 (dnsmasq), freeing up port 53.

---

### Step 6: Successful Banner Grab

**Command:**
```bash
sudo ncat --source-port 53 <target_ip> 50000
```
**What it does:** Opens a raw TCP connection to port 50000 on the target, using port 53 as the source so the firewall/IDS treats it as trusted DNS traffic.

**Result:**
```
220 HTB{flag redacted}
```

> 💡 **What is `220`?** This is the standard FTP server welcome/banner response code. When you connect to an FTP server, it greets you with `220` followed by its banner — which in this case contained the flag. ✅

---

## 🔑 Key Takeaways

| Concept | What I Learned |
|---|---|
| UDP vs TCP | DNS runs on UDP 53 by default. Always use `-sU` for DNS enumeration |
| `-sV` flag | Doesn't just check if a port is open — it actively probes for version info |
| `-Pn` flag | Skips ping/host discovery. Useful when ICMP is blocked by firewall |
| `--source-port 53` | Disguises scan as DNS traffic. Firewalls often whitelist port 53 |
| `tcpwrapped` | Port is open but service is refusing to identify itself to nmap |
| `ncat` for banner grabbing | When nmap can't identify a service, connect directly and let the service announce itself |
| Ports below 1024 | Require root/sudo to bind to them |
| `lsof -i :port` | Find what process is occupying a specific port |
| `sudo kill <PID>` | Kill a process by its PID when systemctl doesn't work |

---

## 🛠️ Commands Reference (Quick Cheat Sheet)

```bash
# DNS version enumeration
nmap -sV -sU -p 53 <target_ip>

# Scan with IDS evasion (source port spoofing)
nmap -sV -p 1-10000 -Pn --source-port 53 <target_ip>
nmap -sV -p 10000-65535 -Pn --source-port 53 <target_ip>

# Find what's using a port
sudo lsof -i :53

# Kill a process by PID
sudo kill <PID>

# Banner grab with trusted source port
sudo ncat --source-port 53 <target_ip> <port>
```

---
*Lab completed as part of self-study towards an entry-level penetration testing role. Currently working through HTB Academy modules alongside CompTIA Pentest+ preparation.*
