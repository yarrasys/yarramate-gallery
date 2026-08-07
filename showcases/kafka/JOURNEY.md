# Apache Kafka — YarraMate discovery journey

## Provenance

| | |
| --- | --- |
| Source | <https://github.com/apache/kafka.git> |
| Tag | `4.3.1` |
| Tag object (annotated) | `a07059eb9b5bac1bfdbb1e74313f2fae4ca20fd9` |
| Commit (deref. HEAD) | `26b251a451ce941d3d7a55e6487bcb7f16b5ad48` |
| Branch | `4.3.1` (shallow clone, detached HEAD) |
| Analyzed | 2026-08-07 |
| Toolchain | yarramate 0.15.0, yarramate-likec4 0.15.0 |
| Profile | `yarramate/core@0.1` |

The freeze target `a07059eb9b5bac1bfdbb1e74313f2fae4ca20fd9` is the **annotated
tag object** for `4.3.1`; `git rev-parse 4.3.1^{commit}` dereferences it to the
actual commit `26b251a451ce941d3d7a55e6487bcb7f16b5ad48`, which is the clone
HEAD. All `repo:` evidence locators are valid at that commit. **Kafka 4.3.1 is
KRaft-only: Apache ZooKeeper was removed in 4.0, so it is deliberately not
modelled** — the KRaft controller quorum and the metadata log replace it.

## Journey and question

**Journey:** Discover an existing project (yarramate-architecture skill).

**Question answered:** How is Apache Kafka organised as a distributed streaming
platform at 4.3.1 under KRaft — its brokers (combined broker+controller or
dedicated controller nodes), topics and partitions, the partition
leader/follower replication model (ISR, high watermark), producers, consumers
with consumer groups and the group coordinator, the KRaft controller quorum and
the metadata log (Raft consensus among controllers), and on-disk segment-based
log storage — as observable from the repository at the pinned commit?

## Observations vs interpretive proposals

### Direct observations (locator-backed)

- **Combined/dedicated roles:** `KafkaRaftServer` constructs an optional
  `BrokerServer` (when `process.roles` has `BrokerRole`) and an optional
  `ControllerServer` (when `ControllerRole`) over one `SharedServer`; a node may
  be combined, broker-only, or controller-only
  (`core/src/main/scala/kafka/server/KafkaRaftServer.scala`,
  `BrokerServer.scala`, `ControllerServer.scala`).
- **Replication engine:** `ReplicaManager` holds `allPartitions`, the delayed
  produce/fetch purgatories, and a `replicaFetcherManager`; it exposes
  `appendRecords`/`fetchMessages`/`becomeLeaderOrFollower` and a scheduled
  `maybeShrinkIsr` (`core/src/main/scala/kafka/server/ReplicaManager.scala`).
- **Partition / ISR / HW:** `kafka.cluster.Partition` manages leader/follower
  replicas, the ISR (`getOutOfSyncReplicas`, `maybeShrinkIsr`, `prepareIsr*`),
  and the high watermark (`maybeIncrementLeaderHW`, min LEO over the maximal
  ISR) (`core/src/main/scala/kafka/cluster/Partition.scala`). The HW is stored
  in `UnifiedLog.highWatermarkMetadata` and advanced monotonically
  (`storage/.../log/UnifiedLog.java`).
- **Follower replication:** `ReplicaFetcherThread.processPartitionData` appends
  the leader's records via `appendRecordsToFollowerOrFutureReplica`, then
  advances the follower's HW from the Fetch response
  (`core/src/main/scala/kafka/server/ReplicaFetcherThread.scala`).
- **ISR → controller:** `AlterPartitionManager.submit` enqueues ISR changes and
  sends at most one in-flight `AlterPartitionRequest` to the controller
  (`core/src/main/scala/kafka/server/AlterPartitionManager.scala`).
- **Request routing:** `KafkaApis.handleProduceRequest` routes to
  `replicaManager.handleProduceAppend`; `handleFetchRequest` routes to
  `replicaManager.fetchMessages` (`core/src/main/scala/kafka/server/KafkaApis.scala`).
- **On-disk logs:** the active log manager is the Scala `kafka.log.LogManager`
  (`getOrCreateLog`, retention/cleaning); `UnifiedLog` appends to the active
  `LogSegment`, which carries offset/time indexes
  (`core/src/main/scala/kafka/log/LogManager.scala`,
  `storage/.../log/UnifiedLog.java`, `LogSegment.java`). (The Java
  `storage/.../log/LogManager.java` is only a static helper.)
- **KRaft consensus:** `KafkaRaftClient<T> implements RaftClient<T>` — "a
  Kafkaesque Raft: leader election is pure Raft, replication is follower-pull".
  `QuorumState` runs the Resigned/Unattached/Prospective/Candidate/Leader/Follower
  state machine and persists the epoch/vote via `FileQuorumStateStore`
  (`raft/src/main/java/org/apache/kafka/raft/`).
