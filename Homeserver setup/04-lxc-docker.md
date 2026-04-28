# 04 — LXC Container and Docker Setup

We run all Docker services inside an LXC container rather than directly on the Proxmox host. This gives isolation — if something goes wrong with Docker or a container, the Proxmox host is unaffected. It also allows Proxmox to snapshot and back up the entire Docker environment.

## Step 1 — Download Ubuntu 24.04 Template

In Proxmox web UI:

**homeserver → local (homeserver) → CT Templates → Templates button → search `ubuntu-24.04` → Download**

Wait for the download to finish.

## Step 2 — Create LXC Container

Click **Create CT** in the top right. Fill in each tab:

| Tab      | Setting   | Value             |
| -------- | --------- | ----------------- |
| General  | Hostname  | docker            |
| General  | Password  | (strong password) |
| Template | Template  | ubuntu-24.04      |
| Disks    | Disk size | 80 GB             |
| CPU      | Cores     | 4                 |
| Memory   | Memory    | 6144 MB (6GB)     |
| Memory   | Swap      | 512 MB            |
| Network  | Bridge    | vmbr0             |
| Network  | IPv4      | Static            |
| Network  | IP        | 10.10.10.2/24     |
| Network  | Gateway   | 10.10.10.1        |

> **Uncheck "Start after created"** before finishing — we need to change a setting before first boot.

## Step 3 — Enable Nesting and keyctl

These features are required for Docker to run inside an LXC container.

In Proxmox web UI:

**docker container → Options → Features → Edit**

Enable:

- ✅ Nesting
- ✅ keyctl

Click OK.

## Step 4 — Enable Auto-Start

So the container starts automatically after a reboot:

**docker container → Options → Start at boot → Yes**

## Step 5 — Start the Container

Right-click the **docker** container in the left panel → **Start**

Then open the console: **docker container → Console**

## Step 6 — Fix DNS

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

This sets Google DNS inside the container so it can resolve domain names.

## Step 7 — Update and Install Docker

```bash
apt update && apt upgrade -y

# Install Docker using the official install script
curl -fsSL https://get.docker.com | sh

# Enable Docker to start on boot
systemctl enable --now docker

# Install Docker Compose plugin
apt install docker-compose-plugin -y
```

Verify:

```bash
docker --version
docker compose version
```

## Step 8 — Create Data Directories

These directories hold all persistent data. Keeping them outside Docker volumes means your data survives container rebuilds.

```bash
mkdir -p /mnt/data/syncthing
mkdir -p /mnt/data/immich/upload
mkdir -p /mnt/data/downloads
mkdir -p /opt/homeserver
mkdir -p /opt/portainer
```

| Directory                 | Purpose                                           |
| ------------------------- | ------------------------------------------------- |
| `/mnt/data/syncthing`     | Syncthing synced files                            |
| `/mnt/data/immich/upload` | Immich photo storage                              |
| `/mnt/data/downloads`     | qBittorrent downloads                             |
| `/opt/homeserver`         | Docker Compose files for services                 |
| `/opt/portainer`          | Docker Compose file for Portainer (kept separate) |
