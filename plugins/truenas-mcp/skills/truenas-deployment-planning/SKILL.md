---
name: truenas-deployment-planning
description: Establish what a TrueNAS NAS is and what the deployment architecture should be, BEFORE deploying anything. Use at the very start of any "deploy / install / ship X to the NAS" request — assesses the NAS state and clarifies the networking and architecture decisions with the user. Hands off to truenas-deploy-and-verify once the picture is complete.
---

# Plan a TrueNAS deployment

This skill makes you competent to deploy onto someone's TrueNAS *before*
you touch a compose file. A deployment that skips this step inherits
whatever default the tooling guessed — and guessed defaults are how a
backend database ends up reachable from the whole LAN.

The MCP server is pure mechanism: it runs TrueNAS API calls. The
judgement — which pool, which network, what gets exposed — is yours.
This skill gives you that judgement.

## When to use this

Run this skill at the **start** of any request like:

- "Deploy <project> on the NAS"
- "Install <app> on TrueNAS"
- "Ship this stack to the NAS"

Do not jump straight to `truenas-deploy-and-verify`. That skill is the
hands; this skill is the head. Only skip this skill if the NAS state and
the architecture are *already established in this conversation* — and even
then, say so explicitly ("NAS already assessed above, architecture per the
loaded docs — proceeding to deploy").

## Core principle — know exactly, never guess

Every fact you act on falls into one of three buckets. Treat them
differently:

| Bucket | Source | How to use it |
|--------|--------|---------------|
| **Exact** | A TrueNAS API call returned it | Use as fact. |
| **Assumed** | You inferred it (e.g. a naming pattern from existing apps) | Present it to the user *as an assumption*. They confirm or correct. Never adopt silently. |
| **Unknown** | Not in any API result, not in context | Ask the user. Never act on a hunch. |

The failure mode this prevents: inferring "the user names apps like
`<service>-prod`" from two samples, then silently creating `paperless-prod`
when they wanted `paperless`. An inference is a question, not an answer.

## Phase A — Assess the NAS

Establish the *exact* state with API calls. Run these and read the results:

1. **`test_connection`** — confirm it's TrueNAS and capture the version.
   - If the version is below 25.04, warn the user: the `app.create`
     schema and the auth flow this tooling targets are 25.04. Older
     firmware may behave differently.
2. **`get_system_info`** — the storage pools that actually exist, plus
   CPU/RAM headroom. You now know *exactly* which pool names are valid.
   Never invent a pool name — only the ones in this list exist.
3. **`list_apps`** — the apps already deployed. Use these to **attempt**
   to infer the user's conventions:
   - Naming: are existing apps `lowercase-with-dashes`? Single words?
     Prefixed?
   - Storage: where do their bind mounts point — which pool, what
     directory depth (`/mnt/<pool>/<app>/` vs `/mnt/<pool>/apps/<app>/`)?
   - Anything you derive here is an **assumption**. Present it. Let the
     user decide. If there are no apps yet, you have nothing to infer —
     ask.
4. **`get_docker_networks`** — the networks that already exist. This tells
   you whether there is already a reverse-proxy network, a macvlan, or
   custom bridges.

Then present a **NAS profile** to the user. Separate the two kinds of
knowledge cleanly:

```
NAS profile — <hostname>
─────────────────────────────────────
Facts (from the API):
  TrueNAS version:  <version>
  Storage pools:    <pool> (<free>/<total>), ...
  Existing apps:    <count> — <names>
  Docker networks:  <name> (<driver>), ...

Assumptions (please confirm):
  Naming scheme:    <inferred pattern, or "no apps yet — undecided">
  Storage layout:   <inferred path convention, or "undecided">
```

Do not proceed until the user has confirmed or corrected the assumptions.

## Phase B — Clarify the architecture

The deployment architecture is usually **already decided** — you just have
to find where. It may live:

- in a `CLAUDE.md` loaded into this session,
- in files the user shared or pasted,
- earlier in this conversation,
- or, if the project follows a documentation methodology, in an arc42
  deployment view (commonly `docs/architecture/ARCH-deployment.md`).

There is **no fixed path**. Use what you already have in context. Only
ask the user about the points that genuinely are not answered anywhere.

The points that must be settled before deploying:

1. **Reverse proxy.** Is there one (Traefik, nginx-proxy-manager, Caddy,
   …)? Should this app be routed through it, or reached directly by
   `IP:port`? If routed: what hostname?
2. **NAS admin UI.** Should the TrueNAS *native* web UI be placed behind
   the reverse proxy? **Sharp edge — warn the user:** TrueNAS does not
   expect its admin UI to be reverse-proxied; the web UI offers no
   first-class setting for it, and getting it wrong can lock you out of
   administration. Most setups keep the admin UI on its own direct
   address. Only proceed if the user explicitly asks for it.
3. **Network interfaces.** Does the NAS have more than one physical NIC?
   Is there a macvlan or a custom bridge? What is the parent interface
   for any macvlan? You captured the *existing* networks in Phase A —
   here you establish what *this deployment* should use.
4. **Per-app IP.** Should each Docker app get its own routable LAN IP
   (macvlan) or share the host's IP with published ports (bridge +
   port mapping)? This is a real architectural fork:
   - *Own IP (macvlan):* clean addressing, but every app sits directly
     on the LAN — a backend service with no business being LAN-reachable
     becomes reachable anyway.
   - *Shared host IP (bridge):* only the ports you publish are reachable;
     unpublished services stay internal. The safer default for stacks
     that have backend services.
5. **Storage placement.** Which pool for this app's data (from the exact
   list in Phase A)? What directory convention for persistent volumes
   versus config (confirmed in Phase A)?

Apply the core principle throughout: exactly evidenced in context → fact;
inferred → confirm as an assumption; not findable → ask.

When the architecture was not written down anywhere, offer to record the
settled answers where the project keeps its architecture — for a project
on the documentation methodology, that is its `ARCH-deployment.md`. This
is optional and the user's call.

## Phase C — Hand off to deployment

Once the NAS profile is confirmed and the five architecture points are
settled, you have everything you need:

- which pool, which directory layout
- which network model (macvlan vs bridge), which network
- which services are exposed and how (reverse proxy / direct ports /
  internal only)

Build the `docker-compose.yml` so it *expresses* those decisions — every
service's `networks:` and `ports:` reflect the architecture you just
established. A backend with no external need gets no published port and
no external network.

Then hand off to **`truenas-deploy-and-verify`** for the mechanical
validate → deploy → verify → diagnose cycle.

## What this skill never does

- Deploy anything itself — that is `truenas-deploy-and-verify`.
- Invent pool names, network names, or hostnames. Every name is either
  from an API result or from the user.
- Adopt an inferred convention without the user confirming it.
- Place a service on an externally-reachable network unless the
  architecture explicitly calls for it.
