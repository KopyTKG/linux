---
tags:
  - linux
  - files
  - etc
  - login
aliases:
  - /etc/issue
  - /etc/motd
  - /etc/issue.net
  - motd
  - issue
---
Two small text files that print **login messages**:

| File            | Shown                                      | Typical content                         |
| --------------- | ------------------------------------------ | --------------------------------------- |
| `/etc/issue`    | **Before** login, on text consoles (`tty`) | OS name, kernel, terminal, IP address   |
| `/etc/issue.net`| Before login over telnet / SSH (if `Banner` is set in [[ssh-config]]) | Legal warning |
| `/etc/motd`     | **After** a successful login               | "Message of the day": news, rules, hints |

> [!info] Quick facts
> - Both are plain text, mode `0644`
> - `/etc/issue` is printed by `agetty`, which can expand **escape codes** (below)
> - `/etc/motd` is printed by `pam_motd` (or by `sshd` when `PrintMotd yes`). No escape codes, just text
> - Example: `/etc/issue` = `\S{PRETTY_NAME} \r (\l)`; `/etc/motd` often does not exist
> - Manual: `man 5 issue`, `man 8 agetty`, `man 5 motd`, `man 8 pam_motd`

# `/etc/issue` escape codes
The line `\S{PRETTY_NAME} \r (\l)` prints something like:
```
Debian GNU/Linux 12 (bookworm) 6.12.8 (tty3)
```

| Code               | Expands to                                             |
| ------------------ | ------------------------------------------------------ |
| `\S` / `\S{KEY}`   | `PRETTY_NAME` / any key from [[os-release]]            |
| `\n`               | Hostname → [[hostname]]                                |
| `\r`               | Kernel release (`uname -r`)                            |
| `\v`               | Kernel version (build date)                            |
| `\m`               | Architecture (`x86_64`)                                |
| `\l`               | Terminal name (`tty3`)                                 |
| `\d`, `\t`         | Date, time                                             |
| `\u` / `\U`        | Number of logged-in users                              |
| `\4` / `\4{enp3s0}`| IPv4 address (of an interface)                         |
| `\6`               | IPv6 address                                           |
| `\e{green}` ... `\e{reset}` | Colors (`\e{bold}`, `\e{red}`, ...)          |
| `\\`               | A literal backslash                                    |

> [!tip] Show the IP on a headless box
> Add `IP: \4` to `/etc/issue`. Plug in a screen, and you see the address to SSH into without logging in.

> [!note] Extra drop-ins
> Newer `agetty` also reads `/etc/issue.d/*.issue`, and `pam_motd` reads `/etc/motd.d/` and `/run/motd.d/`. Ubuntu builds a dynamic motd from scripts in `/etc/update-motd.d/` (that's where its "updates available" text comes from).

# Hiding the messages
- `touch ~/.hushlogin` → no motd and no "last login" line for you ([[login-defs]] `HUSHLOGIN_FILE`).
- `PrintMotd no` in `sshd_config` → `sshd` doesn't print it (PAM may still).

> [!warning] Don't leak information
> `/etc/issue.net` is shown to anyone who connects, before authentication. Don't print kernel versions or hostnames there, only a legal notice.

# Scripting recipes

### Preview `/etc/issue` with codes expanded
```bash
agetty --show-issue 2>/dev/null || cat /etc/issue
```

### Write a motd
```bash
printf 'Backups run nightly at 02:00.\nAsk admin@example.com for access.\n' | sudo tee /etc/motd
```

### Fake the issue line in bash (for your own banner script)
```bash
. /etc/os-release
echo "$PRETTY_NAME $(uname -r) ($(basename "$(tty)"))"
```

### Is the motd printed by PAM?
```bash
grep -l pam_motd /etc/pam.d/* 2>/dev/null
```

# See also
- [[os-release]] · [[hostname]] — values used by `\S` and `\n`
- [[pam-d]] · [[ssh-config]] · [[login-defs]]
- [[dev-special-files]] — `/dev/tty*`
- [[etc]]
