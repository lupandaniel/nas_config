# Home Server Setup

Docker Compose stack for media streaming, downloads, photo management, home automation, camera restreaming/NVR, reverse proxy, backups, and VPN access. All services are defined in [`docker-compose.yml`](docker-compose.yml).

## Application ports

Replace `HOST` with your server IP or hostname.

**Host network** (`plex`, `hass`, `go2rtc`, `nginx`): ports bind on the machine directly — nothing is listed under `ports:` in compose; defaults below are what each app uses.

**Published ports** (everything else): `host:container` mappings from compose.

### Active services

| App | Port | Protocol | Mapping | Purpose |
|-----|------|----------|---------|---------|
| Plex | 32400 | TCP | host | Web UI / API |
| Plex | 32469 | UDP | host | DLNA (if enabled in Plex) |
| Plex | 32443 | TCP | host | HTTPS (if enabled in Plex) |
| Home Assistant | 8123 | TCP | host | Web UI |
| go2rtc | 1984 | TCP | host | Web UI / API |
| go2rtc | 8554 | TCP | host | RTSP restream |
| go2rtc | 8555 | TCP/UDP | host | WebRTC |
| Nginx Proxy Manager | 80 | TCP | host | HTTP (proxied sites) |
| Nginx Proxy Manager | 443 | TCP | host | HTTPS (proxied sites) |
| Nginx Proxy Manager | 81 | TCP | host | Admin UI |
| Frigate | 8971 | TCP | `8971:8971` | Authenticated Web UI / API |
| qBittorrent | 8080 | TCP | `8080:8080` | Web UI |
| qBittorrent | 6881 | TCP | `6881:6881` | BitTorrent peers |
| qBittorrent | 6881 | UDP | `6881:6881` | BitTorrent peers |
| OpenVPN | 1194 | UDP | `1194:1194` | VPN tunnel |
| Duplicati | 8200 | TCP | `8200:8200` | Web UI |
| Radarr | 7878 | TCP | `7878:7878` | Web UI |
| Sonarr | 8989 | TCP | `8989:8989` | Web UI |
| Seerr | 5055 | TCP | `5055:5055` | Web UI |
| Prowlarr | 9696 | TCP | `9696:9696` | Web UI |
| Immich | 2283 | TCP | `2283:2283` | Web UI / API |

### Quick access URLs (active)

| App | URL |
|-----|-----|
| Plex | `http://HOST:32400/web` |
| Home Assistant | `http://HOST:8123` |
| go2rtc | `http://HOST:1984` |
| Frigate | `http://HOST:8971` |
| Nginx Proxy Manager | `http://HOST:81` |
| qBittorrent | `http://HOST:8080` |
| Duplicati | `http://HOST:8200` |
| Radarr | `http://HOST:7878` |
| Sonarr | `http://HOST:8989` |
| Seerr | `http://HOST:5055` |
| Prowlarr | `http://HOST:9696` |
| Immich | `http://HOST:2283` |
| OpenVPN | UDP `HOST:1194` (client `.ovpn` file) |

---

## Overview

- **Storage**: Bind mounts under `PRIMARY_PARTITION` (config, DBs, apps, Frigate recordings) and `SECONDARY_PARTITION` (media libraries, downloads). Set both in `.env`.
- **Networks**:
  - **`host`**: Plex, Home Assistant, go2rtc, Nginx Proxy Manager — use the host’s ports directly.
  - **`my-network`**: Frigate, qBittorrent, OpenVPN — can resolve each other by container name.
  - **Default project bridge**: Duplicati, Radarr, Sonarr, Seerr, Prowlarr, Immich (+ Valkey/Postgres) — published ports only; not on `my-network`.
- **Cameras**: go2rtc owns camera connections and restreams on **8554** / **8555**. Frigate consumes those restreams (does not publish 8554/8555).
- **Restart**: Most services use `restart: always`; Frigate, Duplicati, Radarr, Sonarr, Seerr, and Prowlarr use `restart: unless-stopped`.

---

## Active services

### Plex Media Server

**Image:** `lscr.io/linuxserver/plex:latest`  
**Purpose:** Organize and stream TV, movies, and documentaries.

| Item | Value |
|------|--------|
| Network | `host` |
| Ports | **32400** (web/API; standard Plex port on host) |
| Config | `${PRIMARY_PARTITION}/plex/library` → `/config` |
| Libraries | `${SECONDARY_PARTITION}/plex_data/tv` → `/tv` |
| | `${SECONDARY_PARTITION}/plex_data/movies` → `/movies` |
| | `${SECONDARY_PARTITION}/plex_data/doc` → `/documentaries` |

