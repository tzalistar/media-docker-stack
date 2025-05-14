# Complete Guide to Setting Up a Media Server Stack with GPU Acceleration


This guide provides a detailed walkthrough of setting up a comprehensive media server stack using Docker Compose with NVIDIA GPU acceleration. Each service and configuration option is explained to help you understand how the entire system works together.

> If you dont have a GPU or dont want to use it, skip the step for nvidia and nvidia container toolkit as well as remove the **deploy** block from plex service.


> You can remove any service that you dont want. They way i've set it up is a way that fits my setup best.


> If this stack is deployed as is in one machine, the containers can communicate with each other using the service name `http://radarr:7878`



## Overview of the Media Stack

Before diving into the details, here's a brief overview of what this media stack includes:

- **Plex**: Media server for streaming movies, TV shows, and other media
- **Tautulli**: Analytics and monitoring for your Plex server
- **Bazarr**: Automatic subtitle downloader
- **Radarr**: Movie collection manager and automation
- **Sonarr**: TV series collection manager and automation
- **Overseerr**: Request management system for Plex
- **Prowlarr**: Indexer manager that connects to torrent/usenet sites
- **Readarr**: E-book collection manager
- **Transmission-OpenVPN**: Torrent client protected by VPN
- **Flood**: Modern web UI for the Transmission torrent client

## Prerequisites

- Docker and Docker Compose installed
- NVIDIA GPU with appropriate drivers installed
- NVIDIA Container Toolkit installed (nvidia-docker)
- Sufficient storage for media files


## Understanding Docker Compose Version

```yaml
version: "3.8"
```

This specifies Docker Compose file format version 3.8, which includes support for GPU resources and other advanced features needed for this setup.

## Detailed Service Configuration

### Plex Media Server

```yaml
plex:
  container_name: plex-server
  image: lscr.io/linuxserver/plex:latest
  restart: unless-stopped
  runtime: nvidia
  privileged: true
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            device_ids: ['0']
            capabilities: [gpu]
  environment:
    - TZ=<TZ_COUNTRY>
    - PLEX_CLAIM=<PLEX_CLAIM_PASS>
    - NVIDIA_VISIBLE_DEVICES=all
    - NVIDIA_DRIVER_CAPABILITIES=compute,video,utility
    - CUDA_DRIVER_CAPABILITIES=compute,video,utility
    - VERSION=plexpass
    - PUID=1000
    - PGID=1000
  network_mode: host
  volumes:
    - <SHARE>/Plex/config:/config
    - <SHARE>/Plex/transcode:/transcode
    - <SHARE>/Shared/Movies:/Movies
    - <SHARE>/Shared/Series:/Series
```


#### What It Does

Plex Media Server organizes and streams your media to any device. With GPU acceleration, it can handle hardware transcoding for smoother streaming performance.

#### Configuration Options

- **container_name**: Names the container "plex-server" for easy reference
- **image**: Uses the LinuxServer.io Plex image
- **restart**: Automatically restarts container if it stops
- **runtime**: Sets the container to use the NVIDIA runtime
- **privileged**: Gives the container additional permissions (required for some GPU operations)


#### GPU Configuration

- **deploy > resources > reservations > devices**: Configures GPU access
    - **driver**: Specifies NVIDIA as the driver
    - **device_ids**: Targets specific GPU (in this case, GPU 0) - useful for multi-GPU systems
    - **capabilities**: Enables GPU computation capabilities


#### Environment Variables

- **TZ**: Your timezone (e.g., "Europe/London" or "America/New_York")
- **PLEX_CLAIM**: Claim token from https://plex.tv/claim (expires after 4 minutes)
- **NVIDIA_VISIBLE_DEVICES**: Makes all GPUs visible to the container
- **NVIDIA_DRIVER_CAPABILITIES**: Enables specific driver capabilities
    - **compute**: Required for CUDA applications
    - **video**: Required for video encoding/decoding
    - **utility**: Allows use of nvidia-smi and other tools
- **VERSION**: "plexpass" enables the latest Plex Pass features
- **PUID/PGID**: User/group IDs that own the container processes (usually 1000 for first user)


