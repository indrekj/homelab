# Homelab

Docker Compose manifests for my TrueNAS Community Edition.

This is not meant for public consumption — everything is tuned for my hardware
and domain. I publish it as a backup and so others can crib from it.

## Services

| Service        | Image                                          | Exposed on          |
|----------------|-----------------------------------------------|---------------------|
| traefik        | `traefik:v3.3`                                | `:80`, `:443`       |
| home-assistant | `homeassistant/home-assistant:2024.3.3`       | `homeassistant.urgas.eu` via Traefik |
| mosquitto      | `eclipse-mosquitto:2`                         | LAN `:1883`         |
| plex           | `plexinc/pms-docker`                          | `plex.urgas.eu` via Traefik, LAN `:32400` |
| postgresql     | `bitnamilegacy/postgresql:15.3.0-debian-11-r24` | internal only (`homelab` network) |

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
    └── postgresql/
        ├── docker-compose.yml
        └── override.conf
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
mkdir -p /mnt/ssd-storage/homelab/{traefik,home-assistant/config,plex/config,postgresql/data}

# acme.json must be mode 600 or Traefik refuses to use it
install -m 600 /dev/null /mnt/ssd-storage/homelab/traefik/acme.json

# Secrets
cp .env.example .env
$EDITOR .env
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
