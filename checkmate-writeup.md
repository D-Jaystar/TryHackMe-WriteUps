# 🛡️ Checkmate — TryHackMe Writeup

**Categorie:** Cracking / Authentication Attacks
**Room:** [tryhackme.com/room/checkmate](https://tryhackme.com/room/checkmate)
**Auteur:** Djaystar
**Datum:** september 2026

## Overzicht

Deze room bouwt een authenticatie-aanval op tegen vier gekoppelde services die horen bij dezelfde fictieve sysadmin, **Marco Bianchi**. Elke service vereist een andere aanpak, waardoor informatie uit de ene stap steeds de wordlist voor de volgende stap verbetert:

`Default credentials → OSINT profiling → Custom wordlists → Hash cracking → SSH brute-force`

| Level | Service | Techniek | Resultaat |
|---|---|---|---|
| 1 | Firewall Console (`:5001`) | Default credentials + Hydra | `admin:12345` |
| 2 | Employee Portal (`:5002`) | Website-scraping → custom wordlist | `marco:excellence` |
| 3 | Social Platform (`:5003`) | OSINT profiling met CUPP | `marco:Bianchi2495` |
| 4 | Hash in afbeelding | SHA256 extractie + cracking | `family` |
| 5 | SSH (`:22`) | Wordlist + Hashcat rules | `marco:Security2024!` |

---

## Level 1 — The Firewall Console

**Doel:** toegang krijgen tot de centrale firewall console via `firewall.thm:5001`.

1. DNS-mapping toegevoegd via `sudo nano /etc/hosts` → `firewall.thm` gekoppeld aan `10.112.136.92`.
2. Broncode geïnspecteerd (`Ctrl+U`): het loginformulier stuurt een POST-request naar `/login` met de parameters `username` en `password`.
3. Brute-force uitgevoerd met Hydra tegen `rockyou.txt`:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt firewall.thm -s 5001 \
  http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid credentials." -t 16
```

**Resultaat:** `admin:12345`

**Inzicht:** de console draaide nog op default credentials en fungeerde als toegangspoort tot de rest van het netwerk — een klassiek voorbeeld van waarom standaardwachtwoorden nooit in productie mogen blijven staan.

---

## Level 2 — Employee Login Portal

**Doel:** inloggen op de interne portal via `jobs.thm:5002`.

1. Een hint op de pagina gaf aan dat gebruiker `marco` bedrijfsgerelateerde termen als wachtwoord gebruikt.
2. Formulier-inspectie bevestigde dezelfde POST-structuur naar `/login`.
3. Eerst getest met een kleine, handmatige lijst (`company_passwords.txt`); daarna een bredere lijst gegenereerd door de website-inhoud te scrapen:

```bash
curl -s http://jobs.thm:5002 | sed 's/<[^>]*>/ /g' | tr -cs '[:alnum:]' '\n' \
  | awk 'length >= 4' | sort -u > /tmp/cewl_passwords.txt
```

4. Brute-force met Hydra tegen de gescrapete lijst:

```bash
hydra -l marco -P /tmp/cewl_passwords.txt jobs.thm -s 5002 \
  http-post-form "/login:username=^USER^&password=^PASS^:F=Sign in" -t 6
```

**Resultaat:** `marco:excellence`

**Verzamelde profielinfo:** Marco Bianchi, IT Operations, geboren 14-02-1995, bijnaam "marky" — deze gegevens vormen de basis voor Level 3.

---

## Level 3 — Social Platform (OSINT)

**Doel:** Marco's persoonlijke gegevens omzetten naar een wachtwoord op `social.thm:5003`.

1. **CUPP** (Common User Passwords Profiler) gebruikt om op basis van Marco's PII (naam, geboortedatum, bijnaam) een gerichte wordlist te genereren.
2. Brute-force met Hydra tegen de gegenereerde lijst:

```bash
hydra -l marco -P marco.txt social.thm -s 5003 \
  http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid credentials." -t 4
```

**Resultaat:** `marco:Bianchi2495` — een combinatie van achternaam en geboortejaar, en een goed voorbeeld van hoe voorspelbaar mensen wachtwoorden opbouwen uit persoonlijke informatie.

---

## Level 4 — The Hash Crack (SHA256)

**Doel:** een SHA256-hash uit een PNG-bestand extraheren en kraken naar plaintext.

1. Hash geëxtraheerd uit de afbeelding.
2. Hash gekraakt via CrackStation / Hashcat.

**Resultaat:** `family`

---

## Level 5 — The Final SSH Crack

**Doel:** directe shell-toegang verkrijgen via SSH (poort 22).

1. Een bredere wordlist opgebouwd (`company_words.txt`) via dezelfde scraping-aanpak als Level 2.
2. De lijst gemuteerd met **Hashcat rules** (`marco.rule`) om realistische varianten te genereren (hoofdletters, cijfers, speciale tekens) → `marco_ssh.txt`.
3. Brute-force met Hydra tegen SSH:

```bash
hydra -l marco -P /tmp/marco_ssh.txt 10.114.190.8 ssh -t 4
```

**Resultaat:** `marco:Security2024!`

---

## 📚 Tooling & Concepten

| Tool / Concept | Werking | Toepassing in deze room |
|---|---|---|
| **CUPP** | Zet PII (naam, datum, partner, etc.) om in een gerichte wordlist via interactieve vragen. | Genereren van de wordlist voor Level 3 op basis van Marco's profiel. |
| **CeWL-achtige scraper** | Bash one-liner die woorden van een website scrapt, HTML-tags verwijdert en unieke tokens overhoudt (≥ 4 letters). | Website-inhoud van poort 5002 omgezet naar een bruikbare wordlist voor Level 2. |
| **Hydra** | Netwerk brute-forcer die HTTP POST-formulieren en andere protocollen (SSH, FTP, ...) test tegen een wordlist, op basis van een failure string. | Hoofdtool in vrijwel elk level om gebruikersnaam/wachtwoord-combinaties te testen. |
| **Hashcat rules** | Regels die bepalen hoe een woord muteert (bijv. eerste letter hoofdletter, cijfer toevoegen aan het einde). | Omzetten van simpele woorden naar realistische wachtwoorden zoals `Security2024!`. |
| **wc** | Linux-utility om regels in een bestand te tellen zonder het volledig te openen. | Verifiëren van de omvang van de gescrapete wordlist. |

---

## Reflectie

Deze room laat zien hoe zwakke authenticatie zich stapelt: een default wachtwoord op één service leidt tot informatie die de aanval op de volgende service vergemakkelijkt. Het benadrukt het belang van sterke, unieke wachtwoorden per service, het vermijden van voorspelbare patronen (naam + geboortejaar), en het beperken van hoeveel persoonlijke informatie publiekelijk beschikbaar is.

---

*Deze writeup is met behulp van AI gestructureerd en opgemaakt op basis van mijn eigen testresultaten en aantekeningen. Mijn originele, ruwe documentatie en notities houd ik bij in Obsidian — neem gerust contact met mij op als je die wilt inzien of samen wilt werken.*
