# 🟢 Raspberry Pi 4 — Purpur Minecraft Server

A headless Minecraft server running on a Raspberry Pi 4, managed via SSH. Built as a personal infrastructure project to learn Linux server administration, systemd service management, and network configuration.

![Raspberry Pi](https://img.shields.io/badge/Hardware-Raspberry%20Pi%204-red?style=flat-square)
![Purpur](https://img.shields.io/badge/Server-Purpur-purple?style=flat-square)
![Linux](https://img.shields.io/badge/OS-Raspberry%20Pi%20OS-green?style=flat-square)
![Status](https://img.shields.io/badge/Status-Running-brightgreen?style=flat-square)

## Hardware

| Component | Details |
|---|---|
| Board | Raspberry Pi 4 |
| Storage | MicroSD / USB SSD |
| Network | Ethernet (headless) |
| Access | SSH only |

## Software Stack

- **OS:** Raspberry Pi OS Lite (headless)
- **Server Software:** [Purpur](https://purpurmc.org/) (Minecraft Fork)
- **Runtime:** Java 21
- **Process Management:** systemd

## Setup

### Requirements

- Raspberry Pi 4 with Raspberry Pi OS Lite
- Java 21 installed (`sudo apt install openjdk-21-jre-headless`)
- Git

### Installation
```bash
# Create server directory
sudo mkdir -p /srv/minecraft/server
cd /srv/minecraft/server

# Download Purpur JAR (check https://purpurmc.org for latest version)
wget https://api.purpurmc.org/v2/purpur/1.21.4/latest/download -O purpur.jar

# Accept EULA
echo "eula=true" > eula.txt

# Start the server once to generate configs
java -Xms1G -Xmx2G -jar purpur.jar nogui
```

### systemd Service (Autostart)

Copy `minecraft.service` to `/etc/systemd/system/` and enable it:
```bash
sudo cp minecraft.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable minecraft
sudo systemctl start minecraft
```

### Configuration

Key settings in `server.properties`:
```properties
server-port=25565
max-players=10
view-distance=8
```

Performance is tuned for the Pi's 4GB RAM — `view-distance` kept low for stability.

## What I Learned

- Linux server administration & file system structure (`/srv`, `/etc/systemd`)
- Managing services with `systemctl` (enable, start, stop, status, logs)
- SSH-based remote administration (no monitor/keyboard needed)
- Java memory tuning for constrained hardware (`-Xms`, `-Xmx` flags)
- Network configuration and port forwarding

## Project Structure
```
.
├── server.properties     # Main server config
├── purpur.yml            # Purpur-specific settings
├── minecraft.service     # systemd service definition
└── README.md
```

## Useful Commands
```bash
# Check server status
sudo systemctl status minecraft

# View live logs
sudo journalctl -u minecraft -f

# Restart server
sudo systemctl restart minecraft
```
