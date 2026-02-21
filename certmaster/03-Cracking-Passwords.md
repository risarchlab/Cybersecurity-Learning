Perfect! I've got all 5 sections now. Let me compile this into a professional lab journal entry for your GitHub cybersecurity-learning repo.

Here's your complete lab documentation:

---

# Lab Journal: Password Attacks & Cracking Techniques

**Date:** February 21, 2026  
**Lab Focus:** Offensive Security - Password Attack Methodologies  
**Tools Used:** Kali Linux, John the Ripper, Hydra, aircrack-ng, SSH, SMB  
**Skill Level:** Intermediate (PenTest+ Preparation)

---

## 📋 Lab Overview

This comprehensive lab explored various password attack techniques used in penetration testing. I performed both **online (live)** and **offline** attacks against different authentication systems to understand their strengths, weaknesses, and real-world applications.

**Key Learning Objectives:**
- Differentiate between online vs. offline password attacks
- Understand when to use dictionary vs. brute force methods
- Execute password spraying to avoid account lockout
- Crack wireless network passwords from captured traffic
- Recognize the importance of password length and complexity

---

## 🎯 Section 1: Password Guessing (Online Attack)

### What I Did
Performed manual password guessing against an SSH server to understand live authentication attacks.

### Technical Details

**Target:** `www.structureality.com` (SSH on port 22)  
**Method:** Online/Live password guessing  
**Attack Flow:**
1. Attempted SSH login to `admin@www.structureality.com`
2. Manually tried common passwords from a predefined list
3. SSH allows 3 failed attempts before disconnecting
4. Switched target to `user@www.structureality.com` after admin failed

**Commands Used:**
```bash
ssh admin@www.structureality.com
# Tried passwords: password, 123456, qwerty, etc

ssh [user]@www.structureality.com
# Success with: *redacted*
```

**Successful Credentials:**
- Username: `[user]`
- Password: `*redacted*`
- Prompt after login: `[user]@lamp$`

### Key Takeaways

**Online Attack Characteristics:**
- ✅ Attacks the live authentication system in real-time
- ⏱️ Slow - limited by network latency and authentication processing
- 🚫 Can be stopped by **account lockout** policies
- 🔍 Highly detectable - leaves logs on the target system

**Analogy:** It's like trying to guess someone's house key combination while standing at their front door. The alarm system (account lockout) is watching every failed attempt and can lock you out after too many tries.

**Defense Mechanisms:**
- Account lockout (e.g., 3 failed attempts = locked account)
- Rate limiting
- Multi-factor authentication (MFA)
- Monitoring and alerting on failed login attempts

---

## 🎯 Section 2: Password Spraying Attack

### What I Did
Performed a password spraying attack using Hydra to find which accounts use known passwords from a discovered list.

### Technical Details

**Scenario:** Found passwords in a trash can but no associated usernames  
**Target:** SMB share `//10.6.XX.X/HR` on MS10 system  
**Known Users:** pat, logan, alex, etc (from `users.txt`)  
**Known Passwords:** (redacted, but used 3 passwords)

**Manual Attempt (Failed):**
```bash
mkdir /mnt/HR
mount //[IP address]/HR /mnt/HR -o username=[user2]
# Tried: abc123 (failed)
# Tried: 123456 (failed)
```

**Automated Attack with Hydra:**
```bash
# Created password file
echo abc123 > pass.txt
echo 123456 >> pass.txt
echo Pa55w0rd! >> pass.txt

# Ran Hydra wizard
hydra-wizard
```

**Hydra Configuration:**
| Parameter | Value |
|-----------|-------|
| Service | smb |
| Target | 10.6.XX.X |
| Username | users.txt |
| Password | pass.txt |

**Results Found:**
- ✅ `admin:*redacted*`
- ✅ `user:*redacted*`

**Verification:**
```bash
mount //10.1.16.2/HR /mnt/HR -o username=[user]
# Password: *redacted*
ls /mnt/HR  # Successfully mounted!
```

### Key Takeaways

**Password Spraying vs. Brute Force:**
- Password spraying tries **few passwords against many accounts**
- Traditional brute force tries **many passwords against one account**

**Analogy:** Instead of trying 100 keys on one house (which triggers the alarm), you try 1-2 keys on 50 different houses in the neighborhood. This avoids triggering account lockout while maximizing coverage.

**Why It Works:**
- Bypasses account lockout (only 1-2 attempts per account)
- Exploits password reuse across users
- Common in organizations where users choose similar "compliant" passwords

**Real-World Application:**
- Common in corporate environments
- Effective against cloud services (Office 365, AWS, etc.)
- Used by APT groups for initial access

---

## 🎯 Section 3: Dictionary Password Cracking (Offline Attack)

