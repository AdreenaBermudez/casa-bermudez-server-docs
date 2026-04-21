# Casa Bermudez Server Docs
## NoMachine & Network Setup Guide

**Last Updated:** April 20, 2026
**Author:** Adreena Bermudez
**Network:** Casa Bermudez (192.168.4.x)

---

## Table of Contents

1. [Network Overview](#network-overview)
2. [ASUS RT-BE58U Router Setup](#asus-rt-be58u-router-setup)
3. [DNS Privacy Stack](#dns-privacy-stack)
4. [NoMachine Remote Desktop Setup](#nomachine-remote-desktop-setup)
5. [Remote Access via Tailscale](#remote-access-via-tailscale)
6. [Device Reference](#device-reference)

---

## Network Overview

Casa Bermudez runs a privacy-focused home network built on a layered DNS chain, self-hosted services, and secure remote access via NoMachine and Tailscale.

**DNS Chain:** Devices → Pi-hole :53 → Unbound :5335 → Internet Root Servers

**Subnet:** 192.168.4.x | **Router:** ASUS RT-BE58U at 192.168.4.1

---

## ASUS RT-BE58U Router Setup

The ASUS RT-BE58U replaced an Eero 6 router. The migration was done with minimal downtime by pre-configuring the ASUS on the existing 192.168.4.x subnet before swapping the WAN cable.

### Key Router Configurations

| Setting | Value |
|---|---|
| Admin URL | http://192.168.4.1 |
| HTTPS Admin | https://192.168.4.1:8443 |
| Subnet | 192.168.4.x |
| DHCP Pool Start | 192.168.4.100 |
| DHCP Pool End | 192.168.4.253 |
| Primary DNS | 192.168.4.214 (Pi1 / Pi-hole) |
| Secondary DNS | 192.168.4.216 (Pi2 / Pi-hole) |
| IPv6 | Disabled |
| Remote WAN Access | Disabled |
| Advertise Router IP as DNS | Disabled |
| DNS Rebind Protection | Disabled (required for Pi-hole) |

### DHCP Static Reservations

| Hostname | MAC Address | IP Address |
|---|---|---|
| Proxmox (GMKtec G3 Pro) | E0:51:D8:21:14:52 | 192.168.4.10 |
| Ubuntu Server VM | BC:24:11:64:F0:35 | 192.168.4.11 |
| CasaBermudezPi (Pi1) | DC:A6:32:92:20:CC | 192.168.4.214 |
| CasaBermudezPi2 (Pi2) | 88:A2:9E:80:53:92 | 192.168.4.216 |
| Portainer LXC | BC:24:11:9B:62:DD | 192.168.4.249 |

### WiFi Configuration

- **SSID:** Peanut Loves Butter (preserved from Eero for seamless device reconnection)
- **Security:** WPA3-Personal
- **WiFi 7:** Enabled on supported bands

### Why DNS Rebind Protection Was Disabled

Pi-hole handles its own DNS rebind protection. Leaving it enabled on the ASUS caused conflicts with Pi-hole's DNS responses, resulting in failed lookups. Disabling it at the router level while Pi-hole remains active maintains full protection.

---

## DNS Privacy Stack

### Architecture

```
Devices (all clients)
    ↓ port 53
Pi-hole (192.168.4.214 primary / 192.168.4.216 secondary)
    ↓ port 5335
Unbound (recursive DNS resolver)
    ↓
Internet Root Servers (no upstream DNS provider)
```

### Pi1 — Primary (192.168.4.214 / Raspberry Pi 4)

- **Pi-hole:** Listens on port 53, blocks ads and trackers
- **Unbound:** Listens on port 5335, resolves DNS recursively
- **Blocklists (~258K domains):** StevenBlack, Adaway, Easylist, Easyprivacy, AdGuardDNS

### Pi2 — Secondary / Failover (192.168.4.216 / Raspberry Pi Zero 2W)

- **Pi-hole:** Same configuration as Pi1
- **Unbound:** Same configuration as Pi1
- Automatically used by clients if Pi1 is unreachable

### Why This Stack

- No reliance on upstream DNS providers (Google, Cloudflare, ISP)
- Full recursive resolution from root servers
- Network-wide ad and tracker blocking
- Dual-Pi redundancy for failover

---

## NoMachine Remote Desktop Setup

NoMachine provides full GUI remote desktop access to all Casa Bermudez machines using its NX protocol on port 4000.

### What is NoMachine?

NoMachine streams your remote machine's full desktop in real time using its proprietary NX protocol. It supports audio forwarding, file transfer, clipboard sharing, session recording, and multi-monitor support. The free edition is fully featured for personal use.

### Version Installed

**NoMachine 9.4.14** (latest as of April 2026)

### Installation by Machine

#### Pi1 — CasaBermudezPi (192.168.4.214)

**Architecture:** arm64 (aarch64) | **OS:** Debian GNU/Linux 13 (trixie)
**Desktop:** X11 (switched from Wayland/labwc via raspi-config)
**Display Manager:** LightDM

**Download:** NoMachine for Raspberry Pi DEB (arm64) from nomachine.com/download

Since `wget` redirects to the homepage, download via browser on your Mac then SCP to the Pi:
```bash
scp ~/Downloads/nomachine_9.4.14_1_arm64.deb adreenabermudez87@192.168.4.214:~/
```

Install:
```bash
sudo dpkg -i ~/nomachine_9.4.14_1_arm64.deb
sudo /usr/NX/bin/nxserver --status
```

**Important — Wayland to X11 switch required:**
Pi OS defaults to Wayland (labwc), which NoMachine free edition does not support. Switch to X11:
```bash
sudo raspi-config
# Navigate: Advanced Options → Wayland → W1 X11 → OK → Finish → Reboot
```

Verify services:
```
NX> 111 New connections to NoMachine server are enabled.
NX> 161 Enabled service: nxserver.
NX> 161 Enabled service: nxnode.
NX> 161 Enabled service: nxd.
```

---

#### Pi2 — CasaBermudezPi2 (192.168.4.216)

**Architecture:** armhf (armv7l) | **OS:** Raspbian GNU/Linux 13 (trixie)
**Desktop:** X11 | **Display Manager:** LightDM

**Download:** NoMachine for Raspberry Pi DEB (armhf) — use armhf NOT arm64 for Pi2.

```bash
scp ~/Downloads/nomachine_9.4.14_1_armhf.deb adreenabermudez87@192.168.4.216:~/
sudo dpkg -i ~/nomachine_9.4.14_1_armhf.deb
```

Same Wayland → X11 switch applies via `sudo raspi-config`.

---

#### Ubuntu Server VM (192.168.4.11)

**Architecture:** amd64 | **OS:** Ubuntu 24.04.4 LTS
**Desktop:** XFCE4 (installed post-setup) | **Display Manager:** LightDM

**Download:** NoMachine for Linux DEB (amd64) from nomachine.com/download

```bash
scp ~/Downloads/nomachine_9.4.14_1_amd64.deb adreenabermudez87@192.168.4.11:~/
```

Since Ubuntu Server is headless by default, install XFCE first:
```bash
sudo apt update && sudo apt install xfce4 xfce4-goodies lightdm -y
```

Install NoMachine:
```bash
sudo dpkg -i ~/nomachine_9.4.14_1_amd64.deb
```

**Fix LightDM session (required):** LightDM defaults to the "ubuntu" GNOME session which isn't installed. Force it to use XFCE:
```bash
sudo nano /etc/lightdm/lightdm.conf
```

Add:
```
[Seat:*]
user-session=xfce
greeter-session=unity-greeter
```

Restart LightDM:
```bash
sudo systemctl restart lightdm
sudo reboot
```

---

#### GMKtec G3 Pro / Proxmox Host (192.168.4.10)

NoMachine was intentionally skipped for the Proxmox host. The Proxmox web UI at `https://192.168.4.10:8006` provides full VM and container management. Installing a desktop environment on a hypervisor host adds unnecessary overhead.

---

### NoMachine Client Installation

Install the NoMachine client on each device you connect from:

| Platform | Download |
|---|---|
| Mac | nomachine.com/download → macOS → .dmg |
| Windows | nomachine.com/download → Windows → .exe |
| Linux | nomachine.com/download → Linux → .deb or .rpm |
| Android | Google Play Store → search "NoMachine" |
| iOS | App Store → search "NoMachine" |

### Connecting (Local Network)

NoMachine auto-discovers machines on the same subnet. They'll appear in the Machines list automatically. Double-click to connect and enter your credentials.

To add manually:
- **Host:** machine IP (e.g. 192.168.4.214)
- **Port:** 4000
- **Protocol:** NX

### NoMachine Keyboard Shortcut

Press **Ctrl+Alt+0** at any time during a session to open the NoMachine control menu (resize, audio, disconnect, etc.).

---

## Remote Access via Tailscale

Tailscale creates a private encrypted mesh network (tailnet) so you can connect to your machines from anywhere without exposing ports to the internet.

### Why Tailscale over Port Forwarding

- No port 4000 exposed to the internet
- No dynamic DNS needed
- Encrypted WireGuard tunnels between devices
- Works through firewalls and NAT

### Tailscale Installation

Run on each machine you want to reach remotely:
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
# Open the URL it prints to authenticate
```

Get the machine's Tailscale IP:
```bash
tailscale ip -4
```

### Tailscale IP Reference

| Machine | Tailscale IP |
|---|---|
| Proxmox Host | 100.72.79.47 |
| Pi1 | (run `tailscale ip -4` on Pi1) |
| Pi2 | (run `tailscale ip -4` on Pi2) |

### Connecting Remotely via NoMachine + Tailscale

In NoMachine client, click **Add** and enter:
- **Host:** Tailscale IP of the target machine
- **Port:** 4000
- **Protocol:** NX

Everything else (credentials, session behavior) is identical to local connections.

---

## Device Reference

| Device | Role | Local IP | Port | Access |
|---|---|---|---|---|
| ASUS RT-BE58U | Router | 192.168.4.1 | 80 / 8443 | http://192.168.4.1 |
| GMKtec G3 Pro | Proxmox Host / Jellyfin | 192.168.4.10 | 8006 / 8096 | https://192.168.4.10:8006 |
| Ubuntu Server VM | General lab VM | 192.168.4.11 | 22 / 4000 | SSH / NoMachine |
| Windows 11 VM | Windows lab VM | 192.168.4.12 | 3389 | RDP |
| Pi1 (CasaBermudezPi) | Pi-hole primary / Unbound | 192.168.4.214 | 53 / 5335 / 4000 | NoMachine / SSH |
| Pi2 (CasaBermudezPi2) | Pi-hole secondary / Unbound | 192.168.4.216 | 53 / 5335 / 4000 | NoMachine / SSH |
| Portainer LXC | Docker / container management | 192.168.4.249 | 9443 | https://192.168.4.249:9443 |
