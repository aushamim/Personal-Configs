# 07 — Laptop Optimizations

A laptop used as a server needs specific configuration to run reliably 24/7. These settings prevent it from sleeping, handle the lid correctly, and optimize power consumption.

All commands in this file run on the **Proxmox host shell** (homeserver → Shell in web UI), not inside the Docker LXC.

---

## Lid Behavior

### Prevent suspend on lid close

By default, closing the laptop lid suspends the system. We disable this so the server keeps running with the lid closed.

```bash
nano /etc/systemd/logind.conf
```

Find and uncomment these lines (remove `#`) and set to `ignore`:

```
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

Apply without rebooting:

```bash
systemctl restart systemd-logind
```

### Turn screen off when lid closes, on when lid opens

Install the ACPI event daemon which listens for hardware events like lid open/close:

```bash
apt install acpid -y
```

Create a lid event listener:

```bash
nano /etc/acpi/events/lid
```

```
event=button/lid
action=/etc/acpi/lid.sh
```

Create the script that handles the event:

```bash
nano /etc/acpi/lid.sh
```

```bash
#!/bin/bash
LID_STATE=$(cat /proc/acpi/button/lid/LID/state | awk '{print $2}')
if [ "$LID_STATE" = "closed" ]; then
    # Lid closed — blank the screen
    setterm -blank force > /dev/tty1
else
    # Lid opened — unblank the screen
    setterm -blank poke > /dev/tty1
fi
```

Make the script executable:

```bash
chmod +x /etc/acpi/lid.sh
```

Enable and start acpid:

```bash
systemctl enable acpid
systemctl start acpid
```

### Prevent screen burn — disable console blanking timeout

```bash
nano /etc/default/grub
```

Set:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet consoleblank=0"
```

Apply:

```bash
update-grub
```

`consoleblank=0` disables the kernel's automatic screen blanking, letting our lid script handle it instead.

---

## Sleep and Suspend

Completely disable all forms of sleep so the server never suspends unexpectedly:

```bash
systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

`systemctl mask` is stronger than `disable` — it prevents these targets from ever being activated even by other services.

---

## Power Optimization

### PowerTop auto-tune

PowerTop analyzes power usage and automatically applies optimizations to reduce CPU wakeups and idle power draw.

```bash
apt install powertop -y
```

Create a systemd service to run PowerTop auto-tune on every boot:

```bash
nano /etc/systemd/system/powertop.service
```

```
[Unit]
Description=PowerTop Auto Tune
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/powertop --auto-tune
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable it:

```bash
systemctl daemon-reload
systemctl enable powertop
systemctl start powertop
```

### CPU Governor — Powersave

The CPU governor controls how the CPU scales frequency. `powersave` keeps the CPU at lower frequencies when idle, reducing heat and power consumption. Since this is a server doing mostly I/O work (not CPU-intensive tasks), powersave is appropriate.

Set powersave for all CPU cores immediately:

```bash
echo powersave | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

Make it permanent across reboots:

```bash
nano /etc/systemd/system/cpu-powersave.service
```

```
[Unit]
Description=Set CPU Governor to Powersave
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo powersave | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable cpu-powersave
systemctl start cpu-powersave
```

### Disable Bluetooth

The server doesn't need Bluetooth. Disabling it saves a small amount of power.

Disable immediately:

```bash
rfkill block bluetooth
```

Make permanent:

```bash
nano /etc/rc.local
```

```bash
#!/bin/bash
rfkill block bluetooth
exit 0
```

```bash
chmod +x /etc/rc.local
systemctl enable rc-local
```

### WiFi Power Save — Disable

WiFi power saving causes the adapter to periodically go into low-power mode, which can cause latency spikes and occasional disconnections. Disable it for a stable server connection.

```bash
iw dev <wifi-interface> set power_save off
```

Make permanent:

```bash
nano /etc/NetworkManager/conf.d/wifi-powersave.conf
```

```
[connection]
wifi.powersave = 2
```

`2` = disabled (always on). `3` = enabled (default).

---

## Battery Health (Important for Long-term Use)

Since the laptop is always plugged in, the battery is constantly at 100% charge. This degrades lithium batteries significantly over months and years.

**Recommended: Set charge limit to 80%**

Check if your laptop supports this:

```bash
ls /sys/class/power_supply/BAT0/
```

If you see `charge_control_end_threshold`:

```bash
# Set charge limit to 80%
echo 80 > /sys/class/power_supply/BAT0/charge_control_end_threshold
```

Make permanent:

```bash
nano /etc/systemd/system/battery-limit.service
```

```
[Unit]
Description=Set Battery Charge Limit
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo 80 > /sys/class/power_supply/BAT0/charge_control_end_threshold'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable battery-limit
systemctl start battery-limit
```

> Note: Support for charge limiting varies by laptop manufacturer and BIOS. Some laptops support it natively in BIOS settings instead.

---

## Summary of All Optimizations

| Optimization             | Effect                                              |
| ------------------------ | --------------------------------------------------- |
| Lid close → ignore       | Server keeps running with lid closed                |
| acpid lid script         | Screen turns off/on with lid                        |
| Mask sleep targets       | No accidental suspends                              |
| consoleblank=0           | Kernel doesn't blank screen (handled by lid script) |
| PowerTop auto-tune       | Reduces idle power draw automatically               |
| CPU powersave governor   | Lower CPU frequency at idle = less heat and power   |
| Bluetooth disabled       | Minor power saving, less RF interference            |
| WiFi power save off      | Stable connection, no disconnection spikes          |
| Battery charge limit 80% | Extends battery lifespan for always-plugged-in use  |
