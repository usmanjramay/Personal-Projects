# Samba File Sharing Setup

> Last updated: 2026-03-14
> Purpose: Document the Samba file sharing configuration on the Proxmox host.

---

## Overview

Samba file sharing is installed and configured on the Proxmox host (192.168.4.250) with 5 shares for centralized file access across the home lab network.

---

## Shares Configuration

| Share | Path | Purpose | Notes |
|---|---|---|---|
| `shared` | `/srv/samba/shared` | General purpose folder | On `pve-root` (~58GB available) |
| `frigate-media` | `/srv/frigate-media` | Frigate recordings | Read-only; bind-mounted from LXC 101 |
| `nas` | `/srv/nas` | NAS storage | Dedicated 100GB thin LV; recycle bin enabled |
| `usbdrive1` | `/mnt/usbdrive1` | USB drive slot 1 | Empty until drive plugged in |
| `usbdrive2` | `/mnt/usbdrive2` | USB drive slot 2 | Empty until drive plugged in |

---

## Connection Details

**Connect from Mac:** Finder → Cmd+K → `smb://192.168.4.250` → username: `root`

---

## macOS Compatibility

Configured with `vfs objects = fruit streams_xattr` globally — required for stable Finder connections and proper file metadata handling.

---

## Recycle Bin

Enabled on the `nas` share only. Files up to 2GB are retained for 2 days in the `.recycle` folder. A cron job deletes older files daily at 3am.

---

## USB Drive Management

To add a USB drive to the shares, run the setup script:

```bash
bash add-usb-drive.sh
```

This script configures the USB drive and makes it available via Samba.