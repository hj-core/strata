# TDD-001 — Technical Design Document

| Field | Value |
| --- | --- |
| Document ID | TDD-001 |
| Title | Server-Side Hierarchical Spatial Filtering Engine — Technical Design |
| Version | 0.3.1 |
| Status | Draft |
| Date | 2026-09-20 |
| Owner | Project maintainer |
| Related documents | Upstream: PRD-001 (Product Requirements). Downstream: PLN-001 (Master Execution Sequence). |

## 1. Introduction and Scope

This document describes **how** the filtering engine is built: architecture, data structures, algorithms, transport semantics, client design, and evaluation methodology. It elaborates the settled design decisions and satisfies the requirements in PRD-001, its only upstream dependency. References to execution activities (for example `A7`, `A14`, `M1`) are PLN-001 identifiers.

Guiding rule: **the browser draws; the server thinks.** All heavy work is performed once on the server and shared across viewers; each viewer receives a small, relevant slice per frame.

## 2. Design Goals and Constraints

| Goal | Design consequence |
| --- | --- |
| Sustain ~300k EPS (K1) | Lock-free latest-wins state table; non-blocking ingest path. |
| p99 query ≤ 1.5 ms at 100k (K2) | Constrained parent-first index; frustum pruning; heap-based best-first refinement. |
| Bounded visible workload (K3) | Per-frame entry budget (~5,000) enforced as a frontier cut size. |
| Exact aggregates (K4) | Per-group local channels; same-reduction contributions; accumulator merge (`mean → (sum, count)`); brute-force scalar reference for tests. |
| ≥ 99% egress cut (K5) | f16 quantisation + per-value ε filter (metric/channel) + change suppression. |
| Boot ≤ 30 s at 100k (K8) | Sequential single-pass build; the budget is dominated by config parse and validation (Decision 12). |
| Same engine at 4,280 and 100,000 (G5) | Scale is a property of the data, not the code. |

## 3. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                            SERVER (Rust)                             │
│                                                                      │
│  Telemetry ingest ──► Atomic state table (latest-wins, lock-free)    │
│                              │                                       │
│  Topology config ──► Hierarchical spatial index (flat, parent-first) │
│                              │                                       │
│         ┌────────────────────┴────────────────────┐                  │
│         ▼                                         ▼                  │
│  Shared aggregation sweep (60 Hz, double-buffered)  Visibility query │
│         │                                         │                  │
│         └────────────────┬────────────────────────┘                  │
│                          ▼                                           │
│              Per-viewer selection + encode (f16 ε-filter)            │
│                          │                                           │
│                   WebSocket binary frames                            │
└──────────────────────────┼───────────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     BROWSER CLIENT (Three.js)                        │
│  InstancedMesh renderer · VBO writer · camera publisher · picking    │
└──────────────────────────────────────────────────────────────────────┘

