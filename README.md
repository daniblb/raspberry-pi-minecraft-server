# 🟢 Raspberry Pi 4 — Purpur Minecraft Server

A headless Minecraft server running on a Raspberry Pi 4, managed via SSH. Built as a personal infrastructure project to learn Linux server administration, systemd service management, and network configuration.

![Raspberry Pi](https://img.shields.io/badge/Hardware-Raspberry%20Pi%204-red?style=flat-square)
![Purpur](https://img.shields.io/badge/Server-Purpur-purple?style=flat-square)
![Java](https://img.shields.io/badge/Runtime-Java%2021-orange?style=flat-square)
![Linux](https://img.shields.io/badge/OS-Raspberry%20Pi%20OS%20Lite-green?style=flat-square)
![Status](https://img.shields.io/badge/Status-Running-brightgreen?style=flat-square)

---

## Table of Contents

1. [Hardware](#hardware)
2. [Software Stack](#software-stack)
3. [Prerequisites](#prerequisites)
4. [Installation](#installation)
5. [systemd Service](#systemd-service-autostart)
6. [Configuration](#configuration)
7. [Connecting to the Server](#connecting-to-the-server)
8. [Port Forwarding (LAN → Internet)](#port-forwarding-lan--internet)
9. [Useful Commands](#useful-commands)
10. [Updating Purpur](#updating-purpur)
11. [Backups](#backups)
12. [Troubleshooting](#troubleshooting)
13. [Project Structure](#project-structure)
14. [What I Learned](#what-i-learned)

---

## Hardware

| Component | Details |
|-----------|---------|
| Board | Raspberry Pi 4 (4 GB RAM recommended) |
| Storage | MicroSD (32 GB+) or USB SSD (recommended for better I/O) |
| Network | Ethernet — **wired connection strongly recommended** for stability |
| Access | SSH only (no monitor or keyboard required) |

> **Note:** A USB SSD significantly improves world read/write performance compared to a MicroSD card. If using MicroSD, prefer a high-endurance A2-rated card.

---

## Software Stack

| Layer | Technology |
|-------|-----------|
| OS | Raspberry Pi OS Lite (64-bit, headless) |
| Server | [Purpur](https://purpurmc.org/) — Paper fork with extra config options |
| Runtime | Java 21 (`openjdk-21-jre-headless`) |
| Process Management | systemd |

**Why Purpur?** Purpur extends Paper (which extends Spigot/Bukkit) with additional gameplay toggles and performance tuning options. It's compatible with all Spigot/Paper plugins.

---

## Prerequisites

- Raspberry Pi 4 with **Raspberry Pi OS Lite** flashed and SSH enabled
- Static local IP assigned to the Pi (via router DHCP reservation or `nmtui`)
- Internet access on the Pi
- A machine on the same network to SSH from

### Enable SSH on first boot (headless)

When flashing the SD card with **Raspberry Pi Imager**, click **"OS Customisation"** and:
- Set hostname (e.g. `minecraft-pi`)
- Enable SSH with a password or public key
- Configure your Wi-Fi (or just use Ethernet)

---

## Installation

### 1. SSH into the Pi

```bash
ssh pi@<your-pi-ip>
# Example: ssh pi@192.168.1.42
```

To find the Pi's IP: check your router's DHCP table, or run `hostname -I` directly on the Pi.

### 2. Install Java 21

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y openjdk-21-jre-headless
java -version  # Should output: openjdk version "21..."
```

### 3. Create the server directory

```bash
sudo mkdir -p /srv/minecraft/server
sudo chown $USER:$USER /srv/minecraft/server
cd /srv/minecraft/server
```

> `/srv/` is the standard Linux directory for service data. Keeping it here makes systemd integration cleaner.

### 4. Download Purpur

```bash
# Download latest Purpur build for Minecraft 1.21.4
wget "https://api.purpurmc.org/v2/purpur/1.21.4/latest/download" -O purpur.jar
```

Check [purpurmc.org](https://purpurmc.org/) for the latest Minecraft version and replace `1.21.4` if needed.

### 5. Accept the EULA

```bash
echo "eula=true" > eula.txt
```

Minecraft requires accepting Mojang's End User License Agreement before the server starts.

### 6. First start (generates config files)

```bash
java -Xms1G -Xmx2G -jar purpur.jar nogui
```

Wait for the message `Done (Xs)! For help, type "help"`, then stop the server:

```
stop
```

This generates `server.properties`, `purpur.yml`, `bukkit.yml`, `spigot.yml`, and the `world/` folder.

---

## systemd Service (Autostart)

The server runs as a systemd service so it starts automatically on boot and restarts on crash.

### `minecraft.service`

```ini
[Unit]
Description=Purpur Minecraft Server
After=network.target

[Service]
User=pi
WorkingDirectory=/srv/minecraft/server
ExecStart=/usr/bin/java -Xms1G -Xmx2G -jar purpur.jar nogui
ExecStop=/bin/kill -s SIGINT $MAINPID
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

> **Memory flags:** `-Xms1G` sets the initial heap, `-Xmx2G` the maximum. On a 4 GB Pi, `2G` leaves room for the OS. Do not set `-Xmx` higher than ~2.5G.

### Install the service

```bash
sudo cp minecraft.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable minecraft   # Autostart on boot
sudo systemctl start minecraft    # Start now
```

---

## Configuration

### `server.properties` (key settings)

```properties
server-port=25565
max-players=10
view-distance=8
simulation-distance=6
motd=My Minecraft Server
online-mode=true
difficulty=normal
gamemode=survival
```

| Setting | Value | Why |
|---------|-------|-----|
| `view-distance` | `8` | Reduced from default (10) to save RAM and CPU on Pi |
| `simulation-distance` | `6` | Limits entity processing range; big performance gain |
| `max-players` | `10` | Pi handles 3–5 concurrent players comfortably |
| `online-mode` | `true` | Keep `true` unless you need cracked/offline clients |

### `purpur.yml` (performance tweaks)

Purpur allows per-mob and per-gameplay tuning. Defaults are fine to start — edit only if you experience lag.

---

## Connecting to the Server

### Local network (LAN)

In Minecraft → **Multiplayer → Add Server**:

```
Server Address: <your-pi-ip>:25565
Example:        192.168.1.42:25565
```

If you set up DNS on your router (e.g. `minecraft-pi.local`), you can use that hostname instead.

### Over the internet

See [Port Forwarding](#port-forwarding-lan--internet) below. Then use your **public IP** or a dynamic DNS hostname:

```
Server Address: your.public.ip:25565
```

To find your public IP: [https://ifconfig.me](https://ifconfig.me)

---

## Port Forwarding (LAN → Internet)

> Skip this section if the server is **LAN-only**.

1. Log into your router's admin panel (usually `192.168.1.1` or `192.168.0.1`)
2. Find **Port Forwarding** (sometimes under "NAT" or "Firewall")
3. Create a rule:
   - **External port:** `25565`
   - **Internal IP:** your Pi's local IP (e.g. `192.168.1.42`)
   - **Internal port:** `25565`
   - **Protocol:** TCP (UDP not needed for Minecraft Java Edition)
4. Save and apply

**Static local IP:** Assign a DHCP reservation in your router for the Pi's MAC address, so its local IP never changes.

**Dynamic DNS (optional):** If your public IP changes regularly, use a free service like [DuckDNS](https://www.duckdns.org/) to get a fixed hostname (e.g. `yourname.duckdns.org`).

---

## Useful Commands

### Service management

```bash
# Check if server is running
sudo systemctl status minecraft

# View live logs (Ctrl+C to exit)
sudo journalctl -u minecraft -f

# View last 100 log lines
sudo journalctl -u minecraft -n 100

# Start / Stop / Restart
sudo systemctl start minecraft
sudo systemctl stop minecraft
sudo systemctl restart minecraft
```

### Sending console commands without attaching

Since the server runs as a systemd service (not in a screen/tmux session), you can use **mcrcon** to send commands:

```bash
# Install mcrcon
sudo apt install -y mcrcon

# Send a command (requires rcon to be enabled in server.properties)
mcrcon -H localhost -P 25575 -p <rcon-password> "say Hello from SSH"
```

Enable rcon in `server.properties`:

```properties
enable-rcon=true
rcon.port=25575
rcon.password=yourSecurePassword
```

> ⚠️ Never commit `server.properties` with a real `rcon.password` to a public repository. Add it to `.gitignore` or replace the value before pushing.

### World and disk

```bash
# Check disk usage of the world folder
du -sh /srv/minecraft/server/world/

# Check overall disk space
df -h
```

---

## Updating Purpur

1. Stop the server:
   ```bash
   sudo systemctl stop minecraft
   ```
2. Back up the current JAR:
   ```bash
   cp /srv/minecraft/server/purpur.jar /srv/minecraft/server/purpur.jar.bak
   ```
3. Download the new version:
   ```bash
   wget "https://api.purpurmc.org/v2/purpur/<NEW_VERSION>/latest/download" -O /srv/minecraft/server/purpur.jar
   ```
4. Start the server:
   ```bash
   sudo systemctl start minecraft
   sudo journalctl -u minecraft -f
   ```

---

## Backups

A simple backup script using `rsync` or `tar`:

```bash
#!/bin/bash
# backup.sh — run manually or via cron
TIMESTAMP=$(date +%Y-%m-%d_%H-%M)
BACKUP_DIR="/srv/minecraft/backups"
WORLD_DIR="/srv/minecraft/server/world"

mkdir -p "$BACKUP_DIR"
tar -czf "$BACKUP_DIR/world_$TIMESTAMP.tar.gz" -C /srv/minecraft/server world
echo "Backup saved: world_$TIMESTAMP.tar.gz"
```

### Automate with cron (daily at 4 AM)

```bash
crontab -e
# Add:
0 4 * * * /srv/minecraft/backup.sh
```

> Tip: Save backups to an external USB drive or use `rsync` to sync to another machine.

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| Server won't start | Java not installed or wrong version | `java -version`, ensure Java 21 |
| `Address already in use` | Port 25565 is taken | `sudo lsof -i :25565` to find the process |
| Can't connect from LAN | Firewall blocking port | `sudo ufw allow 25565/tcp` |
| Can't connect from internet | Port forwarding not set up | See [Port Forwarding](#port-forwarding-lan--internet) |
| Server crashes on startup | Not enough RAM allocated | Check `-Xmx` value; don't exceed ~2.5G |
| Extreme lag / TPS drops | Too many players or high view-distance | Lower `view-distance` and `simulation-distance` |
| World not saving | Disk full | `df -h`, clear old backups or logs |
| `eula.txt` error | EULA not accepted | `echo "eula=true" > eula.txt` |

### Check TPS (Ticks Per Second)

A healthy server runs at 20 TPS. Connect to the server console via mcrcon and run:

```
/tps
```

Values below 18 indicate the server is under load.

---

## Project Structure

```
.
├── server.properties     # Main server config (port, view-distance, gamemode, etc.)
├── purpur.yml            # Purpur-specific settings (mob behavior, gameplay toggles)
├── bukkit.yml            # Bukkit/Spigot base settings
├── spigot.yml            # Spigot performance settings
├── minecraft.service     # systemd unit file for autostart
├── backup.sh             # Optional: manual/cron backup script
├── eula.txt              # Mojang EULA acceptance (required)
├── world/                # Overworld chunks and player data
├── world_nether/         # Nether dimension data
├── world_the_end/        # End dimension data
├── plugins/              # Plugin JARs (if any)
├── logs/                 # Server log files (auto-rotated)
└── README.md
```

> `server.properties` and `purpur.yml` are the two files you'll edit most often. Everything in `world*/` is your actual map data — back these up regularly.

---

## What I Learned

- **Linux filesystem structure** — using `/srv/` for service data, `/etc/systemd/` for units
- **systemd service management** — `enable`, `start`, `stop`, `status`, `journalctl`
- **SSH-based remote administration** — fully headless setup, no monitor needed
- **Java memory tuning** — `-Xms`/`-Xmx` heap flags for constrained hardware
- **Network configuration** — port forwarding, DHCP reservations, static IP assignment
- **Security hygiene** — keeping secrets (RCON passwords) out of version control
- **Cron jobs** — automating backups with scheduled tasks

---

*Tested on Raspberry Pi 4 (4 GB) · Purpur 1.21.4 · Java 21 · Raspberry Pi OS Lite 64-bit*