# Network Pivoting with sshuttle

## Overview
This report covers network pivoting techniques using sshuttle to access internal networks through an intermediary system. These are essential skills for penetration testing and understanding network segmentation.

---

## Key Concepts Learned

### 1. Network Segmentation
**What it is:** Internal networks that aren't directly accessible from external networks.

**Why it matters:**
- Good security practice - internal servers shouldn't be directly reachable from the internet
- Common in enterprise environments
- Requires pivoting techniques to assess during penetration tests

**Real-world analogy:** 🏢 An apartment building where you can reach the main entrance (edge router) but need special access to reach individual apartments (internal servers).

---

## Tools & Techniques

### Nmap - Network Scanning

#### Basic Scan Syntax
```bash
nmap [options] [target_ip] -oN [output_file]
```

#### Important Nmap Flags

| Flag | Purpose | When to Use |
|------|---------|-------------|
| `-p [port]` | Scan specific port(s) | When you know which service to check |
| `-F` | Fast scan (top 100 ports) | Quick reconnaissance |
| `--script [name]` | Run NSE script | Detailed service enumeration |
| `-Pn` | Skip ping, assume host is up | Target blocks ICMP or scanning through tunnels |
| `-sT` | TCP connect scan | More reliable through proxies/tunnels |
| `-oN [file]` | Output to normal format | Save results for documentation |

#### Example Commands

**Fast scan to find services:**
```bash
nmap -F 203.0.113.1 -oN edge_scan.nmap
```

**Targeted SMB enumeration:**
```bash
nmap -p 445 --script smb-os-discovery 10.1.16.2 -oN smb_scan.nmap
```
- port 445 (Server Message Block) = Windows file sharing protocol
  
**Scan through tunnel (skip ping):**
```bash
nmap -Pn -sT -p 445 --script smb-os-discovery 10.1.16.2 -oN tunnel_scan.nmap
```

---

### sshuttle - Transparent Proxy via SSH

#### What is sshuttle?
A tool that creates a VPN-like tunnel using SSH, routing network traffic through an intermediary system to reach otherwise inaccessible networks.

**Analogy:** 🚇 Building a subway tunnel through a building's main entrance to reach internal rooms.

#### Basic Syntax
```bash
sshuttle [options] -r [user]@[jump_host] [target_subnet]
```

#### Important sshuttle Flags

| Flag | Purpose | Example |
|------|---------|---------|
| `-r [user@host]` | Remote SSH server to tunnel through | `-r admin@192.168.1.1` |
| `--dns` | Tunnel DNS requests too | Always recommended |
| `--syslog` | Log to system logger | Required for documentation |
| `-v` or `-vv` | Verbose output | Troubleshooting |
| `[subnet]` | Network to route through tunnel | `10.1.16.0/24` |

#### Example Commands

**Basic tunnel setup:**
```bash
sshuttle -r user@jump_host 10.0.0.0/24
```

**With logging and DNS:**
```bash
sshuttle --dns -r user@jump_host 10.0.0.0/24 --syslog
```

**Verbose mode for troubleshooting:**
```bash
sshuttle --dns -r user@jump_host 10.0.0.0/24 --syslog -vv
```

#### How sshuttle Works
1. Connects to remote SSH server (requires SSH credentials)
2. Creates iptables rules to route specified subnets through the SSH connection
3. All traffic to target subnet automatically goes through the tunnel
4. Transparent to applications - no need to configure proxy settings

#### Requirements
- **On your machine:** sshuttle installed
- **On jump host:** SSH server running (typically port 22)
- **Credentials:** Valid SSH username and password/key

---

## Attack Chain Workflow

### Phase 1: Reconnaissance
**Goal:** Identify accessible systems and services

```bash
# Scan edge/perimeter systems
nmap -F [edge_router_ip] -oN edge_scan.nmap

# Look for:
# - SSH (port 22) - potential pivot point
# - Open services that might have vulnerabilities
```

**Key findings to note:**
- IP addresses of accessible systems
- Open ports and services
- Potential pivot points (systems with SSH, RDP, etc.)

---

### Phase 2: Initial Access Assessment
**Goal:** Determine what you can reach directly

```bash
# Try scanning internal targets directly
nmap -p [port] [internal_ip] -oN direct_scan.nmap
```

**Expected results:**
- ✅ **Success:** Target is directly reachable (less common for internal systems)
- ❌ **"Host seems down":** Target is on internal network, need to pivot

---

