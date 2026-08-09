# Homelab

Docker Compose manifests for my TrueNAS Community Edition.

This is not meant for public consumption — everything is tuned for my hardware
and domain. I publish it as a backup and so others can crib from it.

## Services

| Service        | Image                                       | Exposed on          |
|----------------|---------------------------------------------|---------------------|
| traefik        | `traefik:v3.7.10`                           | `:80`, `:443` |
| home-assistant | `homeassistant/home-assistant:2026.8.0`     | `homeassistant.urgas.eu` via Traefik |
| mosquitto      | `eclipse-mosquitto:2.1.2-alpine`            | LAN `:1883` |
| plex           | `plexinc/pms-docker:1.43.3.10861-07dfddaeb` | `plex.urgas.eu` via Traefik, LAN `:32400` |
| postgresql     | `postgres:15.18-trixie`                     | internal only (`homelab` network) |
| qbittorrent    | `lscr.io/linuxserver/qbittorrent:5.2.3`     | `qbittorrent.urgas.eu` via Traefik, LAN `:6881` (peers) |
| prowlarr       | `lscr.io/linuxserver/prowlarr:2.5.2`        | `prowlarr.urgas.eu` via Traefik |
| radarr         | `lscr.io/linuxserver/radarr:6.3.0`          | `radarr.urgas.eu` via Traefik |
| sonarr         | `lscr.io/linuxserver/sonarr:4.0.19`         | `sonarr.urgas.eu` via Traefik |
| bazarr         | `lscr.io/linuxserver/bazarr:1.6.0`          | `bazarr.urgas.eu` via Traefik |
| recyclarr      | `ghcr.io/recyclarr/recyclarr:8.7.0`         | none — daily cron sync, no UI |
| seerr          | `ghcr.io/seerr-team/seerr:v3.4.1`           | `seerr.urgas.eu` via Traefik |
| uptime-kuma    | `louislam/uptime-kuma:2.5.0`                | `uptime.urgas.eu` via Traefik |
| homepage       | `ghcr.io/gethomepage/homepage:v1.13.2`      | `homepage.urgas.eu`, `home.urgas.eu` via Traefik |

Traefik handles TLS for everything under `urgas.eu` using a Let's Encrypt
wildcard cert obtained via the Cloudflare DNS-01 challenge.

Routing is declared centrally in `services/traefik/dynamic/routes.yml` (the
file provider) rather than through per-container `traefik.*` labels. The cost
is that adding a service means editing that file too; there is no
auto-discovery.

**No container in this stack mounts the Docker socket.** `:ro` on a socket
mount protects only the file, not the API, so any container holding it can
create privileged containers and take root on the host. Traefik gave it up by
moving to the file provider; Homepage gave it up by dropping its live
container-status tiles. Keep it that way — if something needs the Docker API,
put a `docker-socket-proxy` in front of it rather than mounting the socket.

**Every container runs with `cap_drop: ALL` and `no-new-privileges:true`**, and
adds back only what it demonstrably needs. New services should follow suit.
There are four shapes:

| Shape | Capabilities added | Services |
|-------|--------------------|----------|
| s6-based images (linuxserver, Plex, Home Assistant) | `CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `SETUID`, `SETGID`, `KILL` | plex, qbittorrent, radarr, sonarr, prowlarr, bazarr, home-assistant |
| Starts as root, drops itself via setuid | same, minus `KILL` | mosquitto |
| Binds a privileged port | `NET_BIND_SERVICE` | traefik |
| Already starts as a non-root uid, or needs nothing | none | postgresql, seerr, recyclarr, homepage, uptime-kuma, whisper, piper |

The s6 capabilities are for the init, not the application — s6 chowns `/config`
and drops to `PUID`/`PGID`, then the app itself runs with none of them. Traefik
needs `NET_BIND_SERVICE` even as root, because the privileged-port check is
capability-based rather than uid-based; without it, `:80` fails to bind.

`NET_RAW` is deliberately absent everywhere. It is what ICMP `ping` monitors and
DHCP-based discovery want, and it also permits ARP spoofing against every other
container on the shared `homelab` bridge. Nothing here currently needs it — add
it back per-service, with a comment, if that changes.

## Layout

```
.
├── docker-compose.yml           # aggregates all services via `include:`
├── .env                         # secrets (gitignored) — see .env.example
├── .github/dependabot.yml       # weekly PRs bumping pinned image tags
└── services/
    ├── traefik/
    │   ├── docker-compose.yml
    │   └── dynamic/              # routing table for every service (file provider)
    ├── home-assistant/docker-compose.yml
    ├── mosquitto/
    │   ├── docker-compose.yml
    │   └── mosquitto.conf
    ├── plex/docker-compose.yml
    ├── postgresql/docker-compose.yml
    ├── qbittorrent/docker-compose.yml
    ├── voice-assist/docker-compose.yml
    ├── prowlarr/docker-compose.yml
    ├── radarr/docker-compose.yml
    ├── sonarr/docker-compose.yml
    ├── bazarr/docker-compose.yml
    ├── recyclarr/
    │   ├── docker-compose.yml
    │   └── config/               # declarative TRaSH-Guides sync config (recyclarr.yml)
    ├── seerr/docker-compose.yml
    ├── uptime-kuma/docker-compose.yml
    └── homepage/
        ├── docker-compose.yml
        └── config/               # declarative dashboard config (settings, services, widgets)