### What I Did
Used John the Ripper to perform offline dictionary attacks against stolen NTLM password hashes from a Windows system.

### Technical Details

**Source:** `ms10-hashes.txt` (extracted from MS10 Windows machine)  
**Tool:** John the Ripper (JtR)  
**Attack Type:** Offline dictionary attack  
**Dictionary Used:** `/usr/share/seclists/Passwords/xato-net-10-million-passwords.txt` (10 million passwords)

**Viewing Stolen Hashes:**
```bash
cat ms10-hashes.txt
```
*Hash format shows: username:RID:LM_hash:NTLM_hash*

**Dictionary Attack Command:**
```bash
john --format=NT --wordlist=/usr/share/seclists/Passwords/xato-net-10-million-passwords.txt ms10-hashes.txt
```

**Results:**
- ✅ **11 passwords cracked** in seconds
- ⏱️ Attack completed in < 1 minute with 10 million password list

**Exporting Results:**
```bash
john --show --format=NT ms10-hashes.txt > dict-cracked.txt
less dict-cracked.txt
```

**Interesting Finding:**
- **12 accounts shown** with cracked passwords (not 11)
- Two accounts shared the same password: *redacted* and *redacted*
- Same password hash = instant crack for both accounts

**Checking Remaining Uncracked Hashes:**
```bash
john --show=left --format=NT ms10-hashes.txt
```

### Key Takeaways

**Offline vs. Online Attacks:**

| Feature | Online Attack | Offline Attack |
|---------|--------------|----------------|
| Speed | Slow (seconds per attempt) | **Billions of attempts/second** |
| Detection | Highly detectable | Undetectable (no network traffic) |
| Account Lockout | ✅ Can stop attack | ❌ No lockout (no auth system involved) |
| Requires | Live access | Stolen hash file |

**Analogy:** Online attack = trying to guess a safe combination while the alarm is active. Offline attack = stealing the safe, taking it home, and cracking it in your garage with unlimited time and no alarms.

**How Dictionary Attacks Work:**
1. Hash each password from the dictionary file
2. Compare hashed password to stolen hash
3. If hashes match (collision) → password found!
4. Move to next password in list

**Why It's So Fast:**
- Modern CPUs can hash billions of passwords per second
- No network latency
- No authentication system delays
- Purely computational problem

**Defense Recommendations:**
- Don't use passwords from common dictionaries
- Implement strong password policies
- Use MFA (even if password is cracked, attacker still blocked)
- Use salted hashes (makes pre-computed attacks harder)

---

## 🎯 Section 4: Brute Force Password Cracking (Offline Attack)

### What I Did
Used John the Ripper's incremental mode to perform brute force attacks, systematically trying every possible password combination.

### Technical Details

**Preparation:**
```bash
# Clear previous cracking history
rm ~/.john/john.pot
```

**First Attempt - MS10 Hashes (Strong Passwords):**
```bash
john --format=NT --incremental ms10-hashes.txt
```
- ⏱️ Ran for 5 minutes
- ❌ **No passwords cracked** (passwords too complex)
- 🛑 Terminated with `q`

**MS10 Password Policy:**
- Minimum 7 characters
- Must include: uppercase, lowercase, number
- **Result:** Too strong for quick brute force

**Second Attempt - Demo Hashes (Weak Passwords):**
```bash
john --format=NT --incremental demo-hashes.txt
```
- ⏱️ Ran for ~5 minutes
- ✅ **10 passwords cracked**
- Quick wins for short/simple passwords (1-4 characters)
- Longer passwords (5-7 chars) took progressively more time

**Viewing Results:**
```bash
john --show --format=NT demo-hashes.txt > brute-cracked.txt
less brute-cracked.txt

john --show=left --format=NT demo-hashes.txt  # Show uncracked
```

**Testing Length Limitations:**

**6-character maximum:**
```bash
john --format=NT --incremental --max-length=6 demo-hashes.txt
```
- ETA: **Many hours**

**7-character maximum:**
```bash
john --format=NT --incremental --max-length=7 demo-hashes.txt
```
- ETA: **Weeks in the future** 📅
- I noticed CertMaster wrote `--maxlength=7` which gives error, so I fixed it as shown above (--max-length=7).

**8-character maximum:**
- ETA: **Years** 🗓️

**9+ characters:**
- ETA: **Too long to calculate** ♾️

### Key Takeaways

**Brute Force Characteristics:**
- Tries **every possible combination** systematically
- Starts with shortest passwords first (1 char, 2 chars, 3 chars...)
- Uses all 95 printable ASCII characters by default

