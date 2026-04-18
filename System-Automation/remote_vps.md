# VPS Setup — Remote Access & MCP Server

> Last updated: 2026-03-22
> Purpose: Document the Hostinger VPS hosting the personal brain MCP server.

---

## VPS Details

| Property | Detail |
|---|---|
| IP | 46.202.159.171 |
| Provider | Hostinger |
| DNS | `mcp.experimentoflife.org` → `46.202.159.171` |
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

## DNS Configuration

The domain `mcp.experimentoflife.org` points to the VPS IP `46.202.159.171`. HTTPS is provisioned via Let's Encrypt and managed by Traefik.

---

## Docker Stack

The MCP server runs in a Docker container managed by `docker-compose.yml`:

```yaml
# Standard docker-compose setup with MCP service
# Container name: root-mcp-1
# Source: /root/personal-brain-mcp/index.ts
```

The service is accessible remotely via `https://mcp.experimentoflife.org`.

---

## Integration with Local Network

Claude can access this VPS remotely via SSH using the AppleScript bridge on the Mac. The MCP server provides a semantic search interface for the personal brain system (stored in Supabase with pgvector).