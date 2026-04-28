# 03 — Tailscale Setup

Tailscale creates a private WireGuard-based mesh VPN. Every device you add gets a stable `100.x.x.x` IP. Traffic between devices is encrypted end-to-end and never touches a public server.

This means your services are never exposed to the internet — you access them through the Tailscale IP from anywhere in the world as if you were on the local network.

## Step 1 — Install Tailscale on Proxmox Host

In **Proxmox host shell**:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

This adds the Tailscale apt repo and installs the package.

## Step 2 — Authenticate

```bash
tailscale up
```

It will print a URL like:

```
https://login.tailscale.com/a/xxxxxx
```

Open that URL on any device, log into your Tailscale account (create one free at tailscale.com if needed), and authorize the server.

## Step 3 — Get Your Tailscale IP

```bash
tailscale ip
```

You'll get a `100.x.x.x` address. This is your server's permanent Tailscale IP. It never changes.

## Step 4 — Install Tailscale on Your Other Devices

| Platform        | Install                         |
| --------------- | ------------------------------- |
| Windows / macOS | https://tailscale.com/download  |
| Android / iOS   | Search "Tailscale" in app store |
| Linux           | Same install script as above    |

Once installed and logged into the same account, all your devices can reach the server at `100.x.x.x`.

## Step 5 — Verify Connection

From any Tailscale-connected device, open a browser and go to:

```
https://100.x.x.x:8006
```

Accept the certificate warning and you should see the Proxmox login screen.

## How to Access Services Remotely

Once Tailscale is running on your phone or laptop, access any service using the Tailscale IP:

```
Proxmox UI:     https://100.x.x.x:8006
Portainer:      http://100.x.x.x:9000
N8N:            http://100.x.x.x:5678
Home Assistant: http://100.x.x.x:8123
Homepage:       http://100.x.x.x:3000
```

## Security Notes

- **Never expose SSH (port 22) to the internet** — always SSH through Tailscale only
- Tailscale is the only way to reach your server from outside your home network
- Even if someone knows your home IP, they cannot access any services without being on your Tailscale network
- You can add ACL rules in the Tailscale admin panel to restrict which devices can reach which services