Supporting: Simulator (Rust binary) → ingest; Demo app (Vite/TS) → client + engine.
```

### 3.1 Component Responsibilities

| Component | Responsibility | PRD coverage |
| --- | --- | --- |
| Simulator | Generate topology-driven telemetry at configurable rates. | FR-CD-07 |
| Ingestor | Accept readings, apply latest-wins to the state table. | FR-IS-01, FR-IS-02 |
| State table | Hold newest value per device; support boot-time layout. | FR-IS-01 |
| Index builder | Build the hierarchical spatial index and validate invariants. | FR-TD-07, FR-TD-08 |
| Aggregator | Compute each group's local channels and its reserved attention channel (and each device's attention level) once per frame from its devices' readings and its child groups' contributions; double-buffered. | FR-TD-05, FR-TD-06, FR-TD-11, FR-AG-01–FR-AG-06 |
| Visibility engine | Frustum-cull and best-first refine the visible hierarchy into a bounded entry cut. | FR-VS-03–FR-VS-08 |
| Encoder | Quantise, ε-filter, and suppress unchanged values per viewer. | FR-TR-02 |
| Transport | WebSocket server + binary framing + sessions. | FR-TR-01, FR-TR-03, FR-TR-04 |
| Client SDK | Render instances, publish camera, pick objects, select display value. | FR-CD-01–FR-CD-06 |
| Demo app | Wire simulator → engine → browser at the building scale (~4,280 devices). | FR-CD-08, FR-CD-09 |
| Benchmark harness | Measure and report KPIs. | FR-BR-01–FR-BR-04 |

## 4. Data Model

### 4.1 Topology Schema

The topology is a single JSON document (FR-TD-09): a **flat list of nodes**, each referencing its parent by id. List order defines sibling order (Decision 8), and ids are unique across the topology.

```json
{
  "version": 1,
  "group_types": {
    "rack_a": {
      "channels": [
        { "id": "avg_temp", "unit": "C", "reduction": "mean", "epsilon": 0.2,
          "contributes_to": ["mean_temp"],
          "limits": [ { "threshold": 35, "side": "high", "level": "warning" } ] }
      ]
    }
  },
  "nodes": [
    { "kind": "group",  "id": "site",   "parent": null,     "channels": [] },
    { "kind": "group",  "id": "bldg-a", "parent": "site",   "channels": [ { "id": "mean_temp", "unit": "C", "reduction": "mean", "epsilon": 0.1 } ] },
    { "kind": "group",  "id": "rack-1", "parent": "bldg-a", "type": "rack_a" },
    { "kind": "device", "id": "dev-1",  "parent": "rack-1", "position": [12.0, 3.5, 8.0],
      "metrics": [
        { "label": "cpu_temp_1", "unit": "C", "epsilon": 0.1, "freshness_ms": 5000,
          "contributes_to": ["avg_temp"],
          "limits": [ { "threshold": 80, "side": "high", "level": "critical" },
                      { "threshold": 5,  "side": "low",  "level": "advisory" } ],
          "absence": "advisory" },
        { "label": "cpu_vib", "unit": "mm/s", "epsilon": 0.05, "freshness_ms": 2000 }
      ] }
  ]
}
```

**Node fields.**

| Field | Required | Notes |
| --- | --- | --- |
| `kind` | yes | `group` or `device` |
| `id` | yes | unique topology-wide (referenced by `parent`) |
| `parent` | yes | a group id, or `null` for the single root |
| `channels` | group | ≤16 local channels, in addition to the reserved attention channel |
| `type` | group, no | shallow template reference; a declared `channels` replaces the template's |
| `position` | device | `[x, y, z]`, finite |
| `metrics` | device | one or more |

**Metric fields.**

| Field | Required | Notes |
| --- | --- | --- |
| `label` | yes | unique within the device (Decision 7) |
| `unit` | yes | opaque string, compared by exact match |
| `epsilon` | yes | absolute noise threshold, in the metric's unit |
| `freshness_ms` | yes | staleness timeout (FR-IS-03) |
| `contributes_to` | no (default `[]`) | containing group's local channel ids |
| `limits` | no (default `[]`) | ordered `{threshold, side, level}` |
| `absence` | no (default `normal`) | severity while unavailable |

**Channel fields.**

| Field | Required | Notes |
| --- | --- | --- |
| `id` | yes | unique within the group |
| `reduction` | yes | `sum`, `mean`, `min`, `max`, `count` |
| `unit` | yes | contributors must match (FR-AG-02); for `count`, the unit of the readings tallied |
| `epsilon` | yes | noise threshold |
| `contributes_to` | no (default `[]`) | parent group's channel ids; may name several |
| `limits` | no (default `[]`) | ordered `{threshold, side, level}` |

Configuration requirements:
- **Single-rooted.** Exactly one node has `parent: null`; every other `parent` resolves; there are no cycles; every device is reachable from the root; ids are unique (FR-TD-07).
- Every device has a fixed, finite position and one or more metrics, each with a **device-local `label`** unique within the device; labels are not a shared vocabulary (FR-TD-03/04, Decision 7).
- A group defines at most 16 channels, in addition to its reserved attention channel; a 17th is a boot error (FR-TD-05).
- A metric contributes to channels of its containing group; a channel contributes to channels of its immediate parent. Both default to none. Fan-out and fan-in are allowed, and a channel may feed several parent channels (FR-TD-04/06, Decision 9).
- Contributions combine only same-unit values, and a channel joins only a parent channel of the same reduction; violations are boot errors (FR-AG-02, FR-TD-06/08).
- `mean` is exact over the subtree's available readings (accumulator `(sum, count)`), and `count` is the total number of available contributing readings in the subtree, merged from child `count` channels by summation (FR-AG-01).
- Severity uses the fixed named scale `normal`, `advisory`, `warning`, `critical`. A limit fires on `value >= threshold` (`high`) or `value <= threshold` (`low`) and inherits the value's unit; a value's level is the maximum over its fired limits; a metric's `absence` level applies while it is unavailable (FR-TD-11).
- Templates are shallow and single-level: a node may reference one `type`; a declared templated field replaces the template's. There is no type-extends-type and no per-channel merge (Decision 9).
- The conventional site → building → room → rack levels are not required; the engine models a generic group tree (FR-TD-01). A physically meaningful hierarchy is assumed for useful level-of-detail; a degenerate or flat hierarchy is accepted and collapses to a single blended entry when zoomed out (FR-VS-05, §8).
- The same hierarchy drives both spatial lookup and aggregation; there is one structure, not two.

### 4.2 Reading Model

- Each reading is `(device_id, metric_label, value)`; at ingest, the device-local `metric_label` resolves to the per-device `metric_slot` bound at boot, and the value is applied to the state table.
- Exactly one current value is retained per `(device, metric)`; concurrent or duplicate values of the same metric are not modelled, and no history is stored.
- **Latest-wins**: an incoming reading replaces the newest state; network jitter degrades update frequency, never correctness.
- A device's metrics may become **unavailable** individually when stale (freshness timeout); this is represented explicitly and encoded as a NaN marker on the wire (FR-IS-03).

#### Simulator Output Contract

The simulator is a device-level reading producer. It emits one ingest record per device/metric sample and nothing derived from the topology:

| Field | Required | Notes |
| --- | --- | --- |
| `device_index` | Yes | Boot-assigned device ordinal (leaf order), bound from the device id in the shared config. |
| `metric_slot` | Yes | Per-device metric index (slot), bound from a `(device, metric_label)` pair at boot using the shared config schema; labels are device-local, so a slot is meaningful only within its device. No string keys at runtime. |
| `value` | Yes | Raw scalar at source precision; quantisation to f16 happens in transport (§9.2), not here. |
| `sample_timestamp` | No | Ignored by the state table (latest-wins); retained for debugging and rate accounting. |

Fields derived from the topology — position, unit, epsilon, group ids — are not emitted; the simulator, server, and client read the same topology definition, so device indices, per-device metric slots, and positions align.

The simulator emits this record in the engine's ingest format directly. There is no runtime middleware; the only translation is the boot-time, per-device `metric_label` → `metric_slot` binding (and the device id → `device_index` binding), performed identically by both sides. The ingest record is a compact little-endian struct: `u32 device_index, u16 metric_slot, f32 value` (10 bytes), optionally followed by a `u64 sample_timestamp`.

The generator must support:

- Configurable global EPS (up to 300,000) or per-device reporting interval.
- Per-metric value models for temperature, vibration, pressure, and power.
- Within-noise jitter (exercises the ε filter), steps/ramps, and short spikes (exercise change suppression and short-lived-event visibility).
- Spatial correlation across neighbouring racks/rooms for realistic LOD blending.
- Offline transitions and dropped/late samples.
- Deterministic seeding for reproducible benchmark traces (R6).

At full scale the simulator emits a compact binary ingest record over a non-blocking channel (or runs in-process); it must not serialise to text at 300k EPS. The JSON representation in PRD §7.1 is the K5 measurement baseline, not the ingest format.

### 4.3 Node and Group Model

The index is a set of nodes laid out in **one flat, parent-first array**. Each node is a group (at any level) or a leaf (device). A leaf stores an **offset into the separate state table**, never its reading (see §4.4). Each group's members occupy **one contiguous subtree**; a group's devices occupy a **contiguous block of state rows** (the contiguous-block invariant, §4.4).

```
Flat array (parent-first):
[ Site ][ Building A ... ][ Room A1 ... ][ Rack A1a: dev,dev,... ][ ... ]
```

Three properties are mandatory (R1, R3):
1. **Contiguous groups** — a group's subtree is a single contiguous range.
2. **Parent-first layout** — a parent always precedes its descendants.
3. **Skip-offset pruning** — each internal node stores the size of its subtree, enabling a whole group to be skipped or emitted without descending.

### 4.4 Memory Layout

The engine separates three arrays by access pattern:

| Array | Contents | Written | Read |
| --- | --- | --- | --- |
| Index | Node kind, AABB min/max, child access + subtree size, channel definitions, contribution sources, leaf state offset | Once at boot | Every frame, per viewer |
| State | Latest value and availability per `(device, metric)` | Continuously, at ingest | Once per frame, per group |
| Aggregates | Per-group per-channel accumulators (including the reserved attention channel); per-device attention level | Once per frame | Per viewer, per frame |

- The **index** is a flat, parent-first, preorder array in structure-of-arrays form to keep traversal cache-friendly; the array position is the node's index and identity, so it is not stored redundantly. Nodes reference state by offset and range, never by value. The per-node field set is listed at the end of this section.
- The **state table** is separate from the index and laid out **device-major in CSR form** — a compact `values` array plus a per-device `row_offsets` array — with devices in index leaf order and each device's declared metrics contiguous within its row in local-slot (declaration) order. Labels are device-local (§4.1), so a slot is meaningful only within its device and there is no global metric column. A device's row is `values[row_offsets[d] .. row_offsets[d+1]]`; a group's devices occupy a contiguous range of rows, so their values occupy one contiguous block. At boot the compiler records, per channel, the sorted absolute offsets of its contributing metric instances, so the aggregation pass walks a bounded, prefetch-friendly gather inside that block (FR-AG-02, K9); a metric feeding several channels appears in several such lists. A device's whole value set is contiguous, which suits the per-entry encode path (§9.3). The layout is fixed by Decision 7.
- Rows are exact: a device's compact row holds only the metrics it declares, so there are no unused slots and the table is `Σ metrics/device` values plus `N+1` `u32` row offsets (~400 KB of offsets at 100k). At ~3 metrics/device the full-park table is ~300,000 values, so the table is small.
- **Channel accumulators** are stored per group for its local channels and double-buffered: the frame reads one buffer while the sweep writes the other, then swaps (§7).

**Index node shape.** The array position is the node's index and identity. Per node:

| Field | Read by | Purpose |
| --- | --- | --- |
| Kind (leaf device / internal group) | Visibility | Finalise a leaf as a device instance and a positioned group as a group entry; subtree size alone cannot distinguish a device from a group (FR-TD-07). Childless groups are excluded from the index (§5.2, §8.2). |
| AABB min/max | Visibility | Frustum test and projection; the only per-node geometry visibility needs. |
| Subtree size (node count) | Visibility | Subtree bounds and skip-offset pruning; `skip_index = index + subtree_size` is derived, not stored. Node count only — no device-count extent is needed (aggregation uses per-channel offset lists, Decision 7). |
| Child access | Visibility | `child_count` gives `deg(v)` in O(1) for the expansion-budget check; `first_child = index + 1` and `next_sibling = sibling + subtree_size` are derived, not stored (Decision 8). |
| Channel definitions | Aggregation, client | A slice `channel_def[channel_base .. +channel_count]` of the global channel-def array: per channel, unit, reduction, ε, and a range into the global limit array. ≤16 per group, fixed at boot. |
| Contribution sources | Aggregation | Per channel, a slice of the global contribution array: sources tagged `seed` (state offset) or `merge` (contributing channel-def index), indexed by target channel (pull, Decision 10). |
| Leaf state offset | Encoder | Start offset of the device's row in the compact state array. |

**Channel, limit, and contribution arrays.** Variable-length per-node data lives in global arrays addressed by ranges rather than inlined. A global **channel-def array** holds one entry per configured channel (`unit`, `reduction`, `epsilon`, `limit_base`, `limit_count`, `contrib_base`, `contrib_count`); a group's channels are contiguous via its `channel_base`/`channel_count`. A global **limit array** holds `{threshold, side, level}`. A global **contribution array** holds one entry per source, tagged `seed` (source = state offset) or `merge` (source = contributing channel-def index); a channel's sources are contiguous via its `contrib_base`/`contrib_count`.

The aggregation plan *is* this contribution array, indexed by **target channel** (pull, Decision 10): the reverse scan reduces each channel's source list — seeds in device-leaf order, then merges in child order — so the reduction is a single vectorisable list and its order is explicit for exact `sum`/`mean` (Decision 14). The reserved attention channel is a per-group accumulator slot outside the channel-def array and is never a contribution target.

### 4.5 Engine Configuration

Parameters that are not part of the topology (PRD §8) live in an engine configuration file:

| Parameter | Default | Requirement |
| --- | --- | --- |
| On-screen-size enter threshold | 120 px | FR-VS-06 |
| On-screen-size exit threshold | 100 px | FR-VS-06 |
| Budget hysteresis `Δ` | 10% of `B` | FR-VS-07, §8.3 |
| Camera-context timeout | 1 s | FR-VS-08 |

These are engine-level settings and do not affect node or metric identities; the topology file (§4.1) remains the single shared source of identities (FR-TD-09).

## 5. Hierarchical Spatial Index

### 5.1 Rationale

A straightforward index of device positions cannot answer *both* "what is on screen?" and "what are this group's aggregate channel values?" within budget. The index therefore mirrors the semantic hierarchy and carries bounding volumes, so one traversal yields visibility and aggregation together.

### 5.2 Build Algorithm

The topology (nodes and parent-child edges) is fixed by the config; the build **derives** the flat index from it and never invents, reparents, rebalances, or spatially reorders the tree (Decision 1, §4.1). A pre-pass resolves templates, buckets each node under its parent in list order, and validates the tree (single root, acyclic, devices reachable); one DFS in that canonical order then produces the layout and all derived per-node fields:

1. **Preorder (on entry):** assign `id = position`, emit the node into the flat array, record its kind (leaf device / internal group) and its channel definitions, and record a leaf's state offset. Recurse into children in canonical (list) order (Decision 8).
2. **Postorder (on exit):** set a group's AABB to the union of its children's AABBs (a leaf's AABB is its device position); set `subtree_size = nodes emitted in this subtree` (node count) and `child_count` = emitted direct children. This also yields the parent-first flattening and the contiguous-subtree invariant directly.
3. Record the **leaf order** (devices in DFS visit order) — the order used to lay out the state table's device dimension (§4.4) — and each group's **contribution sources** (per local channel, the metric instances and child channels that feed it, validated for same unit and same reduction), including each channel's sorted list of contributing state offsets for the aggregation pass (§4.4, §7).
4. Bind each device's metric labels to per-device slots (`metric_label → metric_slot`) and validate the invariants below.
5. Run a **cheap post-build invariant scan** (contiguity, parent-first, skip-offsets, contribution integrity).

A single pass therefore yields ids, layout, AABBs, subtree sizes, child counts, leaf order, channel definitions, contribution sources, and state offsets. Because a parent's AABB is the union of its children, it is **order-invariant**; ordering siblings spatially cannot tighten any bound or improve pruning, so no spatial construction (median split, SAH, or BVH build) is performed. Pruning quality is inherited from the config's containment structure.

A leaf's AABB is the device position (a degenerate point); frustum tests are inclusive and the brute-force reference uses the identical point test (Decision 12). Childless groups are excluded from the index (§8.2).

### 5.3 Invariants and Validation

| Invariant | Check | Failure response |
| --- | --- | --- |
| Contiguous subtrees | Offset/size scan | Abort boot with diagnostics |
| Parent-first ordering | Index ordering scan | Abort boot |
| Skip-offset correctness | Recompute vs stored size | Abort boot |
| Child enumeration consistency | `child_count` equals emitted direct children; `first_child = index + 1` | Abort boot |
| Per-device state-row integrity | Row-offset monotonicity; row `d` aligned to leaf-order device `d`; total = Σ metrics/device | Abort boot |
| Contribution unit and reduction consistency | Each contributor's declared unit matches the channel's unit; each contributing channel's reduction matches the receiving channel's | Abort boot |
| Contribution target existence | Every metric's `contributes_to` names a channel in its containing group; every channel's names a channel in its parent | Abort boot |
| Finite positions | Every device position is finite | Abort boot |
| Group channel-count bound | Each group's configured channels ≤ 16 | Abort boot |
| Aggregate correctness | Brute-force scalar comparison on fixtures | Fail test |

Unit and reduction scans apply to configured channels and their contributions; the reserved attention channel is excluded (it combines only unitless levels).

**Validation order and diagnostics.** Checks run on the resolved config (§4.1) in stages: (1) structural (FR-TD-07), (2) device/metric (FR-TD-04), (3) channels (FR-TD-05), (4) contributions (FR-TD-06/08), (5) limits/absence (FR-TD-11/08), then (6) the post-build invariants above. Validation is **fail-fast**: the first fault aborts boot with a structured diagnostic — `{code, node id, metric label / channel id, message}` — precise enough to locate the fault (FR-TD-07, Decision 12). Codes are stable and prefixed by stage (`E-STRUCT`, `E-METRIC`, `E-CHANNEL`, `E-CONTRIB`, `E-LIMIT`, `E-INVARIANT`).

### 5.4 Correctness Oracle

Correctness is established against an **independent brute-force reference**, not a third-party spatial structure. The reference operates on the **flat device list directly** — it does not walk the index tree, so it does not share any traversal or pruning defect it is meant to catch. For the same `(topology, state, camera, selection)` it frustum-tests every device position (using the same inclusive point test as the engine, Decision 12), computes the ranked cut, and reduces the same readings; the engine's traversal must agree on the visible device-leaf set and its selection must match the reference cut. This mirrors the brute-force scalar reference used for aggregation (§7, §13), giving one independent reference per subsystem (R1).

The semantic n-ary tree has no shared basis with a binary spatial BVH, so the `bvh` crate is not used. Structural invariants (§5.3) cover the layout; the reference covers traversal and selection.

### 5.5 Traversal

- Descend the tree with frustum tests on AABBs.
- **Prune** a subtree when it is outside the frustum or when the group projects below the on-screen-size threshold (skip-offset pruning).
- **Emit** a group's aggregate channel values when it is visible but too small to be shown as individual devices.
- **Expand** to individual devices when the group is large enough on screen and budget remains.
- Keep a **scalar reference path** always available; any optimised path ships behind the same interface and behind the SIMD feature flag (R2, Decision 14). SIMD is applied to aggregation and encoding (§7, §9.2), not to the irregular, early-out-dominated traversal.

## 6. State Management and Ingestion

- A **lock-free, latest-wins state table** indexed by `(device, metric)`, holding exactly one current value per entry. Writers (ingest) store values atomically; readers (aggregation/encode) load the newest value.
- The ingest path is **non-blocking** and does not queue unboundedly; back-pressure is handled by overwriting stale values rather than buffering.
- Readings for an unknown device or metric label, or with non-finite values, are rejected without altering existing state (FR-IS-02).
- Availability is tracked per `(device, metric)`: a metric is current while a reading for it arrived within its configured freshness timeout, and its value is treated as unavailable once stale; a stale metric does not affect the availability of the device's other metrics. A device is marked **offline** when all of its metrics are unavailable, and returns online on the next accepted reading (FR-IS-03, FR-IS-04).
- Each device's metrics are bound at boot to the **local channels of their containing group**. Aggregation reads the state slots of the metrics feeding each channel; a channel combines only same-unit contributions, and a contribution may join only a channel of the same reduction (FR-TD-05, FR-AG-02, FR-TD-06).
- The state table is a dedicated array, separate from the index (leaf nodes store offsets, not values). Its device dimension matches the index leaf order and each device's metrics are contiguous within its row, so each group maps to one contiguous block of rows and a device's value set is contiguous (K9, §9.3, §4.4). Aggregation reads each channel's boot-recorded contributing offsets within that block.

## 7. Shared Aggregation

- **One background pass per 60 Hz frame** computes every group's local channels bottom-up from a **boot-compiled plan** — seed ops (device metric → local channel) and merge ops (child channel → parent channel) — recomputed from scratch each frame. The pass runs continuously while serving, independent of viewer count (idle-gating deferred; Decision 5). The plan is indexed by **target channel** (pull): each channel stores its incoming sources, and the reverse scan reads children's finalised values from the write buffer (Decision 10).
- Each group defines up to **16 local channels** (identity local to the group; reductions `sum`, `mean`, `min`, `max`, `count`). A channel's value is its configured reduction applied to the **available contributions** wired to it: the group's own metric instances and its child groups' contributed channels. A group contributes nothing to its parent by default (FR-TD-05, FR-TD-06).
- **Same-reduction rule.** A channel may contribute only to a parent channel of the same reduction; boot validation rejects mixed-reduction contributions (FR-TD-06, FR-TD-08). With same-reduction contributions and accumulator merge, a group's value is the canonical subtree reduction, so K4's exact reference is well defined (FR-AG-01).
- Each channel carries mergeable accumulator state: `mean` → `(sum, count)`, `sum` → `sum`, `min`/`max` → value, `count` → `count`. A group merges its children's channel accumulators; the published value is the finalised accumulator. `mean` is therefore exact over the subtree's available readings, and `count` is the total number of available contributing readings in the subtree, merged from child `count` channels by summation (FR-AG-01, K4).
- A channel combines one unit only (FR-AG-02), and unavailable (stale or offline) readings are excluded from the accumulation; a channel with no available contribution reports no value (FR-AG-04).
- Results are published as a consistent per-frame snapshot (double-buffered internally); viewers read the published buffer (FR-AG-03).
- Cost model: the shared pass is independent of viewer count; the per-viewer cost beyond it is **O(visible)**, not O(N) (R5, FR-AG-03).
- Aggregates must match a brute-force reduction of the same contributions exactly (FR-AG-01, K4). Because floating-point addition is not associative, exact agreement requires a **canonical reduction order** shared by the scalar reference and any SIMD path: a **blocked-lane order with fixed `W = 8`** — source `i` accumulates into lane `i mod 8`, and the eight lanes fold in a fixed order (Decision 14). The scalar engine and the brute-force reference implement it identically, so they are bit-exact; a SIMD kernel reproduces the same lane pattern. `min`, `max` and `count` are order-free. A channel's reduction over up to 1,000 children targets ≤ 5 µs scalar (K9); the SIMD path aims for ≤ 1 µs as an engineering goal.
- Contingency: reduce the configured channel set if the shared pass cannot meet K9; per-viewer cost must stay independent of viewer count (FR-AG-03).

### 7.1 Attention channel (severity and absence)

Each group carries a **reserved unitless attention channel** (the group's attention value, reduction `max`), computed in the same bottom-up pass as its configured channels and streamed with the node's entry (FR-TD-11, FR-AG-06). Its value is on the fixed severity scale of normal, advisory, warning, and critical. A device's attention level is the greatest of its metrics' contributions. The scale is the fixed named scale of Decision 9 (FR-TD-11).

- **Severity limits.** A metric or channel may declare an ordered list of limits, each `(threshold, side, level)` on the high and/or low side. A value maps to a level by `level = max over limits of ( fired ? level : 0 )`, where `fired` is `value >= threshold` (high) or `value <= threshold` (low). Limits inherit the value's unit and are validated at boot (finite; level on the fixed severity scale).
- **Contributions.** A metric contributes its limit level when available, or its absence level when unavailable (per-metric; availability from FR-IS-03). A configured channel contributes its limit level when it has a value.
- **Composition.** A device's attention level is the max over its metrics' contributions. A group's attention channel is the max over its configured channels' limit levels, its direct devices' attention levels, and its child groups' attention channels. Unavailable values contribute no level; a node with no available values and no nonzero contribution reports normal (0).
- **Propagation.** The attention channel merges as a `max` channel, order-free and exact, and is always contributed to its parent's attention channel, independent of the configured contributions (FR-TD-06, FR-AG-06).
- **Availability stays orthogonal.** NaN availability markers are streamed separately, so the client can distinguish an out-of-limit value from missing data.
- **Cost and storage.** A handful of compare-and-selects per value, folded into the existing seed/merge pass; one small unitless channel per group, double-buffered with the aggregates.
- **Scope.** Fixed bands only — no expressions, rate-of-change, hysteresis, or state. Fraction-based coverage policy is out of scope (PRD §10).

## 8. Visibility Query and LOD Selection

### 8.1 Inputs

Per viewer, per frame: camera pose (position and orientation), screen width/height, pixel ratio, field of view (FR-VS-01). Without these the on-screen-size threshold is ambiguous. If the context is absent or late, the last known context is reused until a timeout, after which that viewer's stream is held (FR-VS-08).

### 8.2 Algorithm

The query produces a per-viewer, per-frame **visibility result**: a bounded list of *entries*, each either a device instance or a blended group entry. It selects a **cut** through the visible hierarchy — the set of nodes that each become one entry — under the entry budget (R5, K3, FR-VS-07). Values are not resolved here; the encoder reads them from the state table and aggregate buffers (§9).

The frontier (the current cut) is held in a **max-heap keyed on exact projected on-screen height**, not in a traversal stack, so selection is best-first: the largest visible node is expanded first, regardless of its depth or position in the array. The heap is a **fixed-capacity, per-viewer scratch buffer**, reused every frame and sized to the entry budget `B` (the frontier can never exceed `B`). The index (node ids, layout, AABBs) is immutable after boot (§4.4), so the heap carries only node references and per-frame heights; each node enters the frontier at most once per query.

**Projected on-screen height** is the vertical pixel extent of the node's AABB: its eight corners are transformed to screen space at the AABB centre depth, and the key is `max_y − min_y` in device pixels (pixel ratio applied). The engine and the brute-force reference compute the identical value, so the frontier's total order is reproducible (§5.4); the monotonicity of projected height down the tree relies on this centre-depth convention.

1. Project the root; if it intersects the frustum, push it onto the frontier with its projected height.
2. Repeat while an expandable candidate remains:
   - Pop the node with the **greatest projected height**. Exact height is the primary key; node id is the terminal tie-break, giving a total, deterministic order.
   - Finalise the node as an entry when any terminal condition holds: a device leaf becomes a device instance entry; a positioned group whose contributors are all currently unavailable becomes a group entry carrying no value (FR-AG-04); a height below the applicable size threshold becomes a blended group entry (enter 120 px, or exit 100 px once expanded; §8.3); and a node whose expansion would breach the budget becomes a blended group entry. Childless groups never enter the index and are never emitted (§5.2).
   - Otherwise **expand** it: remove it; for each direct child, apply the frustum test and push only the visible children with their projected heights. Expansion is permitted only while `entries − 1 + deg(v) ≤ B` (`B` ≈ 5,000), and adds `deg(v) − 1` to `entries`, the running cut size (`finalised entries + frontier nodes`). `deg(v)` is O(1) from the node's `child_count` (§4.4), so a huge-fan-in node is rejected without enumerating its children.
3. Halt when no expandable candidate remains; every node still held is finalised (leaves as device instances, internal nodes as summaries). The output is the union of finalised nodes.

A child's AABB is contained by its parent's, so projected height is **monotone** down the tree: a child is never taller on screen than its parent. The frontier therefore always holds an upper bound for every unexplored subtree, the maximum-height pop cannot miss the largest remaining node, and no node is revisited.

Each entry carries its kind, its identity — the flat-array node index, resolved to a config identity through the connect-time dictionary (§9.3, Decision 11) — and a **transition flag** (appeared, removed, or expanded) relative to the viewer's previous frame; `expanded` is internal to visibility and surfaces on the wire as appeared entries for the newly revealed devices (§8.3, §9.4). Values are not resolved by the query: the encoder maps each index to its source at encode time (§9.3). The result is deterministic given `(topology, state, camera, selection)`, with `entries ≤ B`; the per-viewer frontier is bounded by `B`. The viewer's selected object (FR-VS-02) is force-included every frame, adding at most one entry beyond the cut.

### 8.3 Anti-Flicker

**Size hysteresis:** enter detailed view at 120 px, exit at 100 px (FR-VS-06, R7).

**Budget hysteresis:** the entry budget cuts the frontier independently of on-screen size, so a group near the budget edge can flip between blended and detailed as unrelated scene contents change — frontier ordering cannot absorb this. The running cut size therefore carries its own band: expansion is admitted while `entries < B`, but a node already expanded is retained until the cut falls below `B − Δ` (`Δ` defaults to 10% of `B`, §4.5), so the budget edge does not oscillate. The size and budget bands are independent and configurable.

A group's `expanded` transition in the visibility result (§8.2) causes its newly revealed devices to be transmitted in full (FR-TR-02), covering transitions the hysteresis bands do not absorb.

### 8.4 Latency Strategy

- Explicitly do **not** rely on raw traversal of 100k nodes (≈ 3 ms cache-hot best case, exceeds budget).
- Cost is bounded by the work actually performed: at most `B` expansions, each an `O(log B)` heap push/pop plus a projection, so total work is O(expansions · log B), not O(visible). At `B` ≈ 5,000 the log factor is small (~13 comparisons) and the ordering cost is a minor fraction of the traversal cost.
- Frustum testing dominates the per-expansion cost in aggregate: every direct child is tested while only visible children are pushed, and culled children typically reject on the first plane. Test the frustum before projecting so culled children skip the projection.
- Skip-offset pruning removes whole off-frustum subtrees before they enter the frontier.
- The per-viewer frontier is bounded by the cut size `≤ B` and held in a fixed-capacity scratch heap reused every frame.
- Contingencies: coarse grid first pass; skip re-query for an idle camera; iterate the top ~50 groups directly.

## 9. Viewer Transport and Wire Protocol

### 9.1 Transport

WebSocket carrying a compact **binary** frame format. Sessions are stateful per viewer.

### 9.2 Encoding Pipeline

```
scalar value ──► f16 quantise ──► per-value ε filter ──► emit changed slots (absolute f16) ──► frame
                                        │
                                  raw-threshold guard (~>32 °C)
