# Password Cracking with Hashcat — Hash Identification & Dictionary Attacks

> **Platform:** TryHackMe
> **Category:** Password Cracking / Credential Access
> **Tools:** `hashid`, `hashcat`, `rockyou.txt`
> **Skills demonstrated:** Hash identification, algorithm fingerprinting, dictionary attacks, selecting correct Hashcat modes, understanding cost factors in slow hashing

---

## Overview

This exercise involved identifying and cracking four password hashes, each produced by a **different algorithm**. The goal was not just to recover plaintext passwords, but to demonstrate *why identifying the hash type is a required first step* — feeding a hash to the wrong Hashcat mode fails immediately, regardless of how weak the underlying password is.

A key teaching point: **three of the four hashes share the same plaintext (`<redacted>`) yet look completely different**, because the algorithm transforms the input differently. The algorithm matters as much as the password.

| Hash | Algorithm | Hashcat Mode | Plaintext |
|------|-----------|:------------:|-----------|
| hash1 | MD5 | `0` | `<redacted>` |
| hash2 | SHA-256 | `1400` | `<redacted>` |
| hash3 | NTLM | `1000` | `<redacted>` |
| hash4 | bcrypt | `3200` | `<redacted>` |

---

## Methodology

For each hash the workflow was consistent:

1. **Identify** the algorithm with `hashid`, cross-referenced against the hash's visual characteristics (length, character set, prefix).
2. **Select** the matching Hashcat mode.
3. **Crack** using a straight dictionary attack against `rockyou.txt`.

### Command anatomy

```bash
hashcat -m <MODE> -a 0 <hashfile> /usr/share/wordlists/rockyou.txt
```

| Flag | Meaning |
|------|---------|
| `-m <MODE>` | Hash type (algorithm-specific numeric code) |
| `-a 0` | Attack mode 0 = straight dictionary (try each wordlist entry as-is) |
| `/usr/share/wordlists/rockyou.txt` | ~14.3 million leaked real-world passwords |

**Identifying the algorithm by hash length** (a fast field heuristic):

| Hex length | Likely algorithm(s) |
|:----------:|---------------------|
| 32 | MD5 **or** NTLM |
| 40 | SHA-1 |
| 64 | SHA-256 |
| `$2a$` / `$2b$` prefix | bcrypt |

`hashid` narrows the candidates; length + context makes the final call.

---

## Hash 1 — MD5

**Hash:** `e10adc3949ba59abbe56e057f20f883e`

### Identify
```bash
hashid 'e10adc3949ba59abbe56e057f20f883e'
```
32 hex characters, no prefix → MD5 (or NTLM). `hashid` confirms MD5 among its candidates.

### Crack
```bash
hashcat -m 0 -a 0 hash1.txt /usr/share/wordlists/rockyou.txt
```

### Result
```
e10adc3949ba59abbe56e057f20f883e:<redacted>
Status...........: Cracked
Hash.Mode........: 0 (MD5)
```

**Plaintext: `<redacted>`** — cracked in under a second. MD5 is unsalted and computationally cheap, so a fast GPU/CPU tears through the wordlist instantly.

---

## Hash 2 — SHA-256

**Hash:** `5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8`

### Identify
```bash
hashid hash2.txt
```
64 hex characters → SHA-256.

### Crack
```bash
hashcat -m 1400 -a 0 hash2.txt /usr/share/wordlists/rockyou.txt
```

### Result
```
5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8:<redacted>
Status...........: Cracked
Hash.Mode........: 1400 (SHA2-256)
```

**Plaintext: `<redacted>`**

---

## Hash 3 — NTLM

**Hash:** `8846f7eaee8fb117ad06bdd830b7586c`

### Identify
```bash
hashid hash3.txt
```
32 hex characters — same length as MD5. Context (Windows credential exercise) and `hashid`'s NTLM candidate point to **NTLM**. This is a good example of why length alone isn't enough: MD5 and NTLM are indistinguishable by length.

### Crack
```bash
hashcat -m 1000 -a 0 hash3.txt /usr/share/wordlists/rockyou.txt
```

### Result
```
8846f7eaee8fb117ad06bdd830b7586c:<redacted>
Status...........: Cracked
Hash.Mode........: 1000 (NTLM)
```

**Plaintext: `<redacted>`** — identical plaintext to hash2, but a completely different hash string. NTLM (used by Windows) and SHA-256 process the same input in totally different ways.

---

## Hash 4 — bcrypt

**Hash:** `$2b$05$9I7YCSrgm6aLO7J5YPC9x.Kp08LQ7cSJTmkALhFTgm5UMFAwBr5.e`

### Identify
The `$2b$` prefix is an unmistakable bcrypt signature. The `05` immediately after is the **cost factor** — 2⁵ (32) iterations of the key-setup routine per guess.

### Crack
```bash
hashcat -m 3200 -a 0 hash4.txt /usr/share/wordlists/rockyou.txt
```

### Result
```
$2b$05$9I7YCSrgm6aLO7J5YPC9x.Kp08LQ7cSJTmkALhFTgm5UMFAwBr5.e:<redacted>
Status...........: Cracked
Hash.Mode........: 3200 (bcrypt $2*$, Blowfish (Unix))
Time.Started.....: 1 min, 35 secs
Speed.#1.........: 641 H/s
```

**Plaintext: `<redacted>`**

Note the speed difference: MD5 ran at **~375,000 H/s**, bcrypt at **~641 H/s** — roughly **585× slower**. This is by design. bcrypt is a *deliberately expensive* (adaptive) hashing function, and the cost factor can be raised over time to keep pace with hardware. This is why bcrypt (and similar slow hashes like Argon2 and scrypt) are recommended for storing real user passwords, while MD5/SHA are not.

---

## Troubleshooting notes

A couple of real errors encountered and how they were resolved — worth documenting because they're common:

- **`Token length exception` / `No hashes loaded`** — Hashcat rejects the input when the specified mode doesn't match the hash format (e.g. running `-m 100` SHA-1 against a 64-char SHA-256 hash, or `-m 1000` NTLM against an MD5). Fix: confirm the algorithm with `hashid` and match the correct `-m` mode.
- **Always `cat` the real hash file before cracking.** Verify you're attacking the actual target file and not a placeholder or an example string copied from the task description. Cracking the wrong input wastes time and produces the wrong answer.

---

## Key takeaways

1. **Identify before you crack.** The correct Hashcat mode is non-negotiable — the right password against the wrong mode fails instantly.
2. **The same plaintext produces wildly different hashes across algorithms.** hash1, hash2, and hash3 all hide simple passwords, but MD5, SHA-256, and NTLM outputs share nothing visually.
3. **Slow hashing is a feature, not a bug.** bcrypt's cost factor makes brute-forcing expensive — the defensive counterpart to everything demonstrated here.
4. **Weak passwords fall instantly.** `123456` and `password` are top entries in `rockyou.txt` and crack in milliseconds regardless of algorithm — reinforcing why password *strength* and *slow hashing* must work together.

---

## References

- [Hashcat mode reference (`--help`)](https://hashcat.net/wiki/doku.php?id=example_hashes)
- [`hashid` on GitHub](https://github.com/psypanda/hashID)
- rockyou.txt — standard wordlist bundled with Kali / included in `/usr/share/wordlists/`
