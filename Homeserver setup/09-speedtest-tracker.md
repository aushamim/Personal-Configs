# 09 — Speedtest Tracker

Speedtest Tracker automatically runs internet speed tests on a schedule and stores historical results. Useful for monitoring ISP performance, detecting throttling, and tracking speed trends over time.

## Docker Compose

### Generate APP_KEY

Required for encryption. Run in the **Docker LXC console**:

```bash
echo "base64:$(openssl rand -base64 32 2>/dev/null)"
```

Copy the full output including `base64:`. Store it in your `.env` file — never hardcode it in the compose file.

### Add to Portainer Stack

Add to your `homeserver` stack in Portainer:

```yaml
  speedtest-tracker:
    image: lscr.io/linuxserver/speedtest-tracker:latest
    container_name: speedtest-tracker
    restart: unless-stopped
    ports:
      - "8765:80"
    environment:
      - PUID=1000
      - PGID=1000
      - APP_KEY=${SPEEDTEST_APP_KEY}
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
```

Add to volumes:

```yaml
  speedtest_config:
```

Add to `.env` file:

```
SPEEDTEST_APP_KEY=base64:your_generated_key_here
```

### Port Forward on Proxmox Host

```bash
iptables -t nat -A PREROUTING -i <wifi-interface> -p tcp --dport 8765 -j DNAT --to-destination 10.10.10.2:8765
iptables -t nat -A PREROUTING -i tailscale0 -p tcp --dport 8765 -j DNAT --to-destination 10.10.10.2:8765
netfilter-persistent save
```

## Access

```
http://192.168.x.x:8765
```

Default login:

```
Email:    admin@example.com
Password: password
```

Change these immediately after first login via the user menu.

## Environment Variables Explained

| Variable | Value | Purpose |
|---|---|---|
| `APP_KEY` | `base64:...` | Encryption key — required, never share |
| `APP_URL` | `http://192.168.x.x:8765` | Used for links in notifications |
| `APP_TIMEZONE` | `Asia/Dhaka` | Timezone for database timestamps |
| `DISPLAY_TIMEZONE` | `Asia/Dhaka` | Timezone shown in the UI |
| `SPEEDTEST_SCHEDULE` | `0 */6 * * *` | Run test every 6 hours automatically |
| `PRUNE_RESULTS_OLDER_THAN` | `90` | Delete results older than 90 days |
| `DEFAULT_CHART_RANGE` | `week` | Show last 7 days by default on dashboard |
| `CHART_BEGIN_AT_ZERO` | `true` | Charts start from 0 for accurate comparison |
| `PUBLIC_DASHBOARD` | `true` | Allow viewing dashboard without login |
| `DB_CONNECTION` | `sqlite` | Use SQLite — no separate database needed |

## Schedule Format (Cron)

The `SPEEDTEST_SCHEDULE` uses cron syntax. Common values:

| Schedule | Cron |
|---|---|
| Every hour | `0 * * * *` |
| Every 6 hours | `0 */6 * * *` |
| Every 12 hours | `0 */12 * * *` |
| Once a day at 2AM | `0 2 * * *` |

Use https://crontab.guru to build custom schedules.

## Thresholds

Set threshold alerts in the UI under **Thresholds** in the left sidebar. Configure values based on your internet plan speed:

| Metric | Recommended Threshold |
|---|---|
| Download | 70% of plan speed |
| Upload | 70% of plan speed |
| Ping | 50ms |

When a test result falls below these values, it is flagged in the results table.

## Homepage Widget

In `services.yaml`, the Speedtest Tracker widget works without an API key when `PUBLIC_DASHBOARD=true`:

```yaml
    - Speedtest Tracker:
        icon: speedtest-tracker.png
        href: http://100.x.x.x:8765
        description: Internet Speed Monitor
        server: my-docker
        container: speedtest-tracker
        widget:
          type: speedtest
          url: http://speedtest-tracker:80
```

If you disable the public dashboard, generate an API key in **API Tokens** in the sidebar and add:

```yaml
          key: {{HOMEPAGE_VAR_SPEEDTEST_KEY}}
```

## Notes

- If the server is on WiFi, speedtest results reflect the WiFi connection speed to the router, not the true internet speed. For accurate results use an ethernet connection.
- SQLite is sufficient for personal use. For multi-user or high-frequency testing, consider switching to MariaDB or PostgreSQL.
- `PUBLIC_DASHBOARD=true` allows anyone on your network to view results without logging in — fine for a home network, disable if you share your Tailscale network with others.
