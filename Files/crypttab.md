---
tags:
  - linux
  - files
  - etc
  - disks
  - security
aliases:
  - /etc/crypttab
  - crypttab
---
`/etc/crypttab` lists **encrypted block devices** (LUKS, mostly) to unlock at boot. Each line creates a decrypted device `/dev/mapper/<name>`, which [[fstab]] can then mount. It is like fstab, one step earlier.

> [!info] Quick facts
> - Path: `/etc/crypttab` · mode usually `0600` (may contain key file paths)
> - Read by `systemd-cryptsetup-generator`: each line becomes `systemd-cryptsetup@<name>.service`
> - Often exists only as a commented template when no disks are encrypted
> - The **root** filesystem is usually unlocked earlier, from the initramfs, using kernel parameters (`rd.luks.uuid=...`) or a copy of crypttab inside the initramfs
> - Manual: `man 5 crypttab`

# Format
Four columns:

```
# name     device                                     key        options
cryptdata  UUID=1b2c3d4e-0000-4000-8000-123456789abc  none       luks,discard
cryptswap  /dev/disk/by-partuuid/aaaa-bbbb            /dev/urandom  swap,cipher=aes-xts-plain64,size=512
backup     UUID=...                                   /etc/keys/backup.key  luks,nofail
```

| # | Column  | Meaning                                                                          |
| - | ------- | -------------------------------------------------------------------------------- |
| 1 | name    | Result appears as `/dev/mapper/<name>`                                           |
| 2 | device  | Encrypted device: `UUID=` of the **LUKS container**, `PARTUUID=`, or a path     |
| 3 | key     | `none` / `-` = ask for passphrase. Path = key file. `/dev/urandom` = random key every boot |
| 4 | options | Comma separated                                                                 |

### Options

| Option              | Meaning                                                           |
| ------------------- | ----------------------------------------------------------------- |
| `luks`              | Device is LUKS (default for most)                                 |
| `discard`           | Pass TRIM to SSD. Faster, leaks which blocks are free             |
| `nofail`            | Boot continues if device is missing                               |
| `noauto`            | Don't unlock at boot                                              |
| `swap` / `tmp`      | Format as swap / filesystem after unlocking (random-key setups)   |
| `tpm2-device=auto`  | Unlock with the TPM chip (`systemd-cryptenroll`)                  |
| `fido2-device=auto` | Unlock with a FIDO2 security key                                  |
| `timeout=30s`       | Give up asking after that long                                    |

> [!warning] Key files
> A key file on the **unencrypted** part of the disk protects nothing. Keep key files inside an already-unlocked encrypted filesystem, with mode `0400` root.

> [!note] Random-key swap breaks hibernation
> `/dev/urandom` swap gets a new key each boot, so a hibernation image in it can never be read back. Use a real LUKS swap for hibernate/resume.

# Scripting recipes

### Encrypted devices on the system (no root needed)
```bash
lsblk -o NAME,FSTYPE,TYPE,MOUNTPOINTS | grep -E 'crypto_LUKS|crypt'
```

### Is this device LUKS?
```bash
sudo cryptsetup isLuks /dev/nvme0n1p2 && echo yes
```

### Status of an unlocked mapping
```bash
sudo cryptsetup status cryptdata
ls -l /dev/mapper/
```

### Units generated from crypttab
```bash
systemctl list-units 'systemd-cryptsetup@*'
```

# See also
- [[fstab]] — mounts `/dev/mapper/<name>` afterwards
- [[dev]] — `/dev/mapper`, `/dev/disk/by-uuid`
- [[boot]] — initramfs unlocks the root disk
- [[systemd]] · [[etc]]