**Environment:** `PUID=1000`, `PGID=1000`, `VERSION=docker`, `PLEX_CLAIM` (from [plex.tv/claim](https://www.plex.tv/claim)), plus partition vars passed through.

**Access:** `http://HOST:32400/web` or Plex apps after claiming the server.

**Notes:** Log rotation — JSON driver, max 3 files × 10 MB.

---

### Home Assistant

**Image:** `homeassistant/home-assistant:latest`  
**Purpose:** Smart home automation and device control.

| Item | Value |
|------|--------|
| Network | `host` |
| Ports | **8123** (default UI on host) |
| Config | `${PRIMARY_PARTITION}/hass` → `/config` |
| Serial | `/dev/serial/by-id/` mounted for USB adapters (Z-Wave, Zigbee, etc.) |
| Device | `/dev/ttyACM0` |

**Environment:** `TZ=Europe/Athens`

**Access:** `http://HOST:8123`

**Notes:** Runs **privileged** for hardware access. Adjust `devices` if your USB serial path differs. For Frigate automations, add an MQTT broker and enable MQTT in Frigate + the [Frigate HA integration](https://docs.frigate.video/integrations/home-assistant/).

---

### go2rtc

**Image:** `alexxit/go2rtc`  
**Purpose:** Camera ingest and restream (RTSP/WebRTC) for Home Assistant, Frigate, and browsers.

| Item | Value |
|------|--------|
| Network | `host` |
| Ports | **1984** (UI/API), **8554** (RTSP), **8555** (WebRTC) |
| Config | [`go2rtc/go2rtc.yaml`](go2rtc/go2rtc.yaml) → `/config` |

**Environment:** Loaded from `.env` (`GO2RTC_*`, camera `*_URL` vars). `TZ=Europe/Athens`.

**Access:** `http://HOST:1984` — RTSP restreams at `rtsp://HOST:8554/<stream_name>` (e.g. `fata`, `spate`, `sopru`, `pod`, `terasa`).

**Notes:** Camera credentials live in `.env`, not in git. WebRTC candidate IP is set in `go2rtc.yaml`.

---

### Frigate

**Image:** `ghcr.io/blakeblackshear/frigate:stable`  
**Purpose:** NVR with object detection; cameras come from go2rtc restreams.

| Item | Value |
|------|--------|
| Network | `my-network` (+ `host.docker.internal` → host gateway) |
| Ports | **8971** (authenticated UI/API) — **not** 8554/8555 (owned by go2rtc) |
| Config DB | `${PRIMARY_PARTITION}/frigate` → `/config` |
| Config file | [`frigate/config.yml`](frigate/config.yml) → `/config/config.yml` |
| Media | `${PRIMARY_PARTITION}/frigate_media` → `/media/frigate` |
| Devices | `/dev/dri` (Intel VAAPI + OpenVINO GPU) |
| shm | `512mb` |

**Environment:** `TZ=Europe/Athens`; RTSP auth from `GO2RTC_RTSP_*` (mapped into Frigate as `FRIGATE_RTSP_*`).

**Access:** `http://HOST:8971` — on first start, admin password is printed in `docker logs frigate`.

**Notes:**

- MQTT is disabled in config until you add a broker (needed for Home Assistant integration).
- Uses OpenVINO on GPU and `preset-vaapi` for decode (ThinkCentre / Intel UHD style hosts).
- Do not map Frigate’s 8554/8555 while standalone go2rtc is running.

---

### qBittorrent

**Image:** `lscr.io/linuxserver/qbittorrent:latest`  
**Purpose:** BitTorrent client with web UI; downloads feed Plex libraries and *arr apps.

| Item | Value |
|------|--------|
| Network | `my-network` |
| Ports | **8080** (web UI), **6881/tcp** and **6881/udp** (BitTorrent) |
| Config | `${PRIMARY_PARTITION}/qbt` → `/config` |
| Downloads | `${SECONDARY_PARTITION}/downloads` → `/downloads` |
| Media | Same TV/movies/doc paths as Plex (`/tv`, `/movies`, `/doc`) |

**Environment:** `PUID=1000`, `PGID=1000`, `TZ=Europe/Athens`, `WEBUI_PORT=8080`

**Access:** `http://HOST:8080` — change default credentials (`admin` / `adminadmin`) on first login.

---

### Nginx Proxy Manager

**Image:** `jc21/nginx-proxy-manager:latest`  
**Purpose:** Reverse proxy, host-based routing, and Let’s Encrypt SSL from a web UI.

| Item | Value |
|------|--------|
| Network | `host` |
| Ports | **80** (HTTP), **443** (HTTPS), **81** (admin UI) |
| Data | `${PRIMARY_PARTITION}/nginx/data` → `/data` |
| Certificates | `${PRIMARY_PARTITION}/nginx/letsencrypt` → `/etc/letsencrypt` |

**Access:** `http://HOST:81` — default login `admin@example.com` / `changeme` (change immediately).

**Notes:** Point public DNS at this host and create proxy hosts in the UI for services you want on HTTPS (e.g. Seerr, Frigate, Immich).

---

### OpenVPN

**Image:** `kylemanna/openvpn`  
**Purpose:** Remote access to the home network.

| Item | Value |
|------|--------|
| Network | `my-network` |
| Ports | **1194/udp** |
| Volume | Named volume `ovpn-data-nas` (external) → `/etc/openvpn` |
| Capabilities | `NET_ADMIN` |
| Device | `/dev/net/tun` |

**Environment:** `OVPN_DATA` references `${PRIMARY_PARTITION}/openvpn` for setup scripts; runtime config uses the Docker volume `ovpn-data-nas`.

**Access:** Clients use generated `.ovpn` profiles (see [OpenVPN setup](#openvpn-setup) below).

---

### Duplicati

**Image:** `lscr.io/linuxserver/duplicati:latest`  
**Purpose:** Encrypted incremental backups to cloud or remote targets.

| Item | Value |
|------|--------|
| Network | default bridge |
| Ports | **8200** → `8200` |
| Config | `${PRIMARY_PARTITION}/duplicati/config` → `/config` |
| Backup source | `${PRIMARY_PARTITION}` → `/source` (entire primary partition visible in UI) |

**Environment:** `PUID=1000`, `PGID=1000`, `TZ=Etc/UTC`, `SETTINGS_ENCRYPTION_KEY`, `DUPLICATI__WEBSERVICE_PASSWORD`

**Access:** `http://HOST:8200`

**Notes:** `restart: unless-stopped` — stays stopped if you stop it manually.

---

### Radarr

**Image:** `lscr.io/linuxserver/radarr:latest`  
**Purpose:** Movie collection manager; integrates with download clients and Plex.

| Item | Value |
|------|--------|
| Network | default bridge |
| Ports | **7878** → `7878` |
| Config | `${PRIMARY_PARTITION}/radarr/data` → `/config` |
| Movies | `${SECONDARY_PARTITION}/plex_data/movies` → `/movies` |
| Downloads | `${SECONDARY_PARTITION}/downloads` → `/downloads` |

**Environment:** `PUID=1000`, `PGID=1000`, `TZ=Etc/UTC`

**Access:** `http://HOST:7878`

**Setup:** Add qBittorrent as download client (`HOST:8080` or container IP), set root folder `/movies`, connect Plex in Settings → Connect. Indexers via Prowlarr.

---

### Sonarr

**Image:** `lscr.io/linuxserver/sonarr:latest`  
**Purpose:** TV series manager; same workflow as Radarr for shows.

| Item | Value |
|------|--------|
| Network | default bridge |
| Ports | **8989** → `8989` |
| Config | `${PRIMARY_PARTITION}/sonarr/data` → `/config` |
| TV | `${SECONDARY_PARTITION}/plex_data/tv` → `/tv` |
| Downloads | `${SECONDARY_PARTITION}/downloads` → `/downloads` |

**Environment:** `PUID=1000`, `PGID=1000`, `TZ=Etc/UTC`

**Access:** `http://HOST:8989`

**Setup:** Point download client at qBittorrent; root folder `/tv`; link Plex in Connect. Indexers via Prowlarr.

---

### Seerr

**Image:** `ghcr.io/seerr-team/seerr:latest`  
**Purpose:** Request and discover movies/TV (Overseerr successor); talks to Plex, Radarr, and Sonarr.

| Item | Value |
|------|--------|
| Network | default bridge |
| Ports | **5055** → `5055` |
| Config | `${PRIMARY_PARTITION}/seer` → `/app/config` |

**Environment:** `LOG_LEVEL=debug`, `TZ=Asia/Tashkent`, `PORT=5055`

**Health check:** HTTP `GET /api/v1/settings/public` on port 5055 inside container.

**Access:** `http://HOST:5055`

**Setup:** On first run, connect Plex, then Radarr (`http://HOST:7878`) and Sonarr (`http://HOST:8989`). Use host LAN IP or Docker host gateway IP from containers if discovery fails.

---

### Prowlarr

**Image:** `lscr.io/linuxserver/prowlarr:latest`  
**Purpose:** Indexer manager for Radarr/Sonarr (and other *arr apps).

| Item | Value |
|------|--------|
| Network | default bridge |
| Ports | **9696** → `9696` |
| Config | `${PRIMARY_PARTITION}/prowlarr` → `/config` |

**Environment:** `PUID=1000`, `PGID=1000`, `TZ=Europe/Bucharest`

**Access:** `http://HOST:9696`

**Setup:** Add indexers in Prowlarr, then sync apps to Radarr/Sonarr.

---

### Immich

**Images:** `immich-server`, `immich-machine-learning` (OpenVINO variant), Valkey `redis`, Immich Postgres `database`  
**Purpose:** Self-hosted photo and video backup with ML features (faces, search).

| Item | Value |
|------|--------|
| Network | default project bridge |
| Ports | **2283** (server UI/API) |
| Upload data | `${PRIMARY_PARTITION}/${IMMICH_UPLOAD_LOCATION}` → `/data` |
| Postgres | `${PRIMARY_PARTITION}/${IMMICH_DB_DATA_LOCATION}` → `/var/lib/postgresql/data` |
| Devices | `/dev/dri` on server and ML containers |

**Environment:** See Immich block in [`.env.example`](.env.example) (`IMMICH_*`). ML image tag uses `IMMICH_ML_HWACCEL` (default `openvino`).

**Access:** `http://HOST:2283`

**Notes:** Follow [Immich docs](https://immich.app/docs) for first-run admin user.

---

## Configuration

### Environment variables

Copy [`.env.example`](.env.example) to `.env` and fill in values. Variables are grouped by service in the example file.

**Shared**

| Variable | Purpose |
|----------|---------|
| `COMPOSE_PROJECT_NAME` | Docker Compose project name (default `nas`) |
| `PRIMARY_PARTITION` | Config, DBs, app data |
| `SECONDARY_PARTITION` | Media libraries and downloads |
| `DOMAIN` | Optional (OpenVPN client generation, public hostnames) |

**Plex**

| Variable | Purpose |
|----------|---------|
| `PLEX_CLAIM` | First-time server claim |

**go2rtc**

| Variable | Purpose |
|----------|---------|
| `GO2RTC_API_USERNAME` / `GO2RTC_API_PASSWORD` | Web UI / API auth |
| `GO2RTC_RTSP_USERNAME` / `GO2RTC_RTSP_PASSWORD` | RTSP restream auth (also used by Frigate) |
| `FATA_URL`, `SPATE_URL`, `SOPRU_URL`, `POD_URL`, `TERASA_URL` | Camera RTSP source URLs |

**Duplicati**

| Variable | Purpose |
|----------|---------|
| `DUPLICATI_PASSWORD` | Web UI (`DUPLICATI__WEBSERVICE_PASSWORD`) |
| `SETTINGS_ENCRYPTION_KEY` | Settings encryption |

**Immich**

| Variable | Purpose |
|----------|---------|
| `IMMICH_VERSION` | Image tag (default `release`) |
| `IMMICH_UPLOAD_LOCATION` | Upload path under primary partition |
| `IMMICH_DB_DATA_LOCATION` | Postgres data path under primary partition |
| `IMMICH_TZ` | Timezone |
| `IMMICH_DB_PASSWORD` / `IMMICH_DB_USERNAME` / `IMMICH_DB_DATABASE_NAME` | Postgres credentials |
| `IMMICH_DB_STORAGE_TYPE` | e.g. `HDD` |
| `IMMICH_ML_HWACCEL` | ML image suffix (default `openvino`) |

### External Docker volume (OpenVPN)

Create before first `docker compose up` if not present:

```bash
docker volume create ovpn-data-nas
```

### Volume layout

```
${PRIMARY_PARTITION}/
├── plex/library/
├── hass/
├── frigate/                 # DB / models
├── frigate_media/           # recordings / clips / exports
├── qbt/
├── nginx/data/
├── nginx/letsencrypt/
├── duplicati/config/
├── radarr/data/
├── sonarr/data/
├── seer/                    # Seerr config (compose path name)
├── prowlarr/
├── immich/library/          # IMMICH_UPLOAD_LOCATION
├── immich/postgres/         # IMMICH_DB_DATA_LOCATION
└── openvpn/                 # used by OVPN setup scripts (OVPN_DATA)

${SECONDARY_PARTITION}/
├── plex_data/tv/
├── plex_data/movies/
├── plex_data/doc/
└── downloads/

./go2rtc/                    # go2rtc.yaml (repo)
./frigate/config.yml         # Frigate cameras / detectors (repo)
```

---

## Installation

### Prerequisites

- Docker Engine and Docker Compose plugin
- Disk space on primary and secondary paths
- UID/GID **1000** for LinuxServer images (or change `PUID`/`PGID` in compose)
- For OpenVPN: kernel TUN (`/dev/net/tun`) and external volume `ovpn-data-nas`
- For Frigate / Immich ML on Intel: `/dev/dri` available on the host

### Steps

1. Clone the repo and enter the directory.

2. Create `.env` from the example and set paths and secrets:

   ```bash
   cp .env.example .env
   nano .env
   ```

3. Create data directories (adjust paths to match `.env`):

   ```bash
   mkdir -p "${PRIMARY_PARTITION}"/{plex/library,hass,frigate,frigate_media,qbt,nginx/{data,letsencrypt},duplicati/config,radarr/data,sonarr/data,seer,prowlarr,immich/{library,postgres}}
   mkdir -p "${SECONDARY_PARTITION}"/{plex_data/{tv,movies,doc},downloads}
   ```

4. Set ownership for LinuxServer containers (if needed):

   ```bash
   sudo chown -R 1000:1000 "${PRIMARY_PARTITION}" "${SECONDARY_PARTITION}"
   ```

5. Create OpenVPN volume and start the stack:

   ```bash
   docker volume create ovpn-data-nas
   docker compose up -d
   ```

6. Verify:

   ```bash
   docker compose ps
   docker compose logs -f SERVICE_NAME
   ```

7. Frigate first login: `docker logs frigate` for the generated admin password, then open `http://HOST:8971`.

---

## OpenVPN setup

Uses volume `ovpn-data-nas` mounted at `/etc/openvpn`. Example flow (set `DOMAIN` in `.env`):

```bash
export OVPN_DATA="${PRIMARY_PARTITION}/openvpn"
docker volume create ovpn-data-nas

docker run -v ovpn-data-nas:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://"${DOMAIN}"
docker run -v ovpn-data-nas:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
docker run -v ovpn-data-nas:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full "${DOMAIN}" nopass
docker run -v ovpn-data-nas:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient "${DOMAIN}" > "${DOMAIN}.ovpn"
```

Ensure the `openvpn` service is running and UDP **1194** is forwarded on your router if clients connect from the internet.

---

## Maintenance

**Updates**

```bash
docker compose pull
docker compose up -d
docker image prune
```

**Backups**

- Config and DBs: `${PRIMARY_PARTITION}` (Duplicati can backup `/source` → primary partition)
- Large media: plan separately for `${SECONDARY_PARTITION}`
- Frigate recordings live under `${PRIMARY_PARTITION}/frigate_media` (included in primary backups if Duplicati covers `/source`)
- Immich Postgres: periodic dumps if not fully covered by Duplicati

**Logs**

```bash
docker compose logs -f
docker compose logs -f SERVICE_NAME
docker compose restart SERVICE_NAME
docker compose up -d --force-recreate SERVICE_NAME
```

---

## Security

1. Change default passwords (qBittorrent, Nginx Proxy Manager, Duplicati, Frigate admin).
2. Prefer HTTPS via Nginx Proxy Manager for web UIs accessed remotely.
3. Restrict host firewall to needed ports; VPN for admin access is preferable to wide port forwarding.
4. Keep images updated (`docker compose pull`).
5. OpenVPN: use strong PKI; protect `.ovpn` files.
6. Protect go2rtc API/RTSP credentials; camera URLs in `.env` contain secrets — do not commit `.env`.

---

## Troubleshooting

| Issue | Things to check |
|-------|------------------|
| Permission denied on volumes | `PUID`/`PGID` 1000; `chown` on mount paths |
| Port already in use | `ss -tulpn \| grep PORT` on the host (go2rtc uses 1984/8554/8555) |
| *arr / Seerr cannot reach qBittorrent | Use host IP and port **8080**, not container name (different networks) |
| Plex not visible | Claim with fresh `PLEX_CLAIM`; host networking and firewall on **32400** |
| OpenVPN fails to start | `ovpn-data-nas` exists; `/dev/net/tun`; `NET_ADMIN` |
| Frigate cameras offline | go2rtc up; `GO2RTC_RTSP_*` set; stream names in `frigate/config.yml` |
| Frigate / Immich GPU errors | `/dev/dri` present; OpenVINO/VAAPI supported on the CPU/iGPU |
| Immich won't start | `IMMICH_DB_PASSWORD` set; Postgres path writable |

---

## License

[Add your license information here]

## Contributing

[Add contribution guidelines if applicable]
