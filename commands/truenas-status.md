---
description: Compact dashboard of TrueNAS system info and all apps with their state.
---

Build a short dashboard for the user covering:

1. **System** — call `get_system_info` and surface:
   - TrueNAS version
   - Uptime (in days/hours, not seconds)
   - CPU and memory usage (round to whole percent)
   - Pool list with status (online/degraded) and used/total

2. **Apps** — call `list_apps` (no filter) and group by state:
   - **RUNNING** apps: just the names, comma-separated, one line
   - **STOPPED** apps: same
   - **ERROR / DEPLOYING** apps: full line with name + state

Format the output as a compact text dashboard, not a wall of JSON.
Aim for something like:

```
TrueNAS — nas.pvnkn3t.de — TrueNAS-SCALE-24.10.2.3
Up 10d 3h · CPU 12% · Mem 18.4G / 64G

Pools:
  ✓ nvme-01      ONLINE   421G / 2.0T
  ✓ cluster-01   ONLINE    4.2T / 12T
  ✓ cluster-02   ONLINE    8.1T / 24T

Apps:
  RUNNING (3):  nginx-demo, home-assistant, paperless
  STOPPED (1):  plex-server
  ERROR   (0):
```

If any app is in ERROR or DEPLOYING state for longer than a few
minutes, end the dashboard with a one-line suggestion to investigate
with the `truenas-troubleshoot` skill.

Keep the entire response under 25 lines.
