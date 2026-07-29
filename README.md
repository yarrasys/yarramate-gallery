# YarraMate gallery

Architecture models of well-known open-source repositories, discovered with
[YarraMate](https://github.com/yarrasys/yarramate) and the canonical
`yarramate-architecture` skill — one discovery journey per repository, run by an
agent against a pinned CLI release.

Each showcase is a self-contained, validating YarraMate workspace:

```text
showcases/<repo>/
  .yarramate/       # canonical native model: documents, projections, evidence, likec4 integration
  .yarramate-out/   # derived LikeC4 export
  JOURNEY.md        # discovery report: provenance, observations, evidence, coverage gaps
```

Every model passes the deterministic checks from its showcase directory:

```sh
npx -p yarramate yarramate check showcases/<repo>/.yarramate/workspace.yaml --json
```

## Showcases

| Showcase | Shape / language | Model | Views | Reconciliation |
|---|---|---|---|---|
| [uptime-kuma](showcases/uptime-kuma/JOURNEY.md) | Self-hosted app / Node+Vue | 16 concepts, 20 relationships, 3 projections | 3 (2 dynamic) | 28/29 confirmed, 1 unknown |
| [miniflux](showcases/miniflux/JOURNEY.md) | Server daemon / Go | 20 concepts, 36 relationships, 5 projections | 5 (2 dynamic) | 36/37 confirmed, 1 unknown |
| [fastify](showcases/fastify/JOURNEY.md) | Framework library / Node | 34 concepts, 46 relationships, 3 projections | 3 (2 dynamic) | 59/60 confirmed, 1 not-observed |
| [httpie](showcases/httpie/JOURNEY.md) | CLI tool / Python | 15 concepts, 22 relationships, 3 projections | 3 (2 dynamic) | 25/26 confirmed, 1 not-observed |

All four models were discovered on 2026-07-29 with `yarramate@0.4.0`; source
commit SHAs are in each `JOURNEY.md`.

### uptime-kuma

Uptime Kuma is a self-hosted monitoring tool built as a single Node.js process
serving a Vue 3 SPA over Express and Socket.IO. A per-monitor check loop with 26
pluggable probe types (HTTP, TCP, DNS, ping, MQTT, SNMP, databases, and more)
records heartbeats through knex/redbean into SQLite (or MariaDB) and fans alerts
out through ~99 notification provider integrations. Public status pages, SVG
badges, a passive push endpoint, and a Prometheus `/metrics` endpoint expose
monitor state outward, with the whole system deployed as one Docker container.

### miniflux

Miniflux is a single Go binary that runs an HTTP server, a background feed
scheduler, and a worker pool in one daemon, with PostgreSQL as its only external
dependency for persistence and full-text search. One shared storage layer serves
four HTTP surfaces — the server-rendered web UI plus REST, Fever, and Google
Reader API dialects for third-party clients — while the scheduler-driven
pipeline fetches, sanitizes, and stores feed entries and fans new or saved
entries out to ~30 third-party integration services.

### fastify

Fastify is a single-package Node.js web framework whose factory assembles small
subsystems under `lib/`: a find-my-way-backed router, a 16-hook lifecycle
engine, schema-driven validation/serialization (Ajv + fast-json-stringify),
Request/Reply abstractions, pino logging, and node:http server creation. Its
defining mechanism is avvio-driven plugin registration with encapsulation: each
registered plugin gets a child instance with copied hooks, schemas, parsers and
decorators, isolating plugin effects to a subtree. Every request flows through a
fixed, hook-interleaved pipeline — routing → parsing → validation → handler →
serialization → response — with encapsulated error and 404 handling.

### httpie

HTTPie is a single-process Python CLI whose `http`/`https` entrypoint
orchestrates a pipeline of internal responsibilities: an extended-argparse layer
turns argv and stdin into a resolved namespace, a client core builds and sends
the request through requests/urllib3, and an output pipeline formats and
colourises (via Pygments) the messages for the terminal — or, under
`--download`, streams the body to disk with resume support. Persistent state is
limited to a platform config directory holding `config.json` and per-host
session files, and a plugin manager loads auth/formatter/transport extensions
via package entry points.

## Why this exists

- **Demonstration** — what an agent-driven discovery journey produces on a real
  codebase: the smallest useful current-state model, evidence-grounded, with
  focused projections and derived diagrams.
- **Corpus** — these workspaces seed YarraMate's context benchmark (does a
  deterministic verify loop flatten the capability gap between generator
  models?). See the
  [benchmark design](https://github.com/yarrasys/yarramate/tree/main/docs/research/context-benchmark).

## Reproduction

Each `JOURNEY.md` records the source commit SHA, analysis date, and CLI version.
The recipe: shallow-clone the source repo at that SHA, load the
`yarramate-architecture` skill from the YarraMate repo, and follow the
"Discover an existing project" journey with the pinned CLI. Models are proposals
reviewed via Git — a green `check` is correctness, not completeness, and the
coverage gaps in each journey report are part of the result.

Models here describe third-party projects but are not affiliated with or
endorsed by them; they are interpretive snapshots at the recorded commit.
