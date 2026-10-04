---
tags:
  - linux
  - files
  - etc
  - locale
aliases:
  - /etc/locale.conf
  - locale.conf
  - /etc/locale.gen
  - locale.gen
---
`/etc/locale.conf` sets the **system-wide locale**: language of messages, and how dates, numbers, money and paper sizes are formatted. A locale is a name like `en_US.UTF-8` = language `en`, country `US`, character set `UTF-8`.

> [!info] Quick facts
> - Path: `/etc/locale.conf` · mode `0644` · `KEY=value` lines
> - Read at login by systemd/PAM and turned into environment variables (`LANG`, `LC_*`)
> - Change with `sudo localectl set-locale LANG=en_US.UTF-8 LC_TIME=de_DE.UTF-8`
> - Debian/Ubuntu historically use `/etc/default/locale` with the same format
> - Manual: `man 5 locale.conf`, `man 7 locale`

# Example
```
LANG=en_US.UTF-8
LC_NUMERIC=de_DE.UTF-8
LC_TIME=de_DE.UTF-8
LC_MONETARY=de_DE.UTF-8
LC_PAPER=de_DE.UTF-8
LC_MEASUREMENT=de_DE.UTF-8
...
```
English messages, German formats. That's why `ls -l` shows dates like `4. Okt 17:36`.

# Variables

| Variable          | Controls                                      | `en_US` vs `de_DE`               |
| ----------------- | --------------------------------------------- | -------------------------------- |
| `LANG`            | Default for everything not set below          |                                  |
| `LC_MESSAGES`     | Language of program messages                  | `No such file` / `Adresář nebo soubor neexistuje` |
| `LC_TIME`         | Date and time format                          | `10/04/2026` / `4.10.2026`      |
| `LC_NUMERIC`      | Decimal separator                             | `3.14` / `3,14`                  |
| `LC_MONETARY`     | Currency                                      | `$` / `Kč`                       |
| `LC_COLLATE`      | Sort order (`ls`, `sort`)                     | `ch` sorts after `h` in German    |
| `LC_CTYPE`        | Which characters are letters, upper/lower case |                                 |
| `LC_PAPER`        | Default paper size                            | Letter / A4                      |
| `LC_MEASUREMENT`  | Metric or imperial                            |                                  |
| `LC_ALL`          | **Overrides all** of the above. Don't set it permanently, use it for one command |  |

Priority: `LC_ALL` > `LC_xxx` > `LANG`.

> [!warning] Locale breaks scripts
> Parsing command output in a script? A user with German `LC_NUMERIC` gets `3,14` instead of `3.14`, and translated messages break `grep`. Put this at the top of scripts that parse output:
> ```bash
> export LC_ALL=C
> ```
> `C` (or `C.UTF-8`) = plain English, ASCII sorting, `.` as decimal point, stable everywhere.

# Locales must exist first
A locale only works if its data is compiled on the system. List them with `locale -a`:
```
C
C.utf8
de_DE.utf8
en_US.utf8
POSIX
```
Setting `LANG` to a locale that isn't installed gives warnings like `setlocale: LC_ALL: cannot change locale` and falls back to `C`.

### `/etc/locale.gen` (only some distros)
On distros that compile locales locally (Arch, Gentoo, Debian with `locales` package) you pick them in `/etc/locale.gen` and run `locale-gen`:
```
de_DE.UTF-8 UTF-8      ← uncommented = generated
#fr_FR.UTF-8 UTF-8     ← commented = not generated
```
Other distros ship ready locales as packages (Fedora `glibc-langpack-*`, Ubuntu `language-pack-*`) or use `dpkg-reconfigure locales`.

# Scripting recipes

### Current effective locale
```bash
locale
```

### What locale.conf sets (in a subshell)
```bash
( . /etc/locale.conf; echo "LANG=$LANG LC_TIME=$LC_TIME" )
```

### Run one command in another locale
```bash
LC_ALL=C ls -l             # English month names
LC_ALL=de_DE.UTF-8 date    # German date
LC_TIME=en_GB.UTF-8 date   # only time format changes
```

### Is a locale installed?
```bash
locale -a | grep -qix 'de_DE.utf8' && echo installed
```

### Stable numbers in a script
```bash
LC_ALL=C printf '%.2f\n' 3.14159
```

# See also
- [[vconsole-conf]] — keyboard layout and font of the text console
- [[localtime]] — time zone (not part of locale!)
- [[environment]] · [[profile]] — other places that set variables
- [[etc]]
