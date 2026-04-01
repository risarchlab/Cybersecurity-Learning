# Footprinting Lab — Easy (DNS Server)
**Module:** Penetration Tester - Footprinting
**Difficulty:** Easy
**Date:** 2026-04-01

---

## Scenario
Commissioned by Inlanefreight Ltd to enumerate an internal DNS server.
No aggressive exploitation — passive enumeration only.
Credentials provided: `[username]:[password]`
Hint: SSH keys mentioned on a company forum.

---

## Open Ports & Services

| Port | Service | Version |
|------|---------|---------|
| 21   | FTP | ProFTPD |
| 22   | SSH | OpenSSH 8.2p1 Ubuntu |
| 53   | DNS | ISC BIND 9.16.1 |
| 2121 | FTP | ProFTPD (User's FTP) |

---

## Methodology

### 1. Full Port Scan
```bash
nmap -sV -sC -p- --min-rate 5000 <target-ip>
```
Scanning all 65535 ports revealed a second FTP server on port 2121
that a default scan would have missed.

### 2. DNS Zone Transfer
```bash
dig axfr inlanefreight.htb @<target-ip>
```
Server was misconfigured to allow zone transfers from anyone.
Dumped the full DNS zone, revealing internal infrastructure.

**Hosts discovered:**
| Hostname | IP |
|----------|----|
| app.inlanefreight.htb | 10.129.18.15 |
| internal.inlanefreight.htb | 10.129.1.6 |
| mail1.inlanefreight.htb | 10.129.18.201 |
| ns.inlanefreight.htb | 10.129.34.136 |

### 3. FTP Enumeration — Port 21
```bash
ftp <target-ip>
# [username]:[password]
```
Logged in successfully but directory was empty and chrooted.
Anonymous login was disabled.

### 4. FTP Enumeration — Port 2121
```bash
ftp <target-ip> 2121
# [username]:[password]
```
Accessed user's actual home directory. Found `.ssh` folder.
```bash
cd .ssh
get id_rsa
```
Downloaded user's private SSH key.

### 5. SSH Access
```bash
chmod 600 id_rsa
ssh -i id_rsa user@<target-ip>
```
Successfully authenticated using the retrieved private key.

### 6. Flag
```bash
find / -name "flag.txt" 2>/dev/null
cat /home/flag/flag.txt
```

---

## Key Concepts

| Concept | Explanation |
|---------|-------------|
| DNS Zone Transfer (AXFR) | Dumps entire DNS zone — maps internal hostnames and IPs. Dangerous if unrestricted. |
| FTP on non-standard port | Services hidden on uncommon ports — always scan all ports with `-p-` |
| SSH key auth | Server had password auth disabled — key was the only way in |
| FTP chroot | Port 21 locked user to an empty directory; port 2121 was her real home |

---

## Vulnerabilities Found

| Vulnerability | Risk | Recommendation |
|---------------|------|----------------|
| Unrestricted DNS zone transfer | High — exposes full internal network map | Restrict AXFR to trusted IPs only |
| SSH private key exposed via FTP | Critical — allows full server access | Never store private keys in FTP-accessible directories |
| FTP on port 2121 with no encryption | Medium — credentials sent in plaintext | Replace FTP with SFTP |

---

## Tools Used
- `nmap` — port scanning and service enumeration
- `dig` — DNS zone transfer
- `ftp` — file retrieval
- `ssh` — remote access

---

## Lessons Learned
- Always scan all ports (`-p-`) — critical services hide on non-standard ports
- DNS servers are high-value targets — a misconfigured zone transfer hands you the entire network map
- Follow the hints — "SSH keys on a forum" pointed directly to the attack path
- Check every available service with given credentials, not just the obvious ones
