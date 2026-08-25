# GitLab — YarraMate discovery journey

## Provenance

| | |
| --- | --- |
| Source | <https://github.com/gitlabhq/gitlabhq.git> (GitLab FOSS mirror) |
| Tag | `v19.3.0` |
| Tag ref | `beefb52f223b5f99c2a828124851371287b4c175` |
| Commit (HEAD of shallow clone) | `2c30df7828b86f8ac1fca26e96385fe712b0669f` |
| Analyzed | 2026-08-24 |
| Toolchain | yarramate 1.0.0 |
| Profile | `yarramate/core@0.1` |
| Catalogue | `core-enrichment@1.0` |

The clone is GitLab **FOSS**: no `ee/` directory, so every claim in this
model is checkable against MIT-licensed source. All `repo:` evidence
locators are valid at the commit above.

## Journey and question

**Journey:** Discover an existing project (yarramate-architecture skill).

**Question answered:** How is GitLab organised as a single-application
DevSecOps platform at v19.3.0 — the Rails monolith and the behaviour it
performs, every satellite service the version pins name, the stores and
information they guard, who owns what, what constrains it, and where the
architecture is publicly heading (Cells)?

A second question rides on the first: GitLab publishes its architecture
documentation, so this model treats **the published design as declared
intent and the tree as evidence** — including one place where they
disagree (Praefect, below).

## How the model was produced

By yarramate's own loop, deliberately: concepts and relationships landed
as atomic `apply` batches (~14 batches, ~230 operations), and the
`design` interview drove the enrichment order — motivation, ownership,
statuses, hosting, service realization, contracts. Four batches were
refused whole by the compile gate during authoring (an illegal
goal-to-outcome realization, string owners that were not declared
subjects, an unquoted colon in a generated name, a reference entry
missing its id). Every refusal left the model untouched and made the
next batch better. That behaviour is part of what this showcase
demonstrates.

## Validation at handoff

- `check`: 1 document, 3 projections, 1 evidence document — 84 concepts,
  162 relationships, 2 states, no errors.
- `reconcile`: 49 observations — **48 confirmed, 1 not-observed, 0
  contradicted, 0 subjects without evidence**.
- Interview: **3 open, all deliberate** (see Open by design).

## Interpretations, stated

- **Owners are GitLab's stage groups** (Create, Verify, Package, Deploy,
  the Gitaly team, Core Platform, Global Search), read from the public
  product hierarchy and grounded in-repo by
  `config/feature_categories.yml`. Group-level attribution is an
  interpretation at showcase grain.
- **Motivation is sourced, not invented**: driver, goal, and outcome
  restate GitLab's public single-application positioning, and each
  description says so.
- **The states story** declares `single-instance` (baseline) and
  `cells-target` (target) from in-tree signals (`cells-mailroom/`,
  caproni tooling) plus GitLab's public Cells direction. The cell router
  and topology are not modelled. `presentIn` records one deliberate
  claim: the Elasticsearch indexer does not survive into the target;
  Zoekt (which `supersedes` it) does.
- **Praefect is declared but not observed.** GitLab's architecture page
  documents it; the FOSS tree at this commit carries no pin, directory,
  or client for it. It stays in the model with a `not-observed`
  observation rather than being silently omitted — the one reconcile
  finding is intentional.

## Stated omissions

NGINX and the load-balancing tier; Geo, Consul, PgBouncer/Patroni and
the EE-only estate (not in this tree); the monitoring constellation
(Prometheus, exporters, Alertmanager); OpenBao and the Duo workflow
executor (version-pinned, unmodelled); object storage backends; SMTP and
external identity providers; the 32 internal gems; per-provider
integration models beyond the collapsed framework (54 at this version);
authorization policies below the authentication grain; the Vue frontend
as a distinct component; ActionCable; deployment topology beyond the
single collapsed instance node; the cell router. The `qa/` and test
architecture. Each was seen and left out; none is an oversight.

## Open by design

- `motivation-unattested` (2): no accountable GitLab insider sat in this
  interview, so nobody's name goes on their motivation.
- `kind-untested` (1, Praefect): nothing in this tree can pin what
  Praefect structurally is — fitting, for a component the tree does not
  carry.

## Rendering

Three views ship: the component topology (`system-overview`), the push
path, and a web request. LikeC4 export was deliberately skipped for this
showcase; the intended rendering is the mounted visual editor over this
workspace. Layout sidecars are not committed.

## Product feedback this journey produced

The enrichment process only deepens what exists — whole absent layers
(strategy, events, succession) stayed silent until hunted by hand.
Filed with a worked example from this very model as
<https://github.com/yarrasys/yarramate/issues/272>.
