# Home Network Recon & Attack-Surface Audit — Lab Notes

> **Scope & ethics:** Every step below was performed against my *own* equipment on a
> network I own, in an isolated lab setup. No third-party systems were touched.
> All IP addresses, hostnames, and device identifiers are redacted / replaced with
> placeholders (`<PUBLIC_IP>`, `192.168.X.X`).

## Goal

Map my own home network the way an attacker (and a defender) would:
1. Understand what my network exposes to the *outside* (external attack surface).
2. Map what lives *inside* the LAN (internal attack surface).
3. Practice the recon workflow end-to-end: discovery → port scan → service/version detection.

---

## Setup

- Attacker box: Kali Linux running as a VM.
- Key learning point up front: **know where your traffic exits before you scan anything.**
  If you don't know which network you're egressing through, you risk scanning something
  that isn't yours.

---

## Step 1 — Where am I egressing from?

Two different things I had to separate:

**Internal IP of the VM:**
```bash
ip a
```
The VM first showed a `10.0.2.15/24` address — the classic **VirtualBox NAT** range.
That means the VM was behind VirtualBox's own NAT, isolated from the real LAN.

**Default gateway:**
```bash
ip route | grep default
# default via 10.0.2.2 ...   <-- VirtualBox virtual router, NOT the home router
```

**Public IP as the world sees me:**
```bash
curl -4 ifconfig.me
```

### Lesson: NAT vs. Bridged
In **NAT** mode the VM sits on VirtualBox's private network and cannot see other devices
on the real LAN. To audit the actual home network I switched the adapter to **Bridged**,
so the VM pulls an IP directly from the home router:

```bash
ip a            # eth0 now 192.168.X.X/24  (real LAN)
ip route        # default via 192.168.X.1  (real home router)
```

Now the VM is a first-class citizen on the LAN — it can see (and be seen by) every device.

---

## Step 2 — External attack surface

Scanned my own public IP:

```bash
nmap -F <PUBLIC_IP>       # top 100 ports
nmap -sV <PUBLIC_IP>      # top 1000 + version detection
```

**Result:** all ports `filtered` (no-response) — top 100 *and* top 1000.

### Understanding port states
| State | Meaning |
|---|---|
| **open** | A service is listening and responds. |
| **closed** | No service, but the host replies (RST). |
| **filtered** | No reply at all — a firewall silently drops the probe. |

`filtered` across the board = the router exposes **nothing** to the internet and doesn't even
advertise itself. From a defensive standpoint this is exactly what you want: an attacker doing
recon on the address gets a wall of silence with nothing to start from.

> ⚠️ **Caveat I noted:** scanning my public IP *from inside* the LAN (behind NAT) does not
> perfectly reflect what the world sees — hairpin NAT / router behavior can affect results.
> A true external scan needs a host outside the network (e.g. a cheap VPS).

---

## Step 3 — Internal host discovery

With the VM bridged onto the LAN, I mapped who's alive:

```bash
nmap -sn 192.168.X.0/24     # ping sweep, no port scan
```

This returns live hosts plus MAC addresses, and Nmap often maps the MAC prefix (OUI) to a
**vendor** — which is enough to guess what each device is *before scanning a single port*:

| Device (by vendor OUI) | Likely role |
|---|---|
| Router vendor | Home router / gateway |
| PC motherboard vendor | Desktop / host machine |
| Consumer electronics vendor | Phone / smart TV |
| Set-top / TV box vendor | Media streamer |
| IoT platform vendors (×2) | Smart-home devices (bulbs / switches) |

### Lesson: cheap intel before you touch a port
Vendor identification from a single sweep hands you a lot of intel — "there's an IoT-platform
device here" immediately tells a tester which known platform vulnerabilities to research,
before any port scanning.

> **Terminology note:** a host-discovery sweep is still **active** recon — you're
> transmitting packets to the target. True *passive* recon touches the target zero times
> (e.g. Shodan/Censys lookups, reading public SSL certificates, OSINT). OUI vendor mapping
> is cheap and early, but it is not passive.

---

## Step 4 — Service/version detection on the router

The router is the most interesting target on any home network, and it's built to withstand
scanning, so I picked it (not the fragile IoT devices) as the deep-dive target:

```bash
nmap -sV 192.168.X.1
```

**Findings (LAN-side):**

| Port | State | Service |
|---|---|---|
| 53 | open | DNS (internal resolver) |
| 80 | open | HTTP — router admin web UI |
| 135 | closed | msrpc |
| 139 | closed | netbios-ssn |
| 443 | open | HTTPS — admin UI (encrypted) |
| 445 | open | SMB (microsoft-ds) |
| 49152 | open | **UPnP** — Portable SDK for UPnP; leaked kernel: Linux 4.19.x |

### Reading a "messy" `-sV` output
Nmap couldn't fingerprint several services and dumped raw **service fingerprints** with a
"submit this" message. That's **not an error** — it's Nmap asking for help improving its DB.
Even inside the raw banner you can read useful intel by eye:
- `<title>Gateways</title>` — the admin page title
- `js/jquery.min.js` — front-end stack
- UPnP banner leaked the **kernel version** — great for a version→CVE research exercise

### Notes on the findings
- **UPnP exposed** is worth attention — it's the classic mechanism by which devices punch
  their own port-forwards. Worth auditing / disabling on the router.
- **Port 445 (SMB)** open on a router is slightly unusual (often a USB file-share feature).
  SMB has a heavy vulnerability history (see the [Blue](thm-blue.md) write-up), so it's a
  noteworthy line item even if low-risk internally.
- All of the above are exposed **only to the LAN** — externally everything is `filtered`.
  Open inside, sealed outside = the correct posture.

---

## Recon workflow (the takeaway)

```
1. Host discovery        →  who is alive?          (nmap -sn)
2. Port scan             →  what ports are open?   (nmap -F / -p-)
3. Service/version       →  what runs there?       (nmap -sV)
4. Vuln research         →  known CVE for that service+version?
5. (lab only) exploit
```

A **closed/filtered** port removes itself from the board — you only research vulnerabilities
for services that are **open and responding**, and the vulnerability belongs to the
**software + version**, not the port number. Port 80 being open is just an invitation;
`-sV` tells you *who* you invited.

## Concepts practiced

| Concept | Notes |
|---|---|
| NAT vs. Bridged networking | NAT isolates the VM; bridged puts it on the real LAN |
| External vs. internal attack surface | Router: sealed outside, service-rich inside |
| Port states | open / closed / filtered and what each implies |
| Passive vs. active recon | Sweeps/OUI mapping are active; passive = zero-touch (Shodan, cert/OSINT) |
| Reading raw `-sV` fingerprints | "Unrecognized service" is not an error — read the banner by eye |
| Hairpin NAT caveat | Scanning your public IP from inside ≠ a true external scan |

## Key takeaway

The most valuable result here was **boring on purpose**: nothing exposed externally.
Good home-network hygiene looks like silence from the outside and a well-understood,
minimal surface on the inside. Knowing *why* each open port on the router exists — and
being able to justify or close it — is the whole exercise.
