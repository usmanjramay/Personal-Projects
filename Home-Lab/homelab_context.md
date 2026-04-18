# Home Lab — System Context

> Last updated: 2026-03-22
> Purpose: Provide complete system context for AI agents or humans picking up work on this setup.

---

## 1. Hardware

| Property | Detail |
|---|---|
| Machine | 2016 MacBook Pro |
| OS | Proxmox VE 9.1.1 |
| Kernel | 6.17.2-1-pve (x86_64) |
| Uptime pattern | Runs 24/7 on AC power |
| Battery health | 87.6% (449 charge cycles) |
| Battery runtime | ~68 minutes under full server load |
| Battery runtime on idle | Likely 90–120+ minutes |

The MacBook runs headless. All management is done via the Proxmox web UI at `https://192.168.4.250:8006` or via SSH.

---

## 2. Network

| Property | Detail |
|---|---|
| Subnet | 192.168.4.0/22 (255.255.252.0) |
| Gateway | 192.168.4.1 |
| Proxmox host IP | 192.168.4.250 |
| Bridge interface | vmbr0 |

All VMs and LXC containers are on the same flat subnet — no VLANs. IPv6 is active on the local network (important for Thread/Matter devices in Home Assistant).

---

## 3. Proxmox Host

| Property | Detail |
|---|---|
| IP | 192.168.4.250 |
| Web UI | https://192.168.4.250:8006 |
| Version | Proxmox VE 9.1.1 |
| Storage | `local` (boot/config) + `local-lvm` (VM/LXC disks) |
| Tailscale IP | 100.117.234.128 |
| Samba | Installed and running — `smb://192.168.4.250` |

### Guests at a Glance

| ID | Type | Name | IP | Status |
|---|---|---|---|---|
| 100 | QEMU VM | homeassistant | 192.168.4.150 | Running |
| 101 | LXC | frigate | 192.168.4.151 | Running |

### Storage Layout

| Volume | Size | Purpose | Mount |
|---|---|---|---|
| `pve-root` | 68GB | Proxmox OS + Samba shares | `/` |
| `pve-swap` | 7.6GB | Swap | — |
| `pve/data` (thin pool) | 138GB | VM/LXC virtual disks | — |
| └─ `vm-100-disk-1` | 32GB virtual | Home Assistant VM disk | via VM |
| └─ `vm-101-disk-0` | 20GB virtual | Frigate LXC disk | via LXC |
| └─ `nas-storage` | 100GB thin LV | NAS storage volume | `/srv/nas` |
| VG free | ~10GB | Unallocated buffer | — |
| EFI | ~1GB | Boot partition | `/boot/efi` |

---

## 4. Tailscale Remote Access

Installed 2026-03-14 on the Proxmox host as a subnet router.

| Property | Detail |
|---|---|
| Installation | Proxmox host only (NOT inside HA — would break Thread/Matter) |
| Subnet advertised | `192.168.4.0/22` |
| Proxmox Tailscale IP | `100.117.234.128` |
| Mac Tailscale IP | `100.68.93.126` |
| Tailscale account | `usmanramay@gmail.com` |
| Admin panel | https://login.tailscale.com/admin/machines |

All services (Proxmox UI :8006, Home Assistant :8123, Frigate :5000, Samba) are reachable remotely via their local IPs over the Tailscale tunnel. No port forwarding required.

5. Samba File Sharing
Installed 2026-03-14 on the Proxmox host.

| Share | Path | Purpose | Notes |
|---|---|---|---|
| `shared` | `/srv/samba/shared` | General purpose folder | On `pve-root` (~58GB available) |
| `frigate-media` | `/srv/frigate-media` | Frigate recordings | Read-only; bind-mounted from LXC 101 |
| `nas` | `/srv/nas` | NAS storage | Dedicated 100GB thin LV; recycle bin enabled |
| `usbdrive1` | `/mnt/usbdrive1` | USB drive slot 1 | Empty until drive plugged in |
| `usbdrive2` | `/mnt/usbdrive2` | USB drive slot 2 | Empty until drive plugged in |

**Connect from Mac:** Finder → Cmd+K → `smb://192.168.4.250` → username: `root`

**Recycle bin:** Enabled on `nas` share only. Files up to 2GB retained for 2 days in `.recycle` folder. Cron deletes older files daily at 3am.

**macOS compatibility:** `vfs objects = fruit streams_xattr` configured globally — required for stable Finder connections.

**To add a USB drive:** Run `bash add-usb-drive.sh` from the Homelab folder on the Mac.