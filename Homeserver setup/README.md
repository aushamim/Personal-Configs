## Install Docker

## Create Directory Structure
```sh
mkdir -p /opt/homeserver
mkdir -p /mnt/data/syncthing
mkdir -p /mnt/data/immich/upload
mkdir -p /mnt/data/downloads
```

## Create Docker Compose File
```sh
nano /opt/homeserver/docker-compose.yml
```
Contents
```sh
services:

  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=0.0.0.0
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - GENERIC_TIMEZONE=Asia/Dhaka
    volumes:
      - n8n_data:/home/node/.n8n

  syncthing:
    image: lscr.io/linuxserver/syncthing:latest
    container_name: syncthing
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Dhaka
    ports:
      - "8384:8384"
      - "22000:22000/tcp"
      - "22000:22000/udp"
      - "21027:21027/udp"
    volumes:
      - syncthing_config:/config
      - /mnt/data/syncthing:/data

volumes:
  portainer_data:
  n8n_data:
  syncthing_config:
```

## Start Docker
```sh
cd /opt/homeserver
docker compose up -d
```
