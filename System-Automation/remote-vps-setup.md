# VPS Setup — Remote Access & MCP Server

> Last updated: 2026-03-22
> Purpose: Document the Hostinger VPS hosting the personal brain MCP server.

---

## VPS Details

| Property | Detail |
|---|---|
| IP | 46.202.159.171 |
| Provider | Hostinger |
| HTTPS | Let's Encrypt via Traefik |
| MCP server | `/root/personal-brain-mcp/index.ts` |
| Docker compose | `/root/docker-compose.yml` |
| Container | `root-mcp-1` |

---

## SSH Access

| Property | Detail |
|---|---|
| Address | `46.202.159.171` |
| User | `root` |
| Auth | Key-based (Ed25519), set up 2026-03-22 |
| Key location | `~/.ssh/id_ed25519` on Mac |

SSH is configured for key-based authentication only. No passwords required.

---

## Services on VPS

### n8n Automation

n8n is installed and running on this VPS for automation workflows. For configuration details, credential management, node rules, and workflow setup, see the global n8n-automation-skill files.

### MCP Server (Legacy)

The personal brain MCP server previously ran here as a Docker container but has been decommissioned. MCP is now hosted as a Supabase Edge Function. Container details are preserved below for reference if needed.

---
