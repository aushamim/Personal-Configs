# 🏠 Home Server Setup — Documentation

A complete guide to setting up a home server on a laptop using Proxmox VE, Docker, and various self-hosted services.

## Setup Environment

| Component         | Choice                      |
| ----------------- | --------------------------- |
| Hardware          | Laptop (8GB RAM, 250GB SSD) |
| Hypervisor        | Proxmox VE 8.x              |
| Network           | WiFi (2.4GHz)               |
| Remote Access     | Tailscale VPN               |
| Container         | LXC (Ubuntu 24.04)          |
| Container Runtime | Docker + Docker Compose     |

## Guide Index

| File                                                     | Description                                   |
| -------------------------------------------------------- | --------------------------------------------- |
| [01-proxmox-install.md](01-proxmox-install.md)           | Installing and configuring Proxmox VE         |
| [02-proxmox-wifi.md](02-proxmox-wifi.md)                 | Setting up WiFi on Proxmox                    |
| [03-tailscale.md](03-tailscale.md)                       | Installing Tailscale for secure remote access |
| [04-lxc-docker.md](04-lxc-docker.md)                     | Creating LXC container and installing Docker  |
| [05-portainer.md](05-portainer.md)                       | Setting up Portainer for Docker management    |
| [06-services.md](06-services.md)                         | Deploying N8N, Home Assistant, Homepage       |
| [07-laptop-optimizations.md](07-laptop-optimizations.md) | Power, lid, and sleep optimizations           |

## Network Map

```
Proxmox Host:     192.168.x.x  (WiFi)
Tailscale IP:     100.x.x.x
Docker LXC:       10.10.10.2     (internal NAT)

Services (access via 192.168.x.x or Tailscale IP):
  Proxmox UI:     :8006
  Portainer:      :9000
  N8N:            :5678
  Home Assistant: :8123
  Homepage:       :2000
```

## Architecture

```
Bare Metal (Laptop)
└── Proxmox VE
    ├── LXC: docker (10.10.10.2)
    │   ├── Portainer      :9000
    │   ├── N8N            :5678
    │   ├── Home Assistant :8123
    │   └── Homepage       :3000
    └── (future) VM: Ubuntu Desktop
```
