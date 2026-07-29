# Fastify — YarraMate discovery journey

## Provenance

| | |
|---|---|
| Source | https://github.com/fastify/fastify |
| Commit | `812608e5c9cb0b643b3eb186c2e9e84bbb7ed8ee` |
| Branch | `main` (shallow clone; fastify v5.10.0 per package.json) |
| Analyzed | 2026-07-29 |
| Toolchain | yarramate 0.4.0 (`yarramate`, `yarramate-likec4`) |

## Journey and question

**Journey used:** Discover an existing project (yarramate-architecture skill).

**Question answered:** What is the current architecture of the fastify
framework as a *library* — its core responsibilities, plugin/encapsulation
mechanism, principal information, architecturally significant external
dependencies, and developer-facing API surface? This seed deliberately tests
the library/plugin-architecture shape: there are no end-user actors or
deployed services; the model covers what is there.

## Observations vs interpretive proposals

**Direct observations** (each cited in the evidence overlay):

- Single-package library: `package.json` (fastify 5.10.0), entry `fastify.js`,
  types `fastify.d.ts`, internals under `lib/`.
- Subsystems: routing (`lib/route.js` over find-my-way), hooks
  (`lib/hooks.js`: 10 lifecycle + 6 application hooks), plugin
  boot/encapsulation (avvio + `lib/plugin-override.js`), content-type parsing
  (`lib/content-type-parser.js`), schema validation/serialization
  (`lib/schema-controller.js`, `lib/validation.js`, defaults
  `@fastify/ajv-compiler` and `@fastify/fast-json-stringify-compiler`),
  Request/Reply (`lib/request.js`, `lib/reply.js`), logging
  (`lib/logger-factory.js`, `lib/logger-pino.js` over pino), server creation
  (`lib/server.js` over node:http/https/http2), errors and encapsulated 404
  (`lib/error-handler.js`, `lib/errors.js`, `lib/four-oh-four.js`).
- Request lifecycle order: routing → onRequest → preParsing → parsing →
  preValidation → validation → preHandler → handler → preSerialization →
  serialization → onSend → write → onResponse (code paths in
  `lib/route.js:560-657`, `lib/handle-request.js`, `lib/reply.js`; the same
  order is documented in `docs/Reference/Lifecycle.md`).
- Encapsulation: `lib/plugin-override.js` creates a child instance per plugin
  with copied hooks, schema bucket, parsers, Request/Reply prototypes, prefix
  and log level; `onRegister` fires per instance (line 71).

**Interpretive proposals** (flagged in concept descriptions):

- `fastify#framework` — grouping of `fastify.js` + `lib/` as one component.
- `fastify#app-developer` — the only actor; a library's consumer role, not a
  runtime user.
- `fastify#route-handler` — handler execution phase; handler bodies are user
  code outside the repo.
- The behavior decomposition (`fn-*` applicationFunctions) names lifecycle
  phases that the code executes but does not reify as named modules.
- No target intent was inferred; the model is current-state only, with no
  architecture states declared.

## Evidence results and reconciliation

Evidence overlay: `.yarramate/evidence/repository-inspection.yaml`
(60 observations, provider `repository-inspection`, `repo:` URIs).

Reconciliation summary (`yarramate reconcile`):

| result | count |
|---|---|
| confirmed (supported) | 59 |
| contradicted | 0 |
| unknown | 0 |
| not-observed | 1 |

The single not-observed finding is claim
`fastify#developer-supplies-handlers` — handler implementations live outside
the repository; only the contract/examples exist in `docs/Reference/Routes.md`.
Unevidenced (unobserved at reconcile scope): the 10 `framework-has-*`
composition claims and the 10 `*-performs-*` assignment claims carry no
separate observations — they are interpretation whose evidence is identical to
the already-cited concept locators. Nothing was promoted from evidence into
declared intent.

## Model inventory

- Documents: 1 (`.yarramate/architecture/main.yaml`, id `fastify`)
- Concepts: 34 (1 businessActor, 1 applicationService, 18 applicationComponent
  incl. 7 external dependencies, 11 applicationFunction, 3 dataObject)
- Relationships: 46 · Architecture states: 0 (current-state only)

