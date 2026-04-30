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
      - "2000:2000"
    volumes:
      - homepage_config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - TZ=Asia/Dhaka
      - PORT=2000
      - HOMEPAGE_ALLOWED_HOSTS=192.168.x.x:2000,100.x.x.x:2000
      # Add both local and Tailscale IPs to HOMEPAGE_ALLOWED_HOSTS
      # Use environment variables for sensitive widget credentials (see below)
      - HOMEPAGE_VAR_ADGUARD_USER=${HOMEPAGE_VAR_ADGUARD_USER}
      - HOMEPAGE_VAR_ADGUARD_PASS=${HOMEPAGE_VAR_ADGUARD_PASS}
      - HOMEPAGE_VAR_SPEEDTEST_KEY=${HOMEPAGE_VAR_SPEEDTEST_KEY}

  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    network_mode: host
    volumes:
      - adguardhome_work:/opt/adguardhome/work
      - adguardhome_conf:/opt/adguardhome/conf

  speedtest-tracker:
    image: lscr.io/linuxserver/speedtest-tracker:latest
    container_name: speedtest-tracker
    restart: unless-stopped
    ports:
      - "8765:80"
    environment:
      - PUID=1000
      - PGID=1000
      - APP_KEY=${SPEEDTEST_APP_KEY}   # generate with: echo "base64:$(openssl rand -base64 32)"
      - APP_URL=http://192.168.x.x:8765
      - APP_NAME=Speedtest Tracker
      - APP_TIMEZONE=Asia/Dhaka
      - DISPLAY_TIMEZONE=Asia/Dhaka
      - DB_CONNECTION=sqlite
      - SPEEDTEST_SCHEDULE=0 */6 * * *
      - PRUNE_RESULTS_OLDER_THAN=90
      - DEFAULT_CHART_RANGE=week
      - CHART_BEGIN_AT_ZERO=true
      - PUBLIC_DASHBOARD=true
    volumes:
      - speedtest_config:/config

volumes:
  n8n_data:
  ha_config:
  homepage_config:
  adguardhome_work:
  adguardhome_conf:
  speedtest_config:
```

> ⚠️ **Never hardcode secrets in compose files.** Use a `.env` file alongside the compose file and add `.env` to `.gitignore`. See the secrets section below.

## Secrets Management

Store sensitive values in a `.env` file, never in the compose file directly.

Create `/opt/homeserver/.env`:

```bash
HOMEPAGE_VAR_ADGUARD_USER=your_adguard_username
HOMEPAGE_VAR_ADGUARD_PASS=your_adguard_password
HOMEPAGE_VAR_SPEEDTEST_KEY=your_speedtest_api_key
SPEEDTEST_APP_KEY=base64:your_generated_key
```

Add to `.gitignore`:

```
.env
```

Portainer stacks support `.env` files — paste the env values in the **Environment variables** section at the bottom of the stack editor instead of the compose file.

---

## Service Details

### N8N — Automation

| Setting | Value                     |
| ------- | ------------------------- |
| Port    | 5678                      |
| Access  | `http://192.168.x.x:5678` |

- `N8N_SECURE_COOKIE=false` — required when accessing over HTTP. Without this N8N refuses to set its session cookie and login won't work.
- `GENERIC_TIMEZONE` — sets the timezone for workflow schedules
- `n8n_data` volume — stores all workflows, credentials, and settings persistently

#### Connecting OAuth Services (e.g. Google Calendar)

N8N OAuth requires a public HTTPS callback URL. Use a temporary Cloudflare tunnel:

```bash
# Install cloudflared inside Docker LXC
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o /usr/local/bin/cloudflared
chmod +x /usr/local/bin/cloudflared

# Start tunnel
cloudflared tunnel --url http://localhost:5678
```

Temporarily update N8N environment:
```yaml
- WEBHOOK_URL=https://your-tunnel-url.trycloudflare.com
- N8N_PROTOCOL=https
- N8N_HOST=your-tunnel-url.trycloudflare.com
- N8N_EDITOR_BASE_URL=https://your-tunnel-url.trycloudflare.com
```

