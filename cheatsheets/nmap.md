# Nmap Cheat Sheet

> ⚠️ **Disclaimer — read first.**
> This reference is for **education and authorized testing only**. Scan **only**
> systems you own or have **explicit written permission** to test. Unauthorized
> scanning may be illegal in your jurisdiction. **Use at your own risk and
> responsibility.** The author assumes no liability for misuse.

A quick-reference for the Nmap flags I actually use, organized by *what you're trying
to do* rather than alphabetically. Don't memorize — look it up here or in `man nmap`.

---

## The recon workflow

```
1. Host discovery   →  who is alive?          nmap -sn <range>
2. Port scan        →  what ports are open?   nmap <host>
3. Service/version  →  what runs there?       nmap -sV <host>
4. (OS / scripts)   →  deeper detail          nmap -A <host>
5. Vuln research    →  CVE for service+version (manual)
```

A **closed/filtered** port removes itself from the board — you only research
vulnerabilities for services that are **open and responding**, and the vulnerability
belongs to the **software + version**, not the port number.

---

## Host discovery

| Command | What it does |
|---|---|
| `nmap -sn 192.168.1.0/24` | Ping sweep — list live hosts, no port scan |
| `nmap -sn 192.168.1.0/24` (as root) | Also shows MAC + vendor (OUI) on local segment |
| `nmap -Pn <host>` | Skip discovery, treat host as up (when it blocks pings) |
| `nmap -Pn -sn <host>` | Just resolve/confirm without port scan |

> A host-discovery sweep is **active** recon — you transmit packets to the target.

## Port scanning

| Command | What it does |
|---|---|
| `nmap <host>` | Default: top 1000 TCP ports |
| `nmap -F <host>` | Fast: top 100 ports |
| `nmap -p 80,443,22 <host>` | Specific ports |
| `nmap -p 1-1000 <host>` | Port range |
| `nmap -p- <host>` | **All 65535** TCP ports |
| `nmap -sU <host>` | UDP scan (slow) |
| `nmap --top-ports 20 <host>` | N most common ports |

### Scan types
| Flag | Type | Notes |
|---|---|---|
| `-sS` | SYN "stealth" scan | Default as root; fast, half-open |
| `-sT` | TCP connect scan | Default as non-root; completes handshake |
| `-sU` | UDP scan | Slow but finds UDP services |
| `-sA` | ACK scan | Maps firewall rules (filtered vs unfiltered) |

## Service, version & OS detection

| Command | What it does |
|---|---|
| `nmap -sV <host>` | Service + version detection |
| `nmap -sV --version-intensity 9 <host>` | Max version-detection effort |
| `nmap -O <host>` | OS detection (needs root) |
| `nmap -A <host>` | Aggressive: `-sV` + `-O` + default scripts + traceroute |

> When `-sV` prints raw **service fingerprints** and "unrecognized service" — that's
> not an error. Nmap is asking you to submit it. You can often read the banner (titles,
> versions, kernel strings) by eye anyway.

## Timing & performance

| Flag | Meaning |
|---|---|
| `-T0`–`-T5` | Timing template (0 = paranoid/slow, 5 = insane/fast) |
| `-T4` | Common balance for labs |
| `--min-rate <n>` / `--max-rate <n>` | Packets per second bounds |

## Output

| Flag | Meaning |
|---|---|
| `-oN out.txt` | Normal (human-readable) |
| `-oG out.gnmap` | Greppable |
| `-oX out.xml` | XML |
| `-oA base` | All three formats at once (`base.nmap/.gnmap/.xml`) |
| `-v` / `-vv` | Verbose / more verbose |
| `--reason` | Why Nmap decided each port state |

## Nmap Scripting Engine (NSE)

| Command | What it does |
|---|---|
| `nmap --script default <host>` | Safe default scripts (same as `-sC`) |
| `nmap -sC <host>` | Default scripts shortcut |
| `nmap --script http-title,http-headers <host>` | Specific scripts |
| `nmap --script vuln <host>` | Known-vuln checks (noisy) |
| `nmap --script <category> <host>` | Categories: safe, discovery, auth, vuln, etc. |
| `ls /usr/share/nmap/scripts/` | List installed scripts |

## Firewall / IDS evasion (concept — know the limits)

| Flag | Idea |
|---|---|
| `-f` | Fragment packets (8-byte fragments) |
| `-ff` | 16-byte fragments |
| `--mtu <n>` | Custom fragment size (multiple of 8) |
| `-D ip1,ip2,ME` | Decoys — hide your IP among spoofed sources |
| `-S <ip>` | Spoof source address |
| `--source-port 53` | Masquerade as a trusted port |
| `--data-length <n>` | Pad packets to look innocuous |

> **Reality check:** fragmentation defeats *simple/legacy* filters. Modern IDS/IPS
> (Suricata, Snort, Zeek) **reassemble the stream** before matching rules, so `-f`
> usually doesn't evade them — and abnormal fragmentation can itself flag you. `-f`
> is the "hello world" of evasion, good for understanding the concept.

## Port states (what the output means)

| State | Meaning |
|---|---|
| **open** | A service is listening and responds |
| **closed** | No service, but host replies (RST) |
| **filtered** | No reply — a firewall silently dropped the probe |
| **open\|filtered** | Nmap can't tell (common in UDP) |

---

## Common one-liners

```bash
# Quick situational awareness on a lab box
sudo nmap -sV -sC -oN scan.txt <host>

# Full TCP sweep then version only on what's open
nmap -p- --min-rate 1000 -oG allports.gnmap <host>
nmap -sV -p <comma,separated,open,ports> <host>

# Discover the LAN
sudo nmap -sn 192.168.1.0/24
```

---

*Reference maintained for my own learning. Again: authorized targets only.*
