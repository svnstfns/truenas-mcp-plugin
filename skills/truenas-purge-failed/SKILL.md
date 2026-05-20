---
name: truenas-purge-failed
description: Completely remove a failed, abandoned, or experimental TrueNAS app — including its volumes. Use when the user wants to clean up after a botched deployment, drop an experimental stack, or reset before reinstalling. This is a destructive operation; always confirm with the user before executing.
---

# Purge a TrueNAS app

This skill is the recovery path for deployments that went wrong and
need a clean slate. It removes the app *and* its Docker volumes from
TrueNAS. It does **not** delete host directories under
`/mnt/<pool>/<app-name>/` — Sven decides manually whether to wipe
those.

## When to use this

Trigger on phrases like:

- "Purge nginx-test"
- "Clean up the failed paperless deploy"
- "Remove that experimental stack completely"
- "Reset jellyfin so I can reinstall it"

If the user just says "delete" or "remove" without "fail" / "purge" /
"completely", call `delete_app(app_name, remove_volumes=False)`
instead — that's the gentler path that keeps volumes.

## Inputs

- **app_name** (required).

## Procedure

### Step 1 — Confirm intent (mandatory)

Always show the user what will happen before doing anything:

```
About to purge <app_name>:
  - Stop the app (if running)
  - Delete the app definition
  - Delete all Docker volumes attached to the app
  - KEEP host directories under /mnt/<pool>/<app-name>/

Continue? (y/N)
```

If the user says no or is ambiguous, stop and recommend `delete_app`
with `remove_volumes=False` as the gentler alternative.

### Step 2 — Gather pre-state

Call `get_app_details(app_name)` to record the starting state. Note:

- Current `status` (RUNNING, STOPPED, ERROR, DEPLOYING)
- The list of `volumes` and their host paths
- The `image` name (in case the user wants to redeploy later)

If the app isn't found, stop and tell the user — there's nothing to
purge.

### Step 3 — Execute the purge

Call `purge_app(app_name)` — the MCP tool runs the whole sequence:

1. `stop_app(app_name)` (skipped if already stopped)
2. `app.delete(app_name, {remove_volumes: true})`
3. Polls the job until SUCCESS

The tool returns a structured `steps[]` array showing what worked and
what didn't. Each entry has `action` and `status`. Don't bail on the
first failure — `purge_app` is designed to continue past individual
errors so you don't leave half-cleaned state.

### Step 4 — Report what happened

Format the result for the user:

```
Purge of <app_name>: <ok | partial | failed>

Steps:
  ✓ stop_app          — was already STOPPED  / OK
  ✓ delete_app        — OK
  ✓ remove_volumes    — OK

Kept on host:
  /mnt/nvme-01/<app_name>/   (Sven decides manually)
```

If any step failed, surface it explicitly:

```
✗ delete_app — job 1234 FAILED: "namespace already exists"
```

### Step 5 — Offer follow-up actions

After a successful purge, ask whether they want to:

1. Delete the host directories too (for a true clean slate) — they'd
   do this manually via SMB/SSH; the MCP server intentionally doesn't
   touch host paths.
2. Redeploy from a known-good compose file (use `truenas-deploy-and-verify`).
3. Nothing — leave the state as-is.

## What this skill never does

- Delete host directories under `/mnt/`. The MCP server doesn't do
  this, full stop. The host filesystem is Sven's responsibility.
- Purge multiple apps in one go. Always one at a time, always with
  explicit confirmation for each.
- Skip the confirmation prompt, even if the user typed `--force` or
  similar — there's no `--force` flag in this skill.

## Sequencing relative to other skills

| Situation | Use this skill? | Or which other? |
|---|---|---|
| Deploy failed mid-way, app is in ERROR | yes | — |
| Just want to redeploy a working app | no | `update_app` or `restart_app` |
| Want to delete but keep data for migration | no | `delete_app(..., remove_volumes=false)` |
| Diagnosing why an app keeps crashing | no — diagnose first | `truenas-troubleshoot` |
