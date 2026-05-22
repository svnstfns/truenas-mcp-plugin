---
name: truenas-deploy-and-verify
description: Deploy a Docker Compose stack to TrueNAS Scale and verify it came up healthy. Use after the deployment architecture is settled — wraps validate → deploy → poll job → verify, and on failure pulls logs and reports. For the assessment and architecture decisions that must happen first, use truenas-deployment-planning.
---

# Deploy and verify a TrueNAS app

This skill executes a safe deploy pipeline that catches failures early.
It never just "fires and forgets" — the deploy is always followed by a
verification step.

This skill is the **hands**, not the head. It assumes the deployment
architecture is already settled: which pool, which network model, what
gets exposed. If that has not happened yet, run
**`truenas-deployment-planning`** first — deploying without it means
inheriting whatever default the tooling guessed.

## When to use this

Use this skill once the architecture is settled and the user wants to
ship a concrete compose:

- "Deploy nginx on the NAS" (after planning)
- "Install paperless on TrueNAS" (after planning)
- "Launch this compose file as an app"

If the user only asks to *validate* a compose file (without deploying),
call `validate_compose` directly instead.

## Inputs

- **app_name** — DNS-compatible lowercase name (`[a-z0-9-]`, no
  leading/trailing dash). It is the TrueNAS app name and the basis for
  the host directory under the chosen pool. If it is missing, ask once —
  do not guess it from the compose `services:` block.
- **compose_yaml** — the Docker Compose file as a string. Its `networks:`
  and `ports:` must already reflect the architecture settled during
  planning.

## Procedure

Follow these steps in order. Do **not** skip ahead, do **not** parallelize.

### Step 1 — Validate the compose

Call `validate_compose(compose_yaml)`.

- If `valid == false`: stop. Show the user the `errors` array. Ask
  whether to fix automatically or have them edit.
- If `warnings` is non-empty: show them but continue.
- If `security_issues` is non-empty: stop. These are blocked
  (privileged, dangerous capabilities, system-path mounts). Explain
  why and ask whether to remove them.

### Step 2 — Deploy

Call `deploy_app(app_name, compose_yaml)`.

The tool internally: converts the compose → checks port conflicts →
calls `app.create` → polls the job until `SUCCESS` or `FAILED`.

- If `success == true`: continue to Step 3.
- If `success == false`: skip to Step 4 (failure handling) — do not
  retry blindly.

### Step 3 — Verify the deployment

Call `verify_deployment(app_name)`.

- If `app_running == true` and `health_status == "healthy"`: success.
  Report to the user with app name, status, ports, and — if the compose
  carries a reverse-proxy label — the public URL it routes to.
- If `app_running == true` but `health_status != "healthy"`: ambiguous.
  The container started but health checks are not passing. Continue to
  Step 4 to pull logs.
- If `app_running == false`: continue to Step 4.

### Step 4 — Failure diagnosis

The deployment is in a bad state. Gather context before asking the
user what to do:

1. Call `get_app_details(app_name)` — note the `status` and `config`.
2. Call `get_app_logs(app_name, lines=200)` — read the tail.
3. Identify the failure category from the logs:
   - **Image pull failure** → "image not found" / "manifest unknown" / "unauthorized"
   - **Port conflict** → "address already in use" / "port is already allocated"
   - **Volume mount failure** → "no such file or directory" / "permission denied" on a `/mnt/` path
   - **Config error** → "invalid configuration" / app-specific startup errors
   - **Resource limit** → OOMKilled / insufficient memory
4. Present the user with:
   - The category you identified
   - The relevant log excerpt (10-20 lines max)
   - A specific recommended next action

For each category, the recommended action is:

- **Image pull**: verify the image name and tag; check whether it is a
  private registry that needs credentials.
- **Port conflict**: call `list_apps` to find the colliding app; pick a
  different host port.
- **Volume mount**: check that the host path under the chosen pool
  exists and is owned correctly. The converter normally creates it — if
  it did not, something raced.
- **Config error**: show the relevant compose fragment; ask the user to
  adjust.
- **Resource limit**: suggest adding memory limits or moving heavy apps
  to a pool with more headroom.

### Step 5 — Cleanup on hard failure

If the user wants to abandon the deployment, suggest `purge_app(app_name)`
(handled by the `truenas-purge-failed` skill), which removes the app and
its volumes from TrueNAS. Host directories under `/mnt/` are kept — the
user decides manually whether to delete them.

## Output format

When the deployment succeeds:

```
✓ Deployed <app-name>
  Status:   RUNNING
  Pool:     <pool from deploy result>
  Ports:    <list>
  URL:      <public URL>   (only if a reverse-proxy label is present)
  Job ID:   <job_id>
```

When it fails:

```
✗ Deployment of <app-name> failed
  Category: <category>
  Symptoms: <one-line summary>
  Logs:
    <relevant excerpt>
  Recommended: <specific next action>
```
