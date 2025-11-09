# Home Server Setup

This Docker Compose configuration provides a comprehensive home server setup with various services for media streaming, home automation, file downloading, photo management, web hosting, database management, backup solutions, VPN access, and more.

## Overview

This setup uses Docker Compose to orchestrate multiple containerized services, providing a self-hosted alternative to cloud services. The configuration has been optimized to remove redundancies and improve maintainability.

### Architecture Highlights

- **Network Configuration**: Services use either `host` network mode (for services requiring direct host network access) or a custom bridge network (`my-network`) for inter-service communication
- **Volume Management**: Persistent data is stored using bind mounts to paths defined by `PRIMARY_PARTITION` and `SECONDARY_PARTITION` environment variables
- **Restart Policies**: Most services use `restart: always` to ensure automatic recovery, with `duplicati` using `restart: unless-stopped` for more controlled behavior

## Active Services

### Plex Media Server

[Plex](https://www.plex.tv/) is a media server for organizing and streaming media content to various devices.

**Configuration:**
- **Network**: Host mode (direct access to host network)
- **Ports**: Uses host networking (default port 32400)
- **Volumes**:
  - Configuration: `${PRIMARY_PARTITION}/plex/library`
  - Media libraries: TV shows, movies, documentaries, and camera footage from `${SECONDARY_PARTITION}/plex_data/`
- **Environment Variables**:
  - `PUID=1000` / `PGID=1000`: User/group IDs for file permissions
  - `PLEX_CLAIM`: Required token to claim your server (get from [plex.tv/claim](https://www.plex.tv/claim))
- **Logging**: JSON file driver with rotation (max 3 files, 10MB each)

### Home Assistant

[Home Assistant](https://www.home-assistant.io/) is an open-source platform for smart home automation, allowing you to control and automate IoT devices.

**Configuration:**
- **Network**: Host mode (required for device discovery)
- **Privileged Mode**: Enabled (required for hardware access)
- **Volumes**:
  - Configuration: `${PRIMARY_PARTITION}/hass`
  - Serial devices: `/dev/serial/by-id/` (for Z-Wave, Zigbee, etc.)
- **Devices**: Direct access to `/dev/ttyACM0` (USB serial devices)
- **Timezone**: Europe/Athens

### qBittorrent

[qBittorrent](https://www.qbittorrent.org/) is a BitTorrent client with a web-based interface for downloading and managing torrents.

**Configuration:**
- **Network**: Custom bridge network (`my-network`)
- **Ports**:
  - `8080`: Web UI
  - `6881/tcp` and `6881/udp`: BitTorrent protocol (peer connections)
- **Volumes**:
  - Configuration: `${PRIMARY_PARTITION}/qbt`
  - Downloads: `${SECONDARY_PARTITION}/downloads`
  - Media folders: Shared with Plex (TV, movies, documentaries)
- **Environment Variables**:
  - `PUID=1000` / `PGID=1000`: File ownership
  - `TZ=Europe/Athens`: Timezone
  - `WEBUI_PORT=8080`: Web interface port

### PhotoPrism

[PhotoPrism](https://photoprism.org/) is an AI-powered photo management application that automatically organizes and tags your photos.

**Configuration:**
- **Network**: Custom bridge network (`my-network`)
- **Port**: `2342` (web interface)
- **Database**: Connects to MariaDB for metadata storage
- **Volumes**:
  - Storage: `${PRIMARY_PARTITION}/photoprism`
  - Originals: `${PRIMARY_PARTITION}/photoprism_raw`
- **Environment Variables**:
  - `PHOTOPRISM_ADMIN_PASSWORD`: Admin account password
  - `PHOTOPRISM_DATABASE_DSN`: MySQL connection string
  - `PHOTOPRISM_UPLOAD_NSFW=true`: Allows NSFW content uploads

### Nginx Proxy Manager

[Nginx Proxy Manager](https://nginxproxymanager.com/) provides a web-based interface for managing Nginx reverse proxy configurations, including SSL certificate management via Let's Encrypt.

**Configuration:**
- **Network**: Host mode (required for SSL termination and port management)
- **Volumes**:
  - Data: `${PRIMARY_PARTITION}/nginx/data`
  - SSL Certificates: `${PRIMARY_PARTITION}/nginx/letsencrypt`

### MariaDB

[MariaDB](https://mariadb.org/) is a relational database server, currently used by PhotoPrism for metadata storage.

**Configuration:**
- **Network**: Custom bridge network (`my-network`)
- **Port**: `3306` (MySQL/MariaDB standard port)
- **Volumes**: Database files stored in `${PRIMARY_PARTITION}/mysql`
- **Environment Variables**:
  - `MARIADB_ROOT_PASSWORD`: Root user password
  - `MARIADB_PASSWORD`: Password for the `passbolt` user
  - `MYSQL_DATABASE=passbolt`: Database name (prepared for Passbolt if enabled)
  - `MYSQL_USER=passbolt`: Database user

### OpenVPN

[OpenVPN](https://openvpn.net/) provides a VPN server for secure remote access to your home network.

**Configuration:**
- **Network**: Custom bridge network (`my-network`)
- **Port**: `1194/udp` (OpenVPN standard port)
- **Volumes**: Configuration and certificates in `${PRIMARY_PARTITION}/openvpn`
- **Capabilities**: `NET_ADMIN` (required for network configuration)
- **Devices**: `/dev/net/tun` (TUN/TAP interface)

### Duplicati

[Duplicati](https://www.duplicati.com/) is a backup solution that supports encrypted, incremental backups to various storage backends (cloud storage, network shares, etc.).

**Configuration:**
- **Network**: Default bridge network
- **Port**: `8200` (web interface)
- **Volumes**:
  - Configuration: `${PRIMARY_PARTITION}/duplicati/config`
  - Source: `${PRIMARY_PARTITION}` (entire primary partition available for backup)
- **Environment Variables**:
  - `PUID=1000` / `PGID=1000`: File ownership
  - `TZ=Etc/UTC`: UTC timezone (recommended for backups)
  - `SETTINGS_ENCRYPTION_KEY`: Encryption key for backup settings
  - `DUPLICATI__WEBSERVICE_PASSWORD`: Web interface password
- **Restart Policy**: `unless-stopped` (allows manual stopping without auto-restart)

### Immich

[Immich](https://immich.app/) is a self-hosted photo and video backup solution with features similar to Google Photos, including automatic backup, face recognition, and object detection.

**Components:**
1. **immich-server**: Main application server
   - **Port**: `2283` (API and web interface)
   - **Volumes**: Upload location from `${PRIMARY_PARTITION}/${UPLOAD_LOCATION}`
   - **Dependencies**: Requires `redis` and `database` services

2. **immich-machine-learning**: ML service for face recognition and object detection
   - **Volumes**: Model cache (Docker volume `model-cache`)
   - **Note**: Can be configured for hardware acceleration (CUDA, OpenVINO, etc.)

3. **redis**: In-memory data store (using Valkey fork)
   - Used for caching and job queues
   - Health check: Redis ping command

4. **database**: PostgreSQL database with vector extension
   - **Image**: Custom Immich PostgreSQL with pgvector extension
   - **Volumes**: Database files from `${PRIMARY_PARTITION}/${DB_DATA_LOCATION}`
   - **Shared Memory**: 128MB (required for PostgreSQL)

**Configuration**: Immich services use an `.env` file for configuration. See [Immich documentation](https://immich.app/docs) for required environment variables.

## Optional Services (Currently Commented Out)

The following services are defined but commented out in `docker-compose.yml`. Uncomment them if needed:

### Passbolt
Self-hosted password manager. Requires MariaDB and SMTP configuration.

### n8n
Workflow automation tool for connecting different services and APIs.

### rclone
Command-line program for managing files on cloud storage services.

## Configuration

### Environment Variables

The setup utilizes a `.env` file for managing environment variables. Create a `.env` file in the project root with the following variables:

#### Required Variables

- **`PRIMARY_PARTITION`**: Primary storage path (e.g., `/mnt/storage` or `/media/primary`)
  - Used for: Service configurations, databases, application data
- **`SECONDARY_PARTITION`**: Secondary storage path (e.g., `/mnt/media` or `/media/secondary`)
  - Used for: Media files, downloads, large data files

#### Service-Specific Variables

- **Plex**:
  - `PLEX_CLAIM`: Claim token from [plex.tv/claim](https://www.plex.tv/claim)

- **PhotoPrism**:
  - `PHOTOPRISM_ADMIN_PASSWORD`: Admin password for PhotoPrism
  - `PHOTOPRISM_MYSQL_PASSWORD`: Password for PhotoPrism MySQL user
  - `MYSQL_DB_HOST`: MariaDB hostname (typically `mariadb`)

- **MariaDB**:
  - `MARIADB_ROOT_PASSWORD`: Root password for MariaDB
  - `MARIADB_PASSWORD`: Password for application database users

- **Duplicati**:
  - `SETTINGS_ENCRYPTION_KEY`: Encryption key for backup settings
  - `DUPLICATI_PASSWORD`: Web interface password

- **Immich**:
  - `IMMICH_VERSION`: Image version tag (defaults to `release`)
  - `UPLOAD_LOCATION`: Path relative to `PRIMARY_PARTITION` for uploaded media
  - `DB_PASSWORD`: PostgreSQL password
  - `DB_USERNAME`: PostgreSQL username
  - `DB_DATABASE_NAME`: PostgreSQL database name
  - `DB_DATA_LOCATION`: Path relative to `PRIMARY_PARTITION` for database files
  - Additional variables as per [Immich documentation](https://immich.app/docs/install/environment-variables)

- **OpenVPN**:
  - `DOMAIN`: Your domain name for VPN access

#### Optional Variables (for commented services)

- `EMAIL`: Email address for notifications
- `SMTP_PASSWORD`: SMTP server password
- `DOMAIN`: Domain name for services
- `NAME` / `SURNAME`: For Passbolt user creation

### Network Configuration

The setup uses two network modes:

1. **Host Mode** (`network_mode: host`): Services that need direct access to the host network
   - Plex, Home Assistant, Nginx Proxy Manager
   - These services bind directly to host ports

2. **Bridge Network** (`my-network`): Custom network for inter-service communication
   - qBittorrent, PhotoPrism, MariaDB, OpenVPN
   - Services can communicate using container names as hostnames

### Volume Structure

```
${PRIMARY_PARTITION}/
├── plex/library/          # Plex configuration and metadata
├── hass/                  # Home Assistant configuration
├── qbt/                   # qBittorrent configuration
├── photoprism/            # PhotoPrism storage
├── photoprism_raw/        # PhotoPrism original photos
├── nginx/
│   ├── data/              # Nginx Proxy Manager data
│   └── letsencrypt/       # SSL certificates
├── mysql/                 # MariaDB database files
├── openvpn/               # OpenVPN configuration and certificates
├── duplicati/config/      # Duplicati configuration
└── [immich paths]/        # Immich uploads and database

${SECONDARY_PARTITION}/
├── plex_data/
│   ├── tv/                # TV shows library
│   ├── movies/            # Movies library
│   ├── doc/               # Documentaries library
│   └── cameras/           # Camera footage
└── downloads/             # qBittorrent downloads
```

## Installation Instructions

### Prerequisites

- Docker and Docker Compose installed
- Sufficient disk space for your media and data
- Proper file permissions on storage partitions

### Setup Steps

1. **Clone this repository:**
   ```bash
   git clone <repository-url>
   cd nas_config
   ```

2. **Create and configure `.env` file:**
   ```bash
   # Copy example if available, or create new .env file
   # Edit .env and set all required variables
   nano .env
   ```

3. **Create necessary directories:**
   ```bash
   # Ensure PRIMARY_PARTITION and SECONDARY_PARTITION directories exist
   mkdir -p ${PRIMARY_PARTITION}/{plex/library,hass,qbt,photoprism,photoprism_raw,nginx/{data,letsencrypt},mysql,openvpn,duplicati/config}
   mkdir -p ${SECONDARY_PARTITION}/{plex_data/{tv,movies,doc,cameras},downloads}
   ```

4. **Set proper permissions:**
   ```bash
   # Adjust ownership if needed (PUID=1000, PGID=1000)
   sudo chown -R 1000:1000 ${PRIMARY_PARTITION} ${SECONDARY_PARTITION}
   ```

5. **Start services:**
   ```bash
   docker-compose up -d
   ```

6. **Check service status:**
   ```bash
   docker-compose ps
   docker-compose logs -f [service-name]
   ```

## Service Access

After starting the services, access them at:

- **Plex**: `http://your-server-ip:32400/web` (or use Plex apps)
- **Home Assistant**: `http://your-server-ip:8123`
- **qBittorrent**: `http://your-server-ip:8080` (default credentials: admin/adminadmin)
- **PhotoPrism**: `http://your-server-ip:2342/photoprism/`
- **Nginx Proxy Manager**: `http://your-server-ip:81` (default credentials: admin@example.com/changeme)
- **Duplicati**: `http://your-server-ip:8200`
- **Immich**: `http://your-server-ip:2283`
- **MariaDB**: `your-server-ip:3306` (for database connections)

## Additional Configuration

### OpenVPN Setup

1. **Initialize OpenVPN configuration:**
   ```bash
   export OVPN_DATA=${PRIMARY_PARTITION}/openvpn
   docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://${DOMAIN}
   docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
   ```

2. **Generate client certificate:**
   ```bash
   docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full ${DOMAIN} nopass
   ```

3. **Retrieve client configuration:**
   ```bash
   docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient ${DOMAIN} > ${DOMAIN}.ovpn
   ```

### Passbolt Setup (if enabled)

1. **Change ownership of Passbolt config folder:**
   ```bash
   sudo chown -R www-data:www-data ${PRIMARY_PARTITION}/passbolt
   ```

2. **Create admin user:**
   ```bash
   docker-compose exec passbolt su -m -c "/usr/share/php/passbolt/bin/cake \
                                    passbolt register_user \
                                    -u ${EMAIL} \
                                    -f ${NAME} \
                                    -l ${SURNAME} \
                                    -r admin" -s /bin/sh www-data
   ```

### rclone Setup (if enabled)

```bash
docker run -it \
    -v ${PRIMARY_PARTITION}/rclone:/config/rclone \
    -v ${PRIMARY_PARTITION}/gcs:/data:gcs \
    --net=host \
    --user $(id -u):$(id -g) \
    --privileged \
    rclone/rclone \
    config
```

## Maintenance

### Updating Services

```bash
# Pull latest images
docker-compose pull

# Recreate containers with new images
docker-compose up -d

# Remove unused images
docker image prune
```

### Backup Recommendations

- **Configuration files**: Backup `${PRIMARY_PARTITION}` (excluding large media files)
- **Database**: Use Duplicati or manual database dumps for MariaDB and PostgreSQL
- **Media files**: Consider separate backup strategy for `${SECONDARY_PARTITION}`

### Logs and Troubleshooting

```bash
# View logs for all services
docker-compose logs -f

# View logs for specific service
docker-compose logs -f [service-name]

# Check service status
docker-compose ps

# Restart a service
docker-compose restart [service-name]

# Recreate a service
docker-compose up -d --force-recreate [service-name]
```

## Security Considerations

⚠️ **Important Security Notes:**

1. **Change default passwords** immediately after first login (qBittorrent, Nginx Proxy Manager, etc.)
2. **Use strong passwords** for all services, especially databases
3. **Configure firewall rules** to restrict access to services
4. **Use Nginx Proxy Manager** with SSL certificates for external access
5. **Keep services updated** regularly
6. **Review file permissions** on mounted volumes
7. **Secure OpenVPN** with strong certificates and authentication
8. **Limit network exposure** - only expose necessary ports

## Recent Improvements

The Docker Compose configuration has been optimized by:

- **Removed redundant environment variables**: `PRIMARY_PARTITION` and `SECONDARY_PARTITION` were removed from service environment sections where they were only used in volume paths, not by the containers themselves
- **Fixed duplicate port mapping**: qBittorrent's port 6881 mapping was consolidated into explicit TCP and UDP mappings
- **Improved maintainability**: Cleaner configuration makes it easier to understand and modify

## Troubleshooting

### Common Issues

1. **Permission errors**: Ensure PUID/PGID match your user, or adjust file ownership
2. **Port conflicts**: Check if ports are already in use: `sudo netstat -tulpn | grep [port]`
3. **Network issues**: Verify network mode settings match service requirements
4. **Volume mount errors**: Ensure paths exist and have correct permissions
5. **Database connection issues**: Check if dependent services are running and network connectivity

### Getting Help

- Check service-specific documentation
- Review Docker Compose logs
- Verify environment variables are set correctly
- Ensure all prerequisites are met

## License

[Add your license information here]

## Contributing

[Add contribution guidelines if applicable]