#### Volume Mappings

- **/config**: Stores Plex configuration files
- **/transcode**: Temporary location for transcoding operations
- **/Movies** and **/Series**: Your media libraries


#### Network Configuration

- **network_mode: host**: Uses the host's network directly, simplifies discovery


### Tautulli

```yaml
tautulli:
  image: ghcr.io/tautulli/tautulli
  container_name: tautulli
  restart: unless-stopped
  volumes:
    - <SHARE>/Plex/Tautulli/tautuli-config:/config
  environment:
    - PUID=1000
    - PGID=1000
    - TZ=<TZ_COUNTRY>
  ports:
    - 8181:8181
```


#### What It Does

Tautulli is a monitoring application that provides statistics and usage information about your Plex Media Server.

#### Configuration Options

- **container_name**: Names the container "tautulli"
- **image**: Uses the official Tautulli image
- **restart**: Ensures the container automatically restarts


#### Environment Variables

- **PUID/PGID**: User/group IDs (should match Plex)
- **TZ**: Timezone (should match Plex)


#### Volume Mappings

- **/config**: Stores Tautulli configuration and database


#### Port Mapping

- **8181**: Web interface port (access via http://your-server-ip:8181)


### Bazarr

```yaml
bazarr:
  image: lscr.io/linuxserver/bazarr:latest
  container_name: bazarr
  environment:
    - PUID=1000
    - PGID=1000
    - TZ=<TZ_COUNTRY>
  volumes:
    - <SHARE>/Servarr/Bazarr/bazarr-config:/config
    - <SHARE>/Shared/Movies:/Movies
    - <SHARE>/Shared/Series:/Series
  ports:
    - 6767:6767
  restart: unless-stopped
```


#### What It Does

Bazarr automatically manages and downloads subtitles for your media based on your preferences.

#### Configuration Options

- **container_name**: Names the container "bazarr"
- **image**: Uses the LinuxServer.io Bazarr image


#### Environment Variables

- **PUID/PGID**: User/group IDs (should match your other services)
- **TZ**: Timezone setting


#### Volume Mappings

- **/config**: Stores Bazarr configuration
- **/Movies** and **/Series**: Media libraries (must match the paths used in Radarr/Sonarr)


#### Port Mapping

- **6767**: Web interface port (access via http://your-server-ip:6767)


### Radarr

```yaml
radarr:
  container_name: radarr
  image: lscr.io/linuxserver/radarr:latest
  environment:
    - PUID=1000
    - PGID=1000
    - TZ=<TZ_COUNTRY>
  ports:
    - 7878:7878
  volumes:
    - <SHARE>/Servarr/Radarr/radarr-config:/config
    - <SHARE>/Shared/Movies:/Movies
    - <SHARE>/Shared/Torrents:/Torrents
  restart: "unless-stopped"
```


#### What It Does

Radarr is a movie collection manager that works with various download clients and Plex.

#### Configuration Options

- **container_name**: Names the container "radarr"
- **image**: Uses the LinuxServer.io Radarr image


#### Environment Variables

- **PUID/PGID**: User/group IDs
- **TZ**: Timezone setting


#### Volume Mappings

- **/config**: Stores Radarr configuration
- **/Movies**: Your movie library
- **/Torrents**: Access to downloaded content


#### Port Mapping

- **7878**: Web interface port (access via http://your-server-ip:7878)


### Sonarr

```yaml
sonarr:
  image: lscr.io/linuxserver/sonarr:latest
  container_name: sonarr
  environment:
    - PUID=1000
    - PGID=1000
    - TZ=<TZ_COUNTRY>
  volumes:
    - <SHARE>/Servarr/Sonarr/sonarr-config:/config
    - <SHARE>/Shared/Series:/Series
    - <SHARE>/Shared/Torrents:/Torrents
  ports:
    - 8989:8989
  restart: unless-stopped
```


#### What It Does

Sonarr is a TV series collection manager that works with various download clients and Plex.

#### Configuration Options

- **container_name**: Names the container "sonarr"
- **image**: Uses the LinuxServer.io Sonarr image


#### Environment Variables

- **PUID/PGID**: User/group IDs
- **TZ**: Timezone setting


#### Volume Mappings

- **/config**: Stores Sonarr configuration
- **/Series**: Your TV series library
- **/Torrents**: Access to downloaded content


#### Port Mapping

- **8989**: Web interface port (access via http://your-server-ip:8989)


### Overseerr

```yaml
overseerr:
  image: sctx/overseerr:latest
  container_name: overseerr
  environment:
    - LOG_LEVEL=debug
    - TZ=<TZ_COUNTRY>
    - PORT=5055 #optional
  ports:
    - 5055:5055
  volumes:
    - <SHARE>/Servarr/overseerr/overseerr-config:/app/config
  restart: unless-stopped
```


#### What It Does

Overseerr is a request management and media discovery tool for your Plex ecosystem[^10].

#### Configuration Options

- **container_name**: Names the container "overseerr"
- **image**: Uses the sctx Overseerr image


#### Environment Variables

- **LOG_LEVEL**: Sets logging detail level to debug
- **TZ**: Timezone setting
- **PORT**: Internal port for the application (optional)


#### Volume Mappings

- **/app/config**: Stores Overseerr configuration


#### Port Mapping

- **5055**: Web interface port (access via http://your-server-ip:5055)


### Prowlarr

```yaml
prowlarr:
  container_name: prowlarr
  image: lscr.io/linuxserver/prowlarr:latest
  environment:
    - PUID=1000
    - PGID=1000
    - TZ=<TZ_COUNTRY>
  volumes:
    - <SHARE>/Servarr/Prowlarr/prowlarr-config:/config
  ports:
    - 9696:9696
  restart: unless-stopped
```


#### What It Does

Prowlarr is an indexer manager/proxy that integrates with your media apps (Sonarr, Radarr, etc.).

#### Configuration Options

- **container_name**: Names the container "prowlarr"
- **image**: Uses the LinuxServer.io Prowlarr image


#### Environment Variables

- **PUID/PGID**: User/group IDs
- **TZ**: Timezone setting


#### Volume Mappings

- **/config**: Stores Prowlarr configuration


#### Port Mapping

- **9696**: Web interface port (access via http://your-server-ip:9696)


### Readarr

```yaml
readarr:
  image: lscr.io/linuxserver/readarr:develop
  container_name: readarr
  environment:
    - PUID=1000
    - PGID=1000
    - TZ=<TZ_COUNTRY>
  volumes:
    - <SHARE>/Servarr/Readarr/readarr-config:/config
    - <SHARE>/Shared/Torrents/Downloads:/downloads
  ports:
    - 8787:8787
  restart: unless-stopped
```


#### What It Does

Readarr is a book collection manager and automation tool (similar to Sonarr but for books).

#### Configuration Options

- **container_name**: Names the container "readarr"
- **image**: Uses the LinuxServer.io Readarr image (develop branch)


#### Environment Variables

- **PUID/PGID**: User/group IDs
- **TZ**: Timezone setting


#### Volume Mappings

- **/config**: Stores Readarr configuration
- **/downloads**: Access to downloaded content


#### Port Mapping

- **8787**: Web interface port (access via http://your-server-ip:8787)


### Transmission with OpenVPN

> If you choose so, you can use normal transmission docker that wont use VPN.


```yaml
transmission-openvpn:
  container_name: transmission-vpn
  cap_add:
    - NET_ADMIN
  volumes:
    - <SHARE>/Servarr/Transmission/transmission-opvn-config:/config
    - <SHARE>/Shared/:/Media
    - <SHARE>/Servarr/Transmission/transmission-watch:/watch
  environment:
    - OPENVPN_PROVIDER=<VPN_PROVIDER>
    - NORDVPN_COUNTRY=<COUNTRY_CODE>
    - OPENVPN_CONFIG=<COUNTRY>
    - OPENVPN_USERNAME=<OPENVPN_USERNAME>
    - NORDVPN_CATEGORY=legacy_p2p
    - OPENVPN_PASSWORD=<OPENVPN_PASSWORD>
    - LOCAL_NETWORK=192.168.0.0/16
    - USER=transmission
    - PASS=transm1ss1on
    - TZ=<TZ_COUNTRY>
    - TRANSMISSION_DOWNLOAD_DIR=/Media/Torrents/Downloads
    - TRANSMISSION_INCOMPLETE_DIR=/Media/Torrents/Incomplete
    - TRANSMISSION_WATCH_DIR=/watch
    - TRANSMISSION_RPC_PASSWORD=<RPC_PASSWORD>
    - TRANSMISSION_RPC_USERNAME=<RPC_USER>
    - PUID=1000
    - PGID=1000
  logging:
    driver: json-file
    options:
      max-size: 10m
  # ports:
  #   - '9092:9091'
  restart: unless-stopped
  image: haugene/transmission-openvpn
```


#### What It Does

Transmission-OpenVPN combines the Transmission BitTorrent client with a VPN connection for private downloading.

#### Configuration Options

- **container_name**: Names the container "transmission-vpn"
- **cap_add**: Adds the NET_ADMIN capability required for VPN
- **image**: Uses the haugene/transmission-openvpn image


#### Environment Variables

- **VPN Configuration**:
    - **OPENVPN_PROVIDER**: Your VPN provider name
    - **NORDVPN_COUNTRY**: Country code for NordVPN (if using Nord)
    - **OPENVPN_CONFIG**: Country/server configuration
    - **OPENVPN_USERNAME/PASSWORD**: VPN credentials
    - **NORDVPN_CATEGORY**: Service type for NordVPN
    - **LOCAL_NETWORK**: Your local network range (for bypassing VPN)
- **Transmission Configuration**:
    - **USER/PASS**: Username/password for Transmission web UI
    - **TRANSMISSION_DOWNLOAD_DIR**: Path for completed downloads
    - **TRANSMISSION_INCOMPLETE_DIR**: Path for in-progress downloads
    - **TRANSMISSION_WATCH_DIR**: Directory to monitor for torrent files
    - **TRANSMISSION_RPC_USERNAME/PASSWORD**: Authentication for API access
- **Container Configuration**:
    - **PUID/PGID**: User/group IDs
    - **TZ**: Timezone setting


#### Volume Mappings

- **/config**: Stores Transmission and OpenVPN configuration
- **/Media**: Access to your media directory
- **/watch**: Directory Transmission watches for new .torrent files


#### Logging Configuration

- **driver**: Uses JSON file logging
- **max-size**: Limits log file size to 10MB


#### Port Mapping

- The web interface port (9091) is commented out - access will be through Flood


### Flood

> You may opt out from using Flood and just use the transmission web gui or the standalone client. In that scenrarion you will need to expose the transmission service ports


```yaml
flood:
  container_name: flood-gui
  restart: unless-stopped
  image: jesec/flood
  user: "1000:1000"
  ports:
    - "9092:3000"
  environment:
    - FLOOD_OPTION_rundir=/config
    - FLOOD_OPTION_host=0.0.0.0
    - FLOOD_OPTION_port=3000
    - FLOOD_OPTION_secret="<SECRET>"
  volumes:
    - <SHARE>/Shared/:/Media
    - <SHARE>/Servarr/Transmission/transmission-opvn-config:/config
  depends_on:
    transmission-openvpn:
      condition: service_healthy
      restart: true
```


#### What It Does

Flood provides a modern, responsive web UI for the Transmission torrent client.

#### Configuration Options

- **container_name**: Names the container "flood-gui"
- **image**: Uses the jesec/flood image
- **user**: Sets the user ID to 1000:1000
- **depends_on**: Ensures it starts only after Transmission is healthy


#### Environment Variables

- **FLOOD_OPTION_rundir**: Configuration directory
- **FLOOD_OPTION_host**: Binds to all network interfaces
- **FLOOD_OPTION_port**: Internal port
- **FLOOD_OPTION_secret**: Secret key for session encryption


#### Volume Mappings

- **/Media**: Access to your media directory
- **/config**: Shares the Transmission configuration


#### Port Mapping

- **9092**: Web interface port (access via http://your-server-ip:9092)


## Setting Up the Media Stack

### Step 1: Prepare Your Environment

1. Install Docker and Docker Compose
2. Install NVIDIA drivers and NVIDIA Container Toolkit
3. Create required directory structure:

```bash
mkdir -p <SHARE>/Plex/{config,transcode} \
         <SHARE>/Plex/Tautulli/tautuli-config \
         <SHARE>/Servarr/{Bazarr,Radarr,Sonarr,Prowlarr,Readarr,overseerr}/*/config \
         <SHARE>/Servarr/Transmission/{transmission-opvn-config,transmission-watch} \
         <SHARE>/Shared/{Movies,Series,Torrents/{Downloads,Incomplete}}
```


### Step 2: Edit the Docker Compose File

1. Replace `<TZ_COUNTRY>` with your timezone (e.g., `Europe/London`)
2. Replace `<SHARE>` with your storage path (e.g., `/data` or `/mnt/storage`)
3. Get a Plex claim token from https://plex.tv/claim and replace `<PLEX_CLAIM_PASS>`
4. Configure your VPN settings by replacing:
    - `<VPN_PROVIDER>` (e.g., `nordvpn`)
    - `<COUNTRY_CODE>` (e.g., `US`)
    - `<COUNTRY>` (e.g., `us`)
    - `<OPENVPN_USERNAME>` and `<OPENVPN_PASSWORD>`
5. Set secure credentials for Transmission:
    - `<RPC_USER>` and `<RPC_PASSWORD>`
6. Create a secure random string for `<SECRET>` in the Flood configuration

### Step 3: Start the Media Stack

1. Start all services:

```bash
docker-compose up -d
```

2. Monitor the logs for any errors:

```bash
docker-compose logs -f
```


### Step 4: Configure Individual Services

1. **Plex** (http://your-server-ip:32400/web)
    - Follow the setup wizard to create libraries pointing to /Movies and /Series
2. **Tautulli** (http://your-server-ip:8181)
    - Connect to your Plex server
3. **Radarr** (http://your-server-ip:7878)
    - Add your Movies folder
    - Connect to Prowlarr and Transmission
4. **Sonarr** (http://your-server-ip:8989)
    - Add your Series folder
    - Connect to Prowlarr and Transmission
5. **Bazarr** (http://your-server-ip:6767)
    - Connect to Radarr and Sonarr
6. **Prowlarr** (http://your-server-ip:9696)
    - Add indexers (torrent sites)
    - Connect to Sonarr, Radarr, and Readarr
7. **Overseerr** (http://your-server-ip:5055)
    - Connect to Plex, Radarr, and Sonarr
8. **Readarr** (http://your-server-ip:8787)
    - Configure your book library
    - Connect to Prowlarr and Transmission
9. **Flood** (http://your-server-ip:9092)
    - Login with the credentials set in the compose file

## Troubleshooting NVIDIA GPU Integration

If you encounter issues with GPU acceleration:

1. Verify that NVIDIA drivers are properly installed:

```bash
nvidia-smi
```

2. Ensure NVIDIA Container Toolkit is installed:

```bash
dpkg -l | grep -E "nvidia-container-(runtime|toolkit)"
```

3. Check if Docker can access the GPU:

```bash
docker run --rm --gpus all nvidia/cuda:11.6.2-base-ubuntu20.04 nvidia-smi
```

4. If the above command works but Plex doesn't detect the GPU, check the container logs:

```bash
docker logs plex-server
```

1. Make sure the `NVIDIA_DRIVER_CAPABILITIES` includes the necessary capabilities (`compute,video,utility`)
2. 
## Conclusion

You now have a fully functional media server stack with GPU acceleration for optimal performance. The setup includes media management, automatic downloading, subtitle fetching, and a request system for users to request new content. The NVIDIA GPU integration allows hardware transcoding in Plex, significantly improving performance for multiple simultaneous streams.

