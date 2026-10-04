---
tags:
  - linux
  - files
  - etc
  - network
  - users
aliases:
  - /etc/nsswitch.conf
  - nsswitch.conf
  - NSS
---
`/etc/nsswitch.conf` (Name Service Switch) tells glibc **where to look things up**: users, groups, hostnames, services. For each database it lists sources in order. This is why `getent` is the right way to query users and hosts: it follows this file, while reading [[passwd]] or [[hosts]] directly skips every other source.

> [!info] Quick facts
> - Path: `/etc/nsswitch.conf` · mode `0644`
> - Format: `database: source1 source2 ...`
> - Sources are glibc plugins: `/usr/lib/libnss_<source>.so.2`
> - Read when a program starts its first lookup. Long-running daemons may need a restart after changes
> - Manual: `man 5 nsswitch.conf`

# Example (systemd-based system)
```
passwd:    files systemd
group:     files [SUCCESS=merge] systemd
shadow:    files systemd
gshadow:   files systemd
hosts:     mymachines resolve [!UNAVAIL=return] files myhostname dns
networks:  files
protocols: files
services:  files
```

# Databases

| Database     | Used by                                 | `files` means         |
| ------------ | --------------------------------------- | --------------------- |
| `passwd`     | user lookups (`ls -l`, `id`, `ps`)      | [[passwd]]            |
| `group`      | group lookups                           | [[group]]             |
| `shadow`     | password checks                         | [[shadow]]            |
| `hosts`      | hostname → IP (`ping`, `curl`, browsers)| [[hosts]]             |
| `services`   | port name ↔ number (`ssh` = 22)         | `/etc/services`       |
| `protocols`  | protocol names (`tcp` = 6)              | `/etc/protocols`      |
| `networks`   | network names                           | `/etc/networks`       |
| `netgroup`, `ethers`, `rpc`, `publickey` | rare              | matching `/etc` file  |

# Sources

| Source        | Gives                                                              |
| ------------- | ------------------------------------------------------------------ |
| `files`       | The classic `/etc` file                                            |
| `systemd`     | Dynamic users of services, `systemd-homed` users, root/nobody fallback |
| `resolve`     | Ask `systemd-resolved` (DNS, LLMNR, mDNS)                          |
| `dns`         | Classic DNS using [[resolv-conf]]                                  |
| `myhostname`  | Always resolves the local [[hostname]], `localhost`, `_gateway`    |
| `mymachines`  | Names of local containers/VMs (`systemd-machined`)                 |
| `mdns4_minimal` | `.local` names via Avahi                                         |
| `sss`, `ldap`, `winbind` | Central user directories (company/AD logins)            |

### Action items `[...]`
`[STATUS=action]` changes what happens after a source answers.

| Item                  | Meaning                                                         |
| --------------------- | --------------------------------------------------------------- |
| `[NOTFOUND=return]`   | If this source says "doesn't exist", stop and believe it        |
| `[!UNAVAIL=return]`   | If the source is **available**, stop here, whatever it answered |
| `[SUCCESS=merge]`     | Merge group members from several sources                        |

Read the `hosts` line above: ask containers (`mymachines`), then `systemd-resolved`; **if resolved is running, its answer is final**. Only when resolved is down do `files`, `myhostname` and `dns` get asked.

> [!note] Then how does `/etc/hosts` still work here?
> `systemd-resolved` reads `/etc/hosts` itself. So `files` is still honoured, just through resolved.

# Scripting recipes

### Query any database the right way
```bash
getent passwd alice
getent group wheel
getent hosts localhost
getent services ssh          # ssh  22/tcp
getent protocols tcp
```

### Sources for one database
```bash
awk -F: '$1=="hosts" {print $2}' /etc/nsswitch.conf
```

### Which NSS plugins are installed
```bash
ls /usr/lib/libnss_*.so.2 /lib/x86_64-linux-gnu/libnss_*.so.2 2>/dev/null
```

### Port number from a service name in a script
```bash
port=$(getent services https | awk '{split($2, a, "/"); print a[1]}')   # 443
```

# See also
- [[hosts]] · [[resolv-conf]] · [[dns]] — the hosts database
- [[passwd]] · [[group]] · [[shadow]] — the user databases
- [[hostname]] · [[networking]] · [[etc]]
