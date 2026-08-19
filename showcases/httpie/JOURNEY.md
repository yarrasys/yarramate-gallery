# HTTPie — YarraMate discovery journey

## Provenance

| | |
|---|---|
| Source | https://github.com/httpie/cli |
| Commit | `5b604c37c6c67e18e7c3e9aee6c88a8c22b98345` |
| Branch | `master` (shallow clone) |
| Analyzed | 2026-07-29 |
| Toolchain | yarramate 0.4.0 (`yarramate`, `yarramate-likec4`) |
| Re-verified | yarramate 0.6.0 on 2026-07-30 (check, reconcile, likec4 check; model unchanged) |
| Re-verified | yarramate 0.22.0 on 2026-08-19 (check, reconcile, likec4 check; byte-identical LikeC4 re-export; model unchanged) |

## Journey and question

**Journey:** Discover an existing project (yarramate-architecture skill).
**Question answered:** How is the HTTPie CLI repository organised — which
internal responsibilities make up the `http`/`https` command, which external
dependencies and principal information does it rely on, and how does a request
flow from invocation to rendered output?

This seed exercises the CLI-tool shape: the significant actor is the terminal
user; the "services" are internal responsibilities of a single executable
rather than deployed network services.

## Observations vs interpretation

**Directly observed** (each has a locator in the evidence overlay):

- Console entrypoints in `setup.cfg` `[options.entry_points]`: `http`/`https`
  → `httpie.__main__:main`; `httpie` → `httpie.manager.__main__:main`.
- Responsibility modules: `httpie/cli/` (argument parsing), `httpie/client.py`
  (request building/sending), `httpie/sessions.py` (session management),
  `httpie/output/` (formatting/printing), `httpie/downloads.py` (download
  mode), `httpie/plugins/` (plugin manager), `httpie/manager/` (maintenance
  CLI).
- Principal information: `config.json` via `httpie/config.py`
  (`DEFAULT_CONFIG_DIR`), session files via `httpie/sessions.py`
  (`DEFAULT_SESSIONS_DIR`), downloaded files via `httpie/downloads.py`.
- External dependencies from `setup.cfg` `install_requires`:
  `requests[socks]` (with urllib3), `Pygments`, plus `requests-toolbelt`,
  `multidict`, `charset_normalizer`, `defusedxml`, `rich`.
- Orchestration order in `httpie/core.py` `program()`: parse → collect
  messages (build/send) → write via output pipeline, or hand the final
  response to `Downloader` under `--download`.

**Interpretive proposals** (semantic meaning inferred for review):

- Module boundaries promoted to named application components composed into
  the `http/https command`.
- `Remote HTTP server` — not in the repository; models the destination of the
  user-supplied URL.
- `Terminal user` as the significant business actor; direct invocation is
  documented usage, not statically observable behaviour (recorded
  `not-observed` in evidence, deliberately).

No target intent was inferred; the model is current-state only. No ownership
or constraints were declared — the repository provides no support for either
(Git authorship is explicitly not ownership evidence).

## Model

One native document `.yarramate/architecture/main.yaml` (document id
`httpie`): **15 concepts, 22 relationships, 0 states**.

## Evidence and reconciliation

Overlay: `.yarramate/evidence/repository-inspection.yaml` — 26 observations
(15 concept subjects, 11 relationship claims), all with `repo:` locators.

Reconciliation summary (`yarramate reconcile`):

| result | count |
|---|---|
| confirmed | 25 |
| contradicted | 0 |
| unknown | 0 |
| not-observed | 1 (`httpie#user-invokes-cli` — human invocation is documented usage, not statically observable) |

## Projections and views

| Projection | View type | Purpose |
|---|---|---|
| `.yarramate/projections/httpie-orientation.yaml` | static | Full repository orientation (all 15 concepts, 22 relationships) |
| `.yarramate/projections/request-flow.yaml` | dynamic (7 steps) | invoke → parse → build → send → exchange → format → render |
| `.yarramate/projections/download-flow.yaml` | dynamic (7 steps) | invoke → parse → build → send → exchange → downloader → file |

