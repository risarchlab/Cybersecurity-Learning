# 🔍 Nmap Scripting Engine (NSE)

**Module:** Network Enumeration with Nmap  

**Date:** 2026-02-25  

**Status:** ✅ Completed

---

## 🎯 Objective

Use Nmap's NSE scripts to enumerate a target machine and find a hidden flag embedded in one of its services.

---

## 🖥️ Target

| Field | Value |
|---|---|
| IP Address | target_ip |
| OS | Linux (Ubuntu) |

---

## 🔎 Enumeration

### Step 1 — Initial Service Scan

```bash
sudo nmap target_ip -sV -sC
```

**Flags used:**
- `-sV` — Detect service versions (like asking "what version are you?")
- `-sC` — Run default NSE scripts (automated recon checks)

**Open Ports Discovered:**

| Port | Service | Version |
|---|---|---|
| 22/tcp | SSH | OpenSSH 7.6p1 Ubuntu |
| 80/tcp | HTTP | Apache httpd 2.4.29 |
| 110/tcp | POP3 | Dovecot pop3d |
| 139/tcp | NetBIOS | Samba smbd 3.X - 4.X |
| 143/tcp | IMAP | Dovecot imapd |
| 445/tcp | SMB | Samba smbd 4.7.6 |
| 31337/tcp | Unknown (Elite?) | — |

**Notable finding from port 31337 banner:**
```
220 HTB{pr0F7pDv3r510nb4nn3r}
```

---

### Step 2 — Vulnerability Scan on Port 80

```bash
sudo nmap -sV --script vuln -vv target_ip -p 80
```

**Flags used:**
- `--script vuln` — Run all vulnerability-related NSE scripts against the target port
- `-vv` — Verbose output (more detail)
- `-p 80` — Target port 80 (HTTP/web server) only

**Key findings:**
- Web server: Apache 2.4.29 (Ubuntu)
- `robots.txt` file discovered and flagged by the enumeration scripts

---

### Step 3 — Manual Web Investigation

After the vuln scan flagged `robots.txt`, I navigated directly to the following on FireFox:

```
http://target_ip/robots.txt
```

**Result:** The flag was embedded in the contents of `robots.txt`.

---

## 📚 Key Lessons Learned

### 1. NSE Script Categories
Nmap scripts are organized into categories. The most useful ones for recon and pentesting:

| Category | What it does |
|---|---|
| `default` (-sC) | Safe, commonly useful scripts run automatically |
| `vuln` | Checks for known vulnerabilities |
| `banner` | Grabs service banners (great for finding exposed info) |
| `brute` | Attempts credential brute-forcing |
| `discovery` | Enumerates accessible services |

### 2. Useful Nmap Command Reference

```bash
# Default scripts + service version detection
sudo nmap <target> -sV -sC

# Run a specific script category
sudo nmap <target> --script <category>

# Run specific named scripts
sudo nmap <target> --script banner,smtp-commands

# Aggressive scan (sV + OS detect + traceroute + default scripts)
sudo nmap <target> -p <port> -A

# Vulnerability scan on a specific port
sudo nmap <target> -p 80 -sV --script vuln
```

### 3. Always Check These Web Files Manually
When a web server (port 80/443) is open, immediately check:

- `/robots.txt` — often reveals hidden directories; website owners accidentally expose sensitive paths here
- `/sitemap.xml` — lists all pages on the site
- `/readme.html` — especially common on WordPress sites
- `/.htaccess` — Apache config that may expose directory rules

> **Analogy:** `robots.txt` is like a sign that says "staff only — don't go in the basement." As a pentester, that sign tells you exactly where to look.

### 4. Service Banners
Some services automatically send information the moment you connect, before you even say anything. This is called a **banner**. Banners can reveal:
- Software name and version
- Operating system
- In CTFs — flags!

The `-sC` flag and `--script banner` both capture banners automatically.

### 5. Recon Methodology (Core Workflow)

Enumerate broadly first → Identify interesting services → Dig deep on each one


1. Run a wide scan (`-sV -sC`) to map the attack surface
2. Focus on high-value targets (web servers, SMB, email services)
3. Run targeted scripts (`--script vuln`, `--script banner`) on interesting ports
4. Manually follow up on anything the scripts flag

---

## 🛠️ Tools Used

- nmap 7.94SVN

---

## 🔗 References

- [Nmap NSE Script Reference](https://nmap.org/nsedoc/index.html)
- [HackTheBox](https://www.hackthebox.com)
