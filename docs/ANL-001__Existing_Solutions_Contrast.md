# ANL-001 — Engine Highlights Against Existing Solutions at 30k EPS

| Field | Value |
| --- | --- |
| Document ID | ANL-001 |
| Title | Engine highlights against existing solutions at 30k EPS |
| Version | 0.2.0 |
| Status | Draft — analysis annex to DEC-001 |
| Last Updated | 2026-10-20 |
| Related documents | DEC-001 (v0.4.0), TDD-001, PRD-001, DISCUSSION-SUMMARY |

## 1. Purpose

This annex states the engine’s defensible highlights. It is not a feature list and not a claim of novelty. It asks a narrower question:

> At 30k EPS, which parts of the engine are commodity, and which parts cannot be reproduced by an existing stack without rebuilding the engine’s semantic layer?

The cut is out of scope; DEC-001 covers it. Delivery — cut, bound, per-viewer baseline, encoding, client memory — is not compared with Materialize in §8; it has its own baselines (3D Tiles/Cesium, Potree, deck.gl, interest management) and is treated in §9. This annex concerns the pipeline that remains: hot state, hierarchical rollup, snapshot, and — as a candidate extension — bitmask access control.

All figures are derived or assumed at S0; none are measured.

## 2. The engine’s contract

30k EPS is the S0 rate: 10,000 devices × 3 metrics × 1 Hz = 30,000 readings/s, or 30,540 values per tick including 540 group channels (DEC-001 §3.1). The full state is ~122 KB/tick; the full-state delta is ~65 KB/tick.

| Function | Contract | S0 size |
| --- | --- | --- |
| Hot state | Latest value per (device, metric), update in place, per-metric freshness | 30,000 values, ~120 KB |
| Rollup | Exact reduction over a tree with configured channel wiring, with availability and attention | 180 groups × 3 channels = 540 aggregates |
| Snapshot | One shared frame snapshot, consistent across viewers | ~122 KB/tick; 10 viewers at 1 Hz ≈ 1.2 MB/s |
| Access (candidate) | Per-consumer bitmask; aggregate over authorized contributions only | 10k-bit mask ≈ 1.25 KB per consumer |

The rollup update rate is roughly 30,000 leaf updates × 3–5 ancestors ≈ 90–150k aggregate updates/s. **Throughput is not the constraint.** The constraint is the semantic contract and its guarantees.

**Topology assumption.** The node hierarchy is assumed to be a **tree**: exactly one root; every other node has exactly one parent; there are no cycles; devices are leaves and each device attaches to exactly one group (legacy design references: PRD FR-TD-01, FR-TD-07; TDD §4.5). This is the contract’s assumption while the project is reconsidered, not an external requirement. The channel wiring is layered over that tree: a channel contributes only to channels of its **unique parent group**, so the contribution graph is acyclic, but a channel may fan out to several parent channels and a parent channel may fan in from several child channels. **There are no multi-parent nodes and no cross-links.**

## 3. Highlight 1 — Exact rollup over a tree with channel wiring

The node hierarchy is a tree, but the engine is not a simple tree `GROUP BY`: its aggregation contract (TDD-001 §8) wires many-to-many channel contributions over that tree:

- **Fan-out and fan-in within the parent edge.** A device metric may feed several channels of its containing group; a group channel may feed several channels of its parent group; a parent channel may receive several child channels and local metric contributions. Every contribution goes to the unique parent group; there are no multi-parent nodes.
- **Same unit, same reduction.** Contributions combine only values of the same unit, and a channel may join only a parent channel of the same reduction (`sum`, `mean`, `min`, `max`, `count`); violations abort boot.
- **Exact mean and count.** `mean` is exact over the subtree’s available readings, carried as `(sum, count)`; a `count` channel is the total number of available contributing readings, merged from child `count` channels by summation.
- **Availability.** A metric is available only while its reading is finite and within its `freshness_ms`; stale and offline readings are excluded without affecting the device’s other metrics. A channel with no available contribution reports no value, not zero.
- **Attention and absence.** A reserved unitless channel carries the maximum severity from limits and per-metric absence levels; it propagates independently of the configured contributions.
- **Canonical order.** Bit-exact agreement with a brute-force reference requires a fixed blocked-lane f32 reduction order (`W = 8`); a parallel or plan-dependent order can differ in the last bits. This is a design assumption, not a sourced requirement; §8.4 names the oracle.
- **Shared frame snapshot.** Aggregation runs once per frame, double-buffered, independent of viewer count; all viewers of a frame see the same snapshot.
- **Budget.** ≤5 µs per channel per group with up to 1,000 contributing readings (K9, a legacy PRD planning budget, not a measurement); at S0, 540 channels across 180 groups.

