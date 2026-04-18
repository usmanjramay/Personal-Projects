# Home Assistant — Setup & Integrations

> Last updated: 2026-04-06
> Purpose: Home Assistant instance overview and active integrations.

---

## Instance Details

| Property | Detail |
|---|---|
| Host | VM 100 on Proxmox (192.168.4.250) |
| IP | 192.168.4.150 (DHCP — has been stable) |
| Web UI | http://192.168.4.150:8123 |
| OS | Home Assistant OS (HAOS) |
| VM type | QEMU VM on Proxmox VE 9.1.1 |
| User | Usman |

### Remote Access
- **Nabu Casa HA Cloud** subscription active (taken 2026-04-04)
- Enables remote HTTPS access, Alexa skill, Google Home integration
- Remote domain provisioned via Settings → Home Assistant Cloud

---

## Active Integrations

- **Matter** — smart home devices over Matter protocol (WiFi + Thread)
- **Thread** — Thread network management (border router visibility)
- **Frigate** — NVR integration connected to LXC 101 at 192.168.4.151
- **HACS** — installed (visible in sidebar)
- **Alexa** — via Nabu Casa HA Cloud skill (see decisions/alexa_integration_2026-04-06.md)

---

## Key Commands

```bash
# SSH into HA
ssh root@192.168.4.150

# Check HA core status
ha core info --raw-json

# Validate config before restart
ha core check

# Restart HA core
ha core restart

# Check HA VM status from Proxmox
qm status 100
qm shutdown 100 --timeout 120   # Graceful stop
```

---

## Important Notes

⚠️ **Do NOT install Tailscale inside HA** — modifies HA's IPv6 routing table, breaks Thread/Matter device pairing. Tailscale is installed on the Proxmox host instead.

⚠️ **SSH from Claude Code** — direct SSH (`ssh root@192.168.4.150`) only works from Terminal via osascript, not from the Claude Code bash environment (no route to host). Use osascript or the HA REST API via Nabu Casa URL instead.
