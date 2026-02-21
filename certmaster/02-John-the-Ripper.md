# Lab: Password Cracking with John the Ripper

**Date**: February 21, 2026  
**Course**: CompTIA Pentest+ Prep (CertMaster)  
**Topic**: Password Cracking, Hash Extraction, John the Ripper  
**Difficulty**: Beginner  

---

## Objective
Use John the Ripper to:
1. Crack the root password on the Support machine
2. Crack the password for a protected ZIP file on IT-Laptop

---

## Lab Environment
- **Machine 1**: Support (target for system password cracking)
- **Machine 2**: IT-Laptop (contains protected.zip file)
- **Tools**: John the Ripper, zip2john

---

## Key Concepts

### What is John the Ripper?
A password cracking tool that uses dictionary attacks and brute force to crack password hashes.

**Analogy**: Think of John as a master locksmith who tries thousands of keys per second until finding the one that opens the lock.

### What is a Hash?
A one-way mathematical function that converts a password into a fixed-length string. You can't reverse it, but you can try millions of passwords until you find one that produces the same hash.

### The john.pot File
- Location: `~/.john/john.pot`
- Stores all successfully cracked passwords
- Prevents re-cracking the same hash (saves time)
- Persists across sessions

---

## Commands Used

### Task 1: Crack System Password (Linux)

```bash
# Navigate to the Support machine
# (Already logged in as root in this lab)

# Crack passwords directly from /etc/shadow
john /etc/shadow

# View cracked passwords
john --show /etc/shadow

# Check the john.pot file for all cracked passwords
cat ~/.john/john.pot
```

**Result**: `1worm4b8..sparky`

---

### Task 2: Crack ZIP File Password

```bash
# Navigate to IT-Laptop machine
# Verify the file exists
ls -la protected.zip

# Extract the hash from the ZIP file
zip2john protected.zip > ziphash.txt

# View the extracted hash (optional)
cat ziphash.txt

# Crack the ZIP password
john --format=pkzip ziphash.txt

# Display the cracked password
john --show ziphash.txt
```

**Result**: `p@ssw0rd`

---

## Complete Command Reference

### Basic John the Ripper Syntax

```bash
# Crack a hash file
john [hash_file]

# Crack with specific format
john --format=[format_type] [hash_file]

# Show cracked passwords
john --show [hash_file]

# Use a custom wordlist
john --wordlist=[path_to_wordlist] [hash_file]

# Use wordlist with rules (variations)
john --wordlist=[wordlist] --rules [hash_file]

# List all supported formats
john --list=formats

# View session status (while running)
# Press any key while John is running
```

### Hash Extraction Tools

```bash
# For ZIP files
zip2john [file.zip] > [output_hash.txt]

# For RAR files
rar2john [file.rar] > [output_hash.txt]

# For PDF files
pdf2john [file.pdf] > [output_hash.txt]

# For Linux system passwords (combines /etc/passwd and /etc/shadow)
unshadow /etc/passwd /etc/shadow > [output_hash.txt]
```

---

## Workflow Summary

### System Password Cracking
```
1. Access target machine
2. Extract hashes → john /etc/shadow
3. View results → john --show /etc/shadow
```

### File Password Cracking
```
1. Extract hash → zip2john file.zip > hash.txt
2. Crack hash → john hash.txt
3. View results → john --show hash.txt
```

---

## Common Wordlists

| Wordlist | Location | Size | Use Case |
|----------|----------|------|----------|
| Default | `/usr/share/john/password.lst` | Small | Quick tests, common passwords |
| RockYou | `/usr/share/wordlists/rockyou.txt` | 14M+ passwords | Real-world assessments |
| SecLists | `/usr/share/seclists/Passwords/` | Various | Targeted attacks |

---

## Troubleshooting

### "No password hashes left to crack"
- Hash already cracked
- Check `~/.john/john.pot` for previous results

### Command hangs or takes too long
- Use a wordlist: `john --wordlist=/path/to/wordlist hash.txt`
- Use rules for variations: `john --wordlist=password.lst --rules hash.txt`

### Permission denied
- Use `sudo` for system files
- Copy files to home directory first: `sudo cp /etc/shadow ~/shadow.txt`

### Wrong hash format
- Specify format: `john --format=pkzip hash.txt`
- List formats: `john --list=formats`

---

## Real-World Application

### Penetration Testing
- **Password audits**: Test organizational password policies
- **Post-exploitation**: Crack local passwords after system compromise
- **Lateral movement**: Use cracked credentials to access other systems
- **WiFi testing**: Crack WPA2 handshakes from captured packets

### SOC/Defense Perspective
- **Understand attacker tools**: Know what attackers use to defend better
- **Detect brute-force attacks**: Recognize patterns in logs
- **Policy recommendations**: Demonstrate why strong passwords matter
- **Incident response**: Understand hash cracking in breach investigations

---

## Key Takeaways

1. **Different targets need different tools**: System passwords use `/etc/shadow`, ZIP files need `zip2john`
2. **john.pot is your friend**: Always check it before re-running long cracks
3. **Wordlists matter**: Default wordlist is fast but limited; rockyou.txt is comprehensive
4. **Format specification**: Some hash types require `--format=` flag
5. **Patience pays off**: Complex passwords take time; simple ones crack instantly

---

## Next Steps

- [ ] Practice with different hash types (MD5, SHA-256, NTLM)
- [ ] Experiment with custom wordlists
- [ ] Learn Hashcat for GPU-accelerated cracking
- [ ] Study rainbow tables and how they work
- [ ] Practice on HTB/TryHackMe password cracking rooms

---

## Related Topics to Study
- Hash functions (MD5, SHA-1, SHA-256, bcrypt, scrypt)
- Rainbow tables vs. dictionary attacks
- Password policy best practices
- Multi-factor authentication (MFA)
- Hashcat (GPU-based cracking)
- Hydra (online password cracking)
- Credential stuffing attacks

---

## References
- John the Ripper Documentation: https://www.openwall.com/john/doc/
- CompTIA Pentest+ Objectives (PT0-002/003)
- NIST Password Guidelines: https://pages.nist.gov/800-63-3/
