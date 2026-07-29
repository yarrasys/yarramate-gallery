# Uptime Kuma — YarraMate discovery journey

## Provenance

| | |
|---|---|
| Source | https://github.com/louislam/uptime-kuma |
| Commit | `ca5323c2dd1ed85c8b36b5227d2da6f6c28eeebc` |
| Branch | `master` (shallow clone) |
| Analyzed | 2026-07-29 |
| Toolchain | yarramate 0.4.0 (pinned; `yarramate` + `yarramate-likec4` stable CLIs) |
| App version | uptime-kuma 2.4.0 (package.json) |

## Journey and question

Journey: **Discover an existing project** (yarramate-architecture skill).
Question answered: *What is the current-state architecture of the uptime-kuma
repository — its significant actors, externally meaningful services, principal
components, information, and deployment boundary?*

The model is current-state only. No target intent was inferred and no
architecture states were declared.

## Observations vs interpretive proposals

**Direct observations** (each carries a `repo:` locator in
`.yarramate/evidence/repository.yaml`):

- Node.js server entrypoint `server/server.js`; `UptimeKumaServer` singleton
  (`server/uptime-kuma-server.js`) wiring Express + Socket.IO, auth (JWT +
  optional 2FA), socket handlers, and HTTP routers.
- Vue 3 SPA under `src/` built with Vite (`config/vite.config.js`), served by
  the server; management traffic runs over Socket.IO.
- Monitor check loop in `server/model/monitor.js` with 26 pluggable check
  types under `server/monitor-types/`.
- Notification dispatch in `server/notification.js` with 99 provider modules
  under `server/notification-providers/`.
- Persistence through knex/redbean (`server/database.js`): SQLite by default,
  MariaDB / embedded MariaDB as alternatives; schema in `db/knex_init_db.js`
  and `db/knex_migrations/`.
- Public status pages, RSS, and SVG badges
  (`server/routers/status-page-router.js`, `server/routers/api-router.js`);
  passive push endpoint `/api/push/:pushToken`; Prometheus `/metrics` behind
  API auth (`server/server.js:355`).
- Official Docker deployment: `louislam/uptime-kuma` image, port 3001,
  `/app/data` volume, single Node.js process (`compose.yaml`,
  `docker/dockerfile`).

**Interpretive proposals** (semantic meaning proposed for review, not directly
evidenced as named subjects):

- The two business actors `operator` and `status-visitor` — the roles are
  implied by authenticated vs anonymous surfaces, but no actor is named in the
  repository. They intentionally carry no evidence observations.
- The service groupings `uptime-monitoring` and `status-publication` — an
  architectural reading of the feature set (README + routers), evidenced but
  still an abstraction choice.
- Component boundaries (`server-core` vs `monitor-engine` vs
  `notification-hub`) — the code is one process and one package; the split
  follows stable module responsibilities, not a physical boundary.
- `triggering` semantics for "server starts check loops" and "engine
  dispatches notifications" were modeled as `flow` because the core profile
  restricts `triggering` to behavior-aspect concepts (see product feedback).

## Model contents

- 1 document (`uptime-kuma`), 16 concepts, 20 relationships, 0 states.
- Actors: operator, status page visitor.
- Services: uptime monitoring, status publication, notification channels
  (external).
- Components: dashboard SPA, server core, monitor engine, notification hub,
  monitored target (external), push agent (external), metrics scraper
  (external, assumed).
