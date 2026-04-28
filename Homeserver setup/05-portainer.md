# 05 — Portainer Setup

Portainer is a web UI for managing Docker. It lets you start/stop containers, view logs, edit compose files, and monitor resource usage — all from a browser without needing the terminal.

Portainer is kept in its own separate compose file so it never goes down when you manage other services.

## Step 1 — Create Portainer Compose File

In the **Docker LXC console**:

```bash
nano /opt/portainer/docker-compose.yml
```

Paste:

```yaml
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

volumes:
  portainer_data:
```

### What each part does

| Setting                   | Purpose                                                                 |
| ------------------------- | ----------------------------------------------------------------------- |
| `restart: unless-stopped` | Automatically restarts if it crashes or after a reboot                  |
| `/var/run/docker.sock`    | Gives Portainer access to the Docker daemon so it can manage containers |
| `portainer_data`          | Stores Portainer's own config and user data persistently                |

Save: `Ctrl+X` → `Y` → `Enter`

## Step 2 — Start Portainer

```bash
cd /opt/portainer
docker compose up -d
```

Verify it's running:

```bash
docker ps
```

You should see the portainer container with status `Up`.

## Step 3 — Access Portainer

From your browser (on local network):

```
http://192.168.x.x:9000
```

Or via Tailscale from anywhere:

```
http://100.x.x.x:9000
```

## Step 4 — Create Admin Account

On first launch, Portainer asks you to create an admin account. Set a username and strong password.

It will then show an environment setup screen. Portainer should auto-detect the local Docker instance — click **Get Started** or **Connect**.

You'll see the dashboard showing your running containers.

## Managing Services Through Portainer

All other services (N8N, Home Assistant, etc.) are managed as a **Stack** in Portainer:

- **Stacks** = Docker Compose files managed by Portainer
- You can edit the compose YAML directly in the browser
- Click **Update the stack** to apply changes — no terminal needed

> **Why separate Portainer from other services?**  
> If Portainer were in the same compose file as N8N etc., running `docker compose down` would also kill Portainer, leaving you with no UI to bring things back up. Keeping it separate means Portainer is always available.
