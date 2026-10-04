---
tags:
  - linux
  - files
  - etc
  - locale
aliases:
  - /etc/vconsole.conf
  - vconsole.conf
---
`/etc/vconsole.conf` sets the **keyboard layout and font of the text console**: the black screen you get with `Ctrl+Alt+F3`, in rescue mode, or while typing a disk password at boot. It does not affect your desktop, which has its own layout settings.

> [!info] Quick facts
> - Path: `/etc/vconsole.conf` · mode `0644` · `KEY=value`
> - Read by `systemd-vconsole-setup` at boot, and copied into the initramfs (so the layout also works for the boot password)
> - Change with `sudo localectl set-keymap us` and `sudo localectl set-x11-keymap us`
> - Debian/Ubuntu historically use `/etc/default/keyboard` + `/etc/default/console-setup` instead
> - Manual: `man 5 vconsole.conf`, `man 1 localectl`

# Example
```
KEYMAP=us
XKBLAYOUT=us
```

# Keys

| Key            | Example          | Meaning                                                     |
| -------------- | ---------------- | ----------------------------------------------------------- |
| `KEYMAP`       | `us`, `de-latin1`, `fr` | Console keymap name                           |
| `KEYMAP_TOGGLE`| `us`             | Second keymap to switch to                                  |
| `FONT`         | `ter-v22n`, `eurlatgr` | Console font. Bigger fonts help on 4K screens        |
| `FONT_MAP`     |                  | Console map (rarely needed)                                 |
| `XKBLAYOUT`    | `us`, `de`       | Layout for X11/Wayland when nothing else sets it            |
| `XKBMODEL`, `XKBVARIANT`, `XKBOPTIONS` | `pc105`, `qwerty`, `grp:alt_shift_toggle` | Detail for X11 layout |

> [!tip] Test before saving
> `sudo loadkeys de-latin1` and `sudo setfont ter-v22n` change the running console immediately and are lost at reboot. When happy, write the values into the file.

> [!warning] Disk encryption password
> If the console keymap differs from the layout you used when setting a LUKS password, typing it at boot fails. Keep `KEYMAP` matching, and rebuild the initramfs after changing it (the command differs by distro).

# Scripting recipes

### Available keymaps and fonts
```bash
localectl list-keymaps | grep -i de
ls /usr/share/kbd/consolefonts 2>/dev/null || ls /usr/share/consolefonts
```

### Current settings
```bash
localectl status
```

### Read one key
```bash
( . /etc/vconsole.conf; echo "$KEYMAP" )
```

# See also
- [[locale-conf]] — language and formats
- [[crypttab]] · [[boot]] — where the console layout matters most
- [[dev-special-files]] — `/dev/tty*` consoles
- [[etc]] · [[systemd]]
