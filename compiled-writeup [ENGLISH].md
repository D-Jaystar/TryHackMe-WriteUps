# ⚙️ Compiled — TryHackMe Writeup

**Category:** Reverse Engineering
**Room:** [tryhackme.com/room/compiled](https://tryhackme.com/room/compiled)
**Author:** Djaystar
**Date:** September 2026

## Overview

**Goal:** analyze and reverse-engineer an ELF binary to determine the expected input format and password, through binary triage, static analysis, and decompilation in Ghidra.

**Approach:**
1. **Binary triage** — determine file format and architecture using CLI tools.
2. **Analysis** — inspect embedded strings and functions.
3. **Decompilation** — reconstruct the binary in Ghidra.
4. **Reversal** — break down the logic and verify the correct password.

**Target:** `Compiled.Compiled`, located at `/root/Rooms/Compiled/` (THM VM environment).

---

## Step 1 — String Analysis

**Goal:** check whether the binary contains readable text, such as hardcoded passwords.

```bash
strings Compiled.Compiled | head -n 30
```

**Output (relevant):**
- `/lib64/ld-linux-x86-64.so.2` → indicates an ELF binary.
- `GCC: (Debian 11.3.0-5) 11.3.0` → written in C or C++.
- `Password: DoYouEven%sCTF` → prompt shown to the user.

---

## Step 2 — Profiling and Inspection

**Goal:** determine the file format and properties of the binary.

```bash
file Compiled.Compiled
```

**Output:**
```
Compiled.Compiled: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked,
interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=06dcfaf13fb76a4b556852c5fbf9725ac21054fd,
for GNU/Linux 3.2.0, not stripped
```

**Interpretation:** a 64-bit Linux binary, dynamically linked, and **not stripped** — the symbol names (including function names) are still present, which will make decompilation in Ghidra significantly easier.

---

## Step 3 — Security Check with checksec

**Goal:** map out the compiler-level protections before analyzing the binary in Ghidra.

```bash
checksec --file=Compiled.Compiled
```

**Output:**
| Protection | Status |
|---|---|
| RELRO | Partial RELRO |
| Stack Canary | Not found |
| NX | Enabled |
| PIE | Enabled |
| Stripped | No |

**Interpretation:** the absence of a stack canary means stack-based buffer overflows are theoretically possible, but this room focuses on reading the logic rather than exploitation. PIE confirms the binary's memory address shifts on each run.

---

## Step 4 — Ghidra Project and Analysis

**Goal:** load the file and convert the machine code into readable pseudocode.

**Action:** imported the file into a new Ghidra project and ran the automatic analysis.

---

## Step 5 — Locating Code via Strings

**Goal:** find the relevant function by searching for known text.

**Action:** searched for `CTF` in **Defined Strings**, then jumped to the corresponding code via the cross-reference (XREF).

---

## Step 6 — Jumping to the Main Function via XREF

**Goal:** jump directly from the found string to the controlling logic.

**Action:** double-clicked the `main` reference next to the string `DoYouEven%sCTF` in the listing.

**Result:** the decompiled C code of the main function appears in the Decompiler window.

---

## Step 7 — Logic Analysis

**Goal:** break down the decompiled C code to determine the correct password.

**Action:** identified in the `main` function that `scanf` expects input following the format `DoYouEven%sCTF`, and that the entered value is then compared via `strcmp` against an internal reference — the actual check turned out to revolve around the `_init` part of the string.

---

## Step 8 — First Verification Attempt

**Goal:** test the password against the actual binary.

**Action:**
```bash
chmod +x Compiled.Compiled
./Compiled.Compiled
```
Entered: `DoYouEven_initCTF`

**Result:** ❌ Incorrect — the full format-string pattern was not the correct input.

---

## Step 9 — Correct Input and Verification

**Goal:** correctly apply how the format string in `scanf` actually works.

**Action:** tested the input without the `CTF` suffix: `DoYouEven_init`

**Result:** ✅ Correct — the string matched the required `_init` check in `strcmp`.

---

## Reflection

This room is a good illustration of why you shouldn't just type a prompt string back literally: the visible text (`DoYouEven%sCTF`) is a **format string**, not the literal expected input. The `%s` is a placeholder, and the actual comparison in the code (`strcmp` against `_init`) determined the real password. It highlights the value of decompilation over assuming what a prompt appears to say — static string analysis gives hints, but the actual logic in Ghidra gives the answer.

---

*This writeup was structured and formatted with the help of AI, based on my own test results and notes. My original, raw documentation and notes are kept in Obsidian — feel free to reach out if you'd like to see them or are interested in collaborating.*
