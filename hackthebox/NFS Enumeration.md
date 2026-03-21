# HTB Module: NFS Enumeration

## Overview
Network File System (NFS) is a protocol that allows file systems to be shared over a network, primarily used between Linux/Unix systems. Misconfigured NFS shares can expose sensitive files to anyone on the network — no credentials required in older versions (NFSv2/3).

**Key ports:**
| Port | Protocol | Service |
|------|----------|---------|
| 111 | TCP/UDP | RPC (portmapper) |
| 2049 | TCP/UDP | NFS |

---

## Commands Quick Reference

| Command | Description |
|---------|-------------|
| `sudo nmap -p111,2049 -sV -sC <TARGET_IP>` | Scan NFS ports and run default scripts |
| `sudo nmap --script nfs* -sV -p111,2049 <TARGET_IP>` | Run all NFS-specific NSE scripts |
| `showmount -e <TARGET_IP>` | List exported NFS shares on the target |
| `mkdir target-NFS` | Create local mount point directory |
| `sudo mount -t nfs <TARGET_IP>:/ ./target-NFS/ -o nolock` | Mount the NFS root to local folder |
| `ls target-NFS/` | See top-level directories on mounted share |
| `find target-NFS/ -name "flag.txt" 2>/dev/null` | Recursively search for flag file |
| `cat target-NFS/var/nfs/flag.txt` | Read the flag |
| `ls -l target-NFS/var/nfs/` | List files with usernames/group names |
| `ls -n target-NFS/var/nfs/` | List files with UIDs/GIDs |
| `cd .. && sudo umount ./target-NFS` | Unmount the share (must cd out first) |

---

## Step-by-Step Solution

### 1. Scan for NFS Services
```bash
sudo nmap -p111,2049 -sV -sC <TARGET_IP>
```
Confirms NFS is running on ports 111 (RPC/portmapper) and 2049 (NFS). The RPC portmapper tells clients which port each service is on — think of it as a directory for network services.

Optionally, run NFS NSE scripts for more detail (lists share contents, stats, and mount info):
```bash
sudo nmap --script nfs* -sV -p111,2049 <TARGET_IP>
```

---

### 2. Discover Available Shares
```bash
showmount -e <TARGET_IP>
```
Asks the NFS server "what folders are you sharing?" The `-e` flag means "show exports." Output will list the share path and which subnet is allowed to mount it, e.g.:
```
Export list for <TARGET_IP>:
/mnt/nfs  10.x.x.0/24
```

---

### 3. Mount the NFS Share
```bash
mkdir target-NFS
sudo mount -t nfs <TARGET_IP>:/ ./target-NFS/ -o nolock
```
- `mkdir target-NFS` — creates a local folder to receive the remote share (like a USB mount point)
- `mount -t nfs` — tells the system to use the NFS protocol
- `<TARGET_IP>:/` — mounts from the root of the remote server
- `-o nolock` — disables file locking to avoid errors if the server has no lock manager

---

### 4. Navigate the Share
```bash
ls target-NFS/
```
The directory structure won't always match the share path from `showmount`. Use `ls` to explore, or use `find` to locate the flag directly:
```bash
find target-NFS/ -name "flag.txt" 2>/dev/null
```
> In this lab, the share was located at `target-NFS/var/nfs/` — not the expected `mnt/nfs/` path. Always verify with `ls` rather than assuming the path.

---

### 5. Read the Flag
```bash
cat target-NFS/var/nfs/flag.txt
```
Flag redacted.

---

### 6. Inspect File Ownership (Bonus Enumeration)
```bash
ls -l target-NFS/var/nfs/    # shows usernames and group names
ls -n target-NFS/var/nfs/    # shows raw UIDs and GIDs
```
This is useful for privilege escalation — if `no_root_squash` is set, files owned by root (UID 0) on the server are accessible as root on the client too.

---

### 7. Unmount the Share
```bash
cd ..
sudo umount ./target-NFS
```
Must `cd` out of the directory first, otherwise the system returns a "device is busy" error.

---

## Key Concepts

**Why NFS can be dangerous:**
- NFSv2/3 authenticate the *machine*, not the user — anyone on the allowed subnet can mount the share
- `no_root_squash` — if set, root on the client = root on the share (dangerous)
- `insecure` — allows connections from ports above 1024, bypassing a basic security assumption
- `nohide` — exposes nested mount points that wouldn't normally be visible

**NFS vs SMB:**
- NFS = Linux/Unix to Linux/Unix
- SMB = Windows (also cross-platform via Samba)
- NFSv4 added Kerberos authentication and only requires port 2049, making it firewall-friendly

**UID/GID spoofing:**
Since NFS maps permissions by UID/GID numbers, if you create a local user with the same UID as a file owner on the share, you can read/write that file — even without knowing the original username.

---

## Tools Used
- `nmap` — port scanning and NSE script enumeration
- `showmount` — NFS share discovery
- `mount` / `umount` — mounting and unmounting NFS shares
- `find` — recursive file search on mounted share
