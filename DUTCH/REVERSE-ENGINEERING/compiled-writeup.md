# ⚙️ Compiled — TryHackMe Writeup

**Categorie:** Reverse Engineering
**Room:** [tryhackme.com/room/compiled](https://tryhackme.com/room/compiled)
**Auteur:** Djaystar
**Datum:** september 2026

## Overzicht

**Doel:** een ELF-binary analyseren en reverse-engineeren om het verwachte inputformaat en wachtwoord te achterhalen, via binary triage, statische analyse en decompilatie in Ghidra.

**Plan van aanpak:**
1. **Binary triage** — bestandsformaat en architectuur bepalen via CLI-tools.
2. **Analysis** — embedded strings en functies inspecteren.
3. **Decompilation** — de binary reconstrueren in Ghidra.
4. **Reversal** — de logica ontleden en het correcte wachtwoord verifiëren.

**Target:** `Compiled.Compiled`, lokaal op `/root/Rooms/Compiled/` (THM VM-omgeving).

---

## Stap 1 — Strings-analyse

**Doel:** checken of de binary leesbare tekst bevat, zoals hardcoded wachtwoorden.

```bash
strings Compiled.Compiled | head -n 30
```

**Output (relevant):**
- `/lib64/ld-linux-x86-64.so.2` → wijst op een ELF-binary.
- `GCC: (Debian 11.3.0-5) 11.3.0` → geschreven in C of C++.
- `Password: DoYouEven%sCTF` → prompt die aan de gebruiker getoond wordt.

---

## Stap 2 — Profiling en inspectie

**Doel:** het bestandsformaat en de eigenschappen van de binary vaststellen.

```bash
file Compiled.Compiled
```

**Output:**
```
Compiled.Compiled: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked,
interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=06dcfaf13fb76a4b556852c5fbf9725ac21054fd,
for GNU/Linux 3.2.0, not stripped
```

**Interpretatie:** 64-bit Linux-binary, dynamisch gelinkt, en **niet gestript** — de symboolnamen (waaronder functienamen) zijn dus nog aanwezig, wat de decompilatie in Ghidra straks een stuk makkelijker maakt.

---

## Stap 3 — Beveiliging checken met checksec

**Doel:** de compiler-niveau beveiligingen in kaart brengen voordat de binary in Ghidra wordt geanalyseerd.

```bash
checksec --file=Compiled.Compiled
```

**Output:**
| Beveiliging | Status |
|---|---|
| RELRO | Partial RELRO |
| Stack Canary | Niet aanwezig |
| NX | Enabled |
| PIE | Enabled |
| Stripped | Nee |

**Interpretatie:** geen stack canary betekent dat stack-gebaseerde buffer overflows in theorie mogelijk zijn, maar voor deze room ligt de focus op het lezen van de logica, niet op exploitatie. PIE bevestigt dat het geheugenadres van de binary bij elke run verschuift.

---

## Stap 4 — Ghidra-project en analyse

**Doel:** het bestand inladen en de machinetaal omzetten naar leesbare pseudocode.

**Actie:** het bestand geïmporteerd in een nieuw Ghidra-project en de automatische analyse laten draaien.

---

## Stap 5 — Code vinden via strings

**Doel:** de relevante functie lokaliseren door te zoeken op bekende tekst.

**Actie:** gezocht op `CTF` in **Defined Strings**, en via de cross-reference (XREF) naar de bijbehorende code gesprongen.

---

## Stap 6 — Springen naar de main-functie via XREF

**Doel:** vanuit de gevonden string direct naar de controlerende logica springen.

**Actie:** in de listing gedubbelklikt op de `main`-referentie naast de string `DoYouEven%sCTF`.

**Resultaat:** de gedecompileerde C-code van de hoofdfunctie verschijnt in het Decompiler-venster.

---

## Stap 7 — Logica-analyse

**Doel:** de gedecompileerde C-code ontleden om het juiste wachtwoord te achterhalen.

**Actie:** in de `main`-functie vastgesteld dat `scanf` invoer verwacht volgens het format `DoYouEven%sCTF`, en dat de ingevoerde waarde vervolgens via `strcmp` vergeleken wordt tegen een interne referentie — de eigenlijke check bleek te draaien om het `_init`-deel van de string.

---

## Stap 8 — Eerste verificatiepoging

**Doel:** het wachtwoord testen op de daadwerkelijke binary.

**Actie:**
```bash
chmod +x Compiled.Compiled
./Compiled.Compiled
```
Ingevoerd: `DoYouEven_initCTF`

**Resultaat:** ❌ Fout — het volledige format-string-patroon was niet de juiste invoer.

---

## Stap 9 — Correcte invoer en verificatie

**Doel:** de werking van het format-string in `scanf` correct toepassen.

**Actie:** invoer getest zonder het `CTF`-achtervoegsel: `DoYouEven_init`

**Resultaat:** ✅ Correct — de string matchte de vereiste `_init`-controle in `strcmp`.

---

## Reflectie

Deze room laat goed zien waarom je een prompt-string niet zomaar letterlijk moet overtikken: de zichtbare tekst (`DoYouEven%sCTF`) is een **format string**, geen letterlijke verwachte invoer. De `%s` is een placeholder, en de werkelijke vergelijking in de code (`strcmp` tegen `_init`) bepaalde het echte wachtwoord. Het benadrukt het belang van decompilatie boven het aannemen van wat een prompt lijkt te zeggen — statische string-analyse geeft aanwijzingen, maar de daadwerkelijke logica in Ghidra geeft het antwoord.

---

*Deze writeup is met behulp van AI gestructureerd en opgemaakt op basis van mijn eigen testresultaten en aantekeningen. Mijn originele, ruwe documentatie en notities houd ik bij in Obsidian — neem gerust contact met mij op als je die wilt inzien of samen wilt werken.*
