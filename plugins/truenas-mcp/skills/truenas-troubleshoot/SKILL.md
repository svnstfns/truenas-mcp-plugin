---
name: truenas-troubleshoot
description: Diagnose a TrueNAS app that is broken, stuck, unhealthy, or behaving unexpectedly. Use when the user reports "X is down", "Y isn't responding", "Z keeps restarting", or asks for logs / status of a specific app. Pulls app details, recent logs, and network state, then produces a structured diagnosis report.
---

# Troubleshoot a TrueNAS app

This skill is for **diagnosis only** — it never restarts, redeploys,
or modifies an app. It gathers signal so the user can decide what to
do next.

## When to use this

Trigger on phrases like:

- "Plex isn't working"
- "Why is paperless restarting?"
- "Get me the logs for nginx-demo"
- "What's wrong with home-assistant"
- "Is jellyfin healthy?"

If the user *already* knows the fix and just wants you to apply it
(e.g., "restart paperless"), don't use this skill — call the relevant
tool (`restart_app`) directly.

## Inputs

- **app_name** (required). If missing, call `list_apps()` first and
  ask the user which app they mean.

## Procedure

### Step 1 — Establish current state

Call these three in parallel (they're independent reads):

1. `get_app_details(app_name)` → status, config, container_ids, ports, volumes
2. `get_app_logs(app_name, lines=500)` → recent log tail
3. `get_docker_networks()` → cluster-wide network state

If `get_app_details` returns "app not found": stop. The app isn't
deployed at all. Suggest `list_apps()` to see what exists.

### Step 2 — Classify the state

Map `status` to one of these buckets:

- **RUNNING + healthy** → not actually broken from the NAS's view.
  Show the user logs + ports and ask what symptom they're seeing.
- **RUNNING but unhealthy** → container is up but health checks fail.
  Focus on logs for app-internal errors.
- **STOPPED** → container exited. Find the exit reason in the logs
  (last 50 lines usually have it).
- **ERROR / DEPLOYING / CRASHING** → terminal/transient failure.
  Find the most recent stack trace or error line.

### Step 3 — Scan logs for known patterns

Look for these patterns (in order — first match wins):

| Pattern in logs | Likely cause |
|---|---|
| `permission denied` on `/mnt/`, `/data`, `/config` | Volume ownership wrong (should be `apps:apps`) |
| `database is locked`, `corrupt database` | Underlying SQLite/DB issue |
| `OOMKilled`, `out of memory` | Memory limit too low |
| `bind: address already in use` | Port collision with another app |
| `unable to pull image`, `manifest unknown` | Bad image name/tag |
| `connection refused` to `db:`, `redis:`, `postgres:` | Dependent service down — check sibling app |
| `certificate verify failed`, `SSL` | TLS/cert problem on outbound calls |
| `no route to host`, `network unreachable` | Network misconfig — check `get_docker_networks` |
| Repeating crash loop (same error every ~10s) | Misconfig in compose env vars |

### Step 4 — Cross-check network state

If the issue might be network-related, confirm the app is on `svcs_network`:

```python
networks = get_docker_networks()
svcs = next((n for n in networks if n["name"] == "svcs_network"), None)
if not svcs or container_ids_app_not_in(svcs):
    # Network issue — app isn't on the expected network
```

The pvnkn3t convention is that every app runs on `svcs_network`
(macvlan, `10.121.125.0/24`). If it's on `bridge` instead, the
compose converter didn't apply the network override.

### Step 5 — Report

Produce a structured report. Don't dump raw logs — extract the
relevant 5–15 lines.

```
Troubleshooting report — <app_name>
─────────────────────────────────────
State:        <RUNNING | STOPPED | ERROR | CRASHING>
Health:       <healthy | unhealthy | unknown>
Container:    <id (12 chars)>
Ports:        <list>
Network:      <bridge | svcs_network | other>
Volumes:      <count, with one example>

Most likely cause:
  <one-sentence diagnosis based on Step 3 pattern match>

Evidence:
  <5-15 line log excerpt with the smoking gun highlighted>

Suggested actions (do NOT execute without user approval):
  1. <specific next action>
  2. <alternative if 1 doesn't help>
  3. <escalation if both fail>
```

## What this skill never does

- Restart, redeploy, or update the app
- Delete volumes or host directories
- Modify compose configs
- Run more than 3 tool calls in parallel (rate respect)

If the user explicitly says "fix it" after the report, then hand off
to the right tool (`restart_app`, `update_app`, etc.) or the
`truenas-purge-failed` skill — but make sure they've seen the report
first.