**Important:** Access N8N from the tunnel URL (not local IP) when initiating OAuth. After credentials are saved, revert environment variables back to local settings. Saved OAuth tokens persist and continue working after the tunnel is closed.

---

### Home Assistant — Smart Home

| Setting | Value                     |
| ------- | ------------------------- |
| Port    | 8123                      |
| Access  | `http://192.168.x.x:8123` |

- `network_mode: host` — required for LAN device auto-discovery (mDNS, Zigbee USB adapters, etc.)
- `privileged: true` — gives access to hardware like USB Zigbee/Z-Wave dongles
- Does not appear with an IP in Portainer's container list — this is normal for host networking

---

### Homepage — Dashboard

| Setting | Value                     |
| ------- | ------------------------- |
| Port    | 2000                      |
| Access  | `http://192.168.x.x:2000` |

- `HOMEPAGE_ALLOWED_HOSTS` — must list every IP:port used to access it. Add both local and Tailscale IPs.
- `PORT=2000` — tells Homepage to listen on port 2000 instead of default 3000
- `/var/run/docker.sock:ro` — read-only Docker access for container status widgets
- Use `{{HOMEPAGE_VAR_*}}` syntax in config files to reference environment variables for credentials

#### Config Files Location

```bash
/var/lib/docker/volumes/homeserver_homepage_config/_data/
```

#### settings.yaml

```yaml
title: Home Server
theme: light
color: zinc
useEqualHeights: true

layout:
  Automation & Smart Home:
    style: row
    columns: 2
  Network:
    style: row
    columns: 2
  Management:
    style: row
    columns: 1
```

#### services.yaml

```yaml
- Automation & Smart Home:
    - N8N:
        icon: n8n.png
        href: http://100.x.x.x:5678
        description: Workflow Automation
        server: my-docker
        container: n8n

    - Home Assistant:
        icon: home-assistant.png
        href: http://100.x.x.x:8123
        description: Smart Home
        server: my-docker
        container: homeassistant

- Network:
    - AdGuard Home:
        icon: adguard-home.png
        href: http://100.x.x.x:3001
        description: DNS & Ad Blocking
        server: my-docker
        container: adguardhome
        widget:
          type: adguard
          url: http://10.10.10.2:3001
          username: {{HOMEPAGE_VAR_ADGUARD_USER}}
          password: {{HOMEPAGE_VAR_ADGUARD_PASS}}

    - Speedtest Tracker:
        icon: speedtest-tracker.png
        href: http://100.x.x.x:8765
        description: Internet Speed Monitor
        server: my-docker
        container: speedtest-tracker
        widget:
          type: speedtest
          url: http://speedtest-tracker:80

- Management:
    - Portainer:
        icon: portainer.png
        href: http://100.x.x.x:9000
        description: Docker Manager
        server: my-docker
        container: portainer
```

> **Note:** AdGuard uses `http://10.10.10.2:3001` (container IP) for the widget URL because it runs with `network_mode: host` and is not on the Docker bridge network. Other containers can be referenced by container name.

#### docker.yaml

```yaml
my-docker:
  socket: /var/run/docker.sock
```

#### widgets.yaml

```yaml
- resources:
    cpu: true
    memory: true
    disk: /

- datetime:
    text_size: xl
    format:
      timeStyle: short
      dateStyle: short
      hourCycle: h23
```

Changes to config files apply instantly — just refresh the browser, no restart needed.

---

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

6. Add the new service card to Homepage's `services.yaml`

---

## Planned Future Services

| Service     | Port | Purpose                |
| ----------- | ---- | ---------------------- |
| Syncthing   | 8384 | File sync from main PC |
| Immich      | 2283 | Family photo backup    |
| qBittorrent | 8080 | Torrent downloads      |
| Jellyfin    | 8096 | Media streaming        |
