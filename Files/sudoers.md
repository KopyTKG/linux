---
tags:
  - linux
  - files
  - etc
  - security
aliases:
  - /etc/sudoers
  - /etc/sudoers.d
  - sudoers.d
  - visudo
---
`/etc/sudoers` decides **who may run what as whom** through `sudo`. One wrong line can lock you out of root, so it has its own editor, `visudo`.

> [!info] Quick facts
> - Path: `/etc/sudoers` · owner `root:root` · mode `0440` (read-only even for root, on purpose)
> - Drop-in folder: `/etc/sudoers.d/` · mode `0750`. Put your own rules here, not in the main file
> - Readable only by root. As a user, see **your** rights with `sudo -l`
> - Manual: `man 5 sudoers`, `man 8 visudo`

> [!danger] Always edit with `visudo`
> `sudo visudo` (main file) or `sudo visudo -f /etc/sudoers.d/50-myrules`.
> `visudo` locks the file and **refuses to save broken syntax**. A typo saved with a normal editor means `sudo` stops working for everyone: then you need a root shell, `su`, or a rescue boot to fix it.
> Pick the editor with `sudo EDITOR=nano visudo`.

# Rule format

```
who   where = (as_user:as_group)   [TAGS:]   commands
```

| Part        | Example            | Meaning                                         |
| ----------- | ------------------ | ----------------------------------------------- |
| who         | `alice`, `%wheel`  | User, or `%group`                               |
| where       | `ALL`              | Hostname the rule applies on (`ALL` almost always) |
| as_user     | `(ALL)`            | Users you may become. `(ALL:ALL)` = any user and group |
| tags        | `NOPASSWD:`        | Options for this rule                           |
| commands    | `ALL`, `/usr/bin/systemctl restart nginx` | Full paths, comma separated |

### Typical lines
```
root    ALL=(ALL:ALL) ALL                       # root can do anything
%wheel  ALL=(ALL:ALL) ALL                       # members of wheel, with password
%sudo   ALL=(ALL:ALL) ALL                       # same idea on Debian/Ubuntu
alice   ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx
bob     ALL=(www-data) /usr/bin/php              # bob may run php only as www-data
@includedir /etc/sudoers.d                      # read drop-in folder
```

On most systems the admin rule is **`%wheel`** or **`%sudo`**: you get sudo by being in that [[group]]. Example: `wheel:x:998:alice`.

# Useful tags and Defaults

| Setting                                  | Effect                                                      |
| ---------------------------------------- | ----------------------------------------------------------- |
| `NOPASSWD:`                              | No password prompt for that rule                            |
| `Defaults timestamp_timeout=15`          | Minutes sudo remembers your password (`0` = always ask)     |
| `Defaults env_reset`                     | Clean environment for the command (default, keep it)        |
| `Defaults env_keep += "EDITOR"`          | Keep a variable through sudo                                |
| `Defaults secure_path="..."`             | `PATH` used for commands run by sudo                        |
| `Defaults insults`                       | Funny message on wrong password                             |
| `Defaults pwfeedback`                    | Show `*` while typing the password                          |
| `Defaults:alice !requiretty`             | Defaults for one user                                       |
| `Defaults logfile=/var/log/sudo.log`     | Extra log file (journal logs it anyway)                     |

> [!warning] Security gotchas
> - `NOPASSWD: ALL` = any program that runs as you is root. Keep `NOPASSWD` to exact commands.
> - Allowing editors, `less`, `find`, `vim`, `tar`, `python`... as root lets the user spawn a root shell from inside (see GTFOBins). Use `sudoedit` for editing files.
> - Command arguments: `/usr/bin/systemctl restart nginx` allows exactly that. `/usr/bin/systemctl` alone allows **every** systemctl action.
> - Use full paths. A relative command name could be any binary in `PATH`.

# `/etc/sudoers.d/`
- Every file in the folder is read in **alphabetical order**. Later rules win when two match.
- Files with a `.` in the name or ending in `~` are **ignored** (so `rules.conf` is skipped!). Use names like `50-alice`.
- Mode should be `0440`.

```bash
# create a drop-in safely
sudo visudo -f /etc/sudoers.d/50-nginx
# check every file's syntax
sudo visudo -c
```

# Scripting recipes

### What am I allowed to run?
```bash
sudo -l
```

### Can I sudo without a password right now? (for scripts)
```bash
if sudo -n true 2>/dev/null; then echo "sudo works without prompt"; fi
```

### Ask for the password once at script start, keep it alive
```bash
sudo -v || exit 1
while true; do sudo -n true; sleep 60; kill -0 "$$" || exit; done 2>/dev/null &
```

### Who is in the admin group?
```bash
getent group wheel sudo | cut -d: -f1,4
```

### Re-run the script as root if it isn't
```bash
[[ $EUID -eq 0 ]] || exec sudo "$0" "$@"
```

### Recent sudo use from the journal
```bash
journalctl _COMM=sudo --since today | grep COMMAND
```

# See also
- [[group]] — `wheel` / `sudo` membership
- [[pam-d]] — `/etc/pam.d/sudo` handles the password check
- [[shadow]] · [[etc]] · [[journal]]
