# Prompt: audit the GitLab showcase model for capture fidelity

You are a fresh session. You did not build this model, you carry no
context from the session that did, and your job is to measure — not to
fix. Do not edit the model; report.

## Subject and materials

- **Model under audit:** `showcases/gitlab/.yarramate` in this
  repository (yarramate-gallery), plus its `JOURNEY.md`, which states
  provenance, interpretations, and omissions.
- **Ground truth A (code):** GitLab FOSS at tag `v19.3.0`. Recreate
  the tree with
  `git clone --depth 1 --branch v19.3.0 https://github.com/gitlabhq/gitlabhq.git`
  and verify HEAD is `2c30df7828b86f8ac1fca26e96385fe712b0669f`.
- **Ground truth B (docs):** GitLab's published architecture
  documentation (docs.gitlab.com, development/architecture) and, where
  relevant, their public handbook product hierarchy.
- **Toolchain:** yarramate 1.0.0. Use the verbs (`ask`, `check`,
  `reconcile`, `design`) before reading raw YAML; reading the YAML is
  allowed, but the tool surface is part of what is under audit.

## What "how detailed" means here — score each axis separately

### 1. Documentation coverage

Enumerate every component GitLab's architecture page names. For each,
classify: **modelled** / **omitted and stated** in JOURNEY.md /
**omitted and unstated**. The last category is the finding. Report the
three counts and the unstated list.

### 2. Code coverage, sampled by fixed protocol

So your sampling cannot chase what the model already knows: (a) every
`*_VERSION` file at the repo root; (b) every top-level directory of
`app/`; (c) ten subdirectories of `app/services/` chosen alphabetically
from a seed you state; (d) `lib/api`, `app/graphql`, `app/events`,
`config/feature_flags`, `app/models/integrations`. For each item:
modelled at some grain / collapsed with the collapse stated / absent
and unstated.

### 3. Claim verification

Sample 25 relationships across kinds (include every `serving` between
components and services, all `access` modes, the `triggering` chain,
and the `supersedes` field). For each: verify direction and semantics
against code. ArchiMate serving points provider to consumer; a reversed
edge is a defect, not a style choice. Also resolve **every** evidence
locator (all ~49) at the pinned commit and report any that do not
resolve or whose message misstates what is there.

### 4. The disagreement machinery

The model deliberately contains one non-confirmed observation (Praefect,
`not-observed`). Check whether it is honest, then look for
disagreements the model should have recorded and did not: places where
the architecture page and the FOSS tree diverge. Every one you find
that the model neither records nor states as an omission is a finding.

### 5. Grain honesty

Compare `JOURNEY.md`'s stated omissions against what you actually found
missing. The question is not whether the model is complete (it is
deliberately sampled) but whether **it knows what it does not know**.

### 6. Interview reproducibility

Run `yarramate design` and `yarramate ask --open`. Confirm the
open-by-design set matches JOURNEY.md (2 motivation attestations,
1 kind-untested on Praefect). Anything else open, or anything closed
that your findings say should be open, goes in the report.

## Report format

One markdown report: a scorecard table (axis, counts, verdict), then
findings ordered by severity, each with the evidence path or doc URL
that proves it. Separate two kinds of finding ruthlessly:

- **Model findings** — the builder missed or misstated something.
- **Product findings** — yarramate itself made the gap likely (a
  question the catalogue never asked, a surface that hid information).
  These go in their own section; read
  <https://github.com/yarrasys/yarramate/issues/272> first so you
  extend it rather than rediscover it.

State your token/time spend at the end. Do not soften the verdict; the
builder session explicitly asked to be measured.
