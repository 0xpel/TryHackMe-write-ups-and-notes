# OSINT Cheat Sheet

> ⚠️ **Disclaimer — read first.**
> This reference is for **education, self-audit, and authorized engagements only**.
> Run these techniques on **yourself**, on **organizations with a bug-bounty / written
> scope**, or on **training platforms**. Building a profile on a **private individual**
> without consent (finding where they live, their phone, etc.) is **not** OSINT practice —
> it is stalking/doxxing and may be illegal. **Use at your own risk and responsibility.**
> The author assumes no liability for misuse.

The method matters more than the tools. OSINT is a loop: hold one identifier → ask
"what can I reach from this?" → pick the fitting tool → each result is a new starting
point (**pivoting**).

---

## Golden rules

1. **Single source ≠ certainty.** Verify every tool hit manually.
2. **Assign confidence** to each finding (high / medium / none) — don't state inferences as facts.
3. **"Clean" depends on which vector you checked** — work a checklist, don't stop early.
4. **Tools lie in 4 ways:** false positives, over-reads, tool-rot (API changes), unverified constructed URLs.
5. **Authorization, not technique, is the legal line.** Same tools, legitimate only with scope/consent.

---

## Start by identifier strength

| Starting point | Strength | Why |
|---|---|---|
| Email | 🟢 strong | Unique, tied to services + breaches |
| Username | 🟢 strong | Unique, reused across platforms |
| Phone | 🟡 medium | Unique, rarely indexed publicly |
| First + last name | 🔴 weak alone | Shared by many; needs an anchor |

---

## The checklist — tool per identifier

### Email →
| Tool | Command | Returns |
|---|---|---|
| **holehe** | `holehe target@example.com` | Which of ~120 services the email is registered on |
| **Have I Been Pwned** | web: haveibeenpwned.com | Which breaches the email appears in (+ data categories) |
| **GHunt** (gmail) | `ghunt email target@gmail.com` | Google account name, profile pic, Maps reviews |
| Google dork | `"target@example.com"` | Any public page indexing the address |

> **holehe** is quiet (checks via password-reset/registration flows, doesn't email the target).
> Results vary between runs (rate-limiting) — it's a snapshot, not absolute truth.

### Username →
| Tool | Command | Returns |
|---|---|---|
| **Sherlock** | `sherlock <user>` | Profiles across ~400 sites |
| **Sherlock** (multi) | `sherlock user1 user2 user3` | Try variations of a handle |
| Google dork | `"<username>"` | Public mentions |

> Sherlock false-positives on parked/dead domains that return HTTP 200. **Open every hit.**

### Phone →
| Tool | Command | Returns |
|---|---|---|
| **PhoneInfoga** (CLI) | `phoneinfoga scan -n "+<country><number>"` | Carrier, country, line type, dorks |
| **PhoneInfoga** (web) | `phoneinfoga serve` → localhost:5000 | Same, with a UI |

> PhoneInfoga goes **number → info**, NOT name → number. It does **not** reveal an owner's
> name. Format numbers as E.164: `+<country code><number>`, no leading zero.

### Password →
| Tool | Where | Returns |
|---|---|---|
| **HIBP Pwned Passwords** | haveibeenpwned.com/Passwords | Whether a password is in a dump (checks a hash, never send the password itself) |

### Name / domain / geolocation →
| Need | Tool |
|---|---|
| Starting map of all tools | **OSINT Framework** (osintframework.com) |
| Link-analysis graph of entities | **Maltego** (Community Edition) |
| Modular recon framework | **Recon-ng** |
| Org email/subdomain harvesting | **theHarvester** |
| Geospatial / satellite / historical imagery | **Google Earth Pro** |
| WiFi network geolocation | **WiGLE.net** |

---

## Google Dorking essentials

| Operator | Effect |
|---|---|
| `"exact phrase"` | Exact match (Google may still broad-match — verify) |
| `site:example.com` | Restrict to a domain |
| `-site:example.com` | Exclude a domain |
| `intext:"term"` | Term appears in body |
| `intitle:"term"` | Term in page title |
| `filetype:pdf` | Specific file type |
| `inurl:term` | Term in the URL |

> Result **count is misleading** — "thousands of results" of noise = zero intel. Count
> *relevant* results, not total. A real exact-match hit ranks first, not buried.

---

## Breach data — the boundary

- **HIBP tells you THAT you're exposed and WHICH categories** — by design it does *not*
  show the leaked values. That's the legitimate, defensive use.
- The raw values live in **stolen-data repositories** (DeHashed, Snusbase, breach forums).
  Pulling records from those works identically on **any** person and trades in stolen data.
  Keep that inside an authorized engagement (or a course that frames it legally) — not casual practice.
- For self-audit, the finding *"exposure exists across N breaches"* **is** the result.
  You don't need to exfiltrate stolen records to understand or act on your exposure.

---

## Self-audit → defensive checklist

- Check **hidden name fields** (display name ≠ legal/account name).
- Prune **map reviews** that cluster near home/work.
- Audit **social-profile privacy** (often the biggest exposure).
- Unique passwords + **password manager** + **2FA** (prefer app over SMS).
- Assume more **targeted phishing** once name + email + region are inferable.

---

## Install quick-ref (Kali)

```bash
# pipx keeps each tool isolated (avoids Python dependency clashes)
sudo apt install pipx -y && pipx ensurepath
pipx install sherlock-project
pipx install holehe
pipx install ghunt            # then: ghunt login  (browser cookie auth)
# PhoneInfoga: install script or Docker (see its GitHub repo)
```

> Tools age fast (APIs change). If one crashes or returns odd data, check its GitHub
> issues / install the latest version — that's normal "tool-rot," not your mistake.

---

*Reference maintained for my own learning. Again: yourself, authorized targets, or training only.*
