---
tags:
  - linux
  - files
  - etc
  - identity
aliases:
  - /etc/os-release
  - /usr/lib/os-release
  - os-release
---
`/etc/os-release` says **which Linux distribution and version** this is, in a format scripts can `source` directly. Every systemd-era distro has it, which makes it the standard way to detect the distro.

> [!info] Quick facts
> - Paths: `/etc/os-release`, with `/usr/lib/os-release` as the vendor copy. Usually `/etc/os-release` is a **symlink** to `../usr/lib/os-release`; check `/etc` first, then `/usr/lib`
> - Format: shell-compatible `KEY="value"` lines
> - Shipped by the distro. Don't edit it, upgrades replace it
> - Manual: `man 5 os-release`

# Keys

| Key                | Meaning                                                      | Example (Debian)       | Example (Fedora)   |
| ------------------ | ------------------------------------------------------------ | ---------------------- | ------------------ |
| `NAME`             | Distro name                                                  | `Debian GNU/Linux`     | `Fedora Linux`     |
| `PRETTY_NAME`      | Name for humans, incl. version                               | `Debian GNU/Linux 12 (bookworm)` | `Fedora Linux 41 (Workstation Edition)` |
| `ID`               | Lowercase machine ID. **Use this in scripts**                | `debian`               | `fedora`           |
| `ID_LIKE`          | Distros it is based on / compatible with (space separated)   | *(none)*               | *(none)*           |
| `VERSION_ID`       | Version number. **Missing on rolling releases**              | `12`                   | `41`               |
| `VERSION_CODENAME` | Release codename                                             | `bookworm`             | *(none)*           |
| `BUILD_ID`         | Build identifier                                             | *(none)*               | *(none)*           |
| `HOME_URL`, `BUG_REPORT_URL`, `SUPPORT_URL` | Links                               | `https://www.debian.org/` | `https://fedoraproject.org/` |
| `ANSI_COLOR`       | Color code for the name in terminals                         | *(none)*               | `0;38;2;60;110;180` |
| `LOGO`             | Icon name                                                    | *(none)*               | `fedora-logo-icon` |

> [!tip] Detect a family, not one distro
> Check `ID` **and** `ID_LIKE`. Then your script works on derivatives too (Mint has `ID_LIKE="ubuntu debian"`, Rocky has `ID_LIKE="rhel centos fedora"`).

> [!note] Older systems
> Before `os-release` there were per-distro files: `/etc/lsb-release`, `/etc/debian_version`, `/etc/redhat-release`, `/etc/arch-release`. Some still exist for compatibility. `lsb_release -a` reads them if installed.

# Scripting recipes

### Source it in a subshell (no variables leak into your script)
```bash
( . /etc/os-release; echo "$ID $PRETTY_NAME like=$ID_LIKE" )
# debian Debian GNU/Linux 12 (bookworm) like=
```

### Portable path lookup
```bash
f=/etc/os-release; [[ -r $f ]] || f=/usr/lib/os-release
```

### Branch by distro family
```bash
. /etc/os-release
case " $ID $ID_LIKE " in
  *" debian "*|*" ubuntu "*) family=debian ;;
  *" fedora "*|*" rhel "*)   family=redhat ;;
  *" arch "*)                family=arch ;;
  *" suse "*)                family=suse ;;
  *)                         family=unknown ;;
esac
echo "$family"
```

### One value without sourcing
```bash
grep -oP '^ID=\K.*' /etc/os-release | tr -d '"'
```

### Everything at once
```bash
hostnamectl     # OS, kernel, hardware model
```

# See also
- [[hostname]] · [[machine-id]] — the other identity files
- [[issue-motd]] — `/etc/issue` often prints `PRETTY_NAME` at login
- [[etc]] · [[usr]]
