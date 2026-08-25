# YarraMate gallery

[YarraMate](https://github.com/yarrasys/yarramate) turns a software codebase into
a queryable, evidence-backed architecture model: the concepts, relationships,
and ordered flows of a system, read from the real source, checkable, and
projectable into focused views.

This gallery is the **reference set of those models**. Each one was produced by
an automated discovery journey that reads the actual source, is pinned to a
specific commit, and passes YarraMate's deterministic checks.

**Browse the rendered diagrams below**, open any showcase's native model under
`.yarramate/`, or read its `JOURNEY.md` for how the model was built and what it
deliberately leaves out.

## A small set, kept current

The gallery deliberately holds a **small number of deep showcases** rather than a
broad corpus. A checked model is only worth reading if it still checks, and
YarraMate's own vocabulary moves fast enough that keeping a wide set of models
current costs more than the breadth is worth. Five earlier showcases (httpie,
fastify, miniflux, uptime-kuma, keycloak) were retired for that reason in
August 2026. They remain in this repository's Git history, valid against the
toolchain they were authored under.

Depth is the point: a model that survives a major version, with its evidence
still resolving against the pinned commit, says more than six that do not.

## Showcases

Each showcase is a self-contained, validating YarraMate workspace:

```text
showcases/<repo>/
  .yarramate/       # canonical native model: documents, projections, evidence, likec4 integration
  .yarramate-out/   # derived LikeC4 export, the authoritative rendered source
  diagrams/         # rendered PNG previews of every view (regenerable; see Reproduction)
  JOURNEY.md        # discovery report: provenance, observations, evidence, coverage gaps
```

Every model passes the deterministic checks from its showcase directory:

```sh
npx -p yarramate yarramate check showcases/<repo>/.yarramate/workspace.yaml --json
```

| Showcase | Shape / language | Model | Views | Reconciliation |
|---|---|---|---|---|
| [kafka](showcases/kafka/JOURNEY.md) | Distributed streaming platform / Java+Scala | 26 concepts, 42 relationships, 5 projections | 5 (2 dynamic) | 68/68 confirmed, 0 findings |

### kafka

<img src="showcases/kafka/diagrams/system-overview.png" alt="Kafka system overview" width="640">

Apache Kafka is a distributed event streaming platform written in Java and
Scala, run since 4.0 exclusively in KRaft mode (Apache ZooKeeper removed). A
cluster is a set of brokers, some of which form a KRaft controller quorum that
replicates cluster metadata through a Raft consensus log. Data lives in topics
split into partitions, append-only, segment-backed logs, where each partition
has one leader broker handling all reads and writes and zero or more followers
replicating it; the in-sync replica set and a high watermark govern durability
and visibility. Producers publish to partition leaders; consumers organised in
consumer groups coordinate with a group coordinator (a broker) for partition
assignment and offset commits.

The model was discovered on 2026-08-07 with `yarramate@0.15.0` against Kafka
`4.3.1` (annotated tag object `a07059eb…`, commit `26b251a4…`). It was migrated
to `yarramate@1.0.0` on 2026-08-25: subject references were flattened
([yarramate#223](https://github.com/yarrasys/yarramate/issues/223)) and thirteen
relationships were brought onto the ArchiMate 3.2 relationship table, which
[ADR 0097](https://github.com/yarrasys/yarramate/blob/main/docs/adr/0097-relationship-endpoints-are-validated-against-the-archimate-relationship-table.md)
made the rule for every relationship. No fact about Kafka changed in that
migration; the shapes carrying those facts did. `JOURNEY.md` records what moved.

▶ flows: [produce-path](showcases/kafka/diagrams/produce-path.png) · [consumer-group-coordination](showcases/kafka/diagrams/consumer-group-coordination.png). All views in [`diagrams/`](showcases/kafka/diagrams/)

## Ask the models

A YarraMate model isn't a static picture. It answers questions, deterministically
and offline, against the committed workspace. Two kinds of query:

**"Where in the code is X?"** `yarramate ask <workspace> --where "<X>"` returns
each matching concept with the exact source location its evidence confirms:

```
yarramate ask showcases/kafka/.yarramate/workspace.yaml --where "group coordinator"
Where: "group coordinator" - 7 concepts matched, seeded from the top 5: broker, consumer-client, consumer-offsets, group-coordinator, consumer-application

  group-coordinator
    confirmed  repo:group-coordinator/src/main/java/org/apache/kafka/coordinator/group/GroupCoordinatorService.java  (repository-inspection)
      GroupCoordinatorService implements GroupCoordinator at :162; consumerGroupHeartbeat/joinGroup/syncGroup/heartbeat; per-group shard routing.
  consumer-offsets
    confirmed  repo:clients/src/main/java/org/apache/kafka/common/internals/Topic.java  (repository-inspection)
      GROUP_METADATA_TOPIC_NAME = "__consumer_offsets"; the compacted offsets topic.
```

**"Explain X and what it connects to."** `yarramate ask <workspace> "<X>"`
returns a deterministic prose slice of the matching concepts and their
relationships (try `"consumer group offset commit"`).

Both run with no network and no source checkout beyond the showcase's own
`.yarramate/`. Try any of them:

```sh
yarramate ask showcases/kafka/.yarramate/workspace.yaml --where "partition replication leader"
yarramate ask showcases/kafka/.yarramate/workspace.yaml "your own question"
```

## Why this exists

**Reference models.** A checked, evidence-grounded current-state architecture for
a real system, reviewable as native YAML and as rendered views, and durable
enough to survive the tool's own major versions.

This repository is the **source of truth** for those models and their
deterministic exports. A companion interactive rendering is planned for the
[YarraMate website](https://github.com/yarrasys/yarramate-website); until then,
the `diagrams/` previews and each showcase's `.yarramate-out/likec4` source are
the viewable output.

## Reproduction

Each `JOURNEY.md` records the source commit SHA, analysis date, and CLI version.
The recipe: shallow-clone the source repo at that SHA, load the
`yarramate-architecture` skill from the YarraMate repo, and follow the
"Discover an existing project" journey with the pinned CLI. Models are proposals
reviewed via Git. A green `check` is correctness, not completeness, and the
coverage gaps in each journey report are part of the result.

**Rendering the `diagrams/` PNGs.** These are convenience previews rendered from
each showcase's `.yarramate-out/likec4` source with the [LikeC4](https://likec4.dev)
CLI. They are **not** byte-gated artefacts (Chromium and font rendering are not
reproducible across machines), so the authoritative rendered source remains the
committed `.likec4`. To regenerate:

```sh
npm i likec4 @tanstack/ai                                   # once, in a scratch directory
env -u GEMINI_API_KEY npx likec4 export png \
  showcases/<repo>/.yarramate-out/likec4 -o showcases/<repo>/diagrams
```

If your shell does not export `GEMINI_API_KEY`, drop the `env -u` prefix. The
auto-generated `index.png` is a redundant thumbnail grid and is intentionally not
committed.

Models here describe third-party projects but are not affiliated with or
endorsed by them; they are interpretive snapshots at the recorded commit.
