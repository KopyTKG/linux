---
tags:
  - linux
  - files
  - etc
  - users
  - security
aliases:
  - /etc/shadow
  - shadow file
  - password hashes
---
`/etc/shadow` holds the **password hashes** and **password aging** rules for every account in [[passwd]]. It is split out of `passwd` because `passwd` must be readable by everyone, and hashes must not be.

> [!info] Quick facts
> - Path: `/etc/shadow` · owner `root:root` · mode `0600` (or `0640` group `shadow` on some systems)
> - A normal user **cannot read it**: `cat /etc/shadow` → `Permission denied`
> - Programs that need it (`login`, `su`, `sudo`, `passwd`) are root or setuid root, and go through [[pam-d]]
> - Edit with `passwd`, `chage`, `usermod`, or `sudo vipw -s`. Never with a plain editor
> - Manual: `man 5 shadow`

> [!danger] Never paste real lines from this file anywhere
> Anyone with the hash can try to crack it offline at full speed. Examples below are fake.

# Format
Nine fields separated by `:`. Numbers are in **days since 1 January 1970**.

```
alice:$y$j9T$Fq3...$Xk8...:20730:0:99999:7:14::
```

| #   | Field        | Example       | Meaning                                                                  |
| --- | ------------ | ------------- | ------------------------------------------------------------------------ |
| 1   | name         | `alice`       | Must match a name in [[passwd]]                                          |
| 2   | password     | `$y$j9T$...`  | Hash, or a lock marker (table below)                                     |
| 3   | lastchg      | `20730`       | Day the password was last changed. `0` = must change at next login       |
| 4   | min          | `0`           | Days before the user may change it again                                 |
| 5   | max          | `99999`       | Days until it must be changed. `99999` ≈ never                           |
| 6   | warn         | `7`           | Warn this many days before it expires                                    |
| 7   | inactive     | `14`          | Days after expiry the password still works (then account is locked)      |
| 8   | expire       | *(empty)*     | Day the **account** expires. Empty = never                               |
| 9   | reserved     | *(empty)*     | Unused                                                                   |

# The password field

### Hash format
A modern hash looks like `$id$params$salt$hash`. The `id` tells you the algorithm.

| Prefix      | Algorithm         | Notes                                                       |
| ----------- | ----------------- | ----------------------------------------------------------- |
| `$y$`       | yescrypt          | Current default on most distros. Slow on purpose = good     |
| `$gy$`      | gost-yescrypt     | Rare                                                        |
| `$7$`       | scrypt            | Rare                                                        |
| `$2b$`/`$2y$` | bcrypt          | Common on BSDs, some Linux setups                           |
| `$6$`       | SHA-512 crypt     | Old default. Still OK with high rounds                      |
| `$5$`       | SHA-256 crypt     | Old                                                         |
| `$1$`       | MD5 crypt         | **Broken**. Change these passwords                          |
| no `$`, 13 chars | DES crypt    | **Ancient and broken**                                      |

The default algorithm for new passwords is `ENCRYPT_METHOD` in [[login-defs]] (here `YESCRYPT`).

### Special values

| Value             | Meaning                                                               |
| ----------------- | --------------------------------------------------------------------- |
| *(empty)*         | **No password**: anyone can log in. Dangerous                         |
| `!` + hash        | Password **locked** (`passwd -l`). Hash kept, unlock with `passwd -u` |
| `!` or `!!` alone | No password set yet / locked                                          |
| `*`               | Password login impossible. Typical for system accounts                |

> [!note] Locked password ≠ locked account
> `!` only blocks **password** login. The user may still get in with an SSH key. To block everything, also set an expire date: `usermod -e 1 alice`, or set the shell to `nologin`.

# Day math
All dates are days since epoch. Today as a day number:

```bash
echo $(( $(date +%s) / 86400 ))          # 20730
date -d "@$(( 20730 * 86400 ))" +%F      # day number → date
echo $(( ( $(date -d 2027-01-01 +%s) ) / 86400 ))   # date → day number
```

Password expires on day `lastchg + max`. Warnings start at `lastchg + max - warn`. Account gets locked at `lastchg + max + inactive`.

Example: `lastchg=20730`, `max=90`, `warn=7`, `inactive=14`
- warnings from day 20813
- password expires day 20820
- account locked day 20834

`chage -l <user>` prints all of this as real dates (your own user works without root).

```bash
chage -l alice
```

# Commands that edit it

| Command                     | Effect                                         |
| --------------------------- | ---------------------------------------------- |
| `passwd`                    | Change your own password                       |
| `sudo passwd alice`         | Set someone else's password                    |
| `sudo passwd -l / -u alice` | Lock / unlock password                         |
| `sudo passwd -e alice`      | Force change at next login (sets `lastchg=0`)  |
| `sudo chage -M 90 -W 7 alice` | Max age 90 days, warn 7 days before          |
| `sudo chage -E 2027-01-01 alice` | Account expires on that date              |
| `sudo vipw -s`              | Edit safely (lock + syntax check)              |
| `sudo pwck`                 | Check passwd/shadow consistency                |

# Scripting recipes
These need root (run with `sudo`).

### Accounts with an empty password (should print nothing)
```bash
sudo awk -F: '$2=="" {print $1}' /etc/shadow
```

### Which hash algorithm each user has
```bash
sudo awk -F: '$2 ~ /^\$/ { split($2, p, "$"); print $1, p[2] }' /etc/shadow
# alice y
```

### Users whose password is older than 365 days
```bash
today=$(( $(date +%s) / 86400 ))
sudo awk -F: -v t="$today" '$2 ~ /^\$/ && $3 > 0 && t - $3 > 365 {print $1, t - $3 " days"}' /etc/shadow
```

### Generate a hash without changing anything
```bash
openssl passwd -6           # SHA-512, asks for password
mkpasswd -m yescrypt        # if mkpasswd (whois package) is installed
```

# See also
- [[passwd]] — the public part of the user database
- [[gshadow]] — the same idea for groups
- [[login-defs]] — default hash algorithm and aging
- [[pam-d]] — how login actually checks the hash
- [[sudoers]] · [[etc]]