```

Each `services/<name>/docker-compose.yml` is self-contained — you can paste it
straight into TrueNAS's "Install Custom App" UI, or run `docker compose up`
from that directory. The root `docker-compose.yml` just `include:`s all of them
for convenience. The one exception is HTTP routing: a service pasted on its own
comes up reachable on the `homelab` network but unrouted until its router and
service blocks are added to `services/traefik/dynamic/routes.yml`.

## Bootstrap

One-time setup on the TrueNAS host:

```sh
# Shared network that all services attach to
docker network create homelab

# Persistent directories (bind mounts)
mkdir -p /mnt/ssd-storage/homelab/{traefik,home-assistant/config,plex/config,postgresql/pgdata,qbittorrent/config,prowlarr/config,radarr/config,sonarr/config,bazarr/config,recyclarr/config,seerr/config,uptime-kuma/data}

# Postgres runs as `apps` (568) and its entrypoint will not chown for itself
chown 568:568 /mnt/ssd-storage/homelab/postgresql/pgdata

# Uptime Kuma stays root inside the container, and `cap_drop: ALL` takes away
# DAC_OVERRIDE — so root can only write files it actually owns. A stray
# kuma.db owned by anyone else fails with `SQLITE_READONLY` in a crash loop.
chown -R 0:0 /mnt/ssd-storage/homelab/uptime-kuma/data

# acme.json must be mode 600 or Traefik refuses to use it
install -m 600 /dev/null /mnt/ssd-storage/homelab/traefik/acme.json

# Secrets
cp .env.example .env
$EDITOR .env
```

### qBittorrent WebUI host whitelist

qBittorrent's own settings live in its config directory, not in compose, so
they are not captured by this repo and a recreated config silently reverts to
the insecure default. One matters for security: **Web UI → "Server domains"**
must not be left at `*`. That wildcard disables host-header validation, which
is the check that stops a hostile web page from DNS-rebinding a browser on the
LAN onto the WebUI. Set it to the hostnames that actually reach it:

```
qbittorrent.urgas.eu;qbittorrent
```

Both entries are required — the first is what Traefik forwards, the second is
what Radarr and Sonarr use to reach the API directly over the `homelab`
network. Editing `qBittorrent.conf` by hand instead of via the UI needs the
value **quoted**, or Qt reads the `;` as an INI comment and drops everything
after it, locking out the *arrs with a 401. Stop the container before editing;
qBittorrent rewrites the file on exit.

## Deployment

Sync files to the target server:

```sh
rsync -avz --exclude '.git' ./ root@192.168.1.55:/mnt/ssd-storage/homelab-repo
```

Update services on the remote server:

```sh
ssh root@192.168.1.55 "cd /mnt/ssd-storage/homelab-repo && docker compose pull && docker compose up -d"
```

## Running

```sh
docker compose up -d                          # start everything
docker compose down                           # stop everything
docker compose pull && docker compose up -d   # update images
```

## Image updates

Every image here is pinned, and `.github/dependabot.yml` opens one grouped PR a
week against `master` covering all of `services/*/docker-compose.yml`. Grouped
because fourteen separate PRs for a homelab is noise; `directories: ["/services/*"]`
picks up new services automatically, so adding one needs no config change.

Two things it deliberately does not do. It carries no vulnerability data for
container images — the `docker-compose` ecosystem does version updates only, so
a PR means "a newer tag exists", never "the tag you are on has a CVE". And
merging deploys nothing: the stack is deployed by the manual rsync above, so a
merged PR is only a change of intent until that runs.

Postgres majors are the one exception, ignored in `dependabot.yml`. Moving the
recorder to a new major is a dump and restore rather than a tag change, and a
grouped PR that quietly carries one would crash-loop the container against an
incompatible data directory. Minor and patch bumps still come through.

## Secrets

Secrets live in a gitignored `.env` file at the repo root. See `.env.example`
for the full list. Compose expands `${VAR}` references in each service file
from this one location.

## Home Assistant

### Shelly Devices

For Shelly devices using **CoIoT**, use the following peer address: `192.168.1.55:5683`

Newer Shelly devices (Generation 2 and 3) do not support CoIoT. For these devices, use **Mosquitto (MQTT)** or **WebSockets** instead.
