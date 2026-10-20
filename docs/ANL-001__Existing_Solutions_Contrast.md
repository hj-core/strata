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

The cut is out of scope; DEC-001 covers it. This annex concerns the pipeline that remains: hot state, hierarchical rollup, snapshot, and — as a candidate extension — bitmask access control.

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

**Topology invariant.** The node hierarchy is a **tree**: exactly one root; every other node has exactly one parent; there are no cycles; devices are leaves and each device attaches to exactly one group (PRD FR-TD-01, FR-TD-07; TDD §4.5). The channel wiring is layered over that tree: a channel contributes only to channels of its **unique parent group**, so the contribution graph is acyclic, but a channel may fan out to several parent channels and a parent channel may fan in from several child channels. **There are no multi-parent nodes and no cross-links.**

## 3. Highlight 1 — Exact rollup over a tree with channel wiring

The node hierarchy is a tree, but the engine is not a simple tree `GROUP BY`: its aggregation contract (TDD-001 §8) wires many-to-many channel contributions over that tree:

- **Fan-out and fan-in within the parent edge.** A device metric may feed several channels of its containing group; a group channel may feed several channels of its parent group; a parent channel may receive several child channels and local metric contributions. Every contribution goes to the unique parent group; there are no multi-parent nodes.
- **Same unit, same reduction.** Contributions combine only values of the same unit, and a channel may join only a parent channel of the same reduction (`sum`, `mean`, `min`, `max`, `count`); violations abort boot.
- **Exact mean and count.** `mean` is exact over the subtree’s available readings, carried as `(sum, count)`; a `count` channel is the total number of available contributing readings, merged from child `count` channels by summation.
- **Availability.** A metric is available only while its reading is finite and within its `freshness_ms`; stale and offline readings are excluded without affecting the device’s other metrics. A channel with no available contribution reports no value, not zero.
- **Attention and absence.** A reserved unitless channel carries the maximum severity from limits and per-metric absence levels; it propagates independently of the configured contributions.
- **Canonical order.** Bit-exact agreement with a brute-force reference requires a fixed blocked-lane f32 reduction order (`W = 8`); a parallel or plan-dependent order can differ in the last bits.
- **Shared frame snapshot.** Aggregation runs once per frame, double-buffered, independent of viewer count; all viewers of a frame see the same snapshot.
- **Budget.** ≤5 µs per channel per group with up to 1,000 contributing readings (K9); at S0, 540 channels across 180 groups.

### Why the commodity stack does not provide this out of the box

SQL can model a tree rollup with a recursive CTE or a closure table, and Materialize’s recursive CTEs and MV-on-MV patterns cover the simple case. It does not provide the conjunction:

- The node graph is a tree, but the channel wiring is many-to-many within the parent edge. A parent channel can receive several child channels and local metric contributions; multiple channel paths to the same parent channel can double-count unless the edge semantics are explicit. This is channel wiring, not multiple parents.
- `SUM(float4)`/`AVG(float4)` is not associative and is not guaranteed to match a canonical order under parallel or changing plans.
- Per-metric freshness is a predicate against a frame clock, not a static `NULL`.
- A no-value channel must be distinguishable from zero and from a stale value.
- Attention/absence is a custom max channel, not a generic aggregate.
- A shared frame timestamp across all viewers is a coordination layer, not a materialized view.

**Condition.** The highlight is real only if exact, deterministic, availability-aware rollup is required. If tolerant arithmetic and a simple tree are acceptable, SQL/materialized views are sufficient at 30k EPS.

## 4. Highlight 2 — Shared frame snapshot independent of viewers

The engine computes the rollup once per frame and publishes one snapshot; per-viewer work starts after the snapshot. The cost is independent of viewer count (FR-AG-03). This is what makes the same aggregate usable by the cut, by a policy layer, and by a dashboard without recomputation.

A generic SQL stack can give a consistent snapshot per query, but:

- A materialized view is continuously updated; two viewers querying at different times can see different versions.
- A transaction snapshot is consistent per query, not shared across viewers.
- Flink/Kafka Streams snapshots are per-partition; a global frame snapshot needs coordination.
- A per-viewer query at 60 Hz multiplies the aggregation cost by V.

**Condition.** The highlight matters when V is large, the cadence is 60 Hz, or all viewers must see the same frame. At 1 Hz and V = 10, per-viewer SQL snapshots are comfortable.

## 5. Highlight 3 — Mask-aware aggregation (bitmask access control)

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

So the bitmask does **not rule out SQL as a language**. It rules out the **single shared global rollup** for arbitrary per-consumer masks. The gap is the shared cost and the exact masked semantics, not expressibility.

### 5.4 The engine’s candidate advantage

The engine can apply the mask inside the rollup:

- Bitwise mask against the contribution lists; SIMD masked reduce.
- `sum`/`count`/`mean`: global aggregate plus unauthorized correction; the correction is proportional to the unauthorized set, not the full channel.
- `min`/`max`/attention: a mask-aware tree or a bounded scan, depending on mask structure.
- Per-mask equivalence classes: consumers with the same mask share one aggregate.
- Availability, no-value, and attention remain exact and consistent with the mask.

This is a **shared-cost masked rollup**, not a per-consumer query. It is the candidate highlight.

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

## 8. Failing-baseline tests

The highlight is only real if the commodity stack fails. Build the baseline and test:

1. **Tree + channel wiring.** Single root, one parent per non-root node; fan-out/fan-in within the parent edge; same-unit/same-reduction; duplicate channel paths. Compare to the brute-force reference.
2. **Exactness.** Bit-exact f32 reduction against the oracle. If SQL/Materialize differs in the last bits, record it.
3. **Availability and no-value.** Per-metric freshness, stale/offline exclusion, no-value channels, attention/absence.
4. **Shared frame snapshot.** All viewers at the same frame timestamp at 60 Hz.
5. **Mask-aware aggregation.** Arbitrary per-consumer masks; exact masked sum/count/mean/min/max; measure per-consumer SQL at V = 10 and V = 100 against the engine’s correction/tree.
6. **Cost.** 30k EPS, 540 channels, 1,000 contributions per channel, 60 Hz. Record latency, memory, and whether the shared pass is independent of V.

If the stack passes, the highlight is not real. If it fails, the failure is the contribution.

## 9. Positioning

- **Do not position on “real-time streaming” or “hierarchical rollup.”** At 30k EPS those are commodity.
- **Position on the semantic contract:** exact, deterministic, availability-aware, attention-aware rollup over a tree with channel wiring, with a shared frame snapshot.
- **Position on mask-aware aggregation** if the access requirement is real: the aggregate reflects only authorized contributions, and the cost is shared across consumers with the same mask. This is the candidate differentiator.
- **The cut or policy is a consumer** of the contract, not the whole engine. The contract is the durable asset.
- **Demand caveat.** Access control is a candidate direction (DISCUSSION-SUMMARY §127), not a sourced requirement; the specific demand for exact, masked, live rollups is unproven. The bitmask differentiator must be tested like the cut: name the consumer, the policy, and the failing baseline.

## 10. Summary

- At 30k EPS, hot state, simple rollup, and snapshot are commodity; the scale is not the moat.
- The engine’s highlights are the tree-plus-channel-wiring semantics, the shared frame snapshot, and mask-aware aggregation.
- SQL can express each part, but not the single shared rollup under arbitrary per-consumer masks; per-consumer SQL scales with V, and per-mask views scale with mask cardinality.
- The bitmask does not rule out SQL; it rules out the shared global view. That is the gap to test.
- If the semantic contract is not required, existing frameworks are sufficient at 30k EPS.
