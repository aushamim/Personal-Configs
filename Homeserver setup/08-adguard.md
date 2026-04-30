# 08 — AdGuard Home

AdGuard Home is a network-wide DNS-based ad and tracker blocker. It runs as a DNS server on your home server. When you point your router's DHCP to use it, every device on your network gets ad blocking automatically — no configuration needed on individual devices.

## How It Works

```
Normal DNS flow:
Device → Router → ISP DNS → returns IP → ad loads

With AdGuard Home:
Device → AdGuard Home → checks blocklist
         ├── Ad domain?  → returns nothing → ad never loads
         └── Legit domain? → upstream DNS → returns IP
```

DNS queries happen before any connection is made. If AdGuard blocks the domain, the device never even attempts to load the ad.

## Docker Compose

Add to your Portainer `homeserver` stack:

```yaml
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    network_mode: host
    volumes:
      - adguardhome_work:/opt/adguardhome/work
      - adguardhome_conf:/opt/adguardhome/conf
```

Add to volumes section:

```yaml
  adguardhome_work:
  adguardhome_conf:
```

`network_mode: host` is required because DNS runs on port 53 which needs direct host network access. This also means AdGuard won't appear with an IP in Portainer — this is normal.

## Prerequisites — Disable systemd-resolved

Ubuntu 24.04 uses `systemd-resolved` which occupies port 53 by default. Disable it so AdGuard can take over:

```bash
# In Docker LXC console
systemctl disable systemd-resolved
systemctl stop systemd-resolved

# Set permanent DNS for the container itself
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Create a service to restore DNS on every reboot:

```bash
nano /etc/systemd/system/fix-dns.service
```

```
[Unit]
Description=Fix DNS after systemd-resolved disabled
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable fix-dns
```

## Port Forward on Proxmox Host

AdGuard needs port 53 forwarded from the Proxmox host to the LXC:

```bash
# DNS port (both TCP and UDP)
iptables -t nat -A PREROUTING -i <wifi-interface> -p tcp --dport 53 -j DNAT --to-destination 10.10.10.2:53
iptables -t nat -A PREROUTING -i <wifi-interface> -p udp --dport 53 -j DNAT --to-destination 10.10.10.2:53

# Admin UI
iptables -t nat -A PREROUTING -i <wifi-interface> -p tcp --dport 3001 -j DNAT --to-destination 10.10.10.2:3001
iptables -t nat -A PREROUTING -i tailscale0 -p tcp --dport 3001 -j DNAT --to-destination 10.10.10.2:3001

netfilter-persistent save
```

## Initial Setup Wizard

AdGuard's setup wizard runs on port 3000 on first launch. Access it at:

```
http://192.168.x.x:3000
```

Walk through the wizard:

| Step | Setting | Value |
|---|---|---|
| Admin Web Interface | Listen interface | All interfaces |
| Admin Web Interface | Port | `3001` |
| DNS Server | Listen interface | All interfaces |
| DNS Server | Port | `53` |
| Authentication | Username | (your choice) |
| Authentication | Password | (strong password) |

> Port 3001 avoids conflicts with other services. Do not use port 80 — it conflicts with reverse proxies.

After setup, access the dashboard at:

```
http://192.168.x.x:3001
```

## Configure Router DNS

Point your router's DHCP DNS server to AdGuard so all devices use it automatically.

For **TP-Link Archer C6**:
1. Go to `http://192.168.0.1`
2. **Advanced** → **Network** → **DHCP Server**
3. Set **Primary DNS** to your server's local IP (`192.168.x.x`)
4. Set **Secondary DNS** to `8.8.8.8` (fallback if AdGuard goes down)
5. Click **Save**

Reconnect devices to WiFi or wait for DHCP lease renewal.

## Configure Upstream DNS

Set AdGuard to use your router/ISP as upstream so ISP-specific routing and caching still works:

**Settings → DNS settings → Upstream DNS servers:**

```
192.168.0.1
```

This forwards unblocked domains to your router → ISP DNS. ISP cache features (like fast game downloads from Steam/Epic) are unaffected since those work at the routing level, not DNS level.

## Verify It's Working

From your main PC:

```bash
# Windows
nslookup google.com 192.168.x.x

# Linux/Mac
dig google.com @192.168.x.x
```

Then check **AdGuard Dashboard → Query Log** — you should see DNS queries appearing as you browse.

Also verify your PC is using AdGuard:

```bash
# Windows
ipconfig /all
# Look for DNS Servers: 192.168.x.x
```

## Recommended Blocklists

In AdGuard: **Filters → DNS blocklists → Add blocklist**

| List | URL | Purpose |
|---|---|---|
| AdGuard DNS filter | Built-in | General ads |
| OISD | `https://big.oisd.nl` | Comprehensive |
| Steven Black | `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts` | Ads + malware |

## Tailscale DNS (Optional)

To get ad blocking when you're away from home on mobile data:

1. Go to Tailscale admin panel → **DNS**
2. Add a custom nameserver: `100.x.x.x` (your server's Tailscale IP)
3. Enable **Override local DNS**

All DNS queries on your Tailscale-connected devices will now go through AdGuard regardless of which network you're on.

## Notes

- AdGuard uses `network_mode: host` so it does not appear with a Docker IP in Portainer
- For the Homepage widget, use `http://10.10.10.2:3001` as the URL (not container name) since host networking bypasses Docker's internal DNS
- The secondary DNS on your router (`8.8.8.8`) acts as a fallback — if the server goes down, devices will still resolve DNS through Google
