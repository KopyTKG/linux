---
tags:
  - linux
  - networking
  - sockets
  - ports
aliases:
  - ports
  - sockets
  - ss
  - TCP
  - UDP
---
An IP address gets a packet to the right **machine**. A **port** (0–65535) gets it to the right **program**. A **socket** is the kernel object a program uses to send/receive: `IP + port + protocol`. Every connection is identified by four things: local IP, local port, remote IP, remote port.

> [!info] Example desktop
> `ss -s` → `TCP: 55 (estab 25)`, `UDP: 49`. Some typical listeners:
> | Port   | Bound to          | Program                    | Reachable from LAN? |
> | ------ | ----------------- | -------------------------- | ------------------- |
> | `53`   | `127.0.0.53`      | systemd-resolved (DNS stub) | no                 |
> | `631`  | `127.0.0.1`       | CUPS (printing)            | no                  |
> | `5355` | `0.0.0.0`         | systemd-resolved (LLMNR)   | yes                 |
> | `5432` | `0.0.0.0`         | PostgreSQL in a **Docker** container | **yes**   |
> | `8080` | `127.0.0.1`       | Local dev web server       | no                  |
> | `22`   | `0.0.0.0`         | sshd                       | yes                 |

# TCP vs UDP

|                    | TCP                                     | UDP                                     |
| ------------------ | --------------------------------------- | --------------------------------------- |
| Connection         | Yes — handshake (SYN, SYN-ACK, ACK)     | No — just send packets                  |
| Delivery           | Guaranteed, in order, retransmits       | Best effort, may lose / reorder         |
| Speed / overhead   | More                                    | Less                                    |
| Used by            | HTTP(S), SSH, databases, email          | DNS, video calls, games, QUIC (HTTP/3), DHCP |
| Listening state    | `LISTEN`                                | `UNCONN` (no connections exist)         |

# Port ranges

| Range         | Name                    | Notes                                                   |
| ------------- | ----------------------- | ------------------------------------------------------- |
| `0–1023`      | Well-known / privileged | Need root (or `CAP_NET_BIND_SERVICE`) to listen          |
| `1024–49151`  | Registered              | Apps: `5432` postgres, `8080` dev servers               |
| `32768–60999` | **Ephemeral**           | Random source ports for outgoing connections. See `/proc/sys/net/ipv4/ip_local_port_range` |

### Well-known ports

| Port   | Proto   | Service                        |
| ------ | ------- | ------------------------------ |
| 22     | TCP     | SSH                            |
| 53     | UDP/TCP | DNS → [[dns]]                  |
| 67/68  | UDP     | DHCP server / client           |
| 80     | TCP     | HTTP                           |
| 123    | UDP     | NTP (time sync)                |
| 443    | TCP/UDP | HTTPS / HTTP/3                 |
| 631    | TCP     | IPP / CUPS printing            |
| 3306   | TCP     | MySQL / MariaDB                |
| 5353   | UDP     | mDNS (`.local`)                |
| 5355   | UDP/TCP | LLMNR                          |
| 5432   | TCP     | PostgreSQL                     |
| 6379   | TCP     | Redis                          |
| 51820  | UDP     | WireGuard                      |

Full list: `/etc/services` → `getent services 5432/tcp` → `postgresql 5432/tcp`. → [[etc]]

# Bind address: who can reach a listener

| Listening on        | Reachable from                              |
| ------------------- | ------------------------------------------- |
| `127.0.0.1:631`     | **Only this machine**                       |
| `[::1]:631`         | Only this machine (IPv6)                    |
| `0.0.0.0:5432`      | **Every** IPv4 interface → LAN too          |
| `[::]:5432` or `*:5432` | Every interface, IPv6 (often IPv4 too)  |
| `192.168.1.50:8080`   | Only via that one address                   |

> [!tip] "Works locally, not from my phone"
> Your dev server is probably on `127.0.0.1`. Start it on `0.0.0.0` (e.g. `--host 0.0.0.0`), then check the [[firewall]].

# `ss` — socket statistics
Replacement for old `netstat`. Reads the kernel via netlink (and `/proc/net/*`).

| Flag | Meaning                                             |
| ---- | --------------------------------------------------- |
| `-t` | TCP                                                 |
| `-u` | UDP                                                 |
| `-x` | Unix sockets                                        |
| `-l` | Listening only                                      |
| `-a` | All (listening + connected)                         |
| `-n` | Numeric — don't turn ports into names (faster)      |
| `-p` | Show process (own processes only, unless root)      |
| `-H` | No header (for scripts)                             |
| `-s` | Summary                                             |
| `-4` / `-6` | IPv4 / IPv6 only                             |

```bash
ss -tulpn                     # THE command: what listens, TCP+UDP, with process
ss -tnp state established     # open connections
ss -tn dst 192.168.1.1          # connections to a host
ss -tln 'sport = :5432'       # filter by local port
ss -s                         # summary
```
```
LISTEN 0  4096   127.0.0.1:631    0.0.0.0:*
LISTEN 0  4096     0.0.0.0:5432   0.0.0.0:*
LISTEN 0  511    127.0.0.1:8080   0.0.0.0:*  users:(("python3",pid=3601,fd=3))
```

