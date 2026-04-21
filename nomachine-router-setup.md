# Casa Bermudez Server Docs
## NoMachine & Network Setup Guide

**Last Updated:** April 20, 2026
**Author:** Adreena Bermudez
**Network:** Casa Bermudez (192.168.X.x)

> ⚠️ **Note:** IP addresses, MAC addresses, SSIDs, and Tailscale IPs in this document have been replaced with placeholders for security. Replace `192.168.X.x`, `XX:XX:XX:XX:XX:XX`, and `[TAILSCALE-IP]` with your actual values from your private records.

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

**Subnet:** 192.168.X.x | **Router:** ASUS RT-BE58U

---

## ASUS RT-BE58U Router Setup

The ASUS RT-BE58U replaced an Eero 6 router. The migration was done with minimal downtime by pre-configuring the ASUS on the existing subnet before swapping the WAN cable. All devices reconnected automatically since the SSID and password were preserved.

### Key Router Configurations

| Setting | Value |
|---|---|
| Admin URL | http://192.168.X.1 |
| HTTPS Admin | https://192.168.X.1:8443 |
| Subnet | 192.168.X.x |
| DHCP Pool Start | 192.168.X.100 |
| DHCP Pool End | 192.168.X.253 |
| Primary DNS | 192.168.X.XXX (Pi1 / Pi-hole) |
| Secondary DNS | 192.168.X.XXX (Pi2 / Pi-hole) |
| IPv6 | Disabled |
| Remote WAN Access | Disabled |
| Advertise Router IP as DNS | Disabled |
| DNS Rebind Protection | Disabled (required for Pi-hole) |

### DHCP Static Reservations

| Hostname | MAC Address | IP Address |
|---|---|---|
| Proxmox (GMKtec G3 Pro) | XX:XX:XX:XX:XX:XX | 192.168.X.XX |
| Ubuntu Server VM | XX:XX:XX:XX:XX:XX | 192.168.X.XX |
| CasaBermudezPi (Pi1) | XX:XX:XX:XX:XX:XX | 192.168.X.XXX |
| CasaBermudezPi2 (Pi2) | XX:XX:XX:XX:XX:XX | 192.168.X.XXX |
| Portainer LXC | XX:XX:XX:XX:XX:XX | 192.168.X.XXX |

### WiFi Configuration

- **SSID:** [Your Network Name]
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
Pi-hole (Pi1 primary / Pi2 secondary)
    ↓ port 5335
Unbound (recursive DNS resolver)
    ↓
Internet Root Servers (no upstream DNS provider)
```

### Pi1 — Primary (Raspberry Pi 4)

- **Pi-hole:** Listens on port 53, blocks ads and trackers
- **Unbound:** Listens on port 5335, resolves DNS recursively
- **Blocklists (~258K domains):** StevenBlack, Adaway, Easylist, Easyprivacy, AdGuardDNS

### Pi2 — Secondary / Failover (Raspberry Pi Zero 2W)

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

> ⚠️ **Download note:** NoMachine's download server blocks automated `wget`/`curl` requests and redirects to the homepage. Always download via browser, then SCP the `.deb` file to the target machine.

### Installation by Machine

#### Pi1 — CasaBermudezPi

**Architecture:** arm64 (aarch64) | **OS:** Debian GNU/Linux 13 (trixie)
**Desktop:** X11 (switched from Wayland/labwc via raspi-config)
**Display Manager:** LightDM
**Package:** `nomachine_9.4.14_1_arm64.deb`

Download via browser from [nomachine.com/download](https://www.nomachine.com/download), then SCP to the Pi:
```bash
scp ~/Downloads/nomachine_9.4.14_1_arm64.deb [YOUR-USERNAME]@[PI1-IP]:~/
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

Verify services after reboot:
```
NX> 111 New connections to NoMachine server are enabled.
NX> 161 Enabled service: nxserver.
NX> 161 Enabled service: nxnode.
NX> 161 Enabled service: nxd.
```

---

#### Pi2 — CasaBermudezPi2

