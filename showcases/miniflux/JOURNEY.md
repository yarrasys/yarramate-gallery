# Miniflux — YarraMate discovery journey

## Provenance

| | |
| --- | --- |
| Source | <https://github.com/miniflux/v2> |
| Commit | `c4d54f87a81b30aa173fddf05d7ff83ae7da5796` |
| Branch | `main` (shallow clone) |
| Analyzed | 2026-07-29 |
| Toolchain | yarramate 0.4.0 (`yarramate`, `yarramate-likec4`) |
| Re-verified | yarramate 0.6.0 on 2026-07-30 (check, reconcile, likec4 check; model unchanged) |
| Profile | `yarramate/core@0.1` |

## Journey and question

**Journey:** Discover an existing project (yarramate-architecture skill).

**Question answered:** What is the current-state architecture of the Miniflux
feed-reader server — its externally meaningful services, actors, principal
information, and dependencies — as observable from the repository at the
commit above?

## Observations vs interpretive proposals

### Direct observations (locator-backed)

- Single Go binary: `main.go` delegates to `internal/cli`; the daemon
  (`internal/cli/daemon.go`) starts a worker pool, an optional background
  scheduler, HTTP server(s), and an optional metrics collector in one process.
- HTTP surfaces wired in `internal/http/server/routes.go`: web UI (catch-all,
  `internal/ui`), REST API at `/v1/` (`internal/api`), Fever API at `/fever/`
  (`internal/fever`), Google Reader API at `/reader/api/0/`
  (`internal/googlereader`), `/metrics`, and health probes.
- Background refresh: `internal/cli/scheduler.go` builds batches of due feeds
  (`internal/storage/batch.go`) and pushes jobs to the pool
  (`internal/worker`); each worker runs `RefreshFeed`
  (`internal/reader/handler/handler.go`), which fetches
  (`internal/reader/fetcher`), parses (`internal/reader/parser`), processes/
  sanitizes (`internal/reader/processor`), persists via
  `store.RefreshFeedEntries`, and asynchronously pushes new entries to
  enabled integrations (`go integration.PushEntries`).
- Refresh-all and category refresh, from the UI (`internal/ui/feed_refresh.go`,
  `category_refresh.go`) and the REST API (`internal/api/feed_handlers.go`,
  `category_handlers.go`), enqueue jobs onto the same pool. Single-feed
  refresh is the exception: it calls `reader/handler.RefreshFeed`
  synchronously and never reaches the pool.
- Save-entry: `internal/ui/entry_save.go` and
  `internal/api/entry_handlers.go` invoke `integration.SendEntry`.
- Storage: all SQL lives in `internal/storage`; PostgreSQL only
  (`internal/database/postgresql.go`, `lib/pq` in `go.mod`, migrations in
  `internal/database/migrations.go`).
- ~30 third-party integration client packages under `internal/integration/`.
- Optional Google/OIDC authentication (`internal/oauth2`,
  `internal/ui/oauth2_*.go`); media proxy (`internal/mediaproxy`).
- Deployment: Docker (alpine/distroless), debian, rpm, systemd under
  `packaging/`; Heroku `Procfile`; 86 `*_test.go` files.

### Interpretive proposals (semantic meaning added for review)

- The actor **Reader** and the external systems **Third-party client app**,
  **Feed publisher**, **Third-party integration service**, and
  **OIDC / OAuth2 provider** are interpretations; the code only implies them
  (compat APIs, outbound fetches, provider clients, OIDC callbacks).
- Grouping `internal/reader` (fetcher, parser, processor, sanitizer, …) into
  one **Feed fetching pipeline** component, and `internal/integration` into
  one **Integration hub**, is an abstraction choice.
- Six hand-over relationships originally modelled as `triggering` were
  authored as `flow` with explicit content because the core profile restricts
  `triggering` to behavior elements; the flow content (jobs, entries) is
  observable in the code.
- No ownership and no constraints were declared: the repository contains no
  ownership or governance sources to support them.
- Current state only; no target intent was inferred.

## Model

Canonical inputs (this directory):

- `.yarramate/workspace.yaml` — workspace `miniflux`
- `.yarramate/architecture/miniflux.yaml` — 20 concepts, 36 relationships,
  0 states (1 document)
- `.yarramate/evidence/repository.yaml` — 37 observations
- `.yarramate/projections/*.yaml` — 5 projections
- `.yarramate/integrations/likec4/subject-mapping.yaml` — 56 mappings
  (populated by `map --sync`)
- `.yarramate/integrations/likec4/project.yaml` — 5 views

## Evidence results and reconciliation

`yarramate reconcile` summary (provider `repository-inspection`, evidence
document `miniflux-repository@1.0`):

| result | count |
| --- | --- |
| confirmed | 36 |
| contradicted | 0 |
| unknown | 1 |
| not-observed | 0 |