- **Metadata log:** the single `__cluster_metadata` partition 0
  (`Topic.CLUSTER_METADATA_TOPIC_NAME`, fixed id `Uuid.METADATA_TOPIC_ID`) is the
  Raft log; it is a storage `UnifiedLog` wrapped by `KafkaRaftLog`
  (`clients/.../common/internals/Topic.java`, `raft/.../KafkaRaftLog.java`).
- **Metadata image pipeline:** the Raft client feeds a `MetadataLoader` (a
  `RaftClient.Listener`) which builds `MetadataDelta`/`MetadataImage` and fans
  them out to publishers; `BrokerMetadataPublisher.onMetadataUpdate` calls
  `metadataCache.setImage`. `KRaftMetadataCache` holds a volatile immutable
  `MetadataImage` (`metadata/.../image/loader/MetadataLoader.java`,
  `core/.../server/metadata/BrokerMetadataPublisher.scala`,
  `metadata/.../metadata/KRaftMetadataCache.java`).
- **Active controller:** the Raft leader of the metadata log is the single
  active `QuorumController`; `PartitionRegistration` (replicas/isr/leader/
  leaderEpoch/partitionEpoch) is the per-partition value stored in the log
  (`metadata/.../controller/QuorumController.java`,
  `metadata/.../metadata/PartitionRegistration.java`).
- **Producer:** `KafkaProducer.send` → `RecordAccumulator.append`; the background
  `Sender` drains ready batches and `sendProduceRequest` sets `acks`. The
  `acks` config defaults to `all` (`clients/producer/`).
- **Consumer:** `KafkaConsumer` is a facade over a `ConsumerDelegate`
  (`ClassicKafkaConsumer` uses `ConsumerCoordinator` + `Fetcher`; `AsyncKafkaConsumer`
  is the next-gen path). `ConsumerCoordinator.onLeaderElected` runs
  `assignor.assign`; `Fetcher.sendFetches` pulls from leaders
  (`clients/consumer/`).
- **Assignor:** `CooperativeStickyAssignor` ("cooperative-sticky") confirmed,
  with eager sticky, range, and round-robin alternatives
  (`clients/.../consumer/CooperativeStickyAssignor.java`).
- **Group coordinator:** `GroupCoordinatorService` (broker-side) manages groups;
  `OffsetMetadataManager.commitOffset` writes `OffsetCommitKey/Value` records to
  `__consumer_offsets` (`Topic.GROUP_METADATA_TOPIC_NAME`), a compacted topic
  (`group-coordinator/`).
- **No ZooKeeper:** the only `zookeeper` reference under `core/src/main` is a
  comment in `core/src/main/scala/kafka/tools/TestRaftServer.scala`; there is no
  `kafka.zk` package or ZK client.

### Interpretive proposals (semantic meaning added for review)

- The actors **Producer application** and **Consumer application** are
  interpretations; the repository ships only the client libraries and documents
  producers/consumers in `README.md`.
- Collapsing the producer (`KafkaProducer` + `RecordAccumulator` + `Sender`) into
  one **Producer client** concept, the consumer facade plus its classic
  delegate into one **Consumer client** concept, and the `MetadataLoader` +
  publishers into one **Metadata publisher** concept is an abstraction choice.
- The **Kafka cluster** concept is an interpretation that groups the nodes and
  the single shared metadata log; `SharedServer` is the closest code anchor.
- The leader/follower replication is modelled through the leader **Broker**
  serving follower **Replica fetcher** threads (the observed
  `ReplicaFetcherThread`), rather than a broker→broker self-edge, to avoid an
  ambiguous self-loop. ISR and high watermark are modelled as information
  objects (`Partition`/`UnifiedLog` are the code anchors).
- Current state only; no target intent was inferred. No ownership or
  constraints were declared — the repository contains no governance sources to
  support them at this depth.

## Model

Canonical inputs (this directory):

- `.yarramate/workspace.yaml` — workspace `kafka`
- `.yarramate/architecture/kafka.yaml` — 26 concepts, 42 relationships,
  0 states (1 document)
- `.yarramate/evidence/repository.yaml` — 68 observations (one per concept and
  one per relationship)
- `.yarramate/projections/*.yaml` — 5 projections (3 static, 2 dynamic)
- `.yarramate/likec4-project.yaml` — 5 views (3 static, 2 dynamic)
- `.yarramate/integrations/likec4/subject-mapping.yaml` — 68 mappings
  (populated by `map --sync`)

Concept kinds: 2 business actors, 13 application components, 7 data objects,
4 artifacts. Relationship kinds: 16 composition, 2 aggregation, 11 access,
13 flow.

## Evidence results and reconciliation

`yarramate reconcile` summary (provider `repository-inspection`, evidence
document `kafka-repository@1.0`):

