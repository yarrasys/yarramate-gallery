# GitLab: a YarraMate discovery journey

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
model is checkable against publicly licensed source. `LICENSE` puts
everything outside `doc/` under MIT (Expat); `doc/` itself is CC BY-SA
4.0, and several claims here (Praefect, the Zoekt scope, the stage-group
owners) rest on `doc/` or GitLab's public handbook rather than on code. All `repo:` evidence
locators are valid at the commit above.

## Journey and question

**Journey:** Discover an existing project (yarramate-architecture skill).

**Question answered:** How is GitLab organised as a single-application
DevSecOps platform at v19.3.0: the Rails monolith and the behaviour it
performs, every satellite service the version pins name, the stores and
information they guard, who owns what, what constrains it, and where the
architecture is publicly heading (Cells)?

A second question rides on the first: GitLab publishes its architecture
documentation, so this model treats **the published design as declared
intent and the tree as evidence**, including one place where they
disagree (Praefect, below).

## How the model was produced

By yarramate's own loop, deliberately: concepts and relationships landed
as atomic `apply` batches (~14 batches, ~230 operations), and the
`design` interview drove the enrichment order: motivation, ownership,
statuses, hosting, service realization, contracts. Four batches were
refused whole by the compile gate during authoring (an illegal
goal-to-outcome realization, string owners that were not declared
subjects, an unquoted colon in a generated name, a reference entry
missing its id). Every refusal left the model untouched and made the
next batch better. That behaviour is part of what this showcase
demonstrates.

## Validation at handoff

- `check`: 1 document, 3 projections, 1 evidence document: 85 concepts,
  168 relationships, 2 states, no errors.
- `reconcile`: 53 observations. **53 confirmed, 0 not-observed, 0
  contradicted, 0 subjects without evidence**.
- Interview: **2 open, both deliberate** (see Open by design).

