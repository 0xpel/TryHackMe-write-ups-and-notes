# OSINT Self-Audit: Methodology & the Discipline of Verification

> **Scope & ethics:** This exercise was performed entirely on **my own** identifiers
> (my own email addresses, username, and phone number) as a personal digital-footprint
> audit. No third party was targeted. All real identifiers, results, names, and
> locations are redacted and replaced with placeholders. The goal is to learn the
> methodology and to understand — and reduce — my own exposure.

## Why self-OSINT

The safest and most instructive way to learn reconnaissance is to run it on yourself.
You see exactly what you expose to the world, you can verify every finding against ground
truth, and the output is directly actionable (clean up what leaks). This write-up documents
a full self-audit end to end — including the parts that *failed*, which turned out to be the
most valuable lessons.

---

## The core idea: pivoting

OSINT is not "knowing 50 tools." It's a simple loop: you hold one piece of information,
you ask *"what else can I reach from this?"*, you pick the one tool that fits, and each
thing you discover becomes a new starting point.

```
email  ──► registered services ──► username ──► profiles ──► real name ──► location / habits
```

When a path dead-ends, you don't get stuck — you return to a **checklist** and pick another
vector.

## Strength of a starting identifier

Not every data point is equally useful to start from:

| Starting point | Strength | Why |
|---|---|---|
| Email | 🟢 strong | Unique, tied to many services, appears in breaches |
| Username | 🟢 strong | Unique, people reuse it everywhere |
| Phone | 🟡 medium | Unique, but rarely indexed publicly |
| First + last name | 🔴 weak alone | Shared by many people; needs another anchor to narrow |

---

## The checklist (what stopped me getting stuck)

| Vector | Tool | Direction |
|---|---|---|
| Registered services | holehe | email → services |
| Profile discovery | Sherlock | username → profiles |
| Public footprint | Google dorking | string → indexed pages |
| Breach exposure | Have I Been Pwned | email → breaches (metadata) |
| Password exposure | HIBP Pwned Passwords | password hash → leaked? |
| Google account | GHunt | gmail → Google profile / Maps |
| Phone characteristics | PhoneInfoga | number → carrier / line type / dorks |

The checklist is the real deliverable of the mindset: every time one vector returned
nothing, there was somewhere else to go instead of concluding "clean" too early.

---

## Case walkthrough (sanitized)

**Starting point:** one of my own email addresses + its username portion.

1. **holehe (email → services):** returned a small number of registered services. Each
   registered service is a door — a place that *might* expose more.
2. **Sherlock (username → profiles):** returned a single hit — which turned out to be a
   **false positive** (see lessons). Manually verified: nothing there.
3. **Google dorking:** exact-match search on the username returned only noise; the name
   search surfaced one social profile I own.
4. **HIBP:** the email appeared in multiple breaches. Important nuance below.
5. **Pwned Passwords:** my current password was **not** present — good.
6. **GHunt (gmail → Google account):** this was the breakthrough vector. After a parser
   bug was worked around, it resolved the email to a Google account exposing a real name
   and public Google Maps review activity.

**Result:** starting from "this address looks clean" (holehe/Sherlock returned little), the
*right* vector (GHunt) showed the address actually exposes a real name and location-pattern
data. The lesson writes itself: **"clean" depends on which vector you checked.**

---

## The verification lessons (the real value)

Across one session, tool output was wrong or misleading in **four different ways**. Each
time, manual verification caught it. This is the single most important OSINT skill.

### 1. False positive
Sherlock reported a profile on a service that was actually a **parked domain**. The parking
page returned HTTP 200 for any path, so the tool read "profile exists." Opening the URL
manually showed there was nothing there. *Tools infer existence from responses; dead
services and parking pages lie.*

### 2. Over-reading breach metadata
HIBP confirms two things only: (a) the email string appeared in a breach dataset, and (b)
the breach *as a whole* contained certain data categories. It does **not** confirm that *my
specific record* contained all those fields. "Email is in a breach that contained phone
numbers" ≠ "my phone number leaked." Precision matters.

### 3. Tool-rot
GHunt crashed with a `KeyError` while parsing the response — because the upstream provider
changed its API response shape and the tool hadn't been updated for it. OSINT tools age fast
as platforms change; a crash is not "nothing found."

### 4. Unverified constructed URLs
GHunt listed a public calendar feed URL. But that URL is a **standard template** that exists
in form for every account — it doesn't mean the feed is actually public. Fetching it returned
an error → the feed was **not** public. The tool output a *possible* artifact; ground truth
came from testing it.

> **Single source ≠ certainty.** Every finding was confirmed against ground truth before being
> recorded. That discipline is what separates an analyst from someone who runs tools.

---

## Assigning confidence to findings

A professional finding isn't "they live in city X." It's a claim with a confidence level:

| Finding | Confidence | Basis |
|---|---|---|
| From country Y | 🟢 high | Review written in that country's language |
| General region | 🟡 medium | Clustering of local reviews |
| Exact address | 🔴 none | Not derivable from open sources used here |

Review **clustering** reveals a geographic "center of gravity" (people review places near
where they live/work). That narrows a region — it does not pin an address. Knowing the
difference, and stating confidence honestly, is the craft.

---

## Defensive takeaways

What the audit surfaced about my own hygiene, and what I changed:

- **Hidden name fields.** Changing a *display* name does not change a provider's separate
  **legal/account name** field — which is what leaked. Fixed the hidden field too.
- **Map reviews expose location patterns.** Public reviews cluster near home/work. Reviewed
  and pruned the most location-revealing ones.
- **Social profiles are often the biggest exposure** — more than email. Audited privacy
  settings there.
- **Breaches ≠ password compromise here.** The email is in breach corpora, but the current
  password isn't in any dump. Still: unique passwords + a password manager + 2FA.
- **Realistic risk:** not someone at the door (no exact address found) — **targeted phishing**,
  because name + email + an inferred region is enough to craft a convincing lure.

---

## The ethical line (why this stayed an audit)

The techniques a red teamer uses and the techniques an attacker uses are **identical**. What
makes them legitimate is **authorization** — a signed scope and rules of engagement — not the
technique itself.

This exercise stayed strictly on **my own** identifiers. Two things were deliberately out of
bounds and belong only inside an authorized engagement (or a training platform that simulates
one): pulling raw records from stolen-data repositories, and deanonymizing a person down to a
physical address. The finding that *exposure exists* is the professional result; exfiltrating
stolen data to prove it is not required, and locating a private individual's home is the
doxxing line.

**Where to learn the full attacker toolkit legitimately:** authorized engagements, and
training that builds authorization into the frame — TCM Security's PNPT / OSINT courses,
Hack The Box Pro Labs, TryHackMe red-team paths.

---

## Key takeaway

A thorough audit of a single email reached: a verified real name, country (high confidence),
and region (medium confidence) — but **no phone and no exact address**, and every claim carries
an honest confidence level. More importantly, the session was a live lesson in **not trusting
tool output**: false positives, over-reads, tool-rot, and unverified artifacts all appeared,
and manual verification caught every one. That discipline — not the tool list — is the job.