### Why the commodity stack does not provide this out of the box

SQL can model a tree rollup with a recursive CTE or a closure table, and Materialize’s recursive CTEs and MV-on-MV patterns cover the simple case. It does not provide the conjunction:

- The node graph is a tree, but the channel wiring is many-to-many within the parent edge. A parent channel can receive several child channels and local metric contributions; multiple channel paths to the same parent channel can double-count unless the edge semantics are explicit. This is channel wiring, not multiple parents.
- `SUM(float4)`/`AVG(float4)` is not associative. In Materialize it is accumulated in fixed point at 2⁻²⁴ granularity, which is deterministic and order-independent but does not match a canonical f32 order; a generic parallel SQL plan may instead vary with the plan (§8.4).
- Per-metric freshness is a predicate against a frame clock, not a static `NULL`.
- A no-value channel must be distinguishable from zero and from a stale value.
- Attention/absence is a custom max channel, not a generic aggregate.
- A shared frame timestamp across all viewers is a coordination layer, not a materialized view.

**Condition.** The highlight is real only if exact, deterministic, availability-aware rollup is required. If tolerant arithmetic and a simple tree are acceptable, SQL/materialized views are sufficient at 30k EPS.

## 4. Contract property — Shared frame snapshot independent of viewers

The engine computes the rollup once per frame and publishes one snapshot; per-viewer work starts after the snapshot. The cost is independent of viewer count (FR-AG-03). This is what makes the same aggregate usable by the cut, by a policy layer, and by a dashboard without recomputation.

A generic SQL stack can give a consistent snapshot per query, but:

- A materialized view is continuously updated; two viewers querying at different times can see different versions.
- A transaction snapshot is consistent per query, not shared across viewers.
- Flink/Kafka Streams snapshots are per-partition; a global frame snapshot needs coordination.
- A per-viewer pull query at 60 Hz multiplies the aggregation cost by V; a `SUBSCRIBE` on a shared materialized view does not, so the fair baseline is the shared view.

**Condition.** The property matters when V is large, the cadence is 60 Hz, or all viewers must see the same frame. At 1 Hz and V = 10, per-viewer SQL snapshots are comfortable. It is a contract property rather than a differentiator: a shared materialized view plus `SUBSCRIBE` gives the same consistency across subscribers, and arguably stronger linearizability (§8.2). The residual is the frame clock and its cost model, not consistency itself.

## 5. Conditional highlight — Mask-aware aggregation (bitmask access control)

This is the candidate extension. It is the strongest potential differentiator because access control is a **candidate requirement** (DISCUSSION-SUMMARY §127), not a PRD requirement: PRD-001 §10 places authentication and authorization out of scope, and unlike K3 this is not even self-adopted.

### 5.1 The model

Each consumer `v` has an authorization bitmask `A_v` over the **device set** (10,000 devices; ~1.25 KB per consumer). A metric source is authorized iff its device is authorized; aggregation expands the device mask to its metric slots (30,000 slots; ~3.75 KB if the expanded mask is materialized). For a channel `c` with contribution sources `S_c`, the consumer’s value is the reduction over the authorized sources:

```
value_v(c) = reduce({ s ∈ S_c : device(s) ∈ A_v })
```

Child channels are reduced recursively with the same mask. The mask is applied **before** the reduction, not after serialization. The topology stays a tree; only the channel wiring fans out and fans in. The masked rollup is therefore a bottom-up pass over the tree, with the mask applied to each channel’s local metric sources and child-channel contributions.

The consequence is structural: the same channel can have a different value for each distinct mask. A single global rollup is no longer valid for all consumers if unauthorized contributions must be excluded.

### 5.2 What changes by reduction

| Reduction | Mask-aware computation | Cost |
| --- | --- | --- |
| `sum`, `count` | Global value minus the **available** unauthorized contributions; or a masked scan | `O(unauthorized)` correction, or `O(authorized)` masked scan |
| `mean` | Global sum and count minus the available unauthorized sum and count; `mean = sum / count` | `O(unauthorized)` correction, or `O(authorized)` masked scan |
| `min`, `max`, attention | A global min/max is insufficient; need a mask-aware tree or a scan of authorized sources | `O(log N)` for structured masks; `O(k)` for arbitrary masks |
| `no-value` | A channel with no authorized available contribution reports no value | Per-channel check |

The mask cardinality decides the architecture:

- **Role mask (K small):** one aggregate per role; SQL can maintain one materialized view per role.
- **Subtree mask (hierarchical):** per-subtree aggregates combined per consumer; a closure table can express it, but the combination is custom.
- **Arbitrary per-consumer mask (K ≈ V):** per-consumer aggregates; the shared global view is invalid; cost scales with V unless the engine computes a shared correction.

### 5.3 What SQL can and cannot do

SQL **can express** masked aggregation:

```sql
SELECT m.consumer_id, c.channel_id, SUM(v.value), COUNT(*)
FROM contributions c
JOIN masks m ON m.leaf_id = c.leaf_id AND m.allowed
JOIN values v ON v.leaf_id = c.leaf_id
GROUP BY m.consumer_id, c.channel_id;
```

The policy mask is per device; the `masks` relation is its resolved metric-level expansion.

It can do `SUM`/`COUNT`/`MIN`/`MAX` per consumer, with `FILTER` for availability and `CASE` for attention. It can use bitwise operators if the mask is a bitmap, and bitmap indexes can help. A recursive CTE over the parent-child tree is sufficient for the topology; the channel edge table handles fan-in/fan-out, so no general graph traversal is required.

What is not available is the single shared global rollup:

- A view keyed by `(mask class, channel)` is expressible, but its size and per-update recomputation scale with mask cardinality.
- A parameterized query per consumer is possible, but the cost is `O(V × contributions)` per cadence; at 1 Hz and V = 10 this is fine; at 60 Hz and V = 100 it is not obviously fine.
- A global aggregate plus an output filter leaks unauthorized contributions through the aggregate.
- `MIN`/`MAX`/attention cannot be corrected from a global value; SQL must scan the masked join or maintain a custom structure.

So the bitmask does **not rule out SQL as a language**. It rules out the **single shared global rollup** for arbitrary per-consumer masks. The gap is the shared cost against recompute-per-query SQL, the exact masked semantics, and the constant factors of an IVM mask-class view; because that view is also expressible, the differentiator is narrower than expressibility (§8.6).

### 5.4 The engine’s candidate advantage

The engine can apply the mask inside the rollup:

- Bitwise mask against the contribution lists; SIMD masked reduce.
- `sum`/`count`/`mean`: global aggregate plus unauthorized correction; the correction is proportional to the unauthorized set, not the full channel.
- `min`/`max`/attention: a mask-aware tree or a bounded scan, depending on mask structure.
- Per-mask equivalence classes: consumers with the same mask share one aggregate.
- Availability, no-value, and attention remain exact and consistent with the mask.

This is a **shared-cost masked rollup**, not a per-consumer query. It is the candidate highlight against recompute-per-query SQL; against an IVM mask-class view the difference is in the arithmetic and constants, not in expressibility (§8.6).

**Conditions.** The advantage exists only if:

1. Masks are arbitrary per consumer, not a small role set.
2. The aggregate must exclude unauthorized contributions, not merely hide the output.
3. The cadence is 60 Hz or V is large enough that per-consumer SQL is costly.
4. The aggregate is exact and availability-aware.

If masks are role-based, SQL per-role views are sufficient. If the cadence is 1 Hz and V = 10, per-consumer SQL is sufficient. The engine’s advantage is a narrow band, and it must be measured.

### 5.5 Leakage and policy semantics

Masking the aggregate is not the same as masking raw readings. Four policies are possible, and they have different consequences:

| Policy | What the consumer sees | Leakage |
| --- | --- | --- |
| Raw mask | Only authorized raw readings; aggregates over the full hierarchy | Aggregate may reveal unauthorized children |
| Aggregate mask | Aggregates over authorized contributions only | Exact masked rollup; no direct leakage of unauthorized child values, but inference and differencing remain |
| Suppress | A group is withheld if any child is unauthorized | All-or-nothing; safe but coarse |
| Aggregate + suppression | Masked aggregate, suppressed when the group is too small or too revealing | Disclosure control; not exact for small groups |

The engine’s mask-aware rollup addresses the **aggregate mask**. It does not by itself solve **disclosure** (small-group inference, differencing over time). That is a separate policy layer. The bitmask is authorization, not privacy.

## 6. What remains commodity

At 30k EPS, the following are commodity and should not be the pitch:

- Hot-state ingest and storage: 30,000 values, ~120 KB; a table, KV store, or in-memory map.
- Simple tree rollup: `sum`/`mean`/`count` over a tree; recursive CTE or materialized view.
- Snapshot publication: a query or a cached view at 1 Hz.
- The policy engine: OPA/Cedar/SpiceDB; authorization decisions.
- The broker, the 3D renderer, the twin graph, and the dashboard.