| Column     | Meaning                                                    |
| ---------- | ---------------------------------------------------------- |
| State      | `LISTEN`, `ESTAB`, `TIME-WAIT`, `UNCONN` (UDP) ...         |
| Recv-Q     | For LISTEN: connections waiting to be accepted              |
| Send-Q     | For LISTEN: max backlog                                    |
| Local      | `IP:port` we're bound to                                   |
| Peer       | Remote side (`*` / `0.0.0.0:*` = any)                      |
| `users:`   | Process name, PID, file descriptor ([[proc]] `/proc/<pid>/fd/41`) |

> [!note] Missing process name?
> Without root, `ss -p` only shows **your own** processes. `631` (CUPS) and `5432` (Docker's proxy) belong to root, so the column is empty. Use `sudo ss -tulpn` to see all.

### TCP states

| State         | Meaning                                                        |
| ------------- | -------------------------------------------------------------- |
| `LISTEN`      | Server waiting for connections                                 |
| `SYN-SENT`    | We're connecting, no reply yet (stuck = firewall/host down)    |
| `ESTAB`       | Connected, data flowing                                        |
| `TIME-WAIT`   | Closed by us, waiting ~60 s for stray packets. Normal          |
| `CLOSE-WAIT`  | Remote closed, **our program hasn't**. Many = app bug          |
| `FIN-WAIT-*`  | Closing                                                        |

# `/proc/net/tcp` — the raw data
What `ss`/`netstat` read when netlink isn't used. Hex, little-endian IPs. Format explained in [[proc]] → `/proc/net`. Files: `tcp`, `tcp6`, `udp`, `udp6`, `unix`, `raw`.

Link socket → process: the `inode` column equals `socket:[inode]` in `/proc/<pid>/fd/`.

# Unix sockets
Sockets that are **files**, for processes on the same machine (faster, permission-controlled). Typical examples:

| Socket                                  | Used by                       |
| --------------------------------------- | ----------------------------- |
| `/run/systemd/resolve/io.systemd.Resolve` | systemd-resolved            |
| `/run/dbus/system_bus_socket`           | D-Bus system bus              |
| `/run/user/1000/wayland-0`              | Wayland compositor            |
| `/run/user/1000/pipewire-0`             | Audio                         |
| `/var/run/docker.sock`                  | Docker API                    |

`ss -xl` lists them. → [[run]]

# Scripting recipes

### Listening ports with process (own processes)
```bash
ss -tlnpH | awk '{
  split($4, a, ":"); port = a[length(a)]
  proc = $6; sub(/users:\(\("/, "", proc); sub(/".*/, "", proc)
  print port, (proc == "" ? "?" : proc)
}' | sort -n -u
```
```
53 ?
631 ?
8080 python3
```

### Is a port in use?
```bash
port_in_use() { ss -tlnH "sport = :$1" | grep -q .; }
port_in_use 8080 || echo "8080 free"
```

### Is a remote port open? (no extra tools)
bash has built-in `/dev/tcp/HOST/PORT`:
```bash
if timeout 2 bash -c '</dev/tcp/127.0.0.1/631' 2>/dev/null; then
  echo "open"
else
  echo "closed or filtered"
fi
```
`/dev/tcp` is a bash feature, not a real file in [[dev]]. Alternative: `nc -zv host port` → [[tools]].

### Connections per program
```bash
ss -tnpH state established | grep -o '(("[^"]*' | cut -c4- | sort | uniq -c | sort -rn
```
```
      6 firefox
      4 thunderbird
      2 code
      1 ssh
```

### Port number ↔ service name
```bash
getent services 5432/tcp    # postgresql 5432/tcp
getent services https       # https 443/tcp
```

### Kill whatever listens on a port
```bash
pid=$(ss -tlnpH "sport = :8080" | grep -o 'pid=[0-9]*' | cut -d= -f2 | head -1)
[[ -n $pid ]] && kill "$pid"
```

### Wait until a service starts listening
```bash
until ss -tlnH "sport = :5432" | grep -q .; do sleep 0.5; done
echo "postgres is listening"
```

# Find stuff

| I want ...                          | Command                                    |
| ----------------------------------- | ------------------------------------------ |
| Everything listening                | `sudo ss -tulpn`                           |
| Who uses port X                     | `sudo ss -tlnp 'sport = :X'`, `sudo lsof -i :X` |
| Open connections                    | `ss -tnp state established`                |
| Sockets of one process              | `ls -l /proc/<pid>/fd | grep socket`       |
| Port ↔ name                         | `/etc/services`, `getent services`         |
| Ephemeral port range                | `cat /proc/sys/net/ipv4/ip_local_port_range` |
| Unix sockets                        | `ss -xlp`                                  |

# See also
- [[networking]] — overview
- [[firewall]] — what can reach the listening ports
- [[tools]] — `nc`, `curl` for testing ports
- [[proc]] — `/proc/net/tcp`, `/proc/<pid>/fd`
- [[run]] — Unix socket files
