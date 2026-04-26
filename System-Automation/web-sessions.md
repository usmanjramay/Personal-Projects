# Claude Code Web Sessions

**Status:** Testing
**Last updated:** 2026-04-26

> Reference for how Claude Code web sessions are configured and what still needs to be done to run them reliably across projects.

---

## How It Works

Claude Code on the web reads configuration from the GitHub repository of the project being worked on. It does not use local machine settings.

### What the Web Session Reads

| Source | File | Purpose |
|--------|------|---------|
| GitHub repo | `CLAUDE.md` | Project instructions and rules |
| GitHub repo | `.claude/settings.json` | Permissions and session behavior |
| GitHub repo | `.mcp.json` | MCP server connections |
| claude.ai Environment settings | Environment variables | Secrets and credentials for MCP servers |

### MCP Configuration

MCP servers (e.g., Second Brain, n8n) must be declared in `.mcp.json` in the GitHub repo. The file specifies which servers to connect and how, but sensitive values (API keys, URLs) should be stored as environment variables in claude.ai — not hardcoded in the file.

### Environments

Web sessions run inside named environments configured in claude.ai. Currently one environment is set up: **default**. This environment is shared across all projects running web sessions — any environment variables added here are available to all of them.

For further reference: [Claude Code on the Web documentation](https://code.claude.com/docs/en/claude-code-on-the-web)

---

## Permissions

Permissions are configured in `.claude/settings.json` in each repo. The current default everywhere is **auto mode**, which uses machine-learning classifiers to decide what to approve automatically vs. what to pause and ask.

**Unattended sessions require an extra step.** Because the same `settings.json` is used both locally (where auto mode is fine) and in web sessions (where there is no one to answer prompts), unattended web sessions must be launched with the `--dangerously-skip-permissions` flag. This tells Claude to skip all permission prompts and proceed autonomously. This flag must be set manually at session start — it is not in `settings.json` because it should not apply to local sessions.

---

## Auto-Merge for Scheduled Agents

When a web session runs on a schedule (or any time Claude is not being watched), it works by creating a branch and opening a PR — it does not push directly to main. GitHub then requires a human to review and merge. To allow fully unattended operation, auto-merge needs to be enabled.

### How auto-merge works

1. A PR is created by the agent.
2. GitHub Actions runs any configured checks (CI/CD).
3. When all checks pass (green), GitHub auto-merges the PR to main.

### What is needed to enable it

- Branch protection rules must be enabled on `main` (required by GitHub before auto-merge can be turned on).
- At least one status check must be required — even a trivial one — because auto-merge only triggers after checks pass.
- For repos with no code (like these), no real GitHub Actions workflows are needed. A minimal workflow that always returns green is sufficient to satisfy the requirement.

**Current decision:** Not setting this up yet — there is no current need for scheduled agents that commit to these repos. When the need arises, the steps above are what to follow.

---

## Open Items

- [ ] Add `.mcp.json` to the `EoL-Context` repo so web sessions there have the same MCP access as Personal-Projects.
- [ ] When unattended scheduled sessions are needed: enable branch protection on `main` + add a minimal always-green GitHub Actions workflow + enable auto-merge on the repo.

---

## Change Log

| Date | Change | Why |
|------|--------|-----|
| 2026-04-26 | File created | Document web session configuration after initial setup |
