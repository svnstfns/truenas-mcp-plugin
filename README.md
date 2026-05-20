# truenas-mcp-plugin

Claude Code plugin that bundles the
[`truenas-mcp`](https://github.com/svnstfns/truenas-mcp) server plus
opinionated skills and slash commands for managing TrueNAS Scale apps
in the **pvnkn3t** infrastructure.

## What this plugin gives you

| Component | Purpose |
|---|---|
| **MCP server** (`truenas`) | 15 tools for system info, app lifecycle, deployment, monitoring |
| Skill **`truenas-deploy-and-verify`** | Safe deploy pipeline: validate → deploy → verify → diagnose on failure |
| Skill **`truenas-troubleshoot`** | Diagnose a broken app without modifying it; produces a structured report |
| Skill **`truenas-purge-failed`** | Clean removal of failed deployments with confirmation gates |
| Slash **`/truenas-status`** | Compact dashboard: system info + all apps |
| Slash **`/truenas-list`** | List apps with optional state filter |

The MCP server itself lives in a separate repo
([svnstfns/truenas-mcp](https://github.com/svnstfns/truenas-mcp)) so
it stays usable from any MCP client (Claude Desktop, Cline, Cursor,
…). This plugin is the convenient distribution path for Claude Code
users.

## Installation

### Prerequisites

You need an API key for your TrueNAS. The recommended way is to run
the standalone server's interactive installer once — it creates a
dedicated `mcp-service` user with minimal privileges and a
user-scoped API key:

```bash
uvx --from git+https://github.com/svnstfns/truenas-mcp.git truenas-mcp install
```

This sets `TRUENAS_HOST` and `TRUENAS_API_KEY` in your shell
environment (or in `~/.claude.json`).

### Install the plugin

```bash
claude plugin install https://github.com/svnstfns/truenas-mcp-plugin
```

Restart Claude Code. The `truenas` MCP server will auto-start, and
the three skills + two commands will be discoverable.

### Verify

In a Claude Code session:

```
/truenas-status
```

You should see the system dashboard. If the server can't connect,
check that `TRUENAS_HOST` and `TRUENAS_API_KEY` are set in your
shell.

## Configuration

The bundled MCP server reads four environment variables from your
shell (or from `~/.claude.json` if you ran `truenas-mcp install`):

| Variable | Required | Default | Description |
|---|---|---|---|
| `TRUENAS_HOST` | yes | `nas.pvnkn3t.de` | TrueNAS hostname or IP |
| `TRUENAS_API_KEY` | yes | — | User-scoped API key |
| `TRUENAS_SSL_VERIFY` | no | `true` | Set `false` for self-signed certs |
| `MOCK_TRUENAS` | no | `false` | `true` enables the mock client (for development) |

## How the pieces fit together

```
┌────────────────────────────────────────────────────────────────┐
│  Claude Code session                                           │
│                                                                │
│   ┌───────────────────┐    ┌──────────────────────────────┐   │
│   │  Slash commands   │    │           Skills              │   │
│   │  /truenas-status  │    │  truenas-deploy-and-verify    │   │
│   │  /truenas-list    │    │  truenas-troubleshoot         │   │
│   └─────────┬─────────┘    │  truenas-purge-failed         │   │
│             │              └────────────┬──────────────────┘   │
│             └─────────────┬──────────────┘                     │
│                           │                                    │
│                           ▼                                    │
│                ┌──────────────────────┐                        │
│                │  truenas MCP server  │  (uvx, stdio)          │
│                │  15 tools            │                        │
│                └──────────┬───────────┘                        │
└───────────────────────────┼────────────────────────────────────┘
                            │ WebSocket JSON-RPC
                            ▼
              ┌──────────────────────────────┐
              │   TrueNAS Scale 25.04        │
              │   wss://nas.pvnkn3t.de/...   │
              └──────────────────────────────┘
```

## Tools exposed by the bundled MCP server

| Category | Tool | Purpose |
|---|---|---|
| System | `test_connection` | Verify the connection |
| System | `get_system_info` | Version, uptime, CPU/memory, pools |
| Apps | `list_apps` | List apps with optional state filter |
| Apps | `get_app_details` | Full details: config, container IDs, ports, volumes |
| Apps | `start_app` | Start a stopped app |
| Apps | `stop_app` | Stop a running app |
| Apps | `restart_app` | Redeploy / restart an app |
| Deployment | `validate_compose` | Lint a Compose YAML before deploying |
| Deployment | `deploy_app` | Convert + deploy a Compose stack |
| Deployment | `update_app` | Update an existing app's compose |
| Deployment | `delete_app` | Remove an app (volumes optional) |
| Deployment | `purge_app` | Stop + delete + remove volumes |
| Monitoring | `get_app_logs` | Tail container logs |
| Monitoring | `get_docker_networks` | List Docker networks |
| Monitoring | `verify_deployment` | Check that an app is healthy and reachable |

## Examples

Ask Claude:

> "Deploy this nginx compose on the NAS as `nginx-test` and make sure it comes up."

→ Triggers `truenas-deploy-and-verify`.

> "Plex isn't responding, what's going on?"

→ Triggers `truenas-troubleshoot`.

> "Show me what's running on the NAS."

→ Triggers `/truenas-status`.

> "List only the stopped apps."

→ Triggers `/truenas-list stopped`.

## Uninstall

```bash
claude plugin uninstall truenas-mcp
```

To also remove the service user from the NAS:

```bash
uvx --from git+https://github.com/svnstfns/truenas-mcp.git truenas-mcp uninstall
```

## License

MIT — see [LICENSE](./LICENSE).