### Phase 3: Pivoting
**Goal:** Establish tunnel to reach internal networks

```bash
# Set up sshuttle tunnel
sshuttle --dns -r [user]@[jump_host] [internal_subnet] --syslog
```

**What happens:**
- System prompts for SSH password
- Connection establishes (may show "client: Connected")
- Terminal appears "stuck" - this is normal! Tunnel is active
- Leave this terminal running

**Troubleshooting tips:**
- Verify SSH credentials are correct
- Check that SSH server is running on jump host
- Use `-vv` flag to see detailed connection info
- Make sure you're using correct subnet notation (e.g., 10.1.16.0/24)

---

### Phase 4: Post-Pivot Enumeration
**Goal:** Scan and enumerate internal systems through the tunnel

**Important:** Open a NEW terminal window (keep sshuttle running in the first one!)

```bash
# Scan internal targets through tunnel
# Use -Pn flag to skip ping detection
nmap -Pn -sT -p [port] --script [script] [internal_ip] -oN tunnel_scan.nmap
```

**Why different flags?**
- `-Pn`: Tunnels often block ping, assume host is up
- `-sT`: TCP connect scan works better through proxies/tunnels
- Regular scans may fail through tunnels even if target is reachable

---

## Common Scenarios & Solutions

### Scenario 1: "Host seems down" error
**Problem:** Nmap says target is down but you know it exists

**Solutions:**
1. Add `-Pn` flag to skip ping detection
2. If scanning through tunnel, also add `-sT` for TCP connect scan
3. Verify tunnel is actually active (sshuttle still running?)

**Example:**
```bash
# Instead of this:
nmap -p 445 --script smb-os-discovery 10.1.16.2

# Try this:
nmap -Pn -sT -p 445 --script smb-os-discovery 10.1.16.2
```

---

### Scenario 2: sshuttle won't connect
**Problem:** Connection fails or times out

