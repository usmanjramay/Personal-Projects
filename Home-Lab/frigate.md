# Frigate NVR — Setup & Configuration

> Last updated: 2026-03-14
> Purpose: Document Frigate NVR container setup, networking, and storage configuration.

---

## 1. Container & Host Info

| Property | Detail |
|---|---|
| Type | LXC 101 (privileged container on Proxmox) |
| IP | 192.168.4.151 (static, configured in both Proxmox and `/etc/network/interfaces`) |
| MAC Address | BC:24:11:39:3B:21 |
| Web UI | http://192.168.4.151:5000 |
| Frigate Version | 0.17.0 |
| Runtime | Docker inside LXC |

---

## 2. Network Configuration

### Proxmox LXC Config (`/etc/pve/lxc/101.conf`)
```
net0: name=eth0,bridge=vmbr0,hwaddr=BC:24:11:39:3B:21,ip=192.168.4.151/22,gw=192.168.4.1,type=veth
```

### LXC Internal Config (`/etc/network/interfaces`)
```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.4.151
    netmask 255.255.252.0
    gateway 192.168.4.1
```

---

## 3. Software Stack

- **nginx** – reverse proxy (port 8971 internally)
- **Python backend** – port 5000
- **go2rtc** – RTSP re-streaming
- **ffmpeg workers** – per-camera decode/encode (software transcoding)
- **OpenVINO** – Intel AI inference for object detection (confirmed running)

---

## 4. Storage Configuration

### Bind Mount Architecture

| Layer | Path |
|---|---|
| Docker container | `/media/frigate` |
| LXC mount | `/media/frigate` (bind-mounted from host) |
| Proxmox host | `/srv/frigate-media` (served via Samba) |

### Docker Compose Volume Mapping
```yaml
- /media/frigate:/media/frigate
```

### Storage Specifications

- **LXC disk:** 20GB virtual (`vm-101-disk-0` in thin pool)
- **Samba share:** `frigate-media` — read-only access to `/srv/frigate-media`
- **Old path:** `/opt/frigate/storage` (removed 2026-03-14, space reclaimed via fstrim)

---

## 5. Performance & CPU Usage

| Metric | Value | Notes |
|---|---|---|
| Proxmox CPU | ~15–19% | With one camera |
| Frigate UI CPU | ~1% | Normal — only counts Python process, not nginx/go2rtc/ffmpeg/Docker daemon |
| Root cause of high CPU | Software transcoding | Due to camera rotation filter (now removed) |

---

## 6. Known Issues & Incidents

### Incident 1: LXC 101 IP Changed (2026-03-05)
**Problem:** DHCP lease changed from 192.168.4.151 → 192.168.4.21, making Frigate inaccessible.

**Fix:** Set static IP in both Proxmox LXC config and `/etc/network/interfaces` (see Section 2).

### Incident 2: ffmpeg Crash Loop (2026-03-05)
**Problem:** Camera went offline → ffmpeg crashed → infinite restart loop → Frigate UI unresponsive.

**Fix:** `pct exec 101 -- docker restart frigate`

**Root cause:** Normal behavior — Frigate's watchdog restarts ffmpeg immediately. Just restart Frigate when camera comes back online.

---

## 7. Key Commands

```bash
# Access Frigate LXC shell
pct enter 101

# Run command inside LXC without entering it
pct exec 101 -- <command>

# Restart Frigate Docker container
pct exec 101 -- docker restart frigate

# Check Frigate logs
pct exec 101 -- docker logs frigate --tail 100

# Check LXC network
pct exec 101 -- ip addr show eth0

# Stop LXC gracefully
pct shutdown 101 --timeout 60
```

---

## 8. Remote Access

- **Tailscale:** Accessible at 192.168.4.151:5000 via Tailscale tunnel
- **SSH:** `ssh root@192.168.4.151` (key-based authentication, no password)
- **Samba:** Frigate recordings available via `smb://192.168.4.250/frigate-media` (read-only)