**JtR Incremental Modes:**
- `ASCII` - All 95 printable characters (default)
- `Alnum` - 62 alphanumeric only
- `Alpha` - 52 letters only
- `Lower` - Lowercase only
- `Upper` - Uppercase only
- `Digits` - Numbers only

**Primary Defenses:**
1. **Password Length** - Most critical factor
2. **Password Complexity** - Expands character set
3. **Slow Hash Algorithms** - bcrypt, scrypt, Argon2 (intentionally slow)
4. **MFA** - Even cracked passwords become useless

**Real-World Implications:**
- 8-character passwords = minimum baseline (still weak)
- 12+ characters = recommended
- 16+ characters = strong against current computing power
- Passphrases > random characters (easier to remember, still long)

---

## 🎯 Section 5: Wireless Password Cracking

### What I Did
Used aircrack-ng to crack WPA and WPA2 wireless network passwords from captured network traffic.

### Technical Details

**Setup:**
```bash
# Mounted ISO with pre-captured wireless traffic
cd /media/cdrom0/
ls -l  # Viewed .cap files
```

**Dictionary File:**
```bash
cat /home/kali/passwords14.txt
cat /home/kali/passwords14.txt | wc -l
```
- **107,984 passwords** in dictionary

### Attack 1: WPA Network

```bash
aircrack-ng -w /home/kali/passwords14.txt wpa.cap
```
- ⏱️ **< 1 minute**
- ✅ **Password found:** `dictionary`
- Network type: WPA PSK (Pre-Shared Key)

**Analogy:** WPA is like an old lock - it works, but modern lock-picking tools can open it easily because the lock's design has known weaknesses.

### Attack 2: WPA2-PSK Network

```bash
aircrack-ng -w /home/kali/passwords14.txt wpa2-psk-linksys.cap
```
- ⏱️ **< 1 minute**
- ✅ **Password found:** `password`
- Network type: WPA2 PSK (Linksys router)

**Analogy:** WPA2-PSK is like a better lock than WPA, but if you use a terrible key like "password," even the best lock is useless.

### Attack 3: WPA2 EAPoL Network

```bash
aircrack-ng -w /home/kali/passwords14.txt wpa2.eapol.cap
```
- ⏱️ **< 1 minute**
- ✅ **Password cracked**
- Network type: WPA2 with EAPoL (4-way handshake)

**EAPoL = Extensible Authentication Protocol over LAN**
- More secure than basic WPA2-PSK
- Still vulnerable if password is weak

### Key Takeaways

**How Wireless Cracking Works:**

1. **Capture the 4-way handshake** (authentication between client and AP)
2. **Extract the hashed password** from handshake packets
3. **Offline dictionary/brute force** attack against the hash
4. No need to stay connected - it's an **offline attack**

**Analogy:** It's like recording someone typing their door code, then going home to analyze the recording and figure out the code by trying every possibility.

**Wireless Security Comparison:**

| Protocol | Security Level | Vulnerable? |
|----------|---------------|-------------|
| WEP | 🔴 Very Weak | ✅ Easily cracked (excluded from lab) |
| WPA | 🟡 Weak | ✅ Yes - insufficient crypto |
| WPA2-PSK | 🟡 Moderate | ✅ Yes - weak passwords easily cracked |
| WPA2-EAPoL | 🟠 Better | ✅ Still vulnerable to weak passwords |
| WPA3-SAE | 🟢 Strong | ❌ Resistant to these attacks |

**Why WPA/WPA2 Are Vulnerable:**
- Password hash **is transmitted** during handshake
- Once captured, attacker has unlimited time to crack offline
- Short/simple passwords = quick crack

**WPA3 Fixes This:**
- Uses **SAE (Simultaneous Authentication of Equals)**
- Password hash is **never transmitted**
- Only one-way derived responses sent over network
- Even with captured handshake, can't crack password

### Defense Recommendations

**For WPA/WPA2 Networks:**
1. **Long, complex PSK** (20+ random characters)
2. **Use WPA2-Enterprise (802.1x)** with RADIUS/TACACS+
   - Unique credentials per user
   - MFA support
   - Centralized authentication
3. **Upgrade to WPA3** whenever possible

**For Enterprise:**
- Implement 802.1x authentication
- Require certificate-based auth
- Use RADIUS with MFA
- Regularly rotate credentials

**Analogy for WPA3:** Instead of showing your ID (password hash) that someone could copy, WPA3 is like a challenge-response where you prove you know the password without ever revealing it.

---

## 📊 Lab Summary & Comparison Table

### Attack Methods Comparison