LikeC4 project: `.yarramate/integrations/likec4/project.yaml` (mapping:
`.yarramate/integrations/likec4/subject-mapping.yaml`, 37 entries, populated
by `map --sync`). All three
projections are included as views. Generated output:
`.yarramate-out/likec4/` (`model.likec4`, `specification.likec4`,
`likec4.config.json`, `yarramate.generated.json`). The generated output is
current with the authored inputs (deterministic re-export produced identical
files).

## Rendering coverage

- **Concepts in no projection:** none — the orientation projection selects
  the whole `httpie` document.
- **Ordered chains without a dynamic view:** the maintenance chain
  (`user-invokes-manager` → `manager-influences-plugins`) has no dynamic
  view. *Intentional omission* — a two-step administrative interaction; the
  static orientation view renders it adequately.
- **Projections absent from the LikeC4 project:** none.

Other intentional omissions:

- Session apply/persist ordering around a request is rendered statically
  (`session-serves-builder`, `session-accesses-store`), not as a third
  dynamic view.
- Secondary dependencies (`requests-toolbelt`, `multidict`,
  `charset_normalizer`, `defusedxml`, `rich`, `colorama`) are not modelled as
  concepts; only the two architecture-defining dependencies (requests/urllib3,
  Pygments) are.
- Internal helper modules (`httpie/internal/`, `httpie/legacy/`,
  `httpie/encoding.py`, `httpie/cookies.py`, `httpie/uploads.py` as a
  standalone concept) are absorbed into their owning responsibilities.

## Validation

All commands run with pinned yarramate 0.4.0; ✅ = exit 0, empty diagnostics.

From the analysis clone root:

| Command | Outcome |
|---|---|
| `yarramate check .yarramate/workspace.yaml --json` | ✅ ok, 15 concepts / 22 relationships |
| `yarramate compile .yarramate/workspace.yaml` | ✅ |
| `yarramate evidence .yarramate/evidence/repository-inspection.yaml .yarramate/workspace.yaml` | ✅ 26 observations |
| `yarramate reconcile .yarramate/workspace.yaml` | ✅ 25 confirmed / 1 not-observed |
| `yarramate context <projection> .yarramate/workspace.yaml` (x3) | ✅ |
| `yarramate view <projection> .yarramate/workspace.yaml` (x3) | ✅ |
| `yarramate-likec4 check .yarramate/integrations/likec4/project.yaml --json .yarramate/workspace.yaml` | ✅ |
| `yarramate-likec4 map --sync .yarramate/integrations/likec4/subject-mapping.yaml .yarramate/workspace.yaml` | ✅ 37 mappings added, then "already synchronized" |
| `yarramate-likec4 export-project .yarramate/integrations/likec4/project.yaml .yarramate-out/likec4 .yarramate/workspace.yaml` | ✅ |

Portability from this gallery copy:

| Command | Outcome |
|---|---|
| `yarramate check <showcase>/.yarramate/workspace.yaml --json` (any CWD) | ✅ workspace paths are manifest-relative |
| `cd <showcase> && yarramate-likec4 check .yarramate/integrations/likec4/project.yaml --json .yarramate/workspace.yaml` | ✅ |

ℹ️ Under the 0.4.0 toolchain used for discovery, LikeC4 project references
(`mapping`, `views[].projection`) resolved against the **current working
directory**, so the likec4 check had to run from the showcase root. Since
0.5.0 those references resolve relative to the project document, and the
check passes from any working directory (re-verified under 0.6.0 on
2026-07-30). The core `yarramate check` never had the restriction.

## Unresolved architectural decisions

- None for a current-state discovery model. Candidate refinements if this
  model is maintained: modelling the transport-plugin extension point as an
  explicit interface; deciding whether `httpie/manager/` deserves its own
  decomposition; representing the sessions-upgrade task's access to session
  files.

## Status

Model checks green. A green check is correctness, not completeness — the
coverage section above is the honest completeness statement. Nothing is
committed to Git; this directory is the published gallery copy.
