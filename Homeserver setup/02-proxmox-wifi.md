# 02 — WiFi Setup on Proxmox

Proxmox is designed for ethernet. WiFi requires manual configuration after install. This guide sets up WiFi as the primary network interface and configures NAT so LXC containers can reach the internet through it.

## Step 1 — Install WiFi Tools

In **Proxmox host shell** (homeserver → Shell in web UI):

```bash
apt install wpasupplicant wireless-tools -y
```

- `wpasupplicant` — handles WPA/WPA2 WiFi authentication
- `wireless-tools` — provides `iwlist` for scanning networks

## Step 2 — Find Your WiFi Interface Name

```bash
ip link show
```

Look for an interface starting with `wl` — e.g. `<wifi-interface>`. This is your WiFi adapter.

## Step 3 — Scan for Available Networks

Bring the interface up first, then scan:

```bash
ip link set <wifi-interface> up
iwlist <wifi-interface> scan | grep ESSID
```

Find your network name (SSID) in the list. Copy it exactly including any special characters.

## Step 4 — Create WiFi Config

Replace `YourWiFiName` and `YourWiFiPassword` with your actual credentials:

```bash
wpa_passphrase "YourWiFiName" "YourWiFiPassword" > /etc/wpa_supplicant/wpa_supplicant-<wifi-interface>.conf
```

This generates a config file with your WiFi credentials. The password is stored as a hash.

## Step 5 — Enable wpa_supplicant

```bash
# Enable the service for your specific interface
systemctl enable wpa_supplicant@<wifi-interface>
systemctl start wpa_supplicant@<wifi-interface>
```

`wpa_supplicant` manages the WiFi connection and handles reconnection automatically.

## Step 6 — Configure Network Interfaces

```bash
nano /etc/network/interfaces
```

Replace the entire file with:

```
auto lo
iface lo inet loopback

iface nic0 inet manual

auto vmbr0
iface vmbr0 inet static
        address 10.10.10.1/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        post-up echo 1 > /proc/sys/net/ipv4/ip_forward
        post-up iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o <wifi-interface> -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s 10.10.10.0/24 -o <wifi-interface> -j MASQUERADE

iface nic1 inet manual

auto <wifi-interface>
iface <wifi-interface> inet static
    address 192.168.x.x/24
    gateway 192.168.0.1
    dns-nameservers 8.8.8.8
    wpa-conf /etc/wpa_supplicant/wpa_supplicant-<wifi-interface>.conf

source /etc/network/interfaces.d/*
```

### What this config does

| Block                         | Purpose                                                                                          |
| ----------------------------- | ------------------------------------------------------------------------------------------------ |
| `vmbr0`                       | Virtual bridge for LXC containers. Has IP `10.10.10.1` — acts as the gateway for all containers  |
| `post-up ip_forward`          | Enables Linux kernel IP forwarding so containers can route traffic                               |
| `post-up iptables MASQUERADE` | NAT rule — translates container IPs (`10.10.10.x`) to the WiFi IP when going out to the internet |
| `<wifi-interface>`            | The actual WiFi interface. Has your LAN IP and connects to your router                           |

Save: `Ctrl+X` → `Y` → `Enter`

## Step 7 — Apply and Test

```bash
systemctl restart networking
ping -c 3 google.com
```

All 3 pings should succeed. Then unplug ethernet cable and test again:

```bash
ping -c 3 google.com
```

If ping works without ethernet, WiFi is the sole connection. ✅

## Step 8 — Port Forwarding for Services

Since services run inside LXC containers on `10.10.10.x`, we need to forward ports from the Proxmox host (`192.168.x.x`) to the container. Add a rule for each service port:

```bash
# Forward from LAN/Tailscale to Docker LXC
iptables -t nat -A PREROUTING -i <wifi-interface> -p tcp --dport 9000 -j DNAT --to-destination 10.10.10.2:9000
iptables -t nat -A PREROUTING -i <wifi-interface> -p tcp --dport 5678 -j DNAT --to-destination 10.10.10.2:5678
iptables -t nat -A PREROUTING -i <wifi-interface> -p tcp --dport 8123 -j DNAT --to-destination 10.10.10.2:8123
iptables -t nat -A PREROUTING -i <wifi-interface> -p tcp --dport 3000 -j DNAT --to-destination 10.10.10.2:3000

iptables -t nat -A PREROUTING -i tailscale0 -p tcp --dport 9000 -j DNAT --to-destination 10.10.10.2:9000
iptables -t nat -A PREROUTING -i tailscale0 -p tcp --dport 5678 -j DNAT --to-destination 10.10.10.2:5678
iptables -t nat -A PREROUTING -i tailscale0 -p tcp --dport 8123 -j DNAT --to-destination 10.10.10.2:8123
iptables -t nat -A PREROUTING -i tailscale0 -p tcp --dport 3000 -j DNAT --to-destination 10.10.10.2:3000
```

Make rules persistent across reboots:

```bash
apt install iptables-persistent -y
netfilter-persistent save
```

> **Adding new services:** Every time you add a new Docker service, add two new iptables rules (one for `<wifi-interface>`, one for `tailscale0`) and run `netfilter-persistent save`.
