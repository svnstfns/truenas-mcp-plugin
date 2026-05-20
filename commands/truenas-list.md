---
description: List TrueNAS apps with an optional state filter (RUNNING, STOPPED, ERROR, DEPLOYING).
argument-hint: "[state]"
---

List TrueNAS apps using `list_apps`.

If `$ARGUMENTS` is empty or `all`, list every app.

Otherwise, treat `$ARGUMENTS` as a state filter. Map common spellings:

- `running`, `up`, `live` → `RUNNING`
- `stopped`, `down`, `off` → `STOPPED`
- `error`, `failed`, `broken` → `ERROR`
- `deploying`, `pending`, `starting` → `DEPLOYING`

If the spelling doesn't match any of those, tell the user the valid
values and stop — don't guess.

Call `list_apps(status_filter=<mapped state>)` and format the result
as a table:

```
NAME              STATE     IMAGE                            PORTS
nginx-demo        RUNNING   nginx:latest                     8080:80/TCP
home-assistant    RUNNING   homeassistant/home-assistant     8123:8123/TCP
plex-server       STOPPED   plexinc/pms-docker:latest        32400:32400/TCP
```

Sort by name. Truncate image strings to 32 characters with an
ellipsis if longer. If multiple ports are exposed, comma-separate them.

If no apps match the filter, say so plainly:

> No apps in state `RUNNING`.

Don't follow up with suggestions unless the user asks — the command
is supposed to be a quiet read.