The single `unknown` is deliberate: claim `miniflux#reader-uses-clients`
(readers use third-party client apps) — the compatibility APIs and the
bundled Go client imply it, but actual client usage is not observable from
the repository. Evidence was never promoted into declared intent.

## Projections and views

| Projection | View type | Content |
| --- | --- | --- |
| `system-overview` | static | All 20 concepts and all 36 relationships |
| `http-surfaces` | static | Web UI, three API dialects, media proxy, shared storage, clients |
| `persistence` | static | Storage layer, PostgreSQL, feed/entry/user data objects |
| `feed-refresh-flow` | dynamic (7 steps) | scheduler → pool → pipeline ← publisher → storage → integrations |
| `save-entry-flow` | dynamic (3 steps) | reader → web UI → integration hub → third-party service |

Generated LikeC4 output: `.yarramate-out/likec4/` (`model.likec4`,
`specification.likec4`, `likec4.config.json`, `yarramate.generated.json`).
The generated output is current with the authored inputs.

## Rendering coverage audit

- **Concepts in no projection:** none — `system-overview` covers all 20
  concepts and all 36 relationships.
- **Ordered chains without a dynamic view:**
  - manual refresh (`web-ui`/`rest-api` → `worker-pool` → pipeline) —
    intentional: it converges with the background refresh flow immediately
    after the pool; the enqueue edges are rendered statically.
  - OIDC login (`oidc-provider` → `web-ui`) — intentional: a single serving
    relationship, no multi-step chain worth a dynamic view.
- **Projections absent from the LikeC4 project:** none — all 5 projections
  are listed as views.

### Intentional model omissions

Observed in the repository but deliberately left out of the smallest useful
model: cleanup scheduler and maintenance tasks, Prometheus metrics collector
and `/metrics`, health probes, OPML import/export, full-text search
internals, CLI administration commands (create-admin, reset-password,
export/refresh feeds), WebAuthn/passkey authentication, web session
management detail, favicon fetching, proxy rotation, per-provider integration
components (~30 collapsed into one concept), and deployment packaging
(Docker/debian/rpm/systemd) as technology nodes.

## Validation

All commands used the pinned yarramate 0.4.0 toolchain, and were re-run
under 0.6.0 on 2026-07-30 with the same results. Under 0.4.0 the adapter
checks had to run from the directory containing `.yarramate/`; since 0.5.0
LikeC4 project references resolve relative to the project document, so the
checks pass from any working directory (re-verified from an unrelated one).

| Command | Outcome |
| --- | --- |
| `yarramate check .yarramate/workspace.yaml --json` | ok, exit 0 (1 doc / 20 concepts / 36 relationships) |
| `yarramate compile .yarramate/workspace.yaml` | exit 0 |
| `yarramate evidence .yarramate/evidence/repository.yaml .yarramate/workspace.yaml` | exit 0, 37 observations |
| `yarramate reconcile .yarramate/workspace.yaml` | exit 0, 36 confirmed / 0 contradicted / 1 unknown / 0 not-observed |
| `yarramate context <each of 5 projections> .yarramate/workspace.yaml` | exit 0 (×5) |
| `yarramate view <each of 5 projections> .yarramate/workspace.yaml` | exit 0 (×5) |
| `yarramate-likec4 check .yarramate/integrations/likec4/project.yaml --json .yarramate/workspace.yaml` (pre-sync) | ok:false — expected: empty mapping, 56 missing entries |
| `yarramate-likec4 map --sync .yarramate/integrations/likec4/subject-mapping.yaml .yarramate/workspace.yaml` | exit 0, added 56 mappings (authored diff reviewed) |
| `yarramate-likec4 check` (post-sync) | ok:true, exit 0 |
| `yarramate-likec4 export-project .yarramate/integrations/likec4/project.yaml .yarramate-out/likec4 .yarramate/workspace.yaml` | exit 0 |
| Portability — `yarramate check <showcase>/.yarramate/workspace.yaml --json` from a foreign cwd | ok:true, exit 0 |
| Portability — `yarramate-likec4 check` from within the showcase directory | ok:true, exit 0 |

⚠️ A green check is deterministic correctness, not completeness or
architecture approval. The gaps above are reported, not hidden.

## Unresolved architectural decisions

None pending for the current-state model. Open modelling questions a
maintainer might take up later:

- Whether the Fever and Google Reader APIs should be modelled as one
  compatibility surface or kept separate (kept separate here).
- Whether per-provider integrations deserve individual components (collapsed
  here).
- Whether deployment topology (Docker, systemd, PostgreSQL placement) should
  be modelled as technology/node elements.

## Status in Git

This showcase is a proposed, uncommitted artifact: the model was authored in
a scratch clone and copied here. No commits, pushes, or issues were created.
