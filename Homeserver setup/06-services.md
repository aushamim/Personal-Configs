# 06 — Services Setup

All services except Portainer are managed as a single Stack in Portainer. This makes it easy to add, remove, or update services through the web UI.

## Creating the Stack in Portainer

1. Open Portainer at `http://192.168.x.x:9000`
2. Left panel → **Stacks** → **Add stack**
3. Name it `homeserver`
4. Select **Web editor**
5. Paste the compose content (see below)
6. Click **Deploy the stack**

## Current docker-compose.yml

```yaml
services:
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
      - N8N_SECURE_COOKIE=false
    volumes:
      - n8n_data:/home/node/.n8n

  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    restart: unless-stopped
    privileged: true
    network_mode: host
    environment:
      - TZ=Asia/Dhaka
    volumes:
      - ha_config:/config

  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - homepage_config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - TZ=Asia/Dhaka
      - HOMEPAGE_ALLOWED_HOSTS=192.168.x.x:3000 # Add tailscale IP too: 100.x.x.x:3000

volumes:
  n8n_data:
  ha_config:
  homepage_config:
```

## Service Details

### N8N — Automation

| Setting | Value                     |
| ------- | ------------------------- |
| Port    | 5678                      |
| Access  | `http://192.168.x.x:5678` |

- `N8N_SECURE_COOKIE=false` — required when accessing over HTTP (not HTTPS). Without this N8N refuses to set its session cookie and login won't work.
- `GENERIC_TIMEZONE` — sets the timezone for workflow schedules
- `n8n_data` volume — stores all your workflows, credentials, and settings persistently

### Home Assistant — Smart Home

| Setting | Value                     |
| ------- | ------------------------- |
| Port    | 8123                      |
| Access  | `http://192.168.x.x:8123` |

- `network_mode: host` — Home Assistant runs directly on the host network instead of Docker's internal network. This is required for LAN device auto-discovery (mDNS, Zigbee USB adapters, etc.)
- `privileged: true` — gives Home Assistant access to hardware like USB Zigbee/Z-Wave dongles
- Because it uses host networking, it won't appear with an IP in Portainer's container list — this is normal

### Homepage — Dashboard

| Setting | Value                     |
| ------- | ------------------------- |
| Port    | 3000                      |
| Access  | `http://192.168.x.x:3000` |

- `HOMEPAGE_ALLOWED_HOSTS` — Homepage validates the Host header for security. You must list every IP:port you'll use to access it, separated by commas. Add your Tailscale IP here too.
- `/var/run/docker.sock:ro` — read-only access to Docker so Homepage can show container status
- Config files are in the `homepage_config` volume

## Configuring Homepage

Homepage is configured via YAML files inside its config volume.

### Find the config files

```bash
ls /var/lib/docker/volumes/homeserver_homepage_config/_data/
```

### settings.yaml — Theme and Layout

```bash
nano /var/lib/docker/volumes/homeserver_homepage_config/_data/settings.yaml
```

```yaml
title: Home Server

theme: light
color: zinc

useEqualHeights: true

layout:
  Services:
    style: row
    columns: 3
```

### services.yaml — Service Cards

```bash
nano /var/lib/docker/volumes/homeserver_homepage_config/_data/services.yaml
```

```yaml
- Services:
    - N8N:
        icon: n8n.png
        href: http://192.168.x.x:5678
        description: Automation
        server: my-docker
        container: n8n

    - Home Assistant:
        icon: home-assistant.png
        href: http://192.168.x.x:8123
        description: Smart Home
        server: my-docker
        container: homeassistant

    - Portainer:
        icon: portainer.png
        href: http://192.168.x.x:9000
        description: Docker Manager
        server: my-docker
        container: portainer
```

### docker.yaml — Connect to Docker

```bash
nano /var/lib/docker/volumes/homeserver_homepage_config/_data/docker.yaml
```

```yaml
my-docker:
  socket: /var/run/docker.sock
```

Changes to config files apply instantly — just refresh the browser, no restart needed.

## Adding New Services

1. In Portainer → **Stacks** → **homeserver** → **Editor**
2. Add the new service block under `services:`
3. Add any new volume names under `volumes:`
4. Click **Update the stack**
5. Add iptables port forward rules on the **Proxmox host**:

```bash
iptables -t nat -A PREROUTING -i <wifi-interface> -p tcp --dport NEW_PORT -j DNAT --to-destination 10.10.10.2:NEW_PORT
iptables -t nat -A PREROUTING -i tailscale0 -p tcp --dport NEW_PORT -j DNAT --to-destination 10.10.10.2:NEW_PORT
netfilter-persistent save
```

6. Add the new service to Homepage's `services.yaml`

## Planned Future Services

These are commented out / not yet deployed. Add them when ready:

| Service     | Port | Purpose                |
| ----------- | ---- | ---------------------- |
| Syncthing   | 8384 | File sync from main PC |
| Immich      | 2283 | Family photo backup    |
| qBittorrent | 8080 | Torrent downloads      |
| Jellyfin    | 8096 | Media streaming        |