## Projections and views

| Projection | View type | Purpose |
|---|---|---|
| `.yarramate/projections/framework-overview.yaml` | static | Repository orientation: actor, API surface, subsystems, external deps, principal information |
| `.yarramate/projections/request-lifecycle.yaml` | dynamic (7 steps) | Routing → hooks → parsing → validation → handler → serialization → response |
| `.yarramate/projections/plugin-registration.yaml` | dynamic (3 steps) | register() → avvio boot → encapsulated child context → onRegister |

LikeC4 project: `.yarramate/integrations/likec4/project.yaml` (mapping
`.yarramate/integrations/likec4/subject-mapping.yaml`, 80 entries, synced).
Generated output: `.yarramate-out/likec4/` (`model.likec4`,
`specification.likec4`, `likec4.config.json`, `yarramate.generated.json`) —
current as of this commit. All three intended views are in the project.

## Rendering coverage

- **Concepts in no projection:** none (verified against projection contexts).
- **Ordered chains without a dynamic view:**
  `fastify#routing-triggers-not-found` (404 error branch) is projected in
  `request-lifecycle` but is not an ordered step — intentional: it is a branch,
  not part of the happy-path sequence.
- **Projections absent from the LikeC4 project:** none.
- **Intentional omissions:** the 10 `*-performs-*` assignment relationships and
  `developer-supplies-handlers` appear in no projection — they span the
  structural view and the behavior views; rendering them would mix both layers
  in one diagram. They remain in the compiled model and context output.
  Startup/shutdown application hooks (onReady/onListen/preClose/onClose) are
  modeled inside `fn-app-hooks` but have no dedicated flow. Decorators
  (`lib/decorate.js`) are folded into the encapsulation-context narrative
  rather than modeled as a subsystem.

## Validation commands and outcomes

Run from the model root (CWD at the directory containing `.yarramate/`):

| Command | Outcome |
|---|---|
| `yarramate check .yarramate/workspace.yaml --json` | ok=true, 0 diagnostics (34 concepts / 46 relationships) |
| `yarramate compile .yarramate/workspace.yaml` | exit 0 |
| `yarramate evidence .yarramate/evidence/repository-inspection.yaml .yarramate/workspace.yaml` | exit 0, 60 observations |
| `yarramate reconcile .yarramate/workspace.yaml` | exit 0, 59 confirmed / 1 not-observed / 0 contradicted / 0 unknown |
| `yarramate context <each projection> .yarramate/workspace.yaml` | exit 0 (3/3) |
| `yarramate view <each projection> .yarramate/workspace.yaml` | exit 0 (3/3) |
| `yarramate-likec4 check .yarramate/integrations/likec4/project.yaml --json .yarramate/workspace.yaml` | ok=true (read-only, after authoring-time `map --sync`) |
| `yarramate-likec4 map --sync .yarramate/integrations/likec4/subject-mapping.yaml .yarramate/workspace.yaml` | added 80 mappings (authoring repair, committed as content) |
| `yarramate-likec4 export-project .yarramate/integrations/likec4/project.yaml .yarramate-out/likec4 .yarramate/workspace.yaml` | exit 0, 3 views generated |

Portability (gallery copy): `yarramate check` and `yarramate-likec4 check`
pass in **both** the analysis clone and this showcase directory (ok=true in
each). Note: Core resolves workspace paths relative to the manifest (CWD-
independent); the LikeC4 project's `mapping`/`projection` paths resolve
against the CWD, so run the likec4 check with CWD set to this directory.

## Unresolved architectural decisions

- None blocking: this is a current-state discovery model with no target
  intent, no alternatives, and no architecture states. A green check is
  correctness, not completeness.
- Open modeling questions for a future pass: whether decorators deserve
  first-class treatment; whether the boot/close lifecycle (avvio ready/close,
  onReady/onClose) merits its own flow projection; whether diagnostics_channel
  instrumentation (`fastify.js:85`, `lib/handle-request.js:23`) is
  architecturally significant enough to model.

## Status

Proposed only — this showcase is generated content; nothing has been committed
to Git by the journey. The source clone was scratch and is not retained.
