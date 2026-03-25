## HackTheBox — IMAP/POP3 Footprinting Module

### Overview
IMAP (port 143/993) and POP3 (port 110/995) are email protocols. Higher ports (993/995) use SSL/TLS encryption. These services can leak sensitive info like organization names, email addresses, and credentials during enumeration.

---

### Step 1 — Nmap Scan
```bash
sudo nmap <target-IP> -sV -p110,143,993,995 -sC
```
- Scans all IMAP/POP3 ports
- `-sC` runs default scripts including ssl-cert, which reveals **organization name**, **FQDN (commonName)**, and sometimes **email addresses**

---

### Step 2 — Grab Full Certificate Details
```bash
openssl s_client -connect <target-IP>:imaps
```
- Connects via SSL and dumps the full certificate
- Look for `emailAddress=` in the subject line — reveals admin email
- The IMAP banner at the bottom may also reveal version info or flags

---

### Step 3 — Login and Enumerate Mailboxes
```bash
1 LOGIN robin robin
1 LIST "" *
1 SELECT INBOX
```
- `LOGIN` — authenticates with username:password
- `LIST "" *` — lists all mailbox folders
- `SELECT INBOX` — opens the inbox

---

### Step 4 — Read Emails
```bash
1 FETCH 1 all        # shows email metadata (sender, subject, date)
1 FETCH 1 BODY[]     # shows full email body/content
```
- `FETCH 1 all` — reveals envelope data including sender email addresses
- `FETCH 1 BODY[]` — retrieves the actual email content including flags

---

### Key IMAP Commands Reference
| Command | Description |
|---|---|
| `1 LOGIN user pass` | Authenticate to server |
| `1 LIST "" *` | List all mailboxes |
| `1 SELECT INBOX` | Open a mailbox |
| `1 FETCH <ID> all` | Get email metadata |
| `1 FETCH <ID> BODY[]` | Get full email body |
| `1 LOGOUT` | Close connection |

---

### Key Concepts
- **SSL certificate** is a goldmine — reveals org name, FQDN, and admin email
- **IMAP banner** can leak version info and sometimes flags
- **Credentials found via SMTP enumeration** (`robin:robin`) can be reused on IMAP/POP3
- Always try `FETCH 1 BODY[]` to read actual email contents — metadata alone (`FETCH 1 all`) won't show the full message

---
