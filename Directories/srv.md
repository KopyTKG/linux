---
tags:
  - linux
  - filesystem
  - srv
  - server
aliases:
  - /srv
  - served data
---
`/srv` ("serve") holds **data that this machine serves to others** — website files, FTP shares, Git repositories, etc. On a desktop it's usually empty.

> [!info] Example
> ```
> /srv/
> ├── ftp     (empty)
> └── http    (empty)
> ```
> Some distros pre-create empty `ftp` and `http` folders as defaults for their FTP and web server packages. Here `/srv` is also its own btrfs subvolume `@srv`, so system snapshots don't roll back served data:
> `/srv  /dev/nvme0n1p3[/@srv]  btrfs`

# Typical layout

| Path               | Served by                          |
| ------------------ | ---------------------------------- |
| `/srv/http/` or `/srv/www/` | Web server (nginx, Apache) document root |
| `/srv/ftp/`        | FTP server (vsftpd) anonymous root |
| `/srv/git/`        | Git server repos                   |
| `/srv/nfs/`        | NFS exports                        |
| `/srv/samba/`      | Samba / Windows shares             |
| `/srv/<site-name>/`| One folder per website             |

> [!note] Not strictly enforced
> The FHS says *where* served data **should** go, but every distro and program has its own default:
> - `/var/www/html` — very common web server default
> - `/usr/share/nginx/html` — nginx's built-in default page
> - `/srv/http` or `/srv/www` — FHS-style choice
> - Docker volumes → `/var/lib/docker/volumes`
> Pick one place and point the server config there → see [[etc]] (`/etc/nginx/nginx.conf`).

> [!tip] `/srv` vs `/var/lib` vs [[opt]]
> - `/srv` — content **you** put there to be served (HTML, files)
> - `/var/lib/<app>` — internal state the app manages itself (databases) → [[var]]
> - `/opt/<app>` — the application program itself

# Scripting recipes

### Quick web server for a folder
```bash
cd /srv/http && python -m http.server 8080   # http://localhost:8080
```

### Give your user write access to a site folder
```bash
sudo mkdir -p /srv/http/mysite
sudo chown -R "$USER":http /srv/http/mysite      # group of the web server user (http, www-data, nginx ...)
sudo chmod -R u=rwX,g=rX,o= /srv/http/mysite
```

### Deploy a built site
```bash
rsync -a --delete ./dist/ /srv/http/mysite/
```

# Find stuff
| Looking for                         | Command                                        |
| ----------------------------------- | ---------------------------------------------- |
| Where nginx serves from             | `grep -rn 'root ' /etc/nginx/`                 |
| Where Apache serves from            | `grep -rn DocumentRoot /etc/httpd/ /etc/apache2/ 2>/dev/null` |
| Which process listens on port 80    | `sudo ss -ltnp 'sport = :80'`                  |
| Size of served data                 | `du -sh /srv/*`                                |
| Which user the web server runs as   | `ps -o user= -C nginx,httpd,apache2 \| sort -u` |

# See also
- [[var]] — `/var/www`, `/var/lib`
- [[opt]] — self-contained apps
- [[etc]] — server configs
- `man 7 file-hierarchy`
