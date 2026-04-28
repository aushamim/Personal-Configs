# 01 — Proxmox VE Installation

Proxmox VE is a bare-metal hypervisor. It replaces your laptop's OS entirely and lets you run LXC containers and VMs on top of it. It includes a web UI for managing everything.

## Prerequisites

- USB drive (4GB+, will be wiped)
- Another PC to flash the USB from
- Laptop plugged into power

## Step 1 — Download Proxmox ISO

Go to https://www.proxmox.com/en/downloads and download the latest **Proxmox VE** ISO (~1.1GB).

## Step 2 — Flash USB with Rufus

1. Download Rufus from https://rufus.ie
2. Open Rufus, select your USB drive
3. Click **SELECT** and choose the Proxmox ISO
4. When prompted **"How should this image be written?"** — select **"Write in DD Image mode"**  
   ⚠️ This is critical — ISO mode does not work with Proxmox
5. Click START and wait

## Step 3 — BIOS Settings

Boot into BIOS (spam F2, F10, or Del on startup depending on laptop brand):

- Disable **Secure Boot**
- Set **USB** as first boot device

## Step 4 — Install Proxmox

Boot from USB. The GUI installer will start.

### Network Configuration Page

Fill in these values:

```
Management Interface:  your ethernet NIC (e.g. nic0/e1000e)
Hostname:              homeserver.local
IP Address:            192.168.x.x/24   (adjust to your network range)
Gateway:               192.168.0.1        (your router IP)
DNS:                   8.8.8.8
```

> **Note:** If you only have WiFi, select the ethernet NIC anyway with a static IP. We will configure WiFi after install. The installer does not support WiFi.

### Remaining Steps

- Set a strong **root password** — write it down
- Set email (can be `admin@local.com`)
- Complete the install (~5–10 mins)
- Remove USB when prompted and let it reboot

## Step 5 — Access Web UI

Connect an ethernet cable to the laptop temporarily, then from another PC:

```
https://192.168.x.x:8006
```

Accept the self-signed certificate warning. Login with:

```
Username: root
Password: (your password)
Realm:    Linux PAM standard authentication
```

You will see a "no valid subscription" popup — this is normal. Proxmox is free to use, the subscription is for enterprise support.

## Step 6 — Remove Enterprise Repos

Proxmox ships with enterprise repos enabled by default which require a paid subscription. Remove them and switch to the free community repos.

In the Proxmox web UI, go to **homeserver → Shell** and run:

```bash
# Remove enterprise repo files
rm -f /etc/apt/sources.list.d/pve-enterprise.sources
rm -f /etc/apt/sources.list.d/ceph.sources

# Add free community repos
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-community.list
echo "deb http://download.proxmox.com/debian/ceph-squid bookworm no-subscription" > /etc/apt/sources.list.d/ceph-community.list

# Update package list
apt update
```

## Step 7 — Remove Subscription Nag Popup

```bash
# Patches the JS file that shows the nag popup on every login
sed -i.bak "s/if (res === null || res === undefined || \!res || res/if (false/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
```

Refresh the browser — the popup is gone.

## Step 8 — Update Proxmox

```bash
apt full-upgrade -y
```
