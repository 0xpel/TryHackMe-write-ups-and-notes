# Pickle Rick — TryHackMe Write-up

## Room Info
- **Difficulty:** Easy
- **Topic:** Web enumeration, command execution, privilege escalation
- **OS:** Linux
- **Goal:** Help Rick turn back from a pickle by finding 3 secret ingredients.

> All target IP addresses are redacted (`<IP>`).

---

## Attack chain (the methodology)

```
Recon → Credentials → Access → Command Execution → Enumeration → Privilege Escalation
```

This mirrors a real web pentest: find a foothold, then escalate to full control.

---

## 1. Recon

### Directory brute force
```bash
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

**Key lesson:** my first run *without* `-x php` found nothing useful and I almost went down a
vhost-enumeration rabbit hole. Most "found nothing" in web recon is a **wordlist/extension**
problem, not an absence of content. Adding `-x php,html,txt` surfaced the login portal
(`login.php`).

Findings: `index.html`, `assets/`, `robots.txt`, and a `.php` login portal.

### robots.txt — not a rabbit hole
`robots.txt` was only ~17 bytes — a single odd word (`Wubbalubbadubdub`). A real robots.txt
holds crawler rules, not a lone secret-looking string. **That word was the portal password.**
Lesson: a 200-status file with weird contents is a clue, not a dead end.

### Page source — the username
The login **username** was hidden in an **HTML comment** on the home page (View Source →
`Ctrl+F` "Username"). Developers leave "notes to self" in source and forget they're public.

---

## 2. Access
Logged into the portal with the username (from source) + password (from robots.txt).
Two separate clues, each useless alone — combining them was the point.

---

## 3. Command Execution
The portal had a **command panel** that runs input on the server. That's effectively a shell
through the browser — system-level access via the web app.

Other pages said *"Only the real Rick can view this page."* I initially assumed I needed a
second login — a **rabbit hole**. With command execution I don't need to satisfy the web
app's view restriction at all: I read the data **directly from the filesystem**, bypassing
the UI layer entirely.

> **Lesson:** don't fight a UI restriction when you already have deeper (system) access.

---

## 4. Enumeration → the ingredients

### Ingredient 1 — current directory
```bash
ls -la
cat <ingredient-file>      # found in the web root
```
→ **mr. meeseek hair**

### Ingredient 2 — user's home
```bash
ls -la /home/rick
less "/home/rick/second ingredients"
```
Two gotchas here, both good lessons:
- **`cat` was blocked** on purpose → used `less` instead. Blocking one command is useless
  when `less`/`more`/`head`/`tail`/`grep`/`nl` all read files.
- The filename **had a space** (`second ingredients`). Unquoted, the shell splits it into two
  arguments and the command fails. Fix: **quote it** (`"second ingredients"`) or escape the
  space (`second\ ingredients`).

→ **jerry tear**

### Ingredient 3 — privilege escalation
`/root` is the admin's home — the web user (`www-data`) can't read it. Checked privileges:
```bash
sudo -l
```
→ the low-priv user could run **anything as root with no password** (`NOPASSWD: ALL`) — a
classic misconfiguration. That's the keys to the kingdom:
```bash
sudo ls -la /root
sudo less /root/<ingredient-file>
```
→ **fleeb juice**

---

## Concepts practiced

| Concept | Takeaway |
|---|---|
| Directory brute force | Use `-x` extensions — `login.php` won't show without `-x php` |
| robots.txt | A lone weird string is a credential clue, not crawler rules |
| Source comments | Devs hide usernames/notes in HTML comments |
| Command execution | A command panel = system access through the browser |
| Reading vs downloading | With command exec, read files directly; no need to exfiltrate |
| Blocked commands | `cat` blocked? `less`/`more`/`head`/`grep` still read |
| Spaces in filenames | Quote or escape them |
| `sudo -l` | First privesc check — what can this user run as root? |
| Rabbit holes | vhost enum and "only Rick can view" were dead ends — pick vectors by evidence |

## Key takeaway
A full web kill-chain end to end: recon → creds → foothold → command execution → enumeration
→ privilege escalation. The hardest part wasn't any single exploit — it was **command-line
fluency** (quoting, read-vs-list, command alternatives) and **not chasing rabbit holes**.
Both improve with reps.