```

- Values encoded as **16-bit floats** on the wire.
- Every visible entry defines its **complete value set** — a device instance's metrics or a group entry's local channels — so the client can resolve and change the active display value without a server round-trip (FR-CD-03). Transmission follows FR-TR-02 (the full value set on first send or snapshot, changes thereafter), and the ε filter is applied per value.
- Each entry also carries the node's **attention value** (FR-AG-06) — for a group entry, its reserved attention channel; for a device instance, its attention level — sent on change; availability remains a separate axis.
- Updates computed in the **quantised wire domain** with absolute thresholds — a metric's ε for a device instance, a channel's ε for a group entry.
- The wire carries the **absolute f16 value** for each changed slot; the baseline (the last value actually sent, in f16 wire form) is only the ε-suppression reference. It advances **only on an actual socket write**, so slow drift accumulates until it crosses ε while the rendered value stays within its threshold of the current value (FR-TR-02, Decision 15). A dropped (latest-wins) frame does not advance the baseline.
- A **raw-threshold check** guards readings where 16-bit rounding alone would under-filter (above ~32 °C).
- The quantise → ε → change-suppression pipeline is contiguous and vectorisable (for example F16C conversion); it ships scalar-first behind the same SIMD feature flag (R2, Decision 14).

### 9.3 Session Semantics

| Concern | Behaviour |
| --- | --- |
| Sending state | Each session remembers last-sent values in compact f16 wire form; each entry resolves against its source at encode time — a device entry reads the state table via the leaf's `state_offset`, a group entry reads the aggregate snapshot for the node's local channels (§4.4). |
| Value set | Every visible entry's complete value set is defined (a device instance's metrics, or a group entry's local channels); the client resolves the active display value locally (FR-CD-03), and transmission follows FR-TR-02. |
| Attention | Each entry carries the node's attention value (a group entry's reserved attention channel, or a device instance's attention level); sent on change. |
| Keyframes | Full visible keyframe on (re)connect; appeared entries sent in full on expansion or visibility change; on-demand resync on a sequence gap; no periodic keyframe (Decision 15). |
| Unavailable values | Explicit NaN marker on the wire for an unavailable device metric or a channel with no available contribution, sent regardless of the ε filter. |
| Sequence | Frames carry a monotonic sequence; clients discard duplicates and request a keyframe on a detected gap. |
| Wire identity | Per-frame entries are keyed by the server node index (u32). On (re)connect the server sends a **dictionary** mapping each node index to `(kind, config id)`; the client resolves position, units, metric labels, and channel meanings from the shared config (FR-TD-09). Indices are per-build and not stable across restarts; a reconnect gets a fresh dictionary (Decision 11). |
| Slow clients | Stale frames are dropped (latest-wins on egress) and do not advance the baseline; per-session state is bounded (Decision 15). |
| Inspection target | A selected device or group is streamed every frame at full fidelity until deselected; the ε filter does not suppress it. |

**Session reconciliation.** The last-sent baseline is valid only on the intersection of the current visible set and the previous frame's set, so each frame reconciles the two:

| Case | Session state | Wire |
| --- | --- | --- |
| In both frames | baseline valid | changed slot sent as absolute f16, ε-filtered |
| **Appeared** (current, not previous) | baseline created | full value, `appeared` flag |
| **Removed** (previous, not current) | entry pruned | `removed` flag |

Keying is by node index, which is sufficient because the index is immutable and a node's kind never changes; a device↔blend transition is different indices (the group node leaves the set, its device leaves enter), each handled as appeared/removed. Reappearance after any gap is treated as **appeared** — the client may have dropped the instance, so a stale baseline would be invalid. Pruning on removal keeps per-session state bounded by the current visible set plus the selected object (FR-TR-04).

Implementation is a dense-integer diff — O(entries) per viewer per frame, allocation-free, no hashing. Per session, a `u32` frame **epoch** array indexed by node detects presence and change eligibility without per-frame clearing (only visible nodes are written), and a `u16` **last-sent** array indexed by value slot holds the wire-domain baseline; removals are found by scanning the previous frame's node list (≤ B), never all N nodes. Baseline advancement on the transport occurs only when a frame is actually written, so latest-wins drop and change suppression compose. Table sizes: ~1 MB per viewer at 100k (~300k device-metric slots at 2 B, plus 100k epochs at 4 B).

### 9.4 Frame Layout

Little-endian. Every message begins with a `u8 type`; a frame is `type = 0` (the dictionary is `type = 1`, §9.5). All entries and the per-session diff are keyed by node index (Decision 11).

```
Frame (type = 0):
  u8  type
  u32 sequence
  u16 flags              # bit0 keyframe, bit1 keyframe_end
  u16 entry_count
  Entry[entry_count]:
    u32 node_index
    u8  entry_flags      # bit0 appeared, bit1 removed, bit2 attention-present
    u8  changed_mask[]   # one byte per 8 value slots; bit set = slot included
    f16 values[]         # included slots in slot order; canonical NaN = unavailable
    [u8 attention]       # present only when entry_flags.attention-present