The stack handles the scale comfortably. The gap is the semantic contract, not the EPS.

## 7. Framework landscape

| Category | Representative tools | Hot state | Rollup | Snapshot | Fit |
| --- | --- | --- | --- | --- | --- |
| Streaming SQL / IVM | **Materialize**, RisingWave, Flink SQL, ksqlDB | Maintained views | SQL; Materialize recursive CTEs; RisingWave MV-on-MV; Flink/ksqlDB custom | `SELECT`/`SUBSCRIBE`; pull queries | Closest for simple rollups |
| Stateful stream processors | **Flink**, **Kafka Streams** | Keyed state | Custom topology | External sink; interactive queries | Most control; more code |
| In-memory data grids / NewSQL | **Hazelcast**, Ignite, SingleStore, VoltDB | Native | SQL/entry processors | SQL/map query | Hot state + low-latency snapshots |
| Time-series databases | TimescaleDB, ClickHouse, InfluxDB | Table/cache | Time-window aggregates; tree via SQL | SQL | Time rollups, not semantic tree |
| KV stores | **Redis**, NATS KV | Native | Custom Lua/functions | `GET`/`MGET` | State cache; custom rollup |
| Twin platforms | Eclipse Ditto, Azure Digital Twins, AWS TwinMaker | Twin state | Query/aggregation API | Query | Twin semantics, not 30k rollups |

Feature notes to verify in the current release:

- Materialize recursive CTEs are stable (December 2024); `WITH MUTUALLY RECURSIVE`.
- RisingWave recursive CTE support is limited (tracking issue #15135); use MV-on-MV or flatten the hierarchy.
- Flink Queryable State is deprecated since 1.18; publish snapshots to an external store.
- Kafka Streams interactive queries require an RPC layer across partitions; use `READ_COMMITTED` with transactional state stores for consistency.
- TimescaleDB continuous aggregates are time-windowed, not tree-aware.

## 8. Engine versus Materialize: detailed comparison

Materialize is the closest packaged implementation of the engine’s aggregation layer, so it is the baseline that decides whether the highlight is real. The comparison is only valid if both sides answer the same question, so Materialize is configured to the engine’s contract rather than to its own default deployment.

### 8.1 Configuration

| Setting | Engine | Materialize baseline |
| --- | --- | --- |
| State model | Latest value per `(device, metric)`, overwritten in place (TDD §5.1) | `CREATE TABLE ... FROM SOURCE ... ENVELOPE UPSERT` keyed by `(device, metric)` |
| Topology | Tree: one root, one parent per non-root node, devices as leaves (legacy PRD FR-TD-01/07) | Any relation; a tree is a `parent_id` or closure table, a DAG a recursive CTE |
| History | None | `RETAIN HISTORY FOR '1s'` (Kafka source default per current docs; private preview); current state plus a 1 s window |
| Freshness | `freshness_ms` per metric, arrival-stamped (TDD §5.3) | Temporal filter on an ingest timestamp, before the reduction (§8.5) |
| No-value | Explicit channel state | Left join to the channel dimension; `NULL`, not `0` |
| Masks | Cover, correction, or mask-aware tree over a per-device mask | Mask-class view, cover join, or global-minus-unauthorized (§8.6) |
| Snapshot | Frame clock, double-buffered | Timestamped materialized view; `SUBSCRIBE` |

Materialize defaults and restrictions are stated against the current documentation and are version-dependent; `RETAIN HISTORY` is a private-preview option in the current release.

With this configuration the two systems differ in three things that affect the aggregation contract — the update model, the arithmetic, and the physical layout — and in one that does not at S0: topology generality (§8.3). **Delivery is outside this comparison (§9).** The cut, the bound, the per-viewer baseline, encoding, and client memory have their own competitor set and are not a Materialize capability; scoring them here would be a category error. Everything else is a shared requirement.

### 8.2 Contract by contract

| Requirement | Engine | Materialize (configured) | Where it lands |
| --- | --- | --- | --- |
| Hot state, latest per slot | Flat SoA array; O(1) overwrite; ~0.3 MB at S0 | Upsert table plus arrangement; O(N·m) rows | Tie in order; engine constants expected lower (unmeasured) |
| Tree rollup | Precompiled post-order over contiguous subtrees | Precomputed closure table plus `GROUP BY` | Tie; the tree is not the hard part |
| One-parent topology | Assumed; contiguous subtrees; skip index; unique path per leaf | Not required; closure table over a tree or DAG; no parent rule | Materialize more general; no effect at S0 while the tree assumption holds |
| Channel fan-in/fan-out | Per-parent-edge contribution lists; canonical seed/merge order | Edge table joined to the closure; duplicate edges must be deduplicated | Tie; expressible, with explicit edge semantics |
| Same unit / same reduction | Boot validation; violations abort | Validation query; tables have no `CHECK` constraints | Engine: fail-fast at boot |
| Exact mean and count | `(sum, count)` in f32; count exact | `(sum, count)`; count exact; sum in fixed point | Split: count ties; f32 parity favours the engine |
| Per-metric freshness | `freshness_ms`, arrival-based | Temporal filter on `ingest_ts` | Engine simpler; Materialize expressible |
| No-value versus zero | Explicit | Left join plus `NULL`; guard `count` | Engine simpler; Materialize needs care |
| Attention and absence | Reserved channel; propagated | Anti-join to the expected set; `max` | Engine built in; Materialize expressible |
| Canonical f32 order | Blocked-lane `W = 8`, bit-exact to the oracle | Fixed-point accumulation at 2⁻²⁴, order-independent | Engine, if oracle parity is required |
| Shared frame snapshot | One rollup per frame; double-buffered | Timestamped MV shared across subscribers | Tie; Materialize is arguably stronger on linearizability |
| Masked aggregation | Cover plus exceptions; correction; mask-aware tree | Mask-class view; cover join; global-minus-unauthorized | Tie in structured regimes; engine edge in arbitrary sparse-exception regimes |
| Temporal aggregation | Not supported; no history | Supported with retained history and temporal filters (`RETAIN HISTORY`, private preview) | Materialize, outside the current-state contract |
| Operations | One process | Cluster and object store; consensus if self-managed | Engine simpler at small scale; Materialize more robust |
| Build cost | Aggregation layer sits inside the ~100 h shared substrate (planning) | Rollup is a few views (assumption) | Materialize |

Delivery is excluded from this table (§9). Cut, bound, per-viewer baseline, encoding, and client memory are not Materialize capabilities; their baselines are 3D Tiles/Cesium, Potree, deck.gl, and interest management (DEC-001 §4–§5).

### 8.3 The one-parent-node constraint

The engine assumes a **tree**: one root, exactly one parent per non-root node, no cycles, devices as leaves, and each device attached to exactly one group (legacy PRD FR-TD-01/FR-TD-07, TDD §4.5). Channel wiring is many-to-many, but only within the unique parent edge (§2). Materialize assumes no such thing: a tree is a `parent_id` column or a closure table, and a multi-parent graph is a recursive CTE or a precomputed transitive closure.

The constraint is not a capability either system has. It is a property of the modelled domain, and both receive the same tree.

**What it buys the engine.** Contiguous subtrees, parent-first layout, `skip_index = index + subtree_size`, `first_child = index + 1`, `next_sibling = sibling + subtree_size` (TDD §5.2), a unique path from each leaf to each ancestor, an unambiguous contribution set per channel, a canonical seed/merge order, contiguous device rows in the CSR state table, and a byte-stable preorder manifest. The skip index is an interval labelling of a tree; it is not a generic acceleration.

**What it buys Materialize.** No exclusive capability, but it does simplify the SQL baseline: the tree guarantees that recursion terminates and the closure is acyclic, so the baseline is simpler than a DAG would be. Materialize would handle a DAG as well, but a multi-parent graph would still need a contribution policy the modeller defines: does a device that reaches a group by two paths contribute once or once per path? The traversal is not the hard part; the semantics are.

**Is it an advantage?** No. It is a domain assumption that simplifies both sides. If anything, Materialize is strictly more general on topology, and that generality is free at S0. The engine’s specialisation buys a simpler traversal and a smaller index, not a defensible difference.

**What removing it would cost the engine.** The contiguous layout and the skip-offset arithmetic fail, because a node belongs to several subtrees and no single linearisation makes every subtree contiguous. The replacement is adjacency in CSR, a topological order, a dedup stamp for the cut, per-edge merge lists, and a boot-time reachability pass to deduplicate leaf contributions. The skip index survives only for **closed** subtrees (no incoming edge from outside the range) or by unfolding the DAG into a tree, at the cost of node identity and entry count. K3 (self-set; DEC-001) entry accounting also changes, because a child may already be in the frontier through another parent. These are costs of the topology assumption, not delivery advantages; delivery remains out of scope (§1).

**Where it matters for the comparison.** As a risk boundary, not a feature. The tree is a legacy design assumption (PRD FR-TD-01/07), not a sourced requirement; if the real domain has multi-membership (a device in a spatial group and a functional zone), the engine bakes the simplification into its index and aggregation contract, and Materialize does not. That is a genuine generality advantage for Materialize with a domain consequence. It should be stated plainly rather than counted as an engine strength.

### 8.4 The update model and the arithmetic contract

Materialize represents each effective change as a retraction plus an insertion at the dataflow level, even under `ENVELOPE UPSERT` (a repeated identical value produces no change). That is what lets it maintain views incrementally. Retractions require invertible arithmetic, so `sum` and `avg` over `real` and `double precision` are accumulated in fixed point at 2⁻²⁴ granularity, with each input truncated toward zero. Values with magnitude below 2⁻²⁴ contribute 0, and the result is documented as incorrect once the sum magnitude reaches ~2¹⁰³ — irrelevant at S0, but part of the bound. `min` and `max` return exact input values; `count` is exact; `numeric` is exact but is a different type. The behaviour is documented under the float types page, “Aggregate precision,” and is version-dependent.

The engine recomputes the rollup from leaves each frame in a fixed blocked-lane order. It needs no inverse, so it keeps IEEE f32 addition. That is the causal chain: **invertibility costs precision**.

The consequence is not “SQL is inexact.” It is that the two systems make different promises. Materialize is deterministic and order-independent, but not f32. The engine is deterministic and f32, but only because it does not maintain. Neither equals exact arithmetic.

The oracle has to be named before the test. If the oracle is the engine’s f32 path, the comparison is circular. If it is exact arithmetic, `numeric` wins and both lose. If it is “what an f32 client would compute,” the engine is closer. There is also a benchmark trap: a query evaluated entirely at planning time over a `VALUES` list uses ordinary floating-point addition, while a query through a source or materialized view uses the fixed-point path. The test must go through the dataflow, or it measures the wrong thing.

### 8.5 Availability, no-value, and attention

Freshness is a predicate, not a retention setting. The filter `WHERE mz_now() <= ingest_ts + freshness` sits in the view that feeds the aggregate, so stale rows are retracted and the aggregate updates. Per-metric freshness joins a `metric_defs` relation rather than storing an interval per row. Use an ingest timestamp, not the device’s event timestamp, to match the engine’s arrival-based `arrival_ms`; event time behaves differently under clock skew.

Two SQL-specific traps follow:

- A `GROUP BY` over no available rows emits **no row**, and `count` over no rows is `0`, not “no value.” Left-join from the channel dimension and emit `NULL`; guard count channels so zero contributions are no-value rather than zero.
- Absence and attention need a second view: anti-join the expected `(device, metric)` set against `available`, assign the absence level, then take the maximum with the configured limit breaches.

The cost is that Materialize must maintain future retractions. `mz_now()` clauses are comparison-only, combined with `AND`, and not allowed inside aggregate `FILTER`s. A non-materialized view evaluates per query; the materialized view is the shared-cost version. Retention and freshness are orthogonal: retention is a private-preview reconnect feature, freshness is correctness.

### 8.6 Masked aggregation

There are three primitives, not two:

| Primitive | Per-frame order | Wins when |
| --- | --- | --- |
| Per-class scan | `M·a` | M is small (a role set) |
| Global plus correction | `E + V·u` | Flat masks, allow-list-dominant |
| Cover plus exceptions (sum, count, mean) | `V·(\|cover\|·depth + \|exceptions\|)` | Masks are unions of subtrees |

The cover cost model applies to additive reductions. `min`, `max`, and attention cannot subtract an exception; they need a masked rescan inside the cover or a mask-aware tree, which this row does not price.

With `M ≈ V` and `a = E − u`, correction beats a direct masked scan iff `u < E(V−1)/(2V)`, which is `~E/2` for `V ≫ 1` — authorization is allow-list-dominant. Mask cardinality alone does not decide it; the derivation assumes uniform classes.

All three are expressible in SQL: a mask-class materialized view, a total plus per-consumer unauthorized view, or a cover join plus an exception union for additive reductions (`min`/`max`/attention need a rescan or a mask-aware tree). The engine’s “shared cost” claim separates it from recompute-per-query SQL, not from IVM SQL. If authorization is hierarchical (site → building → floor → room), as is common, it is the cover case and the most SQL-friendly; that is an assumption favouring SQL, not a measured fact. The arbitrary per-device mask is the regime where the engine has an edge, and it is the regime with no named consumer. The exactness caveat still holds: global-minus-unauthorized is not bit-exact to a direct masked reduction in f32; if bit-exact masked aggregation is required, use a direct per-class canonical pass or exact arithmetic.

### 8.7 Memory, operations, and build cost

With current-state Materialize, both sides hold `O(N·m)`: the engine in flat arrays, Materialize as a collection plus arrangements. The remaining gap is fixed database overhead plus arrangement and index state — not an order of magnitude (estimate; unmeasured). The structural memory difference appears only in the mask regime: an `M × E` join state (estimate: about 3M mask-edge pairs at `M = 100`, `E = 30k`) against `|cover| + |exceptions|` per consumer.

Operationally, the engine is one process; Materialize needs a cluster and object storage, plus a consensus store if self-managed. At S0 the engine is simpler to run; Materialize is more robust and better instrumented. On build cost, the engine’s aggregation layer sits inside the ~100 h shared substrate (planning), while Materialize’s rollup is a few views (assumption). The delivery-side D3/D4 build is out of scope here (DEC-001 §5.3).

### 8.8 Where each wins

- **Materialize:** time-to-contract, generality (including topology — no one-parent rule), the SQL surface, durability and recovery, temporal and historical queries when retention is enabled, operational maturity, and stronger isolation.
- **Engine:** f32 canonical parity; recompute determinism; flat, bounded memory; no retraction state; boot-time invariant enforcement; and tail latency, which is a hypothesis rather than a measurement. Delivery is excluded from this comparison (§9).
- **Tie:** 30k EPS throughput, the simple rollup, channel wiring, shared-snapshot consistency, and masked aggregation in structured regimes.

The one-parent rule is a legacy design assumption, not an engine advantage: it simplifies both sides, and it becomes a risk if the real topology is a DAG.

The residual is narrow, and it is narrower once delivery is excluded. Within the aggregation layer, the engine’s defensible claim is not “a faster rollup.” It is “a rollup that keeps the arithmetic contract.” If oracle parity is not required, Materialize is the stronger default; if f32 parity is required, the engine remains required on the arithmetic leg alone. The arbitrary-mask regime is the only other residue, and it has no named consumer. Delivery — cut, bound, per-viewer baseline, encoding — is a separate comparison with its own baselines and is not scored here (§9).

### 8.9 Other aggregation candidates

- **RisingWave:** the closest IVM alternative; PostgreSQL-compatible. Verify its recursive CTE status and its arithmetic in the current release; if it passes the contract, Materialize is not the only SQL baseline.
- **Flink (SQL and DataStream):** strongest for temporal windows, watermarks, and exactly-once. It has no recursive CTE, but the tree is static, so a precomputed closure table is sufficient; canonical order needs a custom operator or UDF. Heavier to operate.
- **Kafka Streams / ksqlDB:** an embedded library and a SQL layer over it. Custom topology and interactive queries; no general recursive rollup. Good when the service is already JVM-native.
- **ClickHouse:** fast OLAP and materialized views, but not per-update IVM with retractions. Best for scan-heavy aggregation, not per-consumer live rollups.
- **TimescaleDB:** continuous aggregates are time-windowed and not tree-aware. A history and rollup companion, not a competitor to the contract.
- **Twin platforms (Ditto, Azure Digital Twins, AWS TwinMaker):** twin semantics and aggregation APIs, not 30k exact per-consumer rollups. Ditto #1525 is the demand evidence, not the baseline.
- **Adjacent, not competitors:** Druid and Pinot (real-time OLAP), Spark Structured Streaming (micro-batch), Hazelcast and VoltDB (in-memory SQL). They change the absolute cost, not the contract.

The closest architectural cousin is a Differential/Timely dataflow; Materialize is its packaged form. A custom dataflow would face the same invertibility and arithmetic trade.

### 8.10 What to measure

This extends §10. Materialize-specific factors are the configuration (current-state versus history), availability (temporal filter versus none), mask representation (per-class, cover, per-consumer), `V = 10/100`, cadence 1/60 Hz, an assumed device mix of 1 s and 5 s reporting intervals with freshness 2 s and 10 s, and the oracle.

Run the contract suite first, as pass/fail. Then record p99, 24 h RSS, and build hours. A pass means the highlight is not real. A fail must name the failure mode — arithmetic, availability, mask cardinality, tail, or operations — not “slower.” Pre-register the configuration and the oracle, or the result will drift.

## 9. Delivery: separate baselines

Delivery is not compared with Materialize. It is a different layer with different baselines, and scoring it against an IVM database would be a category error. It remains part of this annex’s landscape; it is simply not scored in §8.

| Element | Engine | Delivery baselines |
| --- | --- | --- |
| Selection | Server-side geometric cut, hard entry bound | 3D Tiles/CesiumJS (`skipLevelOfDetail`), Potree, OSG/osgEarth `PagedLOD`, deck.gl |
| Per-consumer delivery | Per-viewer baseline, keyframe, encoding | Interest management (SpatialOS, Photon, Mirror, FishNet, Unity NGO) |
| Client memory | `O(B)` with a lazy manifest | Subtree-scoped views, client caches, tileset LOD |

The engine’s cut is not novel against these systems; DEC-001 §4–§5 is the comparison that prices it. A delivery comparison would test selection semantics, bound enforcement, per-viewer baseline and keyframe behaviour, encoding, client memory, and tail latency — none of which are Materialize capabilities. This annex keeps delivery visible as a separate layer, but it does not score it in §8.

## 10. Failing-baseline tests

The highlight is only real if the commodity stack fails. Build the baseline and test:

1. **Tree + channel wiring.** Single root, one parent per non-root node; fan-out/fan-in within the parent edge; same-unit/same-reduction; duplicate channel paths. Compare to the brute-force reference.
2. **Exactness.** Bit-exact f32 reduction against the oracle. Materialize’s fixed-point sum differs by a documented, bounded amount rather than by plan-dependent last bits; record the divergence against the named oracle (§8.4).
3. **Availability and no-value.** Per-metric freshness, stale/offline exclusion, no-value channels, attention/absence.
4. **Shared aggregation snapshot.** All viewers at the same frame timestamp at 60 Hz; this tests the aggregation contract, not the delivery layer.
5. **Mask-aware aggregation.** Arbitrary per-consumer masks; exact masked sum/count/mean/min/max. Measure the IVM mask-class view and the per-consumer pull query separately at V = 10 and V = 100; the IVM view is the fair baseline at 60 Hz/V = 100, while at 1 Hz/V = 10 the per-consumer pull is a legitimate baseline, not a strawman (§8.6).
6. **Cost.** 30k EPS, 540 channels, 1,000 contributions per channel, 60 Hz. Record latency, memory, and whether the shared pass is independent of V.

If the stack passes, the highlight is not real. If it fails, the failure is the contribution. Delivery is not tested here; it has its own baselines in §9.

## 11. Positioning

- **Do not position on “real-time streaming” or “hierarchical rollup.”** At 30k EPS those are commodity.
- **Position on the semantic contract:** exact, deterministic, availability-aware, attention-aware rollup over a tree with channel wiring, with a shared frame snapshot (the snapshot is a contract property, not a differentiator; §8.2).
- **Position on mask-aware aggregation** if the access requirement is real: the aggregate reflects only authorized contributions, and the cost is shared across consumers with the same mask. That sharing is also available as an IVM mask-class view, so the residual is exact f32 semantics and the arbitrary-mask regime (§8.6).
- **The cut or policy is a consumer** of the contract, not the whole engine. The contract is the durable asset.
- **Do not score delivery against Materialize.** The cut, bound, per-viewer baseline, and encoding are a separate comparison (§9); their baselines are 3D Tiles/Cesium, Potree, deck.gl, and interest management, not an IVM database.
- **Demand caveat.** Access control is a candidate direction (DISCUSSION-SUMMARY §127), not a sourced requirement; the specific demand for exact, masked, live rollups is unproven. The bitmask differentiator must be tested like the cut: name the consumer, the policy, and the failing baseline.

## 12. Summary

- At 30k EPS, hot state, simple rollup, and snapshot are commodity; the scale is not the moat.
- The engine’s defensible highlights are the tree-plus-channel-wiring semantics, the arithmetic contract (if oracle parity is required), and, conditionally, mask-aware aggregation. The shared frame snapshot is a contract property, not a differentiator: a shared MV plus `SUBSCRIBE` provides the same consistency (§8.2).
- SQL can express each part, but not the single shared rollup under arbitrary per-consumer masks; per-consumer pull queries scale with V, and IVM mask-class views scale with mask cardinality.
- Materialize configured to the engine’s contract (§8) matches the simple rollup, the channel wiring, and the shared snapshot; it cannot match f32 parity, and it expresses availability and masks at a cost. The engine’s residual is the arithmetic contract and, conditionally, the arbitrary-mask regime — not the rollup. Delivery is excluded from the §8 comparison; it is treated in §9.
- Delivery is not an engine advantage over Materialize: the cut competes with 3D Tiles/Cesium, Potree, deck.gl, and interest management, not with an IVM database.
- The one-parent rule is a legacy design assumption, not an engine advantage; Materialize is topology-agnostic, and the engine would pay in layout and aggregation if the domain were multi-parent.
- The bitmask does not rule out SQL; it rules out the shared global view. That is the gap to test.
- If the semantic contract is not required, existing frameworks are sufficient at 30k EPS.
