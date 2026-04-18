# Tailscale Remote Access Setup

> Last updated: 2026-03-14
> Purpose: Document the Tailscale subnet router configuration for secure remote access.

---

## Overview

Tailscale is installed on the Proxmox host (192.168.4.250) as a subnet router, enabling secure remote access to all home lab services without port forwarding.

---

## Installation & Configuration

| Property | Detail |
|---|---|
| Installed on | Proxmox host (192.168.4.250) |
| Role | Subnet router |
| Subnet advertised | `192.168.4.0/22` |
| Tailscale account | `usmanramay@gmail.com` |
| Admin panel | https://login.tailscale.com/admin/machines |

---

## Proxmox Tailscale IP

`100.117.234.128`

---

## Remote Access to Services

All services are accessible remotely via Tailscale tunnel using their local IPs and ports:

| Service | Address | Port |
|---|---|---|
| Proxmox UI | `192.168.4.250` | 8006 |
| Home Assistant | `192.168.4.150` | 8123 |
| Frigate | `192.168.4.151` | 5000 |
| Samba | `192.168.4.250` | 445 |

---

## Key Benefits

- **No port forwarding required** — Tailscale handles NAT traversal
- **Encrypted tunnel** — All traffic is encrypted end-to-end
- **Easy remote management** — Access home lab from anywhere via Tailscale
- **Subnet coverage** — Entire 192.168.4.0/22 subnet is accessible remotely

---

## Important Notes

- Tailscale is installed **only on the Proxmox host**, not inside Home Assistant VM
- Installing Tailscale inside Home Assistant would break Thread/Matter IPv6 routing
- The subnet router advertises the entire local network, allowing seamless remote access