```

- A **device entry**'s value slots are its metric instances; a **group entry**'s are its local channels; slot order is the config order (Decision 7/9).
- `appeared` entries carry every slot; `removed` entries carry none.
- `appeared`/`removed` are the wire transitions; the visibility query's internal `expanded` transition (§8.2) is not a wire flag — it materialises as `appeared` entries for the newly revealed devices.
- `changed_mask` bounds the payload to changed slots; an entry with nothing changed and no attention change is omitted entirely.
- Attention is sent on change only (Decision 15).
- A keyframe is the full visible set with every slot present, chunked across frames with the `keyframe`/`keyframe_end` flags when it exceeds a size threshold (default ~64 KB). No wire message exceeds 64 KB; a keyframe chunk is cut at that bound.
- Field widths are fixed; the canonical unavailable marker is the quiet-NaN f16 pattern `0x7E00`.

### 9.5 Other Messages

Little-endian; each begins with a `u8 type` (direction-scoped).

```
Dictionary (server → client, type = 1):        # once on (re)connect, before the first frame
  u32 count
  Entry[count]:
    u32 node_index
    u8  kind                # 0 = group, 1 = device
    u16 id_len
    u8  id[id_len]          # config id (UTF-8)

CameraContext (client → server, type = 1):     # every frame
  f32 position[3]
  f32 forward[3]
  f32 up[3]
  u32 width
  u32 height
  f32 pixel_ratio
  f32 fov

