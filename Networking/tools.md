---
tags:
  - linux
  - networking
  - cheatsheet
aliases:
  - network tools
  - curl
  - ssh
  - rsync
  - netcat
---
Cheat sheet of the network commands you'll actually use. Grouped by the question they answer. Theory lives in the other notes — see [[networking]].

> [!info] Typical availability
> | Installed                                                                 | Not installed                       |
> | ------------------------------------------------------------------------- | ----------------------------------- |
> | `ip`, `ss`, `ping`, `tracepath`, `curl`, `wget`, `ssh`/`scp`, `rsync`, `nc` (OpenBSD netcat), `dig`, `host`, `nslookup`, `resolvectl`, `nmcli`, `ethtool`, `iw`, `nft`, `iptables`, `ufw` | `tcpdump`, `nmap`, `mtr`, `traceroute`, `netstat`, `ncat`, `drill`, `wg` |
>
> Missing tools are packaged under the same name in every major distro; install them with your package manager.

# Which tool for which question

| Question                              | Tool                                   |
| ------------------------------------- | -------------------------------------- |
| Interfaces / IPs / routes             | `ip` → [[interfaces]], [[ip-routing]]  |
| Is the host alive?                    | `ping`                                 |
| Where does the path break?            | `tracepath`, `mtr`                     |
| Does the name resolve?                | `getent hosts`, `dig` → [[dns]]        |
| What is listening / connected?        | `ss` → [[sockets-ports]]               |
| Is a remote port open?                | `nc -zv`, bash `/dev/tcp`              |
| Does the web service answer?          | `curl`                                 |
| Download a file                       | `curl -LO`, `wget`                     |
| Log in / run command remotely         | `ssh`                                  |
| Copy files between machines           | `scp`, `rsync`                         |
| See the actual packets                | `tcpdump`, Wireshark                   |
| Scan a network / host                 | `nmap`                                 |
| Firewall                              | `ufw`, `nft` → [[firewall]]            |
| Connection management                 | `nmcli` → [[network-manager]]          |

# `ip` (iproute2)
```bash
ip -br addr                  # IPs, brief
ip -br link                  # interfaces + state
ip route                     # routing table
ip route get 1.1.1.1         # route for one destination
ip neigh                     # ARP cache
ip -s link show enp3s0       # traffic + error counters
ip -j addr | jq              # JSON for scripts
```
Old `ifconfig`, `route`, `arp` (net-tools) are deprecated → use `ip`.

# `ping` / `tracepath` / `mtr`
```bash
ping -c 4 kernel.org      # 4 pings then stop
ping -i 0.2 -c 20 192.168.1.1  # faster interval
ping -4 / -6 host            # force protocol
ping -M do -s 1472 host      # MTU test: 1472 + 28 header = 1500, don't fragment
tracepath -n 1.1.1.1         # path, no root needed, shows pmtu
mtr -rwc 50 1.1.1.1          # report: loss % per hop
```

# `curl`
```bash
curl https://example.com                     # print body
curl -I https://example.com                  # headers only (HEAD)
curl -sS -o /dev/null -w '%{http_code}\n' URL   # status code only
curl -L URL                                  # follow redirects
curl -LO URL                                 # download, keep file name
curl -o out.iso -C - URL                     # resume download
curl -fsSL URL                               # scripts: fail on HTTP error, quiet but show errors
curl -v URL                                  # verbose: DNS, TLS, headers
curl -X POST -H 'Content-Type: application/json' -d '{"a":1}' URL
curl -u user:pass URL                        # basic auth
curl --resolve example.com:443:192.168.1.30 https://example.com   # fake DNS for one request
curl -w '%{time_namelookup} %{time_connect} %{time_total}\n' -o /dev/null -s URL   # timing
```

| Flag | Meaning                                         |
| ---- | ----------------------------------------------- |
| `-s` | Silent (no progress bar)                        |
| `-S` | Show errors even with `-s`                      |
| `-f` | Fail (exit ≠ 0) on HTTP 4xx/5xx                 |
| `-L` | Follow redirects                                |
| `-o` / `-O` | Output to file / keep remote name        |
| `-k` | Ignore TLS certificate errors (testing only!)   |

# `wget`
```bash
wget URL                      # download
wget -c URL                   # continue partial download
wget -q -O - URL              # print to stdout
wget -r -np -k URL            # mirror a site section
```

# `ssh`
```bash
ssh user@host                         # log in
ssh -p 2222 user@host                 # other port
ssh user@host 'df -h'                 # run one command
ssh -J jump@bastion user@internal     # through a jump host
ssh-keygen -t ed25519                 # create key pair (~/.ssh/id_ed25519{,.pub})
ssh-copy-id user@host                 # install your public key on host
```

### Tunnels
```bash
ssh -L 5433:localhost:5432 user@server   # LOCAL: my :5433 → server's :5432
ssh -R 8080:localhost:3000 user@server   # REMOTE: server's :8080 → my :3000
ssh -D 1080 user@server                  # SOCKS proxy through server
ssh -N -f -L ...                         # no shell, go to background
```

### `~/.ssh/config`
```
Host myserver
    HostName 203.0.113.10
    User admin
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
```
Then just `ssh myserver` (also works for `scp`, `rsync`, `git`).