**Architecture:** armhf (armv7l) | **OS:** Raspbian GNU/Linux 13 (trixie)
**Desktop:** X11 | **Display Manager:** LightDM
**Package:** `nomachine_9.4.14_1_armhf.deb`

> ⚠️ Pi2 runs a **32-bit armhf OS** despite having a 64-bit capable processor. Always use the **armhf** package for Pi2, NOT arm64.

```bash
scp ~/Downloads/nomachine_9.4.14_1_armhf.deb [YOUR-USERNAME]@[PI2-IP]:~/
sudo dpkg -i ~/nomachine_9.4.14_1_armhf.deb
```

Same Wayland → X11 switch applies via `sudo raspi-config`.

---

#### Ubuntu Server VM

**Architecture:** amd64 | **OS:** Ubuntu 24.04.4 LTS
**Desktop:** XFCE4 (installed post-setup) | **Display Manager:** LightDM
**Package:** `nomachine_9.4.14_1_amd64.deb`

Ubuntu Server is headless by default — install XFCE first:
```bash
sudo apt update && sudo apt install xfce4 xfce4-goodies lightdm -y
```

SCP and install NoMachine:
```bash
scp ~/Downloads/nomachine_9.4.14_1_amd64.deb [YOUR-USERNAME]@[UBUNTU-IP]:~/
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

Restart and reboot:
```bash
sudo systemctl restart lightdm
sudo reboot
```

---

#### GMKtec G3 Pro / Proxmox Host

NoMachine was intentionally skipped for the Proxmox host. The Proxmox web UI provides full VM and container management. Installing a desktop environment on a hypervisor host adds unnecessary overhead and is not best practice.

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
- **Host:** machine's local IP address
- **Port:** 4000
- **Protocol:** NX

### NoMachine Keyboard Shortcut

Press **Ctrl+Alt+0** at any time during a session to open the NoMachine control menu (resize, audio, disconnect, file transfer, etc.).

---

## Remote Access via Tailscale

Tailscale creates a private encrypted mesh network (tailnet) using WireGuard so you can reach your machines from anywhere without port forwarding or exposing port 4000 to the internet.

### Why Tailscale over Port Forwarding

- No ports exposed to the internet
- No dynamic DNS required
- Encrypted WireGuard tunnels between all devices
- Works through firewalls and NAT automatically

### Tailscale Installation

Run on each machine you want to reach remotely:
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
# Open the URL it prints in your browser to authenticate
```

Get the machine's Tailscale IP:
```bash
tailscale ip -4
```

### Tailscale IP Reference

| Machine | Tailscale IP |
|---|---|
| Proxmox Host (GMKtec) | [TAILSCALE-IP] |
| Pi1 (CasaBermudezPi) | [TAILSCALE-IP] |
| Pi2 (CasaBermudezPi2) | [TAILSCALE-IP] |

### Connecting Remotely via NoMachine + Tailscale

In the NoMachine client, click **Add** and enter the Tailscale IP instead of the local IP. Everything else is identical — port 4000, NX protocol, same credentials.

> When on your home network, use local IPs for best performance. Use Tailscale IPs only when connecting from outside your home.

---

## Device Reference

| Device | Role | Local IP | Port(s) | Access |
|---|---|---|---|---|
| ASUS RT-BE58U | Router | 192.168.X.1 | 80 / 8443 | http://192.168.X.1 |
| GMKtec G3 Pro | Proxmox Host / Jellyfin | 192.168.X.XX | 8006 / 8096 | Proxmox UI / Jellyfin |
| Ubuntu Server VM | General lab VM | 192.168.X.XX | 22 / 4000 | SSH / NoMachine |
| Windows 11 VM | Windows lab VM | 192.168.X.XX | 3389 | RDP |
| Pi1 (CasaBermudezPi) | Pi-hole primary / Unbound | 192.168.X.XXX | 53 / 5335 / 4000 | NoMachine / SSH |
| Pi2 (CasaBermudezPi2) | Pi-hole secondary / Unbound | 192.168.X.XXX | 53 / 5335 / 4000 | NoMachine / SSH |
| Portainer LXC | Docker / container management | 192.168.X.XXX | 9443 | https://[IP]:9443 |
