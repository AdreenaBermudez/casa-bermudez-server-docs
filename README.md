# Casa Bermudez Home Lab — Server Docs

Documentation for the Casa Bermudez home lab infrastructure, including network setup, self-hosted services, and device configurations.

---

## Infrastructure Overview

| Device | Role | IP |
|---|---|---|
| ASUS RT-BE58U | Primary router / DHCP | 192.168.4.1 |
| GMKtec G3 Pro | Proxmox VE hypervisor + Jellyfin | 192.168.4.10 |
| Intel NUC 10 | Nextcloud + backup target | 192.168.4.20 |
| Raspberry Pi 1 | Primary DNS (Pi-hole + Unbound) | 192.168.4.214 |
| Raspberry Pi 2 | Secondary DNS (failover) | 192.168.4.216 |
| ASUS RT-BE58 Go | WiFi 7 travel router | 192.168.4.195 |

---

## Services Running

- **Proxmox VE** — bare-metal hypervisor hosting VMs and LXC containers
- **Jellyfin** — local media streaming server
- **Nextcloud** — self-hosted personal cloud storage (Docker)
- **Pi-hole + Unbound** — network-wide ad blocking and recursive DNS privacy
- **Tailscale** — zero-config VPN for secure remote access
- **Portainer CE** — Docker container management

---

## Network & DNS Stack

All devices on the network route DNS through a two-node Raspberry Pi cluster:

```
Device → Pi-hole :53 → Unbound :5335 → Root DNS servers
```

- No upstream ISP DNS used
- Pi2 runs an identical stack as a failover secondary
- ~258K domains blocked across multiple blocklists

---

## Remote Access

Tailscale is installed on the Proxmox host, Intel NUC, and ASUS RT-BE58 Go travel router, providing secure remote access to all services without port forwarding.

---

## Backup Strategy

| What | How | Schedule |
|---|---|---|
| Proxmox VMs + LXCs | NFS → Intel NUC `/mnt/data/backups` | Nightly 2:00 AM |
| Nextcloud data | rclone → Backblaze B2 | Nightly 3:00 AM |

---

## Documents

| File | Description |
|---|---|
| `Casa_Bermudez_Home_Lab_Final.docx` | Complete setup documentation — all devices, IPs, configs |
| `nomachine-router-setup.md` | Router migration notes (Eero → ASUS RT-BE58U) |

---

## Tech Stack

`Ubuntu Server 24.04` `Proxmox VE` `Docker` `Nextcloud` `Pi-hole` `Unbound` `Tailscale` `Portainer` `Jellyfin` `rclone` `Backblaze B2` `NFS`