| File                      | Purpose                                        |
| ------------------------- | ---------------------------------------------- |
| `~/.ssh/config`           | Per-host settings                              |
| `~/.ssh/id_*`             | **Private** keys — must be `chmod 600`, never share |
| `~/.ssh/id_*.pub`         | Public keys                                    |
| `~/.ssh/authorized_keys`  | Keys allowed to log **in to this** machine     |
| `~/.ssh/known_hosts`      | Fingerprints of servers you trust              |
| `/etc/ssh/sshd_config`    | SSH **server** config → [[etc]]                |

> [!warning] "REMOTE HOST IDENTIFICATION HAS CHANGED"
> The server's key differs from `known_hosts`. Either it was reinstalled (then `ssh-keygen -R host`) or someone is intercepting. Don't blindly remove it.

# `scp` & `rsync`
```bash
scp file.txt user@host:/tmp/              # copy to remote
scp user@host:/var/log/app.log .          # copy from remote
scp -r dir/ user@host:~/                  # directory

rsync -avh --progress src/ user@host:dst/    # sync, only changed parts
rsync -avh --delete src/ dst/                # mirror (deletes extras in dst!)
rsync -avhn --delete src/ dst/               # -n = dry run, check first
rsync -avh -e 'ssh -p 2222' src/ host:dst/   # custom ssh
```

> [!tip] Trailing slash matters in rsync
> `rsync src/ dst/` copies the **contents** of `src` into `dst`.
> `rsync src dst/` creates `dst/src/`.

# `nc` (netcat)
Swiss army knife for raw TCP/UDP. Many systems ship the **OpenBSD** variant (flags differ from GNU netcat / `ncat`).
```bash
nc -zv 192.168.1.1 80            # port open? (-z scan, -v verbose)
nc -zv host 20-25              # range
nc -zvu host 53                # UDP (unreliable result)
nc -l 9000                     # listen on port 9000
nc host 9000                   # connect: type text, appears on the listener
nc -l 9000 > file              # receive a file ...
nc -N host 9000 < file         # ... send it (-N close after EOF)
printf 'GET / HTTP/1.0\r\n\r\n' | nc example.com 80   # raw HTTP
```

# `dig` / `host`
```bash
dig +short kernel.org           # A record
dig +short kernel.org AAAA MX   # several types
dig @9.9.9.9 kernel.org         # specific server
dig -x 1.1.1.1 +short              # reverse
host kernel.org
```
More in [[dns]].

# `ss`
```bash
ss -tulpn                          # listening TCP+UDP with process
ss -tnp state established          # connections
ss -s                              # summary
```
More in [[sockets-ports]].

# `tcpdump` (packet capture, needs root, not installed here)
```bash
sudo tcpdump -i enp3s0 -n                    # everything
sudo tcpdump -i any -n port 53               # DNS traffic
sudo tcpdump -i enp3s0 -n host 192.168.1.30    # one host
sudo tcpdump -i enp3s0 -w cap.pcap           # save for Wireshark
sudo tcpdump -A -i lo port 8080              # print payload as text
```

# `nmap` (scanner, not installed here)
```bash
nmap -sn 192.168.1.0/24          # which hosts are up (ping scan)
nmap 192.168.1.30                # top 1000 TCP ports
nmap -p 22,80,443 -sV host     # specific ports + service version
nmap -p- host                  # all 65535 ports
```

> [!warning] Only scan what you own
> Scanning networks/hosts you don't control may break rules or laws. Your own LAN and machines are fine.

# Scripting recipes

### HTTP health check
```bash
check() {
  local code
  code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 5 "$1")
  [[ $code == 2* || $code == 3* ]] && echo "OK  $code $1" || echo "ERR $code $1"
}
check http://127.0.0.1:631/
# OK  200 http://127.0.0.1:631/
```

### Retry until reachable
```bash
until curl -fsS --max-time 2 http://localhost:8080/health > /dev/null; do
  echo "waiting..."; sleep 2
done
```

### Download only if changed
```bash
curl -fsSL -z file.json -o file.json https://example.com/file.json
```
`-z file` = send `If-Modified-Since` with the local file's date.

### Run a command on many hosts
```bash
for h in web1 web2 db1; do
  echo "== $h"; ssh -o ConnectTimeout=5 "$h" 'uptime'
done
```

### Port check without nc
```bash
timeout 2 bash -c '</dev/tcp/192.168.1.1/80' && echo open || echo closed
```

### Fail a script if offline
```bash
getent hosts example.com > /dev/null || { echo "no DNS / offline" >&2; exit 1; }
```

# Find stuff

| I want ...                         | Command                         |
| ---------------------------------- | ------------------------------- |
| Is tool X installed                | `command -v X`                  |
| Which netcat variant               | `nc -h 2>&1 | head -1`          |
| SSH client config                  | `~/.ssh/config`, `/etc/ssh/ssh_config` |
| SSH server config                  | `/etc/ssh/sshd_config`          |
| curl/wget proxy settings           | `env | grep -i _proxy`          |
| Manual                             | `man curl`, `man ssh_config`, `tldr <cmd>` |

# See also
- [[networking]] — overview and troubleshooting order
- [[interfaces]] · [[ip-routing]] · [[dns]] · [[sockets-ports]] · [[firewall]] · [[network-manager]]
- [[home]] — `~/.ssh` lives in your home
- [[dev]] — why `/dev/tcp` is not really a file