**Troubleshooting checklist:**
- [ ] Is SSH running on jump host? (Port 22 open?)
- [ ] Are credentials correct?
- [ ] Did you type the password correctly? (It won't show on screen!)
- [ ] Are you using correct user@host format?
- [ ] Try with `-vv` flag to see detailed errors

---

### Scenario 3: Can't reach internal targets after establishing tunnel
**Problem:** Tunnel is running but scans still fail

**Solutions:**
1. Verify correct subnet is being tunneled
2. Test basic connectivity: `ping [internal_ip]`
3. Use `-Pn -sT` flags in nmap
4. Check that sshuttle is still running (not crashed)

---

## Important Security Concepts

### Network Segmentation
**Definition:** Separating networks into isolated segments for security

**Why it matters:**
- Limits blast radius of compromises
- Internal servers protected from direct internet access
- Forces attackers to pivot (making detection more likely)

**How to identify:**
- Systems unreachable from external position
- Different IP subnets (e.g., external: 203.x.x.x, internal: 10.x.x.x)
- Services available on jump hosts but not directly accessible

---

### Pivoting (Lateral Movement)
**Definition:** Using a compromised/accessible system to reach other systems

**Common techniques:**
- SSH tunneling (sshuttle, SSH port forwarding)
- Proxy chains
- Metasploit routing
- VPN establishment

**Legal/ethical considerations:**
- Only perform during authorized penetration tests
- Stay within scope of engagement
- Document all pivot points used

---

## Subnet Notation Refresher

### CIDR Notation
Format: `IP_ADDRESS/PREFIX_LENGTH`

**Examples:**

| Notation | Meaning | IP Range | # of Hosts |
|----------|---------|----------|------------|
| 10.1.16.0/24 | Class C network | 10.1.16.0 - 10.1.16.255 | 256 |
| 192.168.1.0/24 | Class C network | 192.168.1.0 - 192.168.1.255 | 256 |
| 10.0.0.0/8 | Class A network | 10.0.0.0 - 10.255.255.255 | 16,777,216 |
| 172.16.0.0/16 | Class B network | 172.16.0.0 - 172.16.255.255 | 65,536 |

**Quick reference:**
- `/24` = 256 addresses (common for small networks)
- `/16` = 65,536 addresses (medium networks)
- `/8` = 16.7 million addresses (large networks)

---

## Port Reference

### Common Ports to Know

| Port | Service | Purpose | Pentest Relevance |
|------|---------|---------|-------------------|
| 22 | SSH | Secure Shell | Pivot point, remote access |
| 25 | SMTP | Email | Email server enumeration |
| 80 | HTTP | Web server | Web application testing |
| 443 | HTTPS | Secure web | Web application testing |
| 445 | SMB | File sharing (Windows) | Windows enumeration, exploits |
| 3389 | RDP | Remote Desktop (Windows) | Windows pivot point |

---

## Best Practices

### Documentation
✅ **Always save scan output to files**
```bash
# Use -oN flag for normal output
nmap [options] [target] -oN descriptive_filename.nmap
```

**Benefits:**
- Creates audit trail
- Required for professional reports
- Easy to review later
- Shows progression of testing

---

### File Naming Conventions
Use descriptive names that include:
- Target identifier
- Test type
- Iteration/version

**Examples:**
- `edge_router_fast_scan.nmap`
- `ms_server_smb_test1.nmap`
- `ms_server_smb_test2_via_tunnel.nmap`

---

### Terminal Management
**When using sshuttle:**
- **Terminal 1:** Run sshuttle (leave it open/running)
- **Terminal 2+:** Run scans and commands through the tunnel

**Why:** sshuttle needs to stay active to maintain the tunnel

---

## Troubleshooting Guide

### Issue: Command not found
**Problem:** `nmap: command not found` or `sshuttle: command not found`

**Solution:**
```bash
# Check if installed
which nmap
which sshuttle

# Install if missing (Debian/Ubuntu/Kali)
sudo apt-get update
sudo apt-get install nmap
sudo apt-get install sshuttle
```

---

### Issue: Permission denied
**Problem:** Can't run certain commands

**Solution:**
```bash
# Run as root (in Kali, often default)
sudo [command]

# Or switch to root
sudo su -
```

---

### Issue: Scan is very slow through tunnel
**This is normal!** Tunneling adds latency.

**Tips:**
- Be patient
- Use `-F` for faster scans when doing reconnaissance
- Scan specific ports instead of full range
- Expect 2-3x slower than direct scans

---

## Key Takeaways

### What You Learned
1. ✅ How to identify network segmentation
2. ✅ How to establish SSH tunnels with sshuttle
3. ✅ How to adapt nmap scans for tunneled environments
4. ✅ Real-world pivoting workflow
5. ✅ Importance of documentation

### Skills Gained
- Network reconnaissance with nmap
- Identifying pivot points (SSH services)
- Establishing transparent proxies
- Scanning through tunnels
- Troubleshooting connection issues

### Commands to Remember
```bash
# Fast edge scan
nmap -F [edge_ip] -oN edge_scan.nmap

# Establish tunnel
sshuttle --dns -r user@jumphost subnet --syslog

# Scan through tunnel (skip ping)
nmap -Pn -sT -p [port] --script [script] [target] -oN scan.nmap
```

---

## Additional Resources

### Official Documentation
- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [sshuttle GitHub](https://github.com/sshuttle/sshuttle)
- [Nmap NSE Scripts](https://nmap.org/nsedoc/)

### Related Topics to Study
- SSH port forwarding (manual tunneling)
- Metasploit routing and pivoting
- ProxyChains
- Network segmentation best practices
- Lateral movement techniques

### CompTIA PenTest+ Domains
This exercise covers:
- **Domain 1:** Planning and Scoping
- **Domain 2:** Information Gathering and Vulnerability Scanning
- **Domain 3:** Attacks and Exploits (pivoting/lateral movement)

---

## Practice Scenarios

To reinforce these skills, practice:

1. **Scenario 1:** Set up two VMs - one as jump host, one as internal target
2. **Scenario 2:** Practice identifying which services can be pivot points
3. **Scenario 3:** Compare direct scans vs tunneled scans
4. **Scenario 4:** Document a full penetration test workflow

---

## Notes & Reminders

### Legal/Ethical Considerations
- ⚠️ Only perform these techniques on systems you own or have written authorization to test
- ⚠️ Unauthorized access is illegal (Computer Fraud and Abuse Act, CFAA)
- ⚠️ Always stay within scope of penetration test agreements

### Exam Tips (PenTest+)
- Know when to use which nmap flags
- Understand network segmentation concepts
- Be able to identify pivot opportunities
- Know common ports and services
- Understand difference between various pivoting techniques

---

## Revision History
- Created: 02-20-2026
- Based on: CertMaster PenTest+ Lab Exercise
- Topics: Network Pivoting, sshuttle, nmap scanning techniques

---

*This study guide focuses on concepts and techniques learned. Always refer to official documentation and course materials for exam preparation.*
