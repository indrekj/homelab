# Homelab

Docker Compose manifests for my TrueNAS Community Edition.

This is not meant for public consumption — everything is tuned for my hardware
and domain. I publish it as a backup and so others can crib from it.

## Services

| Service        | Image                                          | Exposed on          |
|----------------|-----------------------------------------------|---------------------|
| traefik        | `traefik:v3.3`                                | `:80`, `:443`       |
| home-assistant | `homeassistant/home-assistant:2026.7.1`       | `homeassistant.urgas.eu` via Traefik |
| mosquitto      | `eclipse-mosquitto:2`                         | LAN `:1883`         |
| plex           | `plexinc/pms-docker`                          | `plex.urgas.eu` via Traefik, LAN `:32400` |
| postgresql     | `bitnamilegacy/postgresql:15.3.0-debian-11-r24` | internal only (`homelab` network) |
| qbittorrent    | `lscr.io/linuxserver/qbittorrent:5.2.3`       | `qbittorrent.urgas.eu` via Traefik, LAN `:6881` (peers) |
| prowlarr       | `lscr.io/linuxserver/prowlarr:2.4.0`          | `prowlarr.urgas.eu` via Traefik |
| radarr         | `lscr.io/linuxserver/radarr:6.3.0`            | `radarr.urgas.eu` via Traefik |
| sonarr         | `lscr.io/linuxserver/sonarr:4.0.19`           | `sonarr.urgas.eu` via Traefik |
| bazarr         | `lscr.io/linuxserver/bazarr:1.6.0`            | `bazarr.urgas.eu` via Traefik |
| seerr          | `ghcr.io/seerr-team/seerr:v3.3.0`             | `seerr.urgas.eu` via Traefik |
| uptime-kuma    | `louislam/uptime-kuma:2.4.0`                  | `uptime.urgas.eu` via Traefik |

Traefik handles TLS for everything under `urgas.eu` using a Let's Encrypt
wildcard cert obtained via the Cloudflare DNS-01 challenge.

## Layout

```
.
├── docker-compose.yml           # aggregates all services via `include:`
├── .env                         # secrets (gitignored) — see .env.example
└── services/
    ├── traefik/docker-compose.yml
    ├── home-assistant/docker-compose.yml
    ├── mosquitto/
    │   ├── docker-compose.yml
    │   └── mosquitto.conf
    ├── plex/docker-compose.yml
    ├── postgresql/
    │   ├── docker-compose.yml
    │   └── override.conf
    ├── qbittorrent/docker-compose.yml
    ├── voice-assist/docker-compose.yml
    ├── prowlarr/docker-compose.yml
    ├── radarr/docker-compose.yml
    ├── sonarr/docker-compose.yml
    ├── bazarr/docker-compose.yml
    ├── seerr/docker-compose.yml
    └── uptime-kuma/docker-compose.yml
```

Each `services/<name>/docker-compose.yml` is self-contained — you can paste it
straight into TrueNAS's "Install Custom App" UI, or run `docker compose up`
from that directory. The root `docker-compose.yml` just `include:`s all of them
for convenience.

## Bootstrap

One-time setup on the TrueNAS host:

```sh
# Shared network that all services attach to
docker network create homelab

# Persistent directories (bind mounts)
mkdir -p /mnt/ssd-storage/homelab/{traefik,home-assistant/config,plex/config,postgresql/data,qbittorrent/config,prowlarr/config,radarr/config,sonarr/config,bazarr/config,seerr/config,uptime-kuma/data}

# acme.json must be mode 600 or Traefik refuses to use it
install -m 600 /dev/null /mnt/ssd-storage/homelab/traefik/acme.json

# Secrets
cp .env.example .env
$EDITOR .env
```

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

## Secrets

Secrets live in a gitignored `.env` file at the repo root. See `.env.example`
for the full list. Compose expands `${VAR}` references in each service file
from this one location.

## Home Assistant

### Shelly Devices

For Shelly devices using **CoIoT**, use the following peer address: `192.168.1.55:5683`

Newer Shelly devices (Generation 2 and 3) do not support CoIoT. For these devices, use **Mosquitto (MQTT)** or **WebSockets** instead.
