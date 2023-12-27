# Home Server Setup

This Docker Compose configuration provides a comprehensive home server setup with various services for media streaming, home automation, file downloading, photo management, web hosting, database management, password management, VPN access, workflow automation, and cloud storage.

## Services

### Plex

[Plex](https://www.plex.tv/) is a media server for organizing and streaming media content.

### Home Assistant

[Home Assistant](https://www.home-assistant.io/) is an open-source platform for smart home automation.

### qBittorrent

[qBittorrent](https://www.qbittorrent.org/) is a BitTorrent client for downloading and managing torrents.

### PhotoPrism

[PhotoPrism](https://photoprism.org/) is an application for managing and organizing photos.

### Nginx

[Nginx](https://nginx.org/) is a web server and reverse proxy server for handling web traffic.

### MariaDB

[MariaDB](https://mariadb.org/) is a relational database server used for Passbolt.

### Passbolt

[Passbolt](https://www.passbolt.com/) is a self-hosted password manager.

### OpenVPN

[OpenVPN](https://openvpn.net/) provides a VPN server for secure remote access.

### n8n

[n8n](https://n8n.io/) is a workflow automation tool.

### rclone

[rclone](https://rclone.org/) is a command-line program for managing files on cloud storage.

## Configuration

The `docker-compose.yml` file contains configuration settings for each service, including environment variables, volumes, ports, and network settings.

## Instructions

1. Clone this repository:

    ```bash
    git clone https://github.com/your-username/home-server.git
    cd home-server
    ```

2. Set environment variables:

   This setup utilizes a `.env` file for managing environment variables, making it easier to configure and maintain the Docker Compose services. The `.env.example` file contains placeholders for various environment variables, which should be defined with appropriate values. Here's an overview of some key variables:
   
   - `PLEX_CLAIM`: A unique claim token used for Plex Media Server to associate your server with your account.
   - `PRIMARY_PARTITION`: The primary path for storing persistent data used by services, such as media files for Plex.
   - `SECONDARY_PARTITION`: A secondary path for storing additional data, like backups or other media files.
   - `EMAIL`: Your email address, used for notifications or account setups for certain services.
   - `SMTP_PASSWORD`: The password for your SMTP server, used for sending email notifications.
   - `MARIADB_PASSWORD`: The password for the MariaDB database service.
   - `DOMAIN`: The domain name for accessing your services externally.
   - `PHOTOPRISM_ADMIN_PASSWORD`: The admin password for Photoprism, a photo management service.
   - `COMPOSE_PROJECT_NAME`: The name of the Docker Compose project, used for managing and isolating your Docker environment.
   - `NAME`: Your first name, used to generate passbolt user
   - `SURNAME`: Your last name, used to generate passbolt user
   
   Copy the `.env.example` file to a new file named `.env` and fill in the values for these variables to properly configure your services.
   
   ```bash
   source .env
   ```

3. Start services:

    ```bash
    docker-compose up -d
    ```

4. Access services:

   - Plex: [http://your-server-ip:32400](http://your-server-ip:32400)
   - Home Assistant: [http://your-server-ip:8123](http://your-server-ip:8123)
   - qBittorrent: [http://your-server-ip:8080](http://your-server-ip:8080)
   - PhotoPrism: [http://your-server-ip:2342/photoprism/](http://your-server-ip:2342/photoprism/)
   - Passbolt: [http://your-server-ip:8081](http://your-server-ip:8081)
   - OpenVPN: Configured to run on port 1194, ensure the necessary VPN client configurations are generated.
   - n8n: Access n8n at the configured host and port.

## Additional Instructions

- Change Ownership of Passbolt Config Folder to www-data:www-data
- Create an admin user
```bash
docker-compose exec passbolt su -m -c "/usr/share/php/passbolt/bin/cake \
                                 passbolt register_user \
                                 -u ${EMAIL} \
                                 -f ${NAME} \
                                 -l ${SURNAME} \
                                 -r admin" -s /bin/sh www-data
```

- Initialize the \$OVPN_DATA Container
This container will hold the configuration files and certificates. It will prompt for a passphrase to protect the private key used by the newly generated certificate authority.
```bash
docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://${DOMAIN}
docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
```

- Generate a Client Certificate Without a Passphrase
```bash
docker run -v $OVPN_DATA:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full ${DOMAIN} nopass
```

- Retrieve the Client Configuration with Embedded Certificates
```bash
docker run -v $OVPN_DATA:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient ${DOMAIN} > ${DOMAIN}.ovpn
```

- Configure rclone
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

## Note

Make sure to secure your services by configuring appropriate authentication, access controls, and keeping your software up to date.