| Attack Type | Speed | Detectability | Bypasses Lockout? | Best Against | Best Defense |
|-------------|-------|---------------|-------------------|--------------|--------------|
| **Password Guessing** | 🐌 Slow | 🔴 High | ❌ No | Weak passwords, no lockout | Account lockout, MFA |
| **Password Spraying** | 🐌 Slow | 🟡 Medium | ✅ Yes | Password reuse, many accounts | Unique passwords, anomaly detection |
| **Dictionary (Offline)** | 🚀 Very Fast | 🟢 None | ✅ Yes | Common passwords | Avoid dictionary words, MFA |
| **Brute Force (Offline)** | ⏱️ Varies | 🟢 None | ✅ Yes | Short passwords | Length (12+ chars), slow hashes |
| **Wireless Cracking** | 🚀 Fast | 🟢 None | ✅ Yes | WPA/WPA2 with weak PSK | WPA3, long PSK, 802.1x |

### Critical Insights

**Password Length is King:**
- 6 chars = hours to crack
- 7 chars = weeks to crack
- 8 chars = years to crack
- 12+ chars = computationally infeasible (current tech)

**Online vs Offline:**
- **Online:** Slow, detectable, stoppable
- **Offline:** Fast, invisible, unstoppable (once hashes are stolen)

**The MFA Solution:**
Even if passwords are cracked, MFA stops the attack. This is why **multi-factor authentication** is critical.

---

## 🛡️ PenTest Recommendations (What I'd Tell a Client)

### Immediate Actions:
1. ✅ **Reset all cracked passwords** immediately
2. ✅ **Enable MFA** on all accounts (especially admin accounts)
3. ✅ **Implement account lockout** (3-5 failed attempts)
4. ✅ **Monitor for anomalies** (password spraying patterns)

### Policy Improvements:
1. 📏 **Minimum 12-character passwords** (16+ for admins)
2. 🔤 **Complexity requirements** (upper, lower, numbers, symbols)
3. 🚫 **Password blacklisting** (block common passwords)
4. 🔄 **Regular password audits** using tools like this
5. 📚 **Security awareness training** (password hygiene)

### Technical Hardening:
1. 🔐 **Implement slow hash algorithms** (bcrypt, scrypt, Argon2)
2. 🧂 **Salt all password hashes** (prevents rainbow table attacks)
3. 📡 **Upgrade to WPA3** for wireless networks
4. 🏢 **Deploy 802.1x** for enterprise wireless (RADIUS + certificates)
5. 🕵️ **Deploy SIEM** for failed login monitoring

---

## 🎓 Personal Learning Outcomes

### Technical Skills Gained:
- ✅ Hands-on experience with John the Ripper (dictionary + brute force)
- ✅ Password spraying automation with Hydra
- ✅ Wireless password cracking with aircrack-ng
- ✅ Understanding hash formats (NTLM, WPA handshakes)
- ✅ Linux command-line proficiency (mounting shares, file manipulation)

### Conceptual Understanding:
- ✅ **Online vs. Offline attacks** - critical distinction
- ✅ **Time complexity** of password cracking (exponential growth)
- ✅ **Defense-in-depth** - why MFA matters even with strong passwords
- ✅ **Attack surface** - wireless networks are externally accessible

### PenTest+ Exam Relevance:
- Password attack methodologies (online vs. offline)
- Tool usage (JtR, Hydra, aircrack-ng)
- Risk assessment and reporting
- Remediation recommendations

---

## 🔗 Tools Reference

**John the Ripper:**
- Dictionary: `john --format=NT --wordlist=<file> <hashfile>`
- Brute Force: `john --format=NT --incremental <hashfile>`
- Show Results: `john --show --format=NT <hashfile>`

**Hydra:**
- Wizard Mode: `hydra-wizard`
- Manual: `hydra -L users.txt -P pass.txt <service>://<target>`

**Aircrack-ng:**
- `aircrack-ng -w <wordlist> <capfile>`

**SMB Mounting:**
- `mount //<ip>/<share> <mountpoint> -o username=<user>`

---

## 💭 Final Thoughts

This lab really drove home why **password length matters more than complexity**. The exponential time increase with each additional character makes brute force attacks impractical against long passwords, even if they're not complex.

The difference between online and offline attacks is huge - once an attacker has your password hashes, account lockout becomes useless. This is why:
1. **Hash protection** is critical (secure storage, salting)
2. **MFA** is non-negotiable for sensitive systems
3. **Monitoring** for hash extraction attempts is essential

For my career path toward becoming a pentester, understanding these attack vectors from both sides (offensive + defensive) helps me provide better recommendations to clients.

---

**Next Steps:**
- Practice creating custom wordlists with tools like `crunch` and `cewl`
- Learn about rainbow tables and hash cracking optimization
- Study password policy bypass techniques
- Explore GPU-accelerated cracking with Hashcat

---

*Lab completed as part of PenTest+ certification preparation*
