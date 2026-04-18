# Home Lab

## What This Is

A home server running on a 2016 MacBook Pro with Proxmox VE, hosting Home Assistant and Frigate (camera system). Runs headless 24/7, managed via web UI and SSH. Remote access via Tailscale subnet routing.

## Current Focus

- System is stable and operational as of 2026-03-22
- SSH key-based access configured on all systems (`~/.ssh/id_ed25519`), no passwords
- Claude can SSH into all servers via AppleScript bridge (Control your Mac MCP) over Tailscale
- Graceful shutdown watchdog: triggers at ≤20% battery, sequence: Frigate → HA → Samba → Host
- Automated backups: HA daily at 4am (7-day retention), Proxmox weekly Sunday 3am (ZSTD, 2 snapshots)

## Key Decisions

- Proxmox as hypervisor on repurposed MacBook Pro
- Tailscale on Proxmox host only (not inside HA — would break Thread/Matter)
- Flat subnet, no VLANs — IPv6 active for Thread/Matter devices
- Samba on Proxmox host for file sharing (NAS, Frigate recordings, USB drives)
- Frigate runs in LXC container; Home Assistant in QEMU VM
- Frigate media stored on Proxmox host (`/srv/frigate-media`), bind-mounted into LXC, shared via Samba
- LXC 101 (Frigate) uses static IP (was DHCP, caused outage 2026-03-05 — fixed)
- Power loss watchdog handles graceful shutdown sequence

## Reference Docs

- Full system context (hardware, network, storage, guests): `homelab_context.md`
- Frigate camera setup: `frigate.md`
- Home Assistant configuration: `home_assistant.md`
- Samba file sharing: `samba_fileshare.md`
- Tailscale VPN setup: `tailscale_vpn.md`

## Quick Reference

| Resource | Address |
|----------|---------|
| Proxmox Web UI | https://192.168.4.250:8006 |
| Home Assistant | http://192.168.4.150:8123 |
| Frigate | http://192.168.4.151:5000 |
| Samba | smb://192.168.4.250 |
| Proxmox Tailscale IP | 100.117.234.128 |

## Do Not

- Never install Tailscale inside Home Assistant VM — breaks Thread/Matter
- Never assume VLAN segmentation — everything is on flat 192.168.4.0/22