- Information: monitor configuration, heartbeat history.
- Technology: application database (systemSoftware), Docker runtime (node).
- Ownership and constraints: **none declared** — the repository provides no
  supported basis (a single-maintainer OSS project; Git authorship was not
  promoted to ownership per the skill's rules).

## Evidence results

`yarramate evidence .yarramate/evidence/repository.yaml` — 29 observations:
**28 confirmed, 0 contradicted, 1 unknown, 0 not-observed**.

The single `unknown`: `uptime-kuma#metrics-scraper` — the `/metrics` endpoint
exists, but no Prometheus consumer is part of the repository; the scraper is
an assumed external party.

## Reconciliation summary

`yarramate reconcile .yarramate/workspace.yaml`:

| result | count |
|---|---|
| confirmed | 28 |
| contradicted | 0 |
| unknown | 1 |
| not-observed | 0 |

Subjects without evidence observations (deliberate — interpretive, actor-level
or service-plumbing claims): `operator`, `status-visitor`, and the
relationships `ui-serves-operator`, `core-serves-webapp`,
`monitoring-serves-operator`, `engine-realizes-monitoring`,
`status-serves-visitor`. Evidence was not promoted into declared intent at any
point.

## Projections and views

| Projection | View | Kind |
|---|---|---|
| `.yarramate/projections/system-overview.yaml` | `system-overview` | static (repository orientation) |
| `.yarramate/projections/monitor-check-flow.yaml` | `monitor-check-flow` | dynamic, 6 ordered steps |
| `.yarramate/projections/push-heartbeat-flow.yaml` | `push-heartbeat-flow` | dynamic, 3 ordered steps |

LikeC4 project: `.yarramate/integrations/likec4/project.yaml` (mapping:
`subject-mapping.yaml`, 36 entries, populated by `map --sync`). Generated
output: `.yarramate-out/likec4/` (`model.likec4`, `specification.likec4`,
`likec4.config.json`, `yarramate.generated.json`) — current as of this report.

## Rendering coverage audit

- **Concepts in no projection:** none — `system-overview` selects the whole
  `uptime-kuma` document, so all 16 concepts render at least once.
- **Ordered chains without a dynamic view:** the active-check chain and the
  push chain have dynamic views. Intentionally omitted dynamic views:
  status-page delivery (visitor → status publication → heartbeat reads) and
  operator CRUD (dashboard → socket handlers → configuration) — both are
  plain request/response interactions already visible structurally in the
  overview.
- **Projections absent from the LikeC4 project:** none — all three
  projections are listed as views.

## Validation commands and outcomes

Run in the clone (CWD = repo root), all green:

| Command | Outcome |
|---|---|
| `yarramate check .yarramate/workspace.yaml --json` | ok, 0 diagnostics (16 concepts / 20 relationships) |
| `yarramate compile .yarramate/workspace.yaml` | exit 0 |
| `yarramate evidence .yarramate/evidence/repository.yaml .yarramate/workspace.yaml` | 28 confirmed / 1 unknown |
| `yarramate reconcile .yarramate/workspace.yaml` | exit 0, 1 finding (unknown) |
| `yarramate context <each projection> .yarramate/workspace.yaml` | exit 0 (x3) |
| `yarramate view <each projection> .yarramate/workspace.yaml` | exit 0 (x3) |
| `yarramate-likec4 check .yarramate/integrations/likec4/project.yaml --json .yarramate/workspace.yaml` | first run: 16 expected missing-mapping errors (empty mapping); after `map --sync`: ok |
| `yarramate-likec4 map --sync .yarramate/integrations/likec4/subject-mapping.yaml .yarramate/workspace.yaml` | added 36 mappings |
| `yarramate-likec4 export-project ... .yarramate-out/likec4 ...` | exit 0, 3 views written |

Portability verification of this gallery copy:

| Location / CWD | Command | Outcome |
|---|---|---|
| any CWD | `yarramate check <showcase>/.yarramate/workspace.yaml --json` | ok (workspace paths are manifest-relative) |
| CWD = showcase root | `yarramate-likec4 check <showcase>/.yarramate/integrations/likec4/project.yaml --json <workspace>` | ok |
| CWD = clone root (re-verified) | both checks | ok |

⚠️ The LikeC4 project document's `mapping:` and `views[].projection:` paths
resolve against the **current working directory** (verified in
`dist/adapters/likec4-cli.js`: `resolve(cwd, path)`), so `yarramate-likec4`
commands must be run from the repository/showcase root. Core workspace paths
are manifest-relative and CWD-independent.

## Unresolved architectural decisions

- Whether external parties (monitored targets, notification channels, push
  agents, scrapers) should be first-class concepts or metadata is a modeling
  convention this showcase resolves in favor of first-class concepts; a
  gallery-wide convention has not been decided.
- The maintenance-window and status-page incident subsystems
  (`server/model/maintenance.js`, `server/model/incident.js`) are summarized
  inside existing concepts rather than modeled separately — a deliberate
  smallest-useful cut, revisit if the gallery wants deeper drill-down.
- Clustering/HA is out of scope: the repository evidences a single-process,
  single-container deployment only.

## Acceptance status

These files are **proposed** gallery content; nothing has been committed to
Git and no changes were made to the upstream repository. The clone used for
analysis lives in session scratch space and is disposable.
