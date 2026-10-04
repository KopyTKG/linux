---
tags:
  - linux
  - files
  - etc
  - time
aliases:
  - /etc/localtime
  - localtime
  - timezone
  - zoneinfo
---
`/etc/localtime` defines the **system time zone**. It is a **symlink** into the time zone database `/usr/share/zoneinfo/`, which knows every zone's UTC offset and daylight saving rules back through history.

> [!info] Quick facts
> - Path: `/etc/localtime` → `/usr/share/zoneinfo/Europe/Berlin` in this example
> - The target is a **binary** TZif file, not text. Don't `cat` it
> - Change with `sudo timedatectl set-timezone Europe/Berlin`
> - Debian also keeps the name in `/etc/timezone` (text) for old tools
> - Manual: `man 5 localtime`, `man 1 timedatectl`, `man 5 tzfile`

```bash
ls -l /etc/localtime
# /etc/localtime -> /usr/share/zoneinfo/Europe/Berlin
```

# How time works on Linux
- The kernel keeps time in **UTC**. Always.
- The time zone is only used to **display** time. Two users with different `TZ` see different clocks at the same moment.
- The hardware clock (RTC) should also be UTC. `RTC in local TZ: no` in `timedatectl` = good. Dual-boot with Windows is the one case where people set it to local time.

```
timedatectl
               Local time: Sun 2026-10-04 17:49:55 CEST
           Universal time: ne 2026-10-04 15:49:55 UTC
                 RTC time: ne 2026-10-04 15:49:56
                Time zone: Europe/Berlin (CEST, +0200)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

| Line                        | Meaning                                                |
| --------------------------- | ------------------------------------------------------ |
| Local time                  | UTC + zone offset                                      |
| RTC time                    | Clock on the motherboard                               |
| Time zone                   | Zone name, current abbreviation, offset                |
| System clock synchronized   | NTP corrected the clock                                |
| NTP service                 | `systemd-timesyncd` / `chronyd` running                |

> [!tip] `TZ` overrides per process
> `TZ=Asia/Tokyo date` shows Tokyo time without changing anything. Users can also set `TZ` in their shell profile.

> [!warning] Use zone names, not abbreviations
> `CET`, `EST`, `IST` are ambiguous (IST = India, Ireland or Israel) and don't follow DST changes. Use `Region/City` like `Europe/Berlin`.

# Scripting recipes

### Current zone name
```bash
readlink /etc/localtime | sed 's|.*/zoneinfo/||'          # Europe/Berlin
timedatectl show -p Timezone --value
```

### Time in another zone
```bash
TZ=America/New_York date '+%F %H:%M %Z'
```

### Find a zone
```bash
timedatectl list-timezones | grep -i prague
```

### Offset right now
```bash
date +%Z%z         # CEST+0200
```

### Log timestamps that never confuse anyone
```bash
date -u +%FT%TZ    # 2026-10-04T15:49:55Z  (UTC, ISO 8601)
```

### When does DST change this year?
```bash
zdump -v Europe/Berlin | grep "$(date +%Y)"
```

# See also
- [[locale-conf]] — date **format** (not zone)
- [[usr]] — `/usr/share/zoneinfo`
- [[timers]] · [[systemd]] — timers use local time unless told otherwise
- [[etc]]