InspectionTarget (client → server, type = 2):  # on selection change
  u32 node_index            # 0 = deselection
```

The dictionary maps each node index to its kind and config id; the client resolves positions, units, metric labels, and channel meanings from the shared config (FR-TD-09, Decision 11). The camera context is sent every frame (FR-CD-06); the inspection target only on change (FR-VS-02).

## 10. Client SDK Design

The client and demo target the building tier (~4,280 devices). Browser KPIs K6 (frame rate) and K7 (memory) are verified at that scale; the engine itself is benchmarked across all three tiers (§14).

The client loads the same topology definition as the server (FR-TD-09) to map instance ids to device positions and metric units, so positions are never transmitted per reading.

### 10.1 Rendering

- Three.js `InstancedMesh`; per-frame instance transforms and colours written into a pre-allocated VBO to avoid allocation-driven stalls (FR-CD-01, K7). Individual device instances and blended group entries are drawn from the same instanced buffers. Each instance's colour is derived from the selected display value (§10.4).
- Target: stable 60 FPS, main-thread overhead < 3 ms/frame (K6).
- Fallback (R4): rebuild mesh per frame; drop LOD transitions.

### 10.2 Camera Publisher

Each frame the client sends its camera pose (position and orientation) and screen geometry to the server (FR-CD-06), closing the camera-feedback loop that enables server-side culling and blending.

### 10.3 Picking

Object picking / click-to-inspect (FR-CD-04). Primary path uses instance id hit-testing; fallback to CPU `Raycaster` if needed (R4). The inspected value is delivered by the targeted per-session stream (§9.3), not a server query; for an individually streamed instance the client already holds the current value.

### 10.4 Display Mapping

Each rendered object is coloured by one selected value (FR-CD-03). The default is the object's **attention value** (a group's reserved attention channel, or a device's attention level), which gives a consistent fixed severity scale for comparing devices and groups. The viewer may instead select a metric (for a device instance) or one of the group's local channels (for a group entry); because a group's channels are local to it, these selectable values are resolved per group from the shared config. Switching is a client-only change, since every visible entry's complete value set is already streamed (§9.3).

## 11. End-to-End Data Flow

```
simulator ──readings──► state table (latest-wins)
                            │
                            ├──► shared aggregation (60 Hz, double-buffered)
                            │
browser ──camera──► visibility query (frustum + size threshold + entry budget, best-first)
                            │
                            └──► per-viewer select → f16 ε change suppression → WS frame
                                                                        │
browser ◄───────────────────────────────────────────────────────────────┘
   └─ VBO write ─► InstancedMesh draw ─► 60 FPS