These are the numbers after the corrections in "Audit and corrections"
below. At handoff the model reported 84 / 162 / 49 observations with one
not-observed, and that not-observed was wrong.

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
  and topology are not modelled.

  **The two states currently differ in nothing.** `ask --compare` returns
  "0 added, 0 removed". The model originally claimed the Elasticsearch
  indexer does not survive into the target and that Zoekt `supersedes` it,
  and that claim was wrong: `doc/integration/zoekt/_index.md:40` at this
  commit says Zoekt "handles only code search and does not replace
  Elasticsearch or OpenSearch", which every non-code search scope still
  needs. The succession is real but **scoped**, and yarramate cannot
  express a scoped succession
  ([yarramate#281](https://github.com/yarrasys/yarramate/issues/281)), so
  the model records no succession rather than an unqualified one the
  source contradicts. Two declared states that differ in nothing is the
  honest result of that gap, and it is left visible rather than papered
  over with a difference the tree does not support.
- **Praefect is declared and observed.** GitLab's architecture page
  documents it, and the FOSS tree carries the client side:
  `lib/gitlab/gitaly_client/praefect_info_service.rb` defines
  `PraefectInfoService`, whose `replicas` method calls the
  `repository_replicas` RPC; `lib/gitlab/git/repository.rb:1114` holds the
  memoised client and `:1235` consumes it; `lib/gitlab/setup_helper.rb:152`
  carries `module Praefect`; and `doc/administration/gitaly/praefect/`
  documents operating it. There is no `PRAEFECT_*` version pin at the root
  because the server ships separately. **This model originally claimed the
  opposite**. See "Audit and corrections".

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

**The product surface, which the list above does not cover and should
have.** This model is an infrastructure model. It captures the runtime
estate carefully and captures almost none of what GitLab is *for*.
Measured at this commit: `app/services/` holds **1,495 `.rb` files across
130 domains**, of which the three modelled here (`ci`, `merge_requests`,
`search`) account for **252**. The other **1,243** are unmodelled, the
largest being `packages` (107), `projects` (95), `work_items` (71), `mcp`
(53), `users` (52), `import` (48), `groups` (28) and `clusters` (28).
Nothing in this model represents issues, boards, work items, milestones,
epics, environments, releases, wikis, snippets, or security and incident
management.

The sharpest instance: the `package-distribution` capability is evidenced
by a message naming `package_registry`, while `app/services/packages` (107
files, npm, maven, nuget, pypi and more) is neither modelled nor, until
now, listed here. **The model cited the thing it then dropped.**

This paragraph exists because the phase-2 audit found the omission list
fifteen items long, discussing NGINX and Consul at length, and silent on
Plan, Secure, Deploy, Monitor and Govern. On its infrastructure the model
knew what it did not know; on the product it did not.

## Open by design

- `motivation-unattested` (2): no accountable GitLab insider sat in this
  interview, so nobody's name goes on their motivation.

`kind-untested` on Praefect was previously listed here as deliberate. It
was not deliberate; it rested on the mistaken claim that the tree carries
nothing of Praefect. The tree answers it directly, and the model now
records the behaviour Praefect performs, so the question is closed rather
than open.

## Audit and corrections

Phase 2 of this showcase was an independent audit, run in a session that
did not build the model and carried no context from the one that did,
against the same pinned commit. `AUDIT-PROMPT.md` beside this file is the
brief it was given. It found two critical errors, and both were in the
claims this showcase was positioned on.

| Finding | What was wrong | Now |
| --- | --- | --- |
| Praefect | Recorded `not-observed` with a three-clause message; two clauses false. The tree carries `PraefectInfoService`, a `praefect_info_client`, `module Praefect` and an operator doc directory. | `confirmed`, with the client evidence. Reconcile is 53/53. |
| Zoekt succession | `supersedes` and `presentIn` claimed an unqualified replacement; GitLab's own docs say Zoekt does not replace Elasticsearch. | Claim withdrawn; the scope gap is recorded and filed as yarramate#281. |
| Doc/tree divergences | Four found and unrecorded. | Three recorded as keyed observations (Zoekt absent from the component list; KAS and Elasticsearch marked "EE Only" while pinned in FOSS). OpenBao, which has no subject, is noted here: it is pinned at the root and appears **nowhere** in the architecture page. |
| Counts | Four figures came from `ls -1 \| wc -l`, counting directories as files (97 for 68, 81 for 74, 57 for 51). "559 feature flags" matched no counting method; the figure is 544. | All re-derived, and each now says what it counted. |
| Declared constraint | "All git I/O through Gitaly" was violated by three access edges, with `check --strict` green. | The route is now visible as serving edges through `git-storage-service`, and the constraint says so. Nothing checks it mechanically: yarramate#280. |
| Missing edges | Workhorse and the registry each served Rails, and only the reverse was recorded. | Both added, from the evidence the model already cited. |

What the audit found *sound*, stated so the entry is not read as uniformly
negative: 49/49 evidence locators resolved with zero rot, no reversed
relationship in 28 checked, an exactly reproducible open set, and a
`triggering` chain the source code actively enforces
(`lib/gitlab/event_store/store.rb:65` raises unless a subscriber is an
`ApplicationWorker`).

The honest summary of the audit's verdict: **the model was structurally
excellent and evidentially unreliable**, and its two showpiece claims
about knowing what it did not know were the two that turned out to be
false. Both were falsifiable with `grep` in under ten minutes. That is the
finding, and it is recorded here rather than quietly fixed, because a
showcase whose thesis is "the published design is intent, the tree is
evidence, and here is where they disagree" has no business hiding the
place where it got the disagreement backwards.

## Rendering

Three views ship: the component topology (`system-overview`), the push
path, and a web request. LikeC4 export was deliberately skipped for this
showcase; the intended rendering is the mounted visual editor over this
workspace. Layout sidecars are not committed.

## Product feedback this journey produced

The enrichment process only deepens what exists. Whole absent layers
(strategy, events, succession) stayed silent until hunted by hand.
Filed with a worked example from this very model as
<https://github.com/yarrasys/yarramate/issues/272>.