| result | count |
| --- | --- |
| confirmed | 68 |
| contradicted | 0 |
| unknown | 0 |
| not-observed | 0 |

All 26 concepts and all 42 relationships carry a `repo:` observation; none were
left unevidenced (`subjectsWithoutEvidence: 0`). Evidence was never promoted
into declared intent. Every observation is grounded at the pinned commit.

## Projections and views

| Projection | View type | Content |
| --- | --- | --- |
| `system-overview` | static | All 26 concepts and all 42 relationships |
| `replication-and-storage` | static | Broker, replica manager, replica fetcher, log manager, partition log, log segment, topic, partition, ISR, high watermark, record |
| `metadata-plane` | static | Cluster, controller, controller quorum, Raft engine, metadata log, metadata publisher, metadata cache, broker, partition |
| `produce-path` | dynamic (7 steps) | producer → broker → partition log → log segment → replica fetcher (ISR) → high watermark → ack |
| `consumer-group-coordination` | dynamic (6 steps) | consumer join → assignor assignment → coordinator delivers → fetch → commit offsets → `__consumer_offsets` |

Generated LikeC4 output: `.yarramate-out/likec4/` (`model.likec4`,
`specification.likec4`, `likec4.config.json`, `yarramate.generated.json`).
The generated output is current with the authored inputs, and a re-run of
`export-project` is byte-identical to what is committed (verified by SHA-256
before/after — see Validation).

## Rendering coverage audit

- **Concepts in no projection:** none — `system-overview` covers all 26 concepts
  and all 42 relationships.
- **Ordered chains without a dynamic view:** the AlterPartition ISR→controller
  feedback and the metadata-log→cache publication are intentionally rendered
  statically (in `metadata-plane` and `replication-and-storage`); they are
  side-channels to the two primary flows, not end-to-end request journeys worth
  their own dynamic view.
- **Projections absent from the LikeC4 project:** none — all 5 projections are
  listed as views.

### Intentional model omissions

Observed in the repository but deliberately left out of the smallest useful
current-state model: the next-gen async consumer (`AsyncKafkaConsumer`,
KIP-848) and classic/async split detail; Kafka Connect, Kafka Streams, and
MirrorMaker subsystems; transactions and the transaction coordinator/share
coordinator; security (SASL, TLS, ACL authorizer, SCRAM), quotas, and the
KRaft version-upgrade coordinator; the network/socket server and request
handler pool internals; producer idempotence/epoch and exactly-once semantics;
snapshot generation/loading for the metadata log; per-language clients beyond the
Java reference clients; log compaction/cleaner internals; and deployment
packaging/scripts. ZooKeeper is omitted by design (removed in 4.0).

## Validation

All commands used the pinned yarramate 0.15.0 toolchain. LikeC4 project paths
resolve relative to the project document, so the checks pass from any working
directory.

| Command | Outcome |
| --- | --- |
| `yarramate check .yarramate/workspace.yaml --json` | ok, exit 0 (1 doc / 26 concepts / 42 relationships) |
| `yarramate reconcile .yarramate/workspace.yaml` | exit 0, 68 confirmed / 0 contradicted / 0 unknown / 0 not-observed / 0 findings |
| `yarramate ask .yarramate/workspace.yaml <each of 5 projections>` | exit 0 (×5), renders concepts and relationships |
| `yarramate-likec4 check .yarramate/likec4-project.yaml --json .yarramate/workspace.yaml` (pre-sync) | ok:false — expected: empty mapping, 68 missing entries |
| `yarramate-likec4 map --sync .yarramate/integrations/likec4/subject-mapping.yaml .yarramate/workspace.yaml` | exit 0, added 68 mappings (authored diff reviewed) |
| `yarramate-likec4 check` (post-sync) | ok:true, exit 0 |
| `yarramate-likec4 export-project .yarramate/likec4-project.yaml .yarramate-out/likec4 .yarramate/workspace.yaml` | exit 0 |
| Determinism — re-run `export-project`, SHA-256 of all 4 generated files | byte-identical |
| Portability — `yarramate check` from a foreign cwd | ok:true, exit 0 |

⚠️ A green check is deterministic correctness, not completeness or architecture
approval. The omissions above are reported, not hidden.

## Unresolved architectural decisions

None pending for the current-state model. Open modelling questions a maintainer
might take up later:

- Whether the broker and controller should be modelled as separate node kinds
  or as roles of one node concept (modelled as separate concepts here, since
  KRaft allows combined, broker-only, and controller-only nodes).
- Whether the next-gen async consumer deserves its own concept alongside the
  classic consumer (collapsed into one Consumer client concept here).
- Whether Kafka Connect / Streams should be added as first-class subsystems
  (out of scope for this streaming-core model).

## Status in Git

This showcase is a proposed, uncommitted artifact: the model was authored
directly in the gallery tree. No commits, pushes, branches, or issues were
created (per instructions, no git operations were run in the gallery repo).