```

## 12. Performance Budgets

| Stage | Budget | KPI |
| --- | --- | --- |
| Server traversal | ≤ 1.5 ms | K2 |
| Shared aggregation pass | ≤ 0.5 ms for the benchmark topology's configured channel set; scales with channels configured (16 per group, plus the reserved attention channel) | K9 |
| Per-viewer encode | ≤ 0.3 ms | — |
| Network + jitter | ≤ 2 ms | — |
| Browser parse/write | ≤ 3 ms | K6 |
| Rendering | remaining ~10 ms | K6 |

Named budgets sum to ≤ 7.5 ms; overruns do not cascade across frames. K2 measures the server-traversal stage only; these stage budgets compose into the end-to-end freshness target (K11).

## 13. Testing and Validation Strategy

| Layer | Technique | Covers |
| --- | --- | --- |
| Unit | Per-module tests from day one | Build, aggregation, encoding |
| Property | Randomised/permuted fixtures | R1, R3, K4 |
| Composition | Per-group channel contributions, same-reduction joins, mean accumulators | K4 |
| Attention | Limit-band mapping and absence levels vs a scalar reference; max-propagation property tests | FR-TD-11, FR-AG-06 |
| Differential | Engine traversal/selection vs flat-list brute-force reference | R1 |
| Selection | Exact-height heap refinement vs brute-force ranked cut | R5, K3, FR-VS-07 |
| Boundary | Empty subtrees, non-multiple-of-8 counts | R2 |
| Degradation | Flat hierarchy (all devices under the root) returns a single blended entry within budget, with exact aggregates | R5, G2 |
| Invariant | Boot-time structural scan | R1, R3 |
| Integration | Backend alpha / end-to-end | FR-CD-08 |
| Ablation | Sibling ordering on/off (locality hypothesis) | R1 |
| Ablation | SIMD on/off (aggregation and encode) | R2, K9 |
| Ablation | ε suppression on/off (egress) | K5 |

Aggregation is validated against a **brute-force scalar reference** and must match exactly on all fixtures (K4); both use the same blocked-lane order (Decision 14).

## 14. Benchmark Methodology

### 14.1 Benchmark host

The backend host is fixed before benchmarking (FR-BR-01). These fields are recorded at M1; concrete values are filled before A14.

| Field | Value |
| --- | --- |
| CPU model / physical cores / base–boost clock | TBD |
| SIMD features (AVX2 / AVX-512) | TBD |
| Memory (size / speed) | TBD |
| Storage | TBD |
| OS + kernel | TBD |
| Rust toolchain / allocator | TBD |
| Core pinning | `taskset` (simulator and engine isolated) |
| Client-KPI laptop | mid-range laptop, specified separately (K6/K7/K11) |
| Network | localhost / same LAN |

The K5 raw-streaming baseline is fixed in PRD §7.1 (full-key JSON, ~100 B/reading). Baseline performance numbers are produced at A14–A16.

### 14.2 Fixed benchmark topology (FR-BR-05)

The benchmark topology is produced by a **deterministic, seeded generator** (version `bench-v1`) that emits a topology in the schema of §4.1. The generated file is the published artifact; the spec plus seed reproduce it exactly.

- **Tier shapes** (bounded fan-in; exact device counts):

| Tier | Devices | Buildings | Rooms/bldg | Racks/room | Devices/rack |
| --- | --- | --- | --- | --- | --- |
| Building | 4,280 | 1 | 8 | 10 | 54 / 53 (half each) |
| Intermediate | 50,000 | 10 | 8 | 10 | 63 / 62 (half each) |
| Full | 100,000 | 20 | 8 | 10 | 63 / 62 (half each) |

  That is 80 / 800 / 1,600 racks; the half-and-half split lands each tier's total exactly.
- **Metrics** (~3/device): `temp` (C), `vib` (mm/s), `power` (kW), each with ε, a freshness timeout, severity limits, and an absence level.
- **Channels:** per level — for example a rack `rack_avg_temp` (mean, C) fed by its devices, a room `room_avg_temp` merging rack channels, and building and site merging upward, same reduction and explicit contributions.
- **Positions:** deterministic grid — building along x, room on y (floor), racks on an x/z grid within a room, devices at small in-rack offsets.
- **Reading rates:** full park ~1/s per metric instance (~300,000/s); intermediate ~1/s per instance (~150,000/s); building ~2.9/s per instance (~37,000/s), per PRD §7.1.
- **Templates:** rack groups use a `group_types` entry so the generated file stays compact (Decision 9).

### 14.3 Runs

- Tiers: 4,280 / 50,000 / 100,000 devices, each with a fixed metric set per device (~3 metrics/device), so state size and boot cost are reproducible.
- Baseline reading rates per tier are fixed in §14.2; the simulator's per-device reporting profiles must reproduce them (see §4.2).
- Client-side KPIs K6 and K7 are measured in the demo application at the building tier (~4,280 devices), not at the 100k tier.
- End-to-end freshness (K11) is measured in the demo application with instrumented timestamps at the building scale.
- Stress: ≥ 300,000 EPS for ≥ 30 min (K1).
- Concurrent-session run: ≥ 10 browser sessions at 60 Hz at the full-scale tier (100,000 devices); ≤ 50% aggregate CPU; per-client egress within K5 (K10).
- Micro-benchmark: per-channel group aggregation (K9).
- Ablation (FR-BR-03, Should): sibling ordering on/off (locality hypothesis; expected nil, since a parent's AABB is order-invariant); SIMD on/off for aggregation and encoding (informs the A7 ship decision); ε suppression on/off (egress, K5).
- Simulator pinned via `taskset`; non-blocking I/O; pre-generated traces as contingency (R6).

## 15. Risk-to-Design Mapping

| Risk | Design response |
| --- | --- |
| R1 index correctness | Structural invariants + boot scan; independent flat-list brute-force reference for traversal and selection; R5 fallback if pruning is inadequate. |
| R2 SIMD correctness | Scalar reference alongside SIMD; boundary tests; scalar ships first. |
| R3 channel/contribution invariants | Build-time assertions; per-group channel definitions; unit and same-reduction contributor validation; permutation tests; gather/reduce fallbacks. |
| R4 Three.js failures | Early InstancedMesh spike; WebGL inspector; mesh rebuild fallback; CPU raycast. |
| R5 latency | Skip-offset pruning; expansion budget; shared aggregation; grid/idle/top-50 fallbacks. |
| R6 simulator starvation | Core pinning; non-blocking I/O; pre-generated traces. |
| R7 flicker | Hysteresis; keyframe on expand; GPU cross-fade contingency. |
| R8 slippage | Scalar-first; building-scale-only benchmarks; CPU picking; defer 100k tier. |
| R9 configuration validity | Boot-time validation rejects structural and channel/unit inconsistencies before serving (FR-TD-07, FR-TD-08). |

## 16. Build and Tooling

- Rust workspace: `engine`, `transport`, `simulator`, `benches` crates; `cargo nextest` tests; `criterion` benchmarks; `cargo-flamegraph`/`perf` profiling.
- Nightly toolchain for `portable_simd` (optional, behind a feature flag).
- Frontend: Vite + TypeScript + Three.js under `client` and `demo`.
- `taskset` for benchmark core isolation.
- **Run diagnostics (FR-CD-09):** each component logs a structured boot summary and any validation fault, and exposes counters for ingest rate, rejected readings, aggregate-pass time, and per-session egress; the demo surfaces them in a status panel.
- Everything is developed in the open: commits, design notes, and benchmark numbers land in the public GitHub repository as the work happens.

## 17. Requirement Traceability

| PRD requirement | TDD section |
| --- | --- |
| FR-TD-01–FR-TD-11 (topology/channels/index/limits) | §4, §5, §6, §7 (§7.1 for FR-TD-11) |
| FR-IS-01–FR-IS-05 (ingest/state/offline) | §6, §9.3 |
| FR-AG-01–FR-AG-06 (aggregation/attention) | §7 (§7.1 for FR-AG-06) |
| FR-VS-01–FR-VS-08 (visibility/LOD/budget/inspection) | §8, §9.3 |
| FR-TR-01–FR-TR-04 (viewer transport/encoding/sessions) | §9 |
| FR-CD-01–FR-CD-09 (client/demo) | §10, §3 |
| FR-BR-01–FR-BR-05 (benchmark/report) | §14 |
| K1–K11 | §12, §14 |

## 18. Design Decisions

| # | Decision | Recorded in |
| --- | --- | --- |
| 1 | One hierarchy serves both visibility and aggregation; frontier ordered by exact projected height | §2, §8.2–§8.4, §13 |
| 2 | Aggregation by accumulator propagation over per-group channels | §7 (see Decision 4) |
| 3 | Stateful per-viewer change suppression | §9 (Decision 15) |
| 4 | Per-group local channels with explicit contributions | §3.1, §4.1, §4.4, §5.2, §6, §7 |
| 5 | Aggregation execution: always-on 60 Hz background pass | §7 |
| 6 | Attention channel (severity and absence) | §3.1, §4.4, §7.1, §9.2, §9.3, §13 |
| 7 | Metric identity and state layout: device-local labels, per-device instances, device-major state | §4.1, §4.2, §4.4, §5.2, §5.3, §6; amends FR-TD-04/FR-TD-08 |
| 8 | Canonical child order and child enumeration | §4.4, §5.2, §5.3, §8.2 |
| 9 | Config schema: JSON flat node list, explicit fields, shallow templates | §4.1, §5.2 |
| 10 | Index layout for channel definitions and contributions; aggregation direction | §4.4, §7 |
| 11 | Wire identity | §8.2, §9.3 |
| 12 | Hierarchy build details | §2, §5.2–§5.4 |
| 13 | Benchmark host spec and fixed benchmark topology | §14; records FR-BR-05 |
| 14 | Canonical reduction order and SIMD | §7, §9.2 |
| 15 | Transport encoding, keyframes, and egress | §9.2–§9.4 |

### Decision 1 — Frontier ordering

The frontier is a fixed-capacity **max-heap keyed on exact projected on-screen height**, with node id as the terminal tie-break. Recorded in §2, §8.2, §8.3, §8.4, §13.

- The hierarchy is the single structure driving visibility and aggregation; the state table is a separate flat value store aligned to leaf order.
- Exact height is the primary key; no bucket size-error bound or bucket-width parameter applies.
- Ordering is a pure function of `(topology, state, camera, selection)`. Node id is a terminal tie-break for equal heights only. The index is immutable after boot, so the frontier holds node references and per-frame heights.
- Heap operations are `O(log B)`; the frontier is bounded by `B` ≈ 5,000.
- Budget-edge flicker is handled by the entry-count hysteresis in §8.3.

### Decision 2 — Accumulator propagation

Resolved by Decision 4. One shared 60 Hz pass computes each group's local channels by accumulator propagation over its configured contributions (its own device readings and its child groups' contributed channels), with mergeable accumulators (`mean → (sum, count)`), exact over the subtree, published as a consistent per-frame snapshot. Recorded in §7. Alternatives rejected: incremental push-up on ingest; per-viewer or per-request aggregation; event-driven recomputation; GPU/compute offload; alternative state layouts.

### Decision 3 — Stateful per-viewer change suppression

One persistent session per viewer; each frame reconciles the visible set against the previous frame (appeared → full value, removed → prune, otherwise changed slots as absolute f16, ε-filtered in the wire domain); the selected object is streamed at full fidelity; full keyframe on (re)connect; per-session state bounded with latest-wins on egress. Resolved by Decision 15 (the baseline commits on an actual socket write; a dropped frame does not advance it). Recorded in §9.

### Decision 4 — Group channel model

Each group node defines its own local channels — up to 16, in addition to its attention value — whose meaning is local to that group, and configures which of its channels contribute to which of its parent's channels. Recorded in §3.1, §4.1, §4.4, §5.2, §6, §7; PRD FR-TD-05/06/08, FR-AG-01.

- **Local channel identity.** A channel's meaning is local to its group; there is no site-wide channel catalogue. The engine validates units per configured contribution; the client resolves a group entry's channel meanings per group.
- **Explicit contributions, default none.** A group contributes nothing to its parent unless configured (FR-TD-06).
- **Same-reduction contributions.** A channel may contribute only to a parent channel of the same reduction (boot-validated), so a group's value is the canonical subtree reduction.
- **`mean` and `count`.** `mean` is carried as `(sum, count)` and merged as an accumulator, exact over the subtree's available readings; the wire carries the finalised scalar. `count` is the total number of available contributing readings in the subtree, merged from child `count` channels by summation (FR-AG-01).
- **Channel states.** A configured channel is active (has a value), inactive (no available contributor → NaN), or not-used (not configured). All configured channels are carried in a keyframe; steady state carries changes.
- **No `published` flag and no noop channel.** All configured channels are intended aggregates, and the per-frame sender logic governs transmission. Contributions are resolved at boot, so no target-resolution branch or noop channel is required.
- **Terminology.** The canonical term is **aggregate channel**.

The reserved attention channel is not addressable in configured contributions — it propagates implicitly. A status/coverage channel is not adopted (subsumed by per-metric absence). SIMD remains applicable (16 channels ≈ one 512-bit vector; permute/gather plus masked reduce, with effectiveness depending on wiring regularity). Per-group channel definitions plus contributions are the main config-volume cost; templating by group type mitigates it.

### Decision 5 — Aggregation execution

One background pass at 60 Hz computes every group bottom-up from a **boot-built compiled plan** (seed ops: device metric → channel; merge ops: child channel → parent channel), with accumulator layout and reduction order fixed at boot. The pass runs continuously, independent of viewer count. Recorded in §7.

- Freshness is re-evaluated every pass; aggregates change without ingest as metrics cross their freshness timeout (FR-IS-03).
- Bottom-up is a reverse linear scan of the parent-first index; no recursion is required.
- Double-buffered accumulators with an atomic buffer swap give the per-frame consistent snapshot (FR-AG-03).
- Frame boundary (FR-AG-05): atomic latest-wins reads may include a reading accepted during the pass or defer it to the next frame; a reading accepted before the pass cannot be missed.
- Unavailable contributions are skipped; `mean` accumulates `(sum, count)` over available inputs (FR-AG-04).

### Decision 6 — Attention channel (severity and absence)

Each group carries a reserved unitless attention channel (reduction `max`) on the fixed severity scale (normal, advisory, warning, critical), computed by the aggregation pass and streamed per entry; a device's attention level is the greatest of its metrics' contributions. Recorded in §3.1, §4.4, §7.1, §9.2, §9.3, §13; PRD FR-TD-11, FR-AG-06, FR-CD-03.

- Value severity is derived from limits on metrics and channels, evaluated as the maximum over crossed limits; limits inherit the value's unit.
- Absence severity is per metric: a metric contributes its configured absence level while it is unavailable; `0` is permitted. A per-device override is not adopted (it could be added later as a raise-only field).
- The attention channel merges as a `max` channel, evaluated in the same bottom-up pass as aggregation, and is always contributed to its parent's attention channel; it is additional to the ≤16 configured channels.
- Explicit out-of-service/offline signalling is out of scope (PRD §10); a device is offline when all of its metrics are unavailable (FR-IS-04; staleness per FR-IS-03).
- Availability is orthogonal: offline and NaN markers are streamed separately; a node with no data and no nonzero contribution reports normal.
- Fraction-based coverage policy is not adopted (PRD §10); `max` does not distinguish the number of missing devices.

### Decision 7 — Metric identity and state layout

Metric identity is the per-device instance `(device, label)`; labels are chosen freely by each device and need only be unique within it. There is no site-wide metric vocabulary and no global kind slot. Recorded in §4.1, §4.2, §4.4, §5.2, §5.3, §6; implements PRD FR-TD-04/FR-TD-08.

- **Identity.** A reading names `(device_id, metric_label)`; the label binds at boot to a per-device `metric_slot`. A slot is meaningful only within its device, so there is no global metric column.
- **Typing.** Each metric instance declares its own unit and attributes (ε, freshness, limits, absence, contributions). The engine consumes **unit**, not the label and not any separate `quantity` concept; no channel combines contributors of different units (FR-AG-02).
- **State layout.** The state is normalized to one slot per metric instance: a compact `values` array with one entry per `(device, metric)`, plus a per-device row-offset array (CSR) that groups a device's instances and orders them in leaf order and local-slot order. Rows are exact, so there are no unused slots.
- **Aggregation.** At boot each channel records the sorted state offsets of its contributing metric instances; the 60 Hz pass walks those offsets within the group's contiguous block of rows. K9 (≤ 5 µs over ≤ 1,000 contributors) is met by the bounded, prefetch-friendly gather rather than by a single sequential run.
- **Encode.** A device's whole value set is contiguous, matching the per-entry "all values" requirement (FR-TR-02, §9.3).
- **Client.** Cross-object comparison uses the attention channel (FR-CD-03); per-metric selection is per-object by construction, so no global vocabulary is needed.

### Decision 8 — Canonical child order and child enumeration

Canonical child order is the order in which children are declared in the config, and the index carries a per-node child count for O(1) degree. Recorded in §4.4, §5.2, §5.3, §8.2.

- **Child order.** Children are an ordered sequence; the preorder DFS visits them in declaration order. This order defines node indices (wire identity, Decision 11), leaf order (CSR row order and the per-channel offset lists), the canonical float reduction order (Decision 14), and the frontier tie-break. It is server-side only: the client does not replay the build. Order is never derived from ids, map iteration, or filesystem order.
- **Child enumeration.** Node shape: `subtree_size` (node count) and `child_count`. `first_child = index + 1` (parent-first preorder over emitted nodes; excluded childless groups never intervene) and `next_sibling = sibling + subtree_size` are derived, not stored. `deg(v)` is O(1), so the expansion-budget check `entries − 1 + deg(v) ≤ B` is O(1) even for a flat hierarchy with ~100k children; enumeration is O(deg) and only runs when `deg ≤ B`.
- **Subtree-size units.** `subtree_size` is a node count. No per-node device-count extent is needed: aggregation reads per-channel offset lists (Decision 7), and visibility uses projected height and the entry budget.
- **Invariant.** Boot checks that `child_count` equals the emitted direct children and that `first_child = index + 1`; the contiguity scan confirms `subtree_size`.

### Decision 9 — Config schema

The topology is a single JSON document (FR-TD-09) containing a flat list of nodes with parent references; list order is canonical sibling order. Recorded in §4.1, §5.2.

- **Hierarchy.** A flat `nodes` array; each node has `kind` (`group`/`device`), `id` (topology-unique), and `parent` (null for the single root). A pre-pass resolves templates, buckets children by parent in list order, and validates the tree; the preorder DFS then runs in that order.
- **Metrics.** Per metric instance: `label` (device-local, unique), `unit`, `epsilon`, and `freshness_ms` are required; `contributes_to`, `limits`, and `absence` are optional (default none / none / normal).
- **Channels.** Per group: at most 16, each with `id` (group-local, unique), `reduction`, `unit`, and `epsilon` required; `contributes_to` (parent channel ids) and `limits` optional. `count` is unit-typed like any other channel — its unit names the readings tallied.
- **Wiring.** Metric→channel and channel→parent, both default none; fan-out and fan-in are allowed, and a channel may feed several parent channels (same reduction). A 17th channel is a boot error.
- **Limits and absence.** Fixed named scale `normal`/`advisory`/`warning`/`critical`; a limit is `{threshold, side, level}` with the threshold in the value's unit, fired by `>=` (high) or `<=` (low), and a value's level is the max over fired limits; `absence` is per metric, default normal. Attention is stored as one `u8` per group and one per device in the double-buffered snapshot.
- **Units.** Opaque strings compared by exact match.
- **Format and templates.** JSON only. Shallow single-level templates (`group_types`): a group may reference one `type`, and a declared `channels` replaces the template's. No type-extends-type, no per-channel merge.
- **Validation order.** Template resolution precedes FR-TD-08 validation; diagnostics name the source (template or node).

### Decision 10 — Index layout for channels/contributions; aggregation direction

Variable-length per-node data is held in global structure-of-arrays addressed by ranges, and the aggregation plan is indexed by target channel and executed as a pull. Recorded in §4.4, §7.

- **Arrays.** Node arrays carry `channel_base`/`channel_count`. A global **channel-def array** holds one entry per channel (`unit`, `reduction`, `epsilon`, `limit_base`, `limit_count`, `contrib_base`, `contrib_count`); a group's channels are contiguous. A global **limit array** holds `{threshold, side, level}`. A global **contribution array** holds one tagged entry per source: `seed` (source = state offset) or `merge` (source = contributing channel-def index).
- **Pull.** The plan is the contribution array indexed by target channel: each channel lists its incoming sources. The reverse scan — bottom-up, single background thread, writing the write buffer — reduces each channel's list (seeds in device-leaf order, then merges in child order) and finalises it. This vectorises the reduction, gives an explicit canonical order for exact `sum`/`mean` (Decision 14), needs no pre-clear, and leaves the seed reduction parallelisable if the pass is ever split.
- **Reserved attention.** One accumulator slot per group, outside the channel-def array; never a contribution target; computed from the group's channel limit levels, its direct devices' attention levels, and its children's attention channels.
- **Push rejected.** The background-thread/bottom-up design removes push's only advantage (no cross-source races), leaving its serial read-modify-write chain; pull also avoids a pre-pass buffer clear and yields the canonical order.

### Decision 11 — Wire identity

The wire key is the server's flat-array node index; the client maps it to config identities through a connect-time dictionary. Recorded in §8.2, §9.3.

- **Per-frame.** Entries and the per-session diff are keyed by the node index (u32), as in §9.3.
- **Dictionary.** On (re)connect the server sends a dictionary mapping each node index to `(kind, config id)`. The client resolves position, units, metric labels, and channel meanings from the shared config (FR-TD-09). The dictionary is sent once per session, re-sent on reconnect, and may be delta/compressed.
- **Client selection.** The client keys rendered instances by node index, so the inspection target (FR-VS-02) is sent as the node index.
- **Restart / edit.** Indices are per-build; the topology is fixed while serving, so indices are stable within a session. Across a restart or a config edit a fresh dictionary is sent; no cross-restart stability is assumed, and no client index cache survives a reconnect.

### Decision 12 — Hierarchy build details

Recorded in §2, §5.2, §5.3, §5.4.

- **Leaf AABB.** A leaf's AABB is the device position (degenerate point); frustum tests are inclusive, and the brute-force reference uses the identical point test. No inflation. A leaf's projected height is degenerate, which is harmless: a device leaf is terminal by kind, and the size threshold only decides group blending.
- **Positions.** Non-finite positions (NaN/Inf) are rejected at boot; coincident positions are legal and separated by the node-id tie-break.
- **Build parallelism.** The layout DFS is sequential — O(N) and not the K8 bottleneck. K8 is treated as a config-parse and validation budget; parsing is optimised (streaming) only if measurement demands it.
- **Validation.** Checks run on the resolved config in six stages — structural (FR-TD-07), device/metric (FR-TD-04), channels (FR-TD-05), contributions (FR-TD-06/08), limits/absence (FR-TD-11/08), post-build invariants (§5.3) — and are fail-fast: the first fault aborts boot with a structured diagnostic `{code, node id, metric label / channel id, message}`.

### Decision 13 — Benchmark host and topology

Records FR-BR-05. Recorded in §14.

- **Host.** The backend host spec is recorded as a field template (§14.1) at M1, with concrete values filled before A14. The K5 raw baseline is fixed in PRD §7.1; performance numbers are produced at A14–A16.
- **Topology (FR-BR-05).** A deterministic, seeded generator (`bench-v1`) emits each tier's topology in the §4.1 schema, with fixed tier shapes (4,280 / 50,000 / 100,000 devices over 80 / 800 / 1,600 racks), a ~3-metric set, per-level channels with explicit contributions, grid positions, per-tier reading rates, and `group_types` templates. The generated file is versioned and published with the report.

### Decision 14 — Canonical reduction order and SIMD

Recorded in §7, §9.2, §13.

- **Canonical order.** Sums use a blocked-lane order with fixed `W = 8`: source `i` accumulates into lane `i mod 8`, and the eight lanes fold in a fixed order. A channel's sources are already ordered (seeds by ascending state offset, then merges by child order, Decision 10), so the reduction is fully deterministic. The scalar engine and the brute-force reference implement the identical lane scheme and are therefore bit-exact; `min`/`max`/`count` are order-free, and the encode pipeline (f16 quantise, ε filter) is elementwise.
- **SIMD.** Scalar is the default and the reference. SIMD for aggregation and encoding ships behind a feature flag, with the ship decision made at A7 from the K9 measurement; the scalar path remains authoritative.

### Decision 15 — Transport encoding, keyframes, and egress

Recorded in §9.2–§9.4.

- **Value encoding.** The wire carries the absolute f16 value for each changed slot; the last-sent baseline is only the ε-suppression reference. There are no numeric deltas, so there is no client-side accumulation drift — the displayed value is exactly the last-sent f16, within ε + f16 precision of current.
- **Frame layout.** Little-endian; a leading `u8 type`, then a header (sequence, flags, entry count), then entries keyed by node index (Decision 11), each with entry flags, a changed-slot mask, the included f16 values, and attention only when changed. The canonical unavailable marker is the quiet-NaN f16 pattern `0x7E00`.
- **Keyframes.** Full visible keyframe on (re)connect; appeared entries sent in full on expansion or visibility change; on-demand resync when a client detects a sequence gap; no periodic keyframe.
- **Baseline and slow clients.** The baseline advances only on an actual socket write. Egress is latest-wins: a stale pending frame is replaced, and a dropped frame does not advance the baseline, so a client never decodes against a value it did not receive. Per-session state stays bounded by the current visible set plus the selected object.
