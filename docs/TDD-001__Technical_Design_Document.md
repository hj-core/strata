# TDD-001 — Technical Design Document

| Field | Value |
| --- | --- |
| Document ID | TDD-001 |
| Title | Server-Side Hierarchical Spatial Filtering Engine — Technical Design |
| Version | 0.7.0 |
| Status | Draft |
| Last Updated | 2026-09-28 |
| Owner | Project maintainer |
| Related documents | Upstream: PRD-001 (Product Requirements). Downstream: PLN-001 (Master Execution Sequence). |

## 1. Introduction and Scope

This document records and elaborates the design decisions for **how** the system is built: configuration schema and data model (§4), spatial index and aggregation (§5, §7), visibility selection (§8), transport and wire protocol (§9), client and simulator design (§4.2, §10), and testing and benchmark methodology (§13, §14). It addresses the requirements in PRD-001, its only upstream dependency; §16 maps each requirement area to its design section.

Design scope follows PRD-001. Capabilities excluded there (PRD-001 §10) — including field-protocol integration, telemetry persistence, statistical reductions beyond the supported set, alerting, federation, fraction-based coverage policy, explicit out-of-service signalling, occlusion, authentication and authorization, non-browser clients, and cloud deployment — remain out of scope here; PRD-001 §10 is authoritative.

Guiding rule: **the browser draws; the server thinks.** All heavy work is performed once per frame on the server and shared across viewers; each viewer receives a small, relevant slice per frame.

## 2. Design Goals, Constraints, and Technology Choices

| Goal / target | Design consequence | Verified in |
| --- | --- | --- |
| Sustain ≥ 300,000 EPS for ≥ 30 min with stable memory (K1, G1) | Lock-free latest-wins state table and non-blocking ingest path (Decision 16); back-pressure by overwriting stale values rather than buffering, so memory stays bounded (FR-IS-05). | §14.3 |
| Server-side visibility query p99 ≤ 1.5 ms per viewer per 60 Hz frame at 100k, across all camera poses, including max zoom-out; excludes encode, transport, and client (K2) | Constrained parent-first index; frustum pruning; heap-based best-first refinement (Decision 1). | §11, §14.3 |
| ≤ 5,000 active entries per viewer per frame, worst case — an entry being a device instance or a blended group entry (K3, G2) | Per-frame entry budget enforced as a frontier cut size (Decision 1). | §13 |
| Group aggregates match an exact reference; per-unit separation enforced (K4, G3) | Per-group local channels (Decision 4); one unit per channel, never mixed (FR-AG-02); same-reduction contributions; accumulator merge (`mean → (sum, count)`); brute-force scalar reference for tests (§13). | §13 |
| Egress ≤ 0.3 MB/s per client at 100k (≥ 99% cut); ≤ 0.15 MB/s at 50,000 (≥ 99% cut); ≤ 0.05 MB/s at 4,280 (≥ 98.5% cut) — for the benchmark topology's configured channel set, including per-entry attention values (K5, G4) | f16 quantisation + per-value ε filter (metric/channel) + change suppression (Decision 15). | §14.1, §14.3 |
| Stable 60 FPS, main-thread overhead < 3 ms/frame (K6, G2) | InstancedMesh with a pre-allocated VBO; per-frame instance writes without allocation (Decision 17). | §11, §14.3 |
| No monotonic heap growth in steady state; no allocation-driven frame spikes (K7) | Pre-allocated buffers (Decision 17); per-session state bounded (FR-TR-04). | §14.3 |
| Boot ≤ 30 s from topology file to fully loaded and serving at 100k devices (K8) | Sequential single-pass build; the budget is dominated by config parse and validation (Decision 12). | §14.3 |
| ≤ 5 µs per channel per group with up to 1,000 contributing readings (K9) | Boot-recorded per-channel contribution offset lists walked as a bounded gather within the group's contiguous rows (Decision 7). | §11, §14.3 |
| ≥ 10 concurrent browser sessions at 60 Hz at 100k devices; ≤ 50% aggregate server CPU; per-client egress within K5 (K10) | Aggregation pass independent of viewer count (Decision 5); per-viewer cost O(visible); per-session state bounded (FR-TR-04). | §14.3 |
| Update freshness p99 ≤ 100 ms from a reading accepted by the engine to the rendered change in the client, at 60 Hz and the building scale (~4,280 devices) (K11) | Stage budgets compose into an end-to-end path shorter than the freshness target. | §11, §14.3 |
| Same binary serves 4,280 and 100,000 devices with no code change (G5) | One config-driven engine with no scale-specific code paths; scale is a property of the topology, not of the code (Decision 9). | §14.3 |

**Measurement scope (PRD-001 §7).** Product targets (K1–K5, K8–K10) are measured on the fixed benchmark host: K1, K2, K8, and K10 are defined at the full-scale tier (100,000 devices); K5 gives a target for each tier; K3, K4, and K9 are tier-independent. Showcase targets (K6, K7, K11) are measured in the demo application on a mid-range laptop at the building scale (~4,280 devices).

**Constraints (PRD-001 §9).**

- **Browser capability:** a GPU-accelerated browser on a mid-range laptop is the rendering target.
- **Transport:** one persistent connection per viewer (FR-TR-01).
- **Scope tiers:** 4,280 / 50,000 / 100,000 devices are deliberately demanding and testable figures, not measurements from a field study; all three tiers are in scope — the benchmark harness exercises all three, while the demonstration application targets the building tier (~4,280 devices); the tier set is reduced only by an explicit, versioned amendment (PRD-001 §12, R8).
- **Scale unit:** scale is counted in devices (the leaves of the hierarchy); memory and compute also depend on metrics per device and the reading rate (see PRD-001 §7.1).
- **Benchmark host:** a single fixed host, specified before benchmarking begins.
- **Measurement environment:** simulator, engine, and harness run on the fixed host with browsers on the same local network; the demonstration runs the engine and browser together on the mid-range laptop; wide-area network behaviour is out of scope.
- **Topology stability:** fixed at boot, not modified while serving.
- **LOD granularity:** level-of-detail depends on intermediate groups; a flat hierarchy still meets K3 but collapses to large blended entries when zoomed out.
- **Deployment:** no production deployment is assumed.

PRD-001 §9 is authoritative for these constraints.

**Technology choices (this document).** PRD-001 does not prescribe implementation (PRD-001 §1), so the technology is chosen here:

- **Language/runtime:** engine in Rust; browser client in Three.js (§15, Decision 18).
- **Rendering API:** WebGL on the GPU-accelerated browser of PRD-001 §9 (Decision 18).
- **Wire protocol:** WebSocket with a compact binary frame format (§9.1, Decision 18).

## 3. Architecture Overview

The system comprises a server process and a browser client, supported by three additional programs: the simulator, the demo application, and the benchmark harness.

- **Server (Rust).** Owns the state table and the hierarchical spatial index, and runs every stage of the pipeline: telemetry ingest, index build, shared aggregation, per-viewer visibility query, per-viewer encode, and WebSocket transport. All heavy work happens here, once per frame, shared across viewers (Decision 18).
- **Browser client (Three.js).** Renders the entries the server selects, using an `InstancedMesh` over a pre-allocated VBO, and reports back its camera pose and screen geometry every frame, and its selected object and each deselection on change (Decision 18).
- **Supporting programs.** The simulator (Rust binary) feeds ingest; the demo application (Vite/TS) wires simulator → engine → browser; the benchmark harness measures the KPIs (§14). The stack for all three is fixed by Decision 18.

Component responsibilities and PRD-001 coverage are given in §3.1. The order in which the stages execute — the boot path, the ingest path, the frame path, and both client feedback loops — is given as a numbered sequence in §3.2.

### 3.1 Component Responsibilities

Each row lists the component's primary PRD-001 coverage; a requirement that spans components (for example FR-TD-09, FR-TD-10, FR-CD-09) appears on each. §16 is the authoritative requirement traceability.

| Component | Responsibility | PRD coverage |
| --- | --- | --- |
| Simulator | Generate topology-driven telemetry at configurable rates from the shared topology. | FR-TD-09, FR-CD-07, FR-CD-09 |
| Ingestor | Accept readings; apply latest-wins to the state table; reject invalid readings; sustain the offered stream without unbounded buffering. | FR-IS-01, FR-IS-02, FR-IS-05, FR-CD-09 |
| State table | Hold exactly one current value per (device, metric) with its availability; device-major layout per §4.3. | FR-IS-01, FR-IS-03, FR-IS-04, FR-CD-09 |
| Index builder | Load and validate the topology definition and its type libraries; build the hierarchical spatial index; bind metric labels to per-device slots. | FR-TD-01–FR-TD-11, FR-CD-09 |
| Aggregator | Compute each group's channels and attention value once per frame from device readings and child-channel contributions; double-buffered. | FR-TD-05, FR-TD-06, FR-TD-10, FR-TD-11, FR-AG-01–FR-AG-06, FR-CD-09 |
| Visibility engine | Accept camera context and selection; frustum-cull and best-first refine the hierarchy into a bounded entry cut. | FR-VS-01–FR-VS-08, FR-TD-10, FR-CD-09 |
| Encoder | Quantise, ε-filter, and suppress unchanged values per viewer; stream the selected object at full fidelity. | FR-TR-02, FR-VS-02, FR-CD-09 |
| Transport | WebSocket server + binary framing + sessions; ingress for camera context and inspection target. | FR-TR-01, FR-TR-03, FR-TR-04, FR-VS-01, FR-VS-02, FR-CD-09 |
| Browser 3D client | Render instances, publish camera, pick objects, select display value; load the shared topology. | FR-CD-01–FR-CD-06, FR-CD-09, FR-TD-09 |
| Demo app | Wire simulator → engine → browser at the building scale (~4,280 devices). | FR-CD-08, FR-CD-09 |
| Benchmark harness | Produce the fixed benchmark topology; measure and report KPIs. | FR-BR-01–FR-BR-05 |

### 3.2 End-to-End Data Flow

Three paths make up the pipeline: the **boot path**, which runs once before serving; the **ingest path**, which runs continuously and asynchronously to the frame; and the **frame path**, which runs once per 60 Hz cycle. The client closes the loop through two feedback inputs.

**Boot path — once, before serving**

1. The topology definition and its type libraries are loaded and validated, the hierarchical spatial index is built, metric labels are bound to per-device slots, and each channel's contribution list is compiled (§5.2, §5.3). The topology is then fixed until restart (FR-TD-07, FR-TD-08). This boot step runs once and does not repeat on a restart-free server.

**Ingest path — continuous, not frame-bound**

2. The simulator emits readings (`device_index`, `metric_slot`, `value`) to the **ingestor**, which applies latest-wins into the **state table** and discards invalid readings (§4.2, §6). Freshness and offline state follow from the state table (FR-IS-03, FR-IS-04). This path never blocks on a frame.

**Frame path — once per 60 Hz cycle**

3. **Aggregation** reads the state table (metric values) and the index (the boot-compiled contribution plan, Decision 10) — and nothing per-viewer — then writes each group's channels and attention into the write buffer and swaps buffers to publish the frame's snapshot (§7).
4. **Camera context** — each client sends its camera pose and screen geometry every frame, and its selected node index on selection change; the transport passes both to the visibility engine (§9.5).
5. **Visibility** reads the index (geometry boxes, LOD boxes, subtree sizes, child counts) and the camera context from step 4 — and reads no values — then produces that viewer's bounded entry cut of at most `B` entries (§8.2).
6. **Encode** — per viewer, reads three inputs: the entry cut from step 5 (which entries exist), the aggregate snapshot from step 3 (a group entry's channel values), and the state table directly (a device entry's metric values via `state_offset`). It converts each value to its slot's wire width — f16, or f32 for `sum`/`count` — applies the per-value ε filter, suppresses unchanged slots against the last-sent baseline, and sends the selected object at full fidelity (§9.2, §9.3).
7. **Transport** writes one WebSocket binary frame for that viewer (§9.4); the session baseline advances only on an actual socket write (§9.2, §9.3).
8. **Render** — the client writes instance transforms and colours into the pre-allocated VBO and draws the `InstancedMesh` (§10.1).

Steps 3 and 5 are independent branches: they share no data and meet only at step 6. Step 5 cannot substitute for step 3's values, and step 3 does not know which entries step 5 selects.

**Feedback and session paths**

- **Camera loop:** step 4 → step 5, every frame. Without it the visibility query has no screen geometry and cannot size the LOD threshold (FR-VS-01).
- **Selection loop:** step 4 → step 5 (force-include the selected object) and → step 6 (stream its values every frame until deselection), on selection change only (FR-VS-02).
- **Session start:** on (re)connect the transport first sends the node dictionary (§9.5) and a full visible keyframe (FR-TR-03), after which the normal frame path — steps 3 through 8 — resumes carrying incremental frames.

## 4. Data Model

### 4.1 Topology Schema

The topology definition is three JSON documents loaded as one source (FR-TD-09): a **group type library**, a **device type library**, and the **topology document** — a flat list of nodes, each referencing its parent by id. List order defines sibling order (Decision 8), ids are unique across the topology, and a topology that declares a `type` records which version of each library it requires. PRD-001 §8 fixes this interface and its semantics and leaves the configuration schema to this document, so the three-document split and the version contract are design choices (Decision 9).

**Device type library.**

```json
{
  "version": 1,
  "types": {
    "std_rack_node": {
      "metrics": [
        { "label": "cpu_temp_1", "unit": "C" },
        { "label": "cpu_vib",    "unit": "mm/s" }
      ],
      "policy": {
        "cpu_temp_1": { "epsilon": 0.1,  "freshness_ms": 5000 },
        "cpu_vib":    { "epsilon": 0.05, "freshness_ms": 2000 }
      }
    },

    "monitored_rack_node": {
      "metrics": [
        { "label": "cpu_temp_1", "unit": "C" },
        { "label": "cpu_vib",    "unit": "mm/s" }
      ],
      "policy": {
        "cpu_temp_1": {
          "epsilon": 0.1,
          "freshness_ms": 5000,
          "absence": "advisory",
          "limits": [
            { "threshold": 80, "side": "high", "level": "critical" },
            { "threshold":  5, "side": "low",  "level": "advisory" }
          ]
        },
        "cpu_vib":    { "epsilon": 0.05, "freshness_ms": 2000 }
      }
    }
  }
}
```

**Device library fields.**

| Field | Required | Notes |
| --- | --- | --- |
| `version` | yes | library revision; equals the topology's `requires.device` (Decision 9) |
| `types` | yes | map of device type names to type entries — `metrics` (identity) and `policy` (the tunable fields, keyed by metric label); every device's `type` resolves here (Decision 20) |

**Metric identity fields.** What the type fixes for the metric — distinct from the instance identity `(device, label)` (Decision 7).

| Field | Required | Notes |
| --- | --- | --- |
| `label` | yes | unique within the device (Decision 7); supplied by the type, never overridden (Decision 20) |
| `unit` | yes | opaque string, compared by exact match; supplied by the type, never overridden (Decision 9, Decision 20) |

**Metric policy fields.** One `policy` map per type entry, keyed by metric label: every label in `metrics` has an entry, because `epsilon` and `freshness_ms` are required (FR-TD-04); a type entry that omits a label from its `policy`, or names a label absent from `metrics`, is a boot error. The Required column applies to a type entry: a device node carries the same shape but is governed by its own `policy` row — it may name only some of its labels, each with any subset of the four `policy` fields — and identity is the type's alone (Decision 20).

| Field | Required | Notes |
| --- | --- | --- |
| `epsilon` | yes | noise threshold, in the metric's unit (Decision 9) |
| `freshness_ms` | yes | staleness timeout (FR-IS-03) |
| `absence` | no (default `normal`) | severity while unavailable, on the `normal`/`advisory`/`warning`/`critical` scale (FR-TD-11, §7.1, Decision 9) |
| `limits` | no (default `[]`) | array of `{threshold, side, level}`: `threshold` finite in the value's unit, `side` `high` or `low`, `level` on the fixed named scale `normal`, `advisory`, `warning`, `critical` (FR-TD-11, §7.1) |

**Group type library.**

```json
{
  "version": 1,
  "types": {
    "rack_a": {
      "channels": [
        { "id": "avg_temp", "unit": "C", "reduction": "mean" }
      ],
      "policy": {
        "avg_temp": {
          "epsilon": 0.2,
          "limits": [
            { "threshold": 35, "side": "high", "level": "warning" }
          ]
        }
      }
    }
  }
}
```

**Group library fields.**

| Field | Required | Notes |
| --- | --- | --- |
| `version` | yes | library revision; equals the topology's `requires.group` (Decision 9) |
| `types` | yes | map of group type names to type entries — `channels` (identity: `id`, `unit`, `reduction`) and `policy` (the tunable fields, keyed by channel id); a group's `type` resolves here (Decision 9); instantiating a type on a group with no devices is legal and pruned (Decision 21) |

**Channel identity fields.**

| Field | Required | Notes |
| --- | --- | --- |
| `id` | yes | unique within the group |
| `unit` | yes | contributors match (FR-AG-02); for `count`, the unit of the readings tallied — it constrains contributors only, and the value is a dimensionless tally (Decision 9) |
| `reduction` | yes | `sum`, `mean`, `min`, `max`, `count` |

**Channel policy fields.** One `policy` map per type entry, keyed by channel id: every id in `channels` has an entry, because `epsilon` is required (FR-TD-05); a type entry that omits an id from its `policy`, or names an id absent from `channels`, is a boot error. The Required column applies to a type entry: a group node carries the same shape — with `channels` declared its `policy` covers every id, without `channels` it refines its template's fields — and identity is never overridable (Decision 20).

| Field | Required | Notes |
| --- | --- | --- |
| `epsilon` | yes | noise threshold, in the value's unit — counts for `count` (Decision 9) |
| `limits` | no (default `[]`) | array of `{threshold, side, level}`: `threshold` finite in the value's unit, `side` `high` or `low`, `level` on the fixed named scale `normal`, `advisory`, `warning`, `critical` (FR-TD-11, §7.1) |

**Topology document.**

```json
{
  "version": 1,
  "requires": { "group": 1, "device": 1 },
  "nodes": [
    {
      "kind": "group",
      "id": "site",
      "parent": null,
      "channels": []
    },

    {
      "kind": "group",
      "id": "bldg-a",
      "parent": "site",
      "channels": [
        { "id": "mean_temp", "unit": "C", "reduction": "mean" }
      ],
      "policy": { "mean_temp": { "epsilon": 0.1 } }
    },

    {
      "kind": "group",
      "id": "rack-1",
      "parent": "bldg-a",
      "type": "rack_a",
      "aabb": [[11.5, 3.0, 7.5], [13.0, 4.0, 8.5]],
      "policy": { "avg_temp": { "epsilon": 0.3 } },
      "contributes_to": { "avg_temp": ["mean_temp"] }
    },

    {
      "kind": "device",
      "id": "dev-1",
      "parent": "rack-1",
      "type": "monitored_rack_node",
      "position": [12.0, 3.5, 8.0],
      "policy": {
        "cpu_temp_1": { "limits": [ { "threshold": 70, "side": "high", "level": "warning" } ] }
      },
      "contributes_to": { "cpu_temp_1": ["avg_temp"] }
    },

    {
      "kind": "device",
      "id": "dev-2",
      "parent": "rack-1",
      "type": "std_rack_node",
      "position": [12.5, 3.5, 8.0]
    }
  ]
}
```

**Topology document fields.**

| Field | Required | Notes |
| --- | --- | --- |
| `version` | yes | revision of this topology document; incremented when the topology changes (FR-BR-05 for the benchmark topology) |
| `requires` | yes (when any node declares `type`) | one entry per type library — `group` and `device`; each supplied library's `version` equals its entry, and a missing library, an unknown type name, or a mismatch is a boot error (Decision 9) |
| `nodes` | yes | flat list of nodes; only the relative order of siblings is significant — a parent need not precede its children, and subtrees may interleave (Decision 8, Decision 9) |

**Group node fields.**

| Field | Required | Notes |
| --- | --- | --- |
| `kind` | yes | `group` |
| `id` | yes | unique topology-wide (referenced by `parent`) |
| `parent` | yes | a group id, or `null` for the single root |
| `type` | no | shallow template reference into the group type library (Decision 9, Decision 20) |
| `aabb` | no | `[[min],[max]]`, finite with min ≤ max per axis; contains every descendant device position, checked at boot (Decision 22); never templated |
| `channels` | no | ≤16 local channels (`id`, `unit`, `reduction`), in addition to the reserved attention channel; supplied by `type` when the node omits the key — an explicit array, `[]` included, replaces the template's channels and `policy` wholesale (Decision 9, Decision 20) |
| `policy` | no | map from channel ids to any subset of `epsilon`, `limits`; with `channels` declared it covers every id (`epsilon` has no default), without `channels` it refines the template's fields per channel; an id naming no channel of this node is a boot error (Decision 20) |
| `contributes_to` | no | map from this group's channel ids to its parent's channel ids; a channel may name several targets; never carried by a group type (Decision 9) |

**Device node fields.**

| Field | Required | Notes |
| --- | --- | --- |
| `kind` | yes | `device` |
| `id` | yes | unique topology-wide (referenced by `parent`) |
| `parent` | yes | a group id, or `null` for the single root |
| `type` | yes | shallow template reference into the device type library (Decision 9, Decision 20) |
| `position` | yes | `[x, y, z]`, finite |
| `metrics` | yes | one or more, supplied by `type` — a device node that declares `metrics` is a boot error; at most 15,000, so its complete value set — slot mask plus f16 values, ≈31 KB — fits within half the 64 KB message bound (FR-TR-02, §9.4, Decision 7, Decision 20) |
| `policy` | no | map from this device's metric labels to any subset of `epsilon`, `freshness_ms`, `absence`, `limits`; merge, defaults, and the unknown-label rule are stated in the Metric policy override bullet below (Decision 20) |
| `contributes_to` | no | map from this device's metric labels to its containing group's channel ids; a metric may name several targets; never carried by a device type (Decision 9) |

**What the example shows.**

- **Structure.** Three documents: the device library holds `std_rack_node` and `monitored_rack_node`, the group library `rack_a`, and the topology holds one root (`site`) and four descendants; key order inside an object is insignificant, but the order of the `nodes` array defines sibling order (Decision 8).
- **Inline channels.** `bldg-a` declares `mean_temp` directly — its identity in `channels`, its ε in the companion `policy`, because `epsilon` has no default; its parent, the root, declares an explicit empty `channels`, so the chain ends there.
- **Group template.** `rack-1` declares `type: "rack_a"` to inherit `avg_temp`'s channels and `policy`, refines only its ε to 0.3 with a node-level `policy`, and declares `contributes_to` itself: a group type carries definitions only, never wiring, because upward wiring names the parent's channels (Decision 9, Decision 20).
- **Device templates.** Both types supply the labels `cpu_temp_1` and `cpu_vib` with the same units, and they differ only in `policy`: `monitored_rack_node` adds an `absence` and two `limits` for `cpu_temp_1` that `std_rack_node` omits, so `dev-2`'s merged metrics fall back to `normal` and `[]`. `dev-1` takes `monitored_rack_node` and declares its contribution map plus a `policy` override on the node — the override replaces `cpu_temp_1`'s two limits with a single 70/high warning limit for that device alone, while its ε and freshness inherit from the type; `dev-2` takes `std_rack_node` and contributes nothing. Labels are device-local, so the two types may carry the same ones (Decision 7, Decision 20).
- **Contribution chain.** `dev-1.cpu_temp_1` (C) → `rack-1.avg_temp` (C, mean) → `bldg-a.mean_temp` (C, mean): every hop matches unit, and the channel-to-channel hop matches reduction. `dev-1.cpu_vib` (mm/s) contributes nowhere.
- **Boot order.** The libraries load first and each library's `version` is checked against its `requires` entry; group templates then expand before device templates, so contribution targets exist when validation runs (Decision 9, Decision 20).
- **Declared extent.** `rack-1` declares an `aabb` containing both of its devices; `site` and `bldg-a` declare none, so theirs is derived from their devices (Decision 22).

Configuration requirements:

- **Structure and validity.**
  - Unknown fields, and fields not applicable to a node's `kind`, are boot errors; the diagnostic names the document and the node or template (FR-TD-08).
  - The three documents load as one definition: both type libraries first, then the topology. A topology that declares a `type` carries a `requires` entry per library, and each library's `version` equals its entry; a missing library, an unknown type name, or a mismatch is a boot error (FR-TD-09, Decision 9).
  - **Single-rooted.** Exactly one node has `parent: null`; every other `parent` resolves; there are no cycles; every device is reachable from the root; ids are unique. Groups may be empty: a group with no device link — empty, or inherited from a type with nothing attached — is accepted and excluded from the index at build, not rejected at boot (FR-TD-07, Decision 21).
  - Every device has a fixed, finite position and one or more metrics, each with a **device-local `label`** unique within the device; a duplicate label is a boot error, and labels are not a shared vocabulary (FR-TD-03/04, Decision 7).
  - A group's declared `aabb` contains every device position in its subtree; a position outside it is a boot error (Decision 22).
  - `epsilon` is finite and ≥ 0 for every metric and channel, and every `freshness_ms` is a positive integer; violations are boot errors (Decision 9).
- **Channels and contributions.**
  - A group defines at most 16 channels, in addition to its reserved attention channel; a 17th is a boot error (FR-TD-05).
  - A metric contributes to channels of its containing group, declared on its device node as a `contributes_to` map from metric labels; a group's `contributes_to` map declares which of its channels feed which of its parent's channels. Both default to none. Fan-out and fan-in are allowed, and a channel may feed several parent channels (FR-TD-04/06, Decision 9).
  - Contribution wiring is not template content: neither type library carries it — a device node maps its metric labels to its containing group's channels, a group node maps its channel ids to its parent's channels, because both maps name another node's namespace (Decision 9, Decision 20). Absent `contributes_to` is legal: the group's channels keep their values and feed nothing above (FR-TD-06).
  - Contributions combine only same-unit values, and a channel joins only a parent channel of the same reduction; violations are boot errors (FR-AG-02, FR-TD-06/08).
- **Templates and expansion.**
  - A group declaring neither `type` nor `channels` has no channels (Decision 9).
  - Every device declares a `type`, and its metric set comes from that entry in the device type library; a device node that declares `metrics` is a boot error, as is a `type` that yields zero metrics after expansion (FR-TD-03, Decision 20).
  - Expansion merges the type's identity, the type's `policy` entry, the node's `policy` fields, and the node's `contributes_to` into one metric instance: after boot each instance carries its `label`, unit, noise threshold, freshness timeout, limits, absence, and its contributions — FR-TD-04 requires the label, unit, noise threshold, and freshness timeout of every instance, and its contributions if any (Decision 20).
  - **Metric policy override.** A device node may declare `policy`: a map from its metric labels to any subset of `epsilon`, `freshness_ms`, `absence`, `limits`. Each declared field replaces the type entry's value for that label — a field is replaced whole, so an array field is never merged element by element — and each absent field inherits it; `absence` and `limits` absent from both take their defaults. A key naming none of the device's metrics is a boot error, and neither `label` nor `unit` is overridable: identity stays with the type (FR-TD-04, FR-TD-11, Decision 20).
  - **Channel policy override.** Wherever channels are declared — a type entry or the node itself — a companion `policy` map keyed by channel id supplies `epsilon` (required) and `limits` (default `[]`) for every declared id; an id the map omits, or one naming no channel of its declaring source — the type entry's `channels` or the node's — is a boot error. A group node may declare `policy` without `channels`, refining its template's fields per channel: a declared field replaces the type's value whole, an absent field inherits it; a node that declares `channels` replaces the template's channels and `policy` together. `policy` never reaches `id`, `unit`, or `reduction` (FR-TD-05, FR-TD-11, Decision 20).
  - Geometry is never templated: a device's `position` and a group's `aabb` belong to the node itself, because absolute coordinates hold only for that instance (Decision 20, Decision 22).
  - Templates are shallow and single-level: a node may reference one `type` — a group's in the group library, a device's in the device library; a declared `channels` array replaces the template's channels and `policy` for a group, and a device declares no metric set of its own — its `policy` map refines only the four policy fields, never the metric list or identity. There is no type-extends-type and no per-item merge of channel definitions or metric sets; policy fields refine whole, never element by element (Decision 9, Decision 20).
  - The asymmetry is deliberate: a device's `type` is required, because the simulator configuration addresses device types and no instance ids (§4.2); a group's `type` is optional, because no external configuration addresses groups (Decision 9, Decision 20).
  - Templates expand at boot before validation: group templates first — with each group node's own `channels` and `policy` resolved into its channel set — then device templates, so a device node's `contributes_to` resolves against its containing group's channels as they stand after the group template (Decision 20).
  - Expansion is deterministic and identical for the simulator, the engine, and the client (FR-TD-09); the runtime still holds one metric instance per `(device, label)` — no template or kind reaches the runtime (Decision 7, Decision 20).
- **Hierarchy scope.**
  - The conventional site → building → room → rack levels are not required; the engine models a generic group tree (FR-TD-01). A physically meaningful hierarchy is assumed for useful level-of-detail; a degenerate or flat hierarchy is accepted and collapses to a single blended entry when zoomed out (FR-VS-05, §8).
  - The same hierarchy drives both spatial lookup and aggregation; there is one structure, not two (Decision 1).

### 4.2 Reading Model and Simulator Contract

- Each reading is `(device_index, metric_slot, value)`; the device id → `device_index` and the `(device, metric_label)` → `metric_slot` bindings are resolved at boot from the shared config, identically by the simulator and the engine, so no string key reaches the runtime, and the value is applied to the state table (Decision 7).
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

The simulator emits this record in the engine's ingest format directly. There is no runtime middleware; the only translation is the boot-time, per-device `metric_label` → `metric_slot` binding (and the device id → `device_index` binding), performed identically by both sides. The ingest record is a compact little-endian struct: `u32 device_index, u16 metric_slot, f32 value` (10 bytes), optionally followed by a `u64 sample_timestamp` (Decision 7).

The generator supports:

- Configurable global EPS (up to 300,000) or a per-device-type reporting interval.
- Per-metric value models for temperature, vibration, pressure, and power, resolved by the metric's unit (simulator profile).
- Within-noise jitter (exercises the ε filter), steps/ramps, and short spikes (exercise change suppression and short-lived-event visibility).
- Spatial correlation across neighbouring racks/rooms for realistic LOD blending.
- Offline transitions and dropped/late samples.
- Deterministic seeding from the profile's seed for reproducible benchmark traces (FR-CD-07).

At full scale the simulator emits a compact binary ingest record over a non-blocking channel (or runs in-process); it does not serialise to text at 300k EPS. The JSON representation in PRD-001 §7.1 is the K5 measurement baseline, not the ingest format.

#### Simulator Profile

The simulator's operating parameters live in a **simulator profile** — the simulator's own configuration file — a versioned artifact published with the benchmark report (FR-BR-04, FR-BR-05) and shipped with the demonstration packages.

- **Keyed by `unit`.** A model describes physics rather than identity, so metrics that share a unit share a model; metric labels are device-local and are not a key (Decision 7).
- **Per unit:** baseline and range, noise scale, step and spike cadence, offline probability, and the seed that makes traces deterministic (FR-CD-07).
- **Noise scale.** A unit's noise scale is set at or above the ε of the metrics that use it, so within-noise jitter both exercises suppression and crosses it.
- **Rates.** `reading_rate` and `reporting_interval_ms` are read from this configuration and never from the topology, which stays the single source of identities, positions, and channels (FR-TD-09, Decision 23).

**Rate fields.**

| Field | Level | Required | Notes |
| --- | --- | --- | --- |
| `reading_rate` | top-level | yes | `{ "global_eps": N }` — target readings per second across the topology, divided evenly across metric instances |
| `reporting_interval_ms` | per device type | no | keyed by a name in the device type library; emit period applied to each metric of every device of that type — one reading per metric per interval — replacing those devices' even share, which is not redistributed |

- **Division.** `reading_rate.global_eps` is a positive target for the whole topology, divided evenly across metric instances; a device type's positive `reporting_interval_ms` gives each of its devices one reading per metric per interval, replacing those shares, which are not redistributed, so offered EPS equals `global_eps` plus the net effect of the overridden types (FR-BR-05, Decision 23).
- **Validation.** The simulator rejects a non-positive `global_eps` or `reporting_interval_ms`, and an override key that names no entry in the device library, before it starts emitting; the topology validator never sees either field, so a rate fault surfaces at simulator start-up rather than at engine boot (Decision 23).

### 4.3 Memory Layout

The index is a set of nodes laid out in **one flat, parent-first array** (Decision 8). Each node is a group (at any level) or a leaf (device). A leaf stores an **offset into the separate state table**, never its reading (see below). Each group's members occupy **one contiguous subtree**; a group's devices occupy a **contiguous block of state rows** (the contiguous-block invariant, below).

Three properties hold (R1, R3), each checked at boot (§5.3):
1. **Contiguous groups** — a group's subtree is a single contiguous range.
2. **Parent-first layout** — a parent always precedes its descendants.
3. **Skip-offset pruning** — each internal node stores the size of its subtree, enabling a whole group to be skipped or emitted without descending.

The engine separates its memory by access pattern into the index — the flat node array plus the boot-written global channel-def, limit, and contribution arrays — the state table, and the per-frame aggregate buffers:

| Array | Contents | Written | Read |
| --- | --- | --- | --- |
| Index | Node kind, geometry box, LOD box, child access + subtree size, channel definitions, limits, contribution sources, leaf state offset | Once at boot | Every frame: per viewer (visibility, encode); once per frame, shared (aggregation) |
| State | Latest value and availability per `(device, metric)` | Continuously, at ingest | Once per frame, per group (aggregation); per viewer, per device entry (encode) |
| Aggregates | Per-group per-channel accumulators (including the reserved attention channel) for groups retained in the index; per-device attention level | Once per frame | Per viewer, per frame |

- The **index** is a flat, parent-first, preorder array in structure-of-arrays form to keep traversal cache-friendly; the array position is the node's index and identity (Decision 11), so it is not stored redundantly. Nodes reference state by offset and range, never by value. The per-node field set is listed at the end of this section.
- The **state table** is separate from the index and laid out **device-major in CSR form** — a compact **f32** `values` array (source precision; f16 quantisation happens only in transport, §4.2) plus a per-device `row_offsets` array — with devices in index leaf order and each device's declared metrics contiguous within its row in local-slot (declaration) order. Labels are device-local (§4.1), so a slot is meaningful only within its device and there is no global metric column. A device's row is `values[row_offsets[d] .. row_offsets[d+1]]`; a group's devices occupy a contiguous range of rows, so their values occupy one contiguous block. At boot the compiler records, per channel, the sorted absolute offsets of its contributing metric instances, so the aggregation pass walks a bounded, prefetch-friendly gather inside that block (FR-AG-02, K9); a metric feeding several channels appears in several such lists. A device's whole value set is contiguous, which suits the per-entry encode path (§9.3). The layout is fixed by Decision 7.
- Rows are exact: a device's compact row holds only the metrics it declares, so there are no unused slots and the table is `Σ metrics/device` values plus `N+1` `u32` row offsets (~400 KB of offsets at 100k). At ~3 metrics/device the full-park table is ~300,000 values, so the table is small.
- **Channel accumulators** are stored per group for its local channels and double-buffered: the frame reads one buffer while the sweep writes the other, then swaps (§7, Decision 5).

**Index node shape.** The array position is the node's index and identity (Decision 11). Per node:

| Field | Read by | Purpose |
| --- | --- | --- |
| Kind (leaf device / internal group) | Visibility | Finalise a leaf as a device instance and a group as a group entry; subtree size alone cannot distinguish a device from a group (FR-TD-07). Groups with no device descendant are excluded from the index (§5.2, §8.2, Decision 21). |
| Geometry box (AABB min/max) | Visibility | Frustum test and the extent the client draws; the declared `aabb` when present, else the derived box; contains every descendant device position (Decision 22). |
| LOD box (AABB min/max) | Visibility | Projected on-screen height, expansion order, and the entry-budget ranking; always the union of the node's devices' positions (Decision 22). |
| Subtree size (node count) | Visibility | Subtree bounds and skip-offset pruning; `skip_index = index + subtree_size` is derived, not stored. Node count only — no device-count extent is needed (aggregation uses per-channel offset lists, Decision 7). |
| Child access | Visibility | `child_count` gives `deg(v)` in O(1) for the expansion-budget check; `first_child = index + 1` and `next_sibling = sibling + subtree_size` are derived, not stored (Decision 8). |
| Channel definitions | Aggregation, Encoder | A slice `channel_def[channel_base .. +channel_count]` of the global channel-def array: per channel, unit, reduction, ε, and a range into the global limit array. ≤16 per group, fixed at boot. |
| Contribution sources | Aggregation | Per channel, a slice of the global contribution array: sources tagged `seed` (state offset) or `merge` (contributing channel-def index), indexed by target channel (pull, Decision 10). |
| Leaf state offset | Encoder | Start offset of the device's row in the compact state array. |

**Channel, limit, and contribution arrays.** Variable-length per-node data lives in global arrays addressed by ranges rather than inlined. A global **channel-def array** holds one entry per configured channel (`unit`, `reduction`, `epsilon`, `limit_base`, `limit_count`, `contrib_base`, `contrib_count`); a group's channels are contiguous via its `channel_base`/`channel_count`. A global **limit array** holds `{threshold, side, level}`. A global **contribution array** holds one entry per source, tagged `seed` (source = state offset) or `merge` (source = contributing channel-def index); a channel's sources are contiguous via its `contrib_base`/`contrib_count`.

The aggregation plan *is* this contribution array, indexed by **target channel** (pull, Decision 10): the reverse scan reduces each channel's source list — seeds in device-leaf order, then merges in child order — so the reduction is a single vectorisable list and its order is explicit for exact `sum`/`mean` (Decision 14). A seed contributes according to the target channel's reduction — the reading's value for `sum`, `mean`, `min`, and `max`, and `1` for `count` — while a merge adds the child channel's finalised accumulator (FR-AG-01, Decision 10). The reserved attention channel is a per-group accumulator slot outside the channel-def array and is never a contribution target.

### 4.4 Engine Configuration

Parameters that are not part of the topology (PRD-001 §8) live in an engine configuration file:

| Parameter | Default | Requirement |
| --- | --- | --- |
| On-screen-size enter threshold | 120 px | FR-VS-06 |
| On-screen-size exit threshold | 100 px | FR-VS-06 |
| Budget hysteresis `Δ` | 10% of `B` | §8.3, Decision 19 |
| Camera-context timeout | 1 s | FR-VS-08, Decision 19 |

These are engine-level settings and do not affect node or metric identities; the topology file (§4.1) remains the single shared source of identities (FR-TD-09).

## 5. Hierarchical Spatial Index

### 5.1 Rationale

A straightforward index of device positions cannot answer *both* "what is on screen?" and "what are this group's aggregate channel values?" within budget. The index therefore mirrors the semantic hierarchy and carries bounding volumes, so one traversal yields visibility and aggregation together (Decision 1).

### 5.2 Build Algorithm

The topology (nodes and parent-child edges) is fixed by the config; the build **derives** the flat index from it and never invents, reparents, rebalances, or spatially reorders the tree (Decision 1). A pre-pass resolves `type` references against the loaded libraries, buckets each node under its parent in list order, and validates the tree (single root, acyclic, devices reachable); one DFS in that canonical order then produces the layout and all derived per-node fields:

1. **Preorder (on entry):** assign `id = position`, emit the node into the flat array, record its kind (leaf device / internal group) and its channel definitions, and record a leaf's state offset. Recurse into children in canonical (list) order (Decision 8).
2. **Postorder (on exit):** set a group's **LOD box** to the union of its children's LOD boxes (a leaf's box is its device position) and its **geometry box** to the declared `aabb` when present, else to that same union (Decision 22); set `subtree_size = nodes emitted in this subtree` (node count) and `child_count` = emitted direct children. This also yields the parent-first flattening and the contiguous-subtree invariant directly.
3. Record the **leaf order** (devices in DFS visit order) — the order used to lay out the state table's device dimension (§4.3) — and each group's **contribution sources** (per local channel, the metric instances and child channels that feed it, validated for same unit and same reduction), including each channel's sorted list of contributing state offsets for the aggregation pass (§4.3, §7).
4. Bind each device's metric labels to per-device slots (`metric_label → metric_slot`) and validate the invariants below.
5. Run a **cheap post-build invariant scan** (contiguity, parent-first, skip-offsets, contribution integrity).

A single pass therefore yields ids, layout, AABBs, subtree sizes, child counts, leaf order, channel definitions, contribution sources, and state offsets. Because a parent's LOD box is the union of its children's, it is **order-invariant**; ordering siblings spatially cannot tighten any bound or improve pruning, so no spatial construction (median split, SAH, or BVH build) is performed. Pruning quality is inherited from the config's containment structure.

A leaf's AABB is the device position (a degenerate point); frustum tests are inclusive and the brute-force reference uses the identical point test (Decision 12). Groups with no device descendant are excluded from the index, decided bottom-up after device placement, so a retained group's LOD box is always the union of a non-empty set. Merge sources whose child group is excluded are dropped at build — the child channel is valueless by construction (FR-AG-04) — so every contribution source resolves to a recorded channel definition (Decision 21).

### 5.3 Invariants and Validation

| Invariant | Check | Failure response |
| --- | --- | --- |
| Contiguous subtrees | Offset/size scan | Abort boot with diagnostics |
| Parent-first ordering | Index ordering scan | Abort boot |
| Skip-offset correctness | Recompute vs stored size | Abort boot |
| Child enumeration consistency | `child_count` equals emitted direct children; `first_child = index + 1` | Abort boot |
| Per-device state-row integrity | Row-offset monotonicity; row `d` aligned to leaf-order device `d`; total = Σ metrics/device | Abort boot |
| Contribution unit and reduction consistency | Each contributor's declared unit matches the channel's unit; each contributing channel's reduction matches the receiving channel's | Abort boot |
| Contribution wiring existence | Every device's `contributes_to` key names one of its metrics and each target a channel in its containing group; every group's `contributes_to` key names one of its own channels and each target a channel in its parent | Abort boot |
| Metric policy key existence | Every device type entry's `policy` covers exactly its `metrics` labels, each carrying `epsilon` and `freshness_ms`; every device node's `policy` key names one of its metrics | Abort boot |
| Channel policy key existence | Wherever channels are declared, the companion `policy` covers exactly those ids with `epsilon`; a refining group node's `policy` names only existing channels | Abort boot |
| Finite positions | Every device position is finite | Abort boot |
| Device containment | Every declared group `aabb` contains all of its descendant device positions | Abort boot |
| Metric-count ceiling | Each device's configured metrics ≤ 15,000, so a complete value set (slot mask plus f16 values, ≈31 KB) stays within half a 64 KB message | Abort boot |
| Epsilon and freshness | Every metric and channel ε is finite and ≥ 0; every `freshness_ms` is a positive integer | Abort boot |
| Group channel-count bound | Each group's configured channels ≤ 16 | Abort boot |
| Aggregate correctness | Brute-force scalar comparison on fixtures | Fail test |

Unit and reduction scans apply to configured channels and their contributions; the reserved attention channel is excluded (it combines only unitless levels).

**Validation order and diagnostics.** Checks run on the resolved config (§4.1) in stages: (1) schema — unknown and kind-inapplicable fields (FR-TD-08), (2) structural (FR-TD-07), (3) device/metric (FR-TD-04), (4) channels (FR-TD-05), (5) contributions (FR-TD-06/08), (6) limits/absence (FR-TD-11/08), then (7) the post-build invariants above. Validation is **fail-fast**: the first fault aborts boot with a structured diagnostic — `{code, node id, metric label / channel id, message}` — precise enough to locate the fault (FR-TD-07, Decision 12). Codes are stable and prefixed by stage (`E-SCHEMA`, `E-STRUCT`, `E-METRIC`, `E-CHANNEL`, `E-CONTRIB`, `E-LIMIT`, `E-INVARIANT`).

### 5.4 Correctness Oracle

Correctness is established against an **independent brute-force reference**, not a third-party spatial structure. The reference operates on the **flat device list directly** — it does not walk the index tree, so it does not share any traversal or pruning defect it is meant to catch. For the same `(topology, state, camera, selection)` it frustum-tests every device position (using the same inclusive point test as the engine, Decision 12), computes the ranked cut, and reduces the same readings; the engine's traversal agrees on the visible device-leaf set and its selection matches the reference cut. This mirrors the brute-force scalar reference used for aggregation (§7, §13), giving one independent reference per subsystem (R1).

The semantic n-ary tree has no shared basis with a binary spatial BVH, so the `bvh` crate is not used. Structural invariants (§5.3) cover the layout; the reference covers traversal and selection.

### 5.5 Traversal

- Descend the tree with frustum tests on geometry boxes.
- **Prune** a subtree when it is outside the frustum or when the group projects below the on-screen-size threshold (skip-offset pruning).
- **Emit** a group's aggregate channel values when it is visible but too small to be shown as individual devices.
- **Expand** to individual devices when the group is large enough on screen and budget remains.
- Keep a **scalar reference path** always available; any optimised path ships behind the same interface and behind the SIMD feature flag (R2, Decision 14). SIMD is applied to aggregation and encoding (§7, §9.2), not to the irregular, early-out-dominated traversal.

## 6. State Management and Ingestion

- A **lock-free, latest-wins state table** indexed by `(device, metric)`, holding exactly one current value per entry. Writers (ingest) store values atomically; readers (aggregation/encode) load the newest value (Decision 16).
- The ingest path is **non-blocking** and does not queue unboundedly; back-pressure is handled by overwriting stale values rather than buffering (Decision 16).
- A record naming an unknown `device_index`, a `metric_slot` outside that device's row, or a non-finite value is rejected without altering existing state (FR-IS-02); device ids and labels are bound at boot, so no reading carries a string key (Decision 7).
- Availability is tracked per `(device, metric)`: a metric is current while a reading for it arrived within its configured freshness timeout, and its value is treated as unavailable once stale; a stale metric does not affect the availability of the device's other metrics. A device is marked **offline** when all of its metrics are unavailable, and returns online on the next accepted reading (FR-IS-03, FR-IS-04).
- Each device's metrics are bound at boot to the **local channels of their containing group**. Aggregation reads the state slots of the metrics feeding each channel; a channel combines only same-unit contributions, and a contribution may join only a channel of the same reduction (FR-TD-05, FR-AG-02, FR-TD-06).
- The state table is a dedicated array, separate from the index (leaf nodes store offsets, not values). Its device dimension matches the index leaf order and each device's metrics are contiguous within its row, so each group maps to one contiguous block of rows and a device's value set is contiguous (K9, §9.3, §4.3). Aggregation reads each channel's boot-recorded contributing offsets within that block.

## 7. Shared Aggregation

- **One background pass per 60 Hz frame** computes every group's local channels bottom-up from a **boot-compiled plan** — seed ops (device metric → local channel) and merge ops (child channel → parent channel) — recomputed from scratch each frame. The pass runs continuously while serving, independent of viewer count (idle-gating deferred; Decision 5). The plan is indexed by **target channel** (pull): each channel stores its incoming sources, and the reverse scan reads children's finalised values from the write buffer (Decision 10).
- Each group defines up to **16 local channels** (identity local to the group; reductions `sum`, `mean`, `min`, `max`, `count`). A channel's value is its configured reduction applied to the **available contributions** wired to it: the group's own metric instances and its child groups' contributed channels. A group contributes nothing to its parent by default (FR-TD-05, FR-TD-06).
- **Same-reduction rule.** A channel may contribute only to a parent channel of the same reduction; boot validation rejects mixed-reduction contributions (FR-TD-06, FR-TD-08). With same-reduction contributions and accumulator merge, a group's value is the canonical subtree reduction, so K4's exact reference is well defined (FR-AG-01).
- Each channel carries mergeable accumulator state in **f32** lanes (16 channels ≈ one 512-bit vector, Decision 4): `mean` → `(sum, count)`, `sum` → `sum`, `min`/`max` → value, `count` → `count`. Seeds follow the same table: a seed contributes the reading's value, except in a `count` channel, where each available seed contributes `1` (FR-AG-01). A `count` accumulator is therefore exact below 16,777,216, the same bound as its wire encoding (§9.2). A group merges its children's channel accumulators; the published value is the finalised accumulator. `mean` is therefore exact over the subtree's available readings, and `count` is the total number of available contributing readings in the subtree, merged from child `count` channels by summation (FR-AG-01, K4).
- A channel combines one unit only (FR-AG-02), and unavailable (stale or offline) readings are excluded from the accumulation; a channel with no available contribution reports no value (FR-AG-04).
- Results are published as a consistent per-frame snapshot (double-buffered internally); viewers read the published buffer (FR-AG-03).
- Cost model: the shared pass is independent of viewer count; the per-viewer cost beyond it is **O(visible)**, not O(N) (R5, FR-AG-03).
- Aggregates match a brute-force reduction of the same contributions exactly (FR-AG-01, K4). Because floating-point addition is not associative, exact agreement requires an identical type (**f32**) and a **canonical reduction order** shared by the scalar reference and any SIMD path: a **blocked-lane order with fixed `W = 8`** — source `i` accumulates into lane `i mod 8`, and the eight lanes fold in a fixed order (Decision 14). The scalar engine and the brute-force reference implement it identically, so they are bit-exact; a SIMD kernel reproduces the same lane pattern. `min`, `max` and `count` are order-free. A channel's reduction over up to 1,000 contributing readings targets ≤ 5 µs scalar (K9); the SIMD path aims for ≤ 1 µs as an engineering goal.
- Contingency: reduce the configured channel set if the shared pass cannot meet K9; per-viewer cost stays independent of viewer count (FR-AG-03).

### 7.1 Attention channel (severity and absence)

Each group carries a **reserved unitless attention channel** (the group's attention value, reduction `max`), computed in the same bottom-up pass as its configured channels and streamed with the node's entry (FR-TD-11, FR-AG-06). Its value is on the fixed severity scale of normal, advisory, warning, and critical. A device's attention level is the greatest of its metrics' contributions. The scale is the fixed named scale of Decision 9 (FR-TD-11).

- **Severity limits.** A metric or channel may declare a list of limits, each `(threshold, side, level)` on the high and/or low side. A value maps to a level by `level = max over limits of ( fired ? level : 0 )`, where `fired` is `value >= threshold` (high) or `value <= threshold` (low). Limits inherit the value's unit and are validated at boot (finite threshold; `side` `high`/`low`; level on the fixed severity scale).
- **Contributions.** A metric contributes its limit level when available, or its absence level when unavailable (per-metric; availability from FR-IS-03). A configured channel contributes its limit level when it has a value.
- **Composition.** A device's attention level is the max over its metrics' contributions. A group's attention channel is the max over its configured channels' limit levels, its direct devices' attention levels, and its child groups' attention channels. Unavailable values contribute no level; a node with no available values and no nonzero contribution reports normal (0).
- **Propagation.** The attention channel merges as a `max` channel, order-free and exact, and is always contributed to its parent's attention channel, independent of the configured contributions (FR-TD-06, FR-AG-06).
- **Availability stays orthogonal.** NaN availability markers are streamed separately, so the client can distinguish an out-of-limit value from missing data.
- **Cost and storage.** A handful of compare-and-selects per value, folded into the existing seed/merge pass; one small unitless channel per group, double-buffered with the aggregates.
- **Scope.** Fixed bands only — no expressions, rate-of-change, hysteresis, or state. Fraction-based coverage policy is out of scope (PRD-001 §10).

## 8. Visibility Query and LOD Selection

### 8.1 Inputs

Per viewer, per frame: camera pose (position and orientation), screen width/height, pixel ratio, field of view (FR-VS-01). Without these the on-screen-size threshold is ambiguous. If the context is absent or late, the last known context is reused until a timeout, after which that viewer's stream is held (FR-VS-08).

### 8.2 Algorithm

The query produces a per-viewer, per-frame **visibility result**: a bounded list of *entries*, each either a device instance or a blended group entry. It selects a **cut** through the visible hierarchy — the set of nodes that each become one entry — under the entry budget (R5, K3, FR-VS-07). Values are not resolved here; the encoder reads them from the state table and aggregate buffers (§9).

The frontier (the current cut) is held in a **max-heap keyed on exact projected on-screen height**, not in a traversal stack, so selection is best-first: the largest visible node is expanded first, regardless of its depth or position in the array. The heap is a **fixed-capacity, per-viewer scratch buffer**, reused every frame and sized to the entry budget `B` (the frontier can never exceed `B`). The index (node ids, layout, AABBs) is immutable after boot (§4.3), so the heap carries only node references and per-frame heights; each node enters the frontier at most once per query.

**Projected on-screen height** is the vertical pixel extent of the node's **LOD box** — the union of its devices' positions (Decision 22): its eight corners are transformed to screen space at that box's centre depth, and the key is `max_y − min_y` in device pixels (pixel ratio applied). The engine and the brute-force reference compute the identical value, so the frontier's total order is reproducible (§5.4); the monotonicity of projected height down the tree relies on this centre-depth convention.

1. Project the root; if it intersects the frustum, push it onto the frontier with its projected height.
2. Repeat while an expandable candidate remains:
   - Pop the node with the **greatest projected height**. Exact height is the primary key; node id is the terminal tie-break, giving a total, deterministic order.
   - Finalise the node as an entry when any terminal condition holds: a device leaf becomes a device instance entry; a group whose contributors are all currently unavailable becomes a group entry carrying no value (FR-AG-04); a height below the applicable size threshold becomes a blended group entry (enter 120 px, or exit 100 px once expanded; §8.3); and a node whose expansion would breach the budget becomes a blended group entry. Groups with no device descendant never enter the index and are never emitted (§5.2).
   - A group whose **LOD box is degenerate** (a single device, or coincident devices) projects to 0 px at every distance, so the size threshold would blend it forever: it is exempt from the size threshold and expands when its parent expands, subject to the entry budget (Decision 21).
   - Otherwise **expand** it: remove it; for each direct child, apply the frustum test and push only the visible children with their projected heights. Expansion is permitted only while `entries − 1 + deg(v) ≤ B` (`B` ≈ 5,000), and adds `deg(v) − 1` to `entries`, the running cut size (`finalised entries + frontier nodes`). `deg(v)` is O(1) from the node's `child_count` (§4.3), so a huge-fan-in node is rejected without enumerating its children.
3. Halt when no expandable candidate remains; every node still held is finalised (leaves as device instances, internal nodes as summaries). The output is the union of finalised nodes.

A child's LOD box is contained by its parent's LOD box, so projected height is **monotone** down the tree: a child is never taller on screen than its parent. Geometry boxes need not nest and take no part in this ordering. The frontier therefore always holds an upper bound for every unexplored subtree, the maximum-height pop cannot miss the largest remaining node, and no node is revisited.

Each entry carries its kind, its identity — the flat-array node index, resolved to a config identity through the connect-time dictionary (§9.3, Decision 11) — and a **transition flag** (appeared, removed, or expanded) relative to the viewer's previous frame; `expanded` is internal to visibility and surfaces on the wire as appeared entries for the newly revealed devices (§8.3, §9.4). Values are not resolved by the query: the encoder maps each index to its source at encode time (§9.3). The result is deterministic given `(topology, state, camera, selection)`, with `entries ≤ B`; the per-viewer frontier is bounded by `B`. The viewer's selected object (FR-VS-02) is force-included every frame, adding at most one entry beyond the cut.

### 8.3 Anti-Flicker

**Size hysteresis:** enter detailed view at 120 px, exit at 100 px (FR-VS-06, R7). A group whose LOD box is degenerate has no measurable size and is not subject to this band (Decision 21).

**Budget hysteresis:** the entry budget cuts the frontier independently of on-screen size, so a group near the budget edge can flip between blended and detailed as unrelated scene contents change — frontier ordering cannot absorb this. The running cut size therefore carries its own band: expansion is admitted while `entries < B`, but a node already expanded is retained until the cut falls below `B − Δ` (`Δ` defaults to 10% of `B`, §4.4), so the budget edge does not oscillate. The size and budget bands are independent and configurable.

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
scalar value ──► width convert ──► per-value ε filter ──► emit changed slots (absolute) ──► frame
                                         │
                                   raw-threshold guard (f16 slots only, ~>32 °C)
```

- Wire widths are set by reduction: metric values and `mean`, `min`, `max` channels are **16-bit floats**; `sum` and `count` channels are **32-bit floats**, whose range covers what f16 cannot hold (largest finite f16: 65,504).
- **Count exactness.** f32 represents integers exactly only below 16,777,216; a `count` channel is therefore exact while its subtree holds fewer contributing readings than that, and rounds above it.
- Every visible entry defines its **complete value set** — a device instance's metrics or a group entry's local channels — so the client can resolve and change the active display value without a server round-trip (FR-CD-03). Transmission follows FR-TR-02 (the full value set on first send or snapshot, changes thereafter), and the ε filter is applied per value.
- Each entry also carries the node's **attention value** (FR-AG-06) — for a group entry, its reserved attention channel; for a device instance, its attention level — sent on change; availability remains a separate axis.
- Updates computed in the **quantised wire domain** with absolute thresholds — a metric's ε for a device instance, a channel's ε for a group entry.
- The wire carries the **absolute value in its slot's wire width** for each changed slot; the baseline (the last value actually sent, in wire form) is only the ε-suppression reference. It advances **only on an actual socket write**, so slow drift accumulates until it crosses ε while the rendered value stays within its threshold of the current value (FR-TR-02, Decision 15). A dropped (latest-wins) frame does not advance the baseline.
- A **raw-threshold check** guards f16 slots where 16-bit rounding alone would under-filter (above ~32 °C); f32 slots need no guard.
- The convert → ε → change-suppression pipeline is contiguous and vectorisable (F16C for f16 slots; f32 slots pass through unchanged); it ships scalar-first behind the same SIMD feature flag (R2, Decision 14).

### 9.3 Session Semantics

| Concern | Behaviour |
| --- | --- |
| Sending state | Each session remembers last-sent values in wire form (f16 for most slots, f32 for `sum`/`count`); each entry resolves against its source at encode time — a device entry reads the state table via the leaf's `state_offset`, a group entry reads the aggregate snapshot for the node's local channels (§4.3). |
| Value set | Every visible entry's complete value set is defined (a device instance's metrics, or a group entry's local channels); the client resolves the active display value locally (FR-CD-03), and transmission follows FR-TR-02. |
| Attention | Each entry carries the node's attention value (a group entry's reserved attention channel, or a device instance's attention level); sent on change. |
| Keyframes | Full visible keyframe on (re)connect; appeared entries sent in full on expansion or visibility change; on-demand resync on a sequence gap; no periodic keyframe (Decision 15). |
| Unavailable values | Explicit NaN marker on the wire for an unavailable device metric or a channel with no available contribution, sent regardless of the ε filter. |
| Sequence | Frames carry a monotonic sequence; clients discard duplicates and request a keyframe on a detected gap. |
| Wire identity | Per-frame entries are keyed by the server node index (u32). On (re)connect the server sends the **dictionary** — every node in the index, in chunks (§9.5) — mapping each node index to `(kind, config id)`; the client resolves position, units, metric labels, and channel meanings from the shared config (FR-TD-09). Indices are per-build and not stable across restarts; a reconnect gets a fresh dictionary (Decision 11). |
| Slow clients | Stale frames are dropped (latest-wins on egress) and do not advance the baseline; per-session state is bounded (Decision 15). |
| Inspection target | A selected device or group is streamed every frame at full fidelity until deselected; the ε filter does not suppress it. |

**Session reconciliation.** The last-sent baseline is valid only on the intersection of the current visible set and the previous frame's set, so each frame reconciles the two:

| Case | Session state | Wire |
| --- | --- | --- |
| In both frames | baseline valid | changed slot sent as absolute, in its wire width, ε-filtered |
| **Appeared** (current, not previous) | baseline created | full value, `appeared` flag |
| **Removed** (previous, not current) | entry pruned | `removed` flag |

Keying is by node index, which is sufficient because the index is immutable and a node's kind never changes; a device↔blend transition is different indices (the group node leaves the set, its device leaves enter), each handled as appeared/removed. Reappearance after any gap is treated as **appeared** — the client may have dropped the instance, so a stale baseline would be invalid. Pruning on removal keeps per-session state bounded by the current visible set plus the selected object (FR-TR-04).

Implementation is a dense-integer diff — O(visible value slots) per viewer per frame, allocation-free, no hashing. Per session, a `u32` frame **epoch** array indexed by node detects presence and change eligibility without per-frame clearing (only visible nodes are written), and a `u16` **last-sent** array indexed by value slot holds the f16 wire-domain baseline, with a parallel f32 array for `sum`/`count` slots; removals are found by scanning the previous frame's node list (≤ B), never all N nodes. Baseline advancement on the transport occurs only when a frame is actually written, so latest-wins drop and change suppression compose. Table sizes: ~1 MB per viewer at 100k (~300k device-metric slots at 2 B, plus 100k epochs at 4 B); a group entry's `sum`/`count` slots add 2 B each in the f32 baseline.

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
    values[]             # slots in slot order; f16 = metric/mean/min/max,
                          # f32 = sum/count; quiet NaN of that width = unavailable
    [u8 attention]       # present only when entry_flags.attention-present
```

- A **device entry**'s value slots are its metric instances; a **group entry**'s are its local channels; slot order is the config order (Decision 7/9).
- `appeared` entries carry every slot; `removed` entries carry none.
- `appeared`/`removed` are the wire transitions; the visibility query's internal `expanded` transition (§8.2) is not a wire flag — it materialises as `appeared` entries for the newly revealed devices.
- `changed_mask` bounds the payload to changed slots; an entry with nothing changed and no attention change is omitted entirely.
- Attention is sent on change only (Decision 15).
- A keyframe is the full visible set with every slot present, chunked across frames with the `keyframe`/`keyframe_end` flags when it exceeds a size threshold (default ~64 KB). No wire message exceeds 64 KB; keyframe and dictionary chunks are cut at that bound.
- **Message filling.** Every message is filled to the 64 KB bound and cut between entries — a frame entry is never split — and the remainder continues in the next message with its own sequence, so a normal diff cycle may span messages. Each message is a complete diff unit: the client applies it directly, and the baseline advances only on the messages actually written (§9.3).
- Field widths are fixed per reduction: f16 slots use the quiet-NaN pattern `0x7E00`, f32 slots use the f32 quiet NaN. The client derives each slot's width from the shared config (FR-TD-09), so no per-slot type tag is sent on the wire.

### 9.5 Other Messages

Little-endian; each begins with a `u8 type` (direction-scoped).

```
Dictionary (server → client, type = 1):        # in chunks, before the first frame
  u8  dict_flags            # bit0 dictionary, bit1 dictionary_end
  u32 entry_count           # entries in this chunk
  Entry[entry_count]:
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

The dictionary covers **every node in the index** — `entry_count` is the total node count, ~101,781 at the full tier (≈ 2.4 MB with ~17-byte ids) — delivered in chunks cut at the 64 KB wire-message bound (§9.4), in ascending node-index order, complete before the first frame; the client accepts frames only after `dictionary_end`. It maps each node index to its kind and config id, and the client resolves positions, units, metric labels, and channel meanings from the shared config (FR-TD-09, Decision 11). The camera context is sent every frame (FR-CD-06); the inspection target only on change (FR-VS-02).

## 10. Browser 3D Client Design

The client and demo target the building tier (~4,280 devices). Browser KPIs K6 (frame rate) and K7 (memory) are verified at that scale; the engine itself is benchmarked across all three tiers (§14).

The client loads the same topology definition as the server (FR-TD-09) to map instance ids to device positions and metric units, so positions are never transmitted per reading.

A group entry's transform comes from the same config: its position and extent are those of the group's **geometry box** — the declared `aabb` when present, otherwise the union of its descendant devices' positions (FR-TD-09) — so what is drawn is what is culled. The LOD decision uses the derived box instead (§8.2), so expansion still follows the spread of a group's devices (Decision 22). Shape and marker size are client-side rendering choices, with a minimum visible size for a degenerate box; a group's AABB is a bounding box, not a model of the object.

### 10.1 Rendering

- Three.js `InstancedMesh`; per-frame instance transforms and colours written into a pre-allocated VBO to avoid allocation-driven stalls (FR-CD-01, K7). Individual device instances and blended group entries are drawn from the same instanced buffers. Each instance's colour is derived from the selected display value (§10.4) (Decision 17).
- Target: stable 60 FPS, main-thread overhead < 3 ms/frame (K6).
- Fallback (R4): rebuild mesh per frame; drop LOD transitions.

### 10.2 Camera Publisher

Each frame the client sends its camera pose (position and orientation) and screen geometry to the server (FR-CD-06), closing the camera-feedback loop that enables server-side culling and blending.

### 10.3 Picking

Object picking / click-to-inspect (FR-CD-04). Primary path uses instance id hit-testing; fallback to CPU `Raycaster` if needed (R4). The inspected value is delivered by the targeted per-session stream (§9.3), not a server query; for an individually streamed instance the client already holds the current value.

### 10.4 Display Mapping

Each rendered object is coloured by one selected value (FR-CD-03). The default is the object's **attention value** (a group's reserved attention channel, or a device's attention level), which gives a consistent fixed severity scale for comparing devices and groups. The viewer may instead select a metric (for a device instance) or one of the group's local channels (for a group entry); because a group's channels are local to it, these selectable values are resolved per group from the shared config. Switching is a client-only change, since every visible entry's complete value set is already streamed (§9.3). The selected object's values are shown with their units (FR-CD-05), except a `count` channel, whose tally is shown with no unit (Decision 9).

## 11. Performance Budgets

Every budget below is derived at the benchmark topology (measurement scope, §2); each stage scales with the quantity it walks — index nodes, entries, channels and contributors, or value slots.

| Stage | Budget | KPI |
| --- | --- | --- |
| Server traversal | ≤ 1.5 ms | K2 |
| Shared aggregation pass | ≤ 0.5 ms for the benchmark topology's configured channel set; scales with channels configured (16 per group, plus the reserved attention channel) | K9 |
| Per-viewer encode | ≤ 0.3 ms at the benchmark's visible value slots; scales linearly with visible value slots | — |
| Network + jitter | ≤ 2 ms | — |
| Browser parse/write | ≤ 3 ms | K6 |
| Rendering | remaining ~10 ms | K6 |

Named budgets sum to ≤ 7.5 ms; overruns do not cascade across frames. K2 measures the server-traversal stage only; these stage budgets compose into the end-to-end freshness target (K11).

## 12. Risk-to-Design Mapping

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

## 13. Testing and Validation Strategy

| Layer | Technique | Covers |
| --- | --- | --- |
| Unit | Per-module tests from day one | Build, aggregation, encoding |
| Property | Randomised/permuted fixtures | R1, R3, K4 |
| Ingestion | Reject unknown `device_index`, out-of-row `metric_slot`, and non-finite value; assert prior state is unchanged and liveness is not refreshed; latest-wins overwrite and freshness timeout on fixtures | FR-IS-01, FR-IS-02, FR-IS-03, FR-IS-04 |
| Composition | Per-group channel contributions, same-reduction joins, mixed-unit rejection, mean accumulators | K4 |
| Attention | Limit-band mapping and absence levels vs a scalar reference; max-propagation property tests | FR-TD-11, FR-AG-06 |
| Differential | Engine traversal/selection vs flat-list brute-force reference | R1 |
| Selection | Exact-height heap refinement vs brute-force ranked cut | R5, K3, FR-VS-07 |
| Boundary | Empty and device-less groups (accepted, pruned from the visible set, parents report the remaining contributions per FR-AG-04); degenerate groups (single device, coincident devices) expand with their parent; non-multiple-of-8 counts | R2 |
| Degradation | Flat hierarchy (all devices under the root) returns a single blended entry within budget, with exact aggregates | R5, G2 |
| Invariant | Boot-time structural scan; declared-`aabb` containment; geometry box ⊇ LOD box | R1, R3 |
| Integration | Backend alpha / end-to-end | FR-CD-08 |
| Transport | Wide-slot encode/decode round-trip: f32 `sum`/`count`, ε after width conversion, per-width unavailable marker, count above 65,504; dictionary chunk reassembly across the 64 KB bound | FR-TR-02, FR-TR-03, K5 |
| Ablation | Sibling ordering on/off (locality hypothesis) | R1 |
| Ablation | SIMD on/off (aggregation and encode) | R2, K9 |
| Ablation | ε suppression on/off (egress) | K5 |

Aggregation is validated against a **brute-force scalar reference** and matches exactly on all fixtures (K4); both use the same blocked-lane order (Decision 14).

## 14. Benchmark Methodology

### 14.1 Benchmark host

The backend host is fixed before benchmarking (FR-BR-01). These fields are recorded when the architecture is frozen; concrete values are filled before the first benchmark run.

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

The K5 raw-streaming baseline is fixed in PRD-001 §7.1 (full-key JSON, ~100 B/reading). Baseline performance numbers are produced by the benchmark runs of §14.

### 14.2 Fixed benchmark topology and simulator configuration (FR-BR-05)

The benchmark topology and its simulator configuration are produced by a **deterministic, seeded generator** (version `bench-v1`): it emits each tier's topology and the type libraries shared by all three tiers in the schema of §4.1, plus the tier's simulator profile (§4.2). The generated files are the published artifacts; the spec plus seed reproduce them exactly (Decision 13).

- **Tier shapes** (bounded fan-in; exact device counts):

| Tier | Devices | Buildings | Rooms/bldg | Racks/room | Devices/rack |
| --- | --- | --- | --- | --- | --- |
| Building | 4,280 | 1 | 8 | 10 | 54 / 53 (half each) |
| Intermediate | 50,000 | 10 | 8 | 10 | 63 / 62 (half each) |
| Full | 100,000 | 20 | 8 | 10 | 63 / 62 (half each) |

  That is 80 / 800 / 1,600 racks; the half-and-half split lands each tier's total exactly.
- **Metrics** (~3/device): `temp` (C), `vib` (mm/s), `power` (kW), each with ε, a freshness timeout, an absence level, and severity limits.
- **Channels:** per level — for example a rack `rack_avg_temp` (mean, C) fed by its devices, a room `room_avg_temp` merging rack channels, and building and site merging upward, same reduction and explicit contributions.
- **Positions:** deterministic grid — building along x, room on y (floor), racks on an x/z grid within a room, devices at small in-rack offsets.
- **Reading rates:** PRD-001 §7.1's aggregate is normative for each tier — ~300,000 / ~150,000 / ~37,000 readings/s — and each tier's simulator profile writes it as `reading_rate` (§4.2); the per-instance rate is `global_eps` ÷ metric instances (~1/s at full and intermediate, ~2.9/s at building). No benchmark device type declares `reporting_interval_ms`, so offered EPS equals the tier's `global_eps`.
- **Templates:** the libraries are identical across tiers and supply `rack_a` for rack groups and a device type entry for each device kind, so every device resolves one and each generated topology stays compact (Decision 9, Decision 20).

### 14.3 Runs

- Tiers: 4,280 / 50,000 / 100,000 devices, each with a fixed metric set per device (~3 metrics/device), so state size and boot cost are reproducible. All tiers run the same engine binary with no code change (G5).
- Baseline reading rates per tier are fixed in §14.2 and carried in each tier's simulator profile as `reading_rate` (§4.2); the simulator reproduces them.
- Client-side KPIs K6 and K7 are measured in the demo application at the building tier (~4,280 devices), not at the 100k tier.
- End-to-end freshness (K11) is measured in the demo application with instrumented timestamps at the building scale.
- Stress: ≥ 300,000 EPS for ≥ 30 min (K1).
- Visibility latency: p99 ≤ 1.5 ms per viewer per 60 Hz frame at 100k, across camera poses including max zoom-out, engine instrumentation only (K2).
- Boot: ≤ 30 s from topology file to fully loaded and serving at 100k devices (K8).
- Concurrent-session run: ≥ 10 browser sessions at 60 Hz at the full-scale tier (100,000 devices); ≤ 50% aggregate server CPU — engine-process CPU across all its threads, summed and expressed as a fraction of total host capacity, taken as the mean over the steady-state window with the maximum reported alongside (K10); per-client egress within K5.
- Micro-benchmark: per-channel group aggregation (K9). The measured channel is constructed with 1,000 contributing readings, since no channel of the benchmark topology reaches that fan-in — the widest is a rack channel at ~63 devices × 1 same-unit metric each ≈ 63 (§14.2, FR-AG-02) — so the ceiling is exercised synthetically, not sampled from the topology.
- Ablation (FR-BR-03, Should): sibling ordering on/off (locality hypothesis; expected nil, since a parent's LOD box is order-invariant); SIMD on/off for aggregation and encoding (informs the SIMD ship decision, Decision 14); ε suppression on/off (egress, K5).
- Simulator pinned via `taskset`; non-blocking I/O; pre-generated traces as contingency (R6).

## 15. Build and Tooling

- Rust workspace: `engine`, `transport`, `simulator`, `benches` crates; `cargo nextest` tests; `criterion` benchmarks; `cargo-flamegraph`/`perf` profiling.
- Nightly toolchain for `portable_simd` (optional, behind a feature flag).
- Frontend: Vite + TypeScript + Three.js under `client` and `demo`.
- `taskset` for benchmark core isolation.
- **Run diagnostics (FR-CD-09):** each component logs a structured boot summary and any validation fault, and exposes counters for ingest rate, rejected readings, aggregate-pass time, and per-session egress; the demo surfaces them in a status panel.
- Everything is developed in the open: commits, design notes, and benchmark numbers land in the public GitHub repository as the work happens.

## 16. Requirement Traceability

| PRD requirement | TDD section |
| --- | --- |
| FR-TD-01–FR-TD-11 (topology/channels/index/limits) | §4, §5, §6, §7 (§7.1 for FR-TD-11) |
| FR-IS-01–FR-IS-05 (ingest/state/offline) | §6, §9.3 |
| FR-AG-01–FR-AG-06 (aggregation/attention) | §7 (§7.1 for FR-AG-06) |
| FR-VS-01–FR-VS-08 (visibility/LOD/budget/inspection) | §8, §9.3 |
| FR-TR-01–FR-TR-04 (viewer transport/encoding/sessions) | §9 |
| FR-CD-01–FR-CD-09 (client/demo) | §10, §3, §4.2 |
| FR-BR-01–FR-BR-05 (benchmark/report) | §14, §4.1, §4.2 |
| K1–K11 | §4.3, §7 (K9), §11, §13, §14 |

## 17. Design Decisions

Each decision states its decision in the first sentence, gives labelled aspects as bullets, records PRD-001 mappings as **Implements** or **Amends**, and closes with **Alternatives rejected.** Section coverage is given once, in the table above.

| # | Decision | Recorded in |
| --- | --- | --- |
| 1 | One hierarchy serves both visibility and aggregation; frontier ordered by exact projected height | §2, §4.1, §5.1, §5.2, §8.2–§8.4, §13 |
| 2 | Aggregation by accumulator propagation over per-group channels | §7 |
| 3 | Stateful per-viewer change suppression | §9 |
| 4 | Per-group local channels — identity and policy — with explicit contributions | §2, §3.1, §4.1, §4.3, §5.2, §6, §7 |
| 5 | Aggregation execution: always-on 60 Hz background pass | §2, §4.3, §7 |
| 6 | Attention channel (severity and absence) | §3.1, §4.3, §7.1, §9.2, §9.3, §13 |
| 7 | Metric identity and state layout: device-local labels, per-device instances, device-major state | §2, §4.1, §4.2, §4.3, §5.2, §5.3, §6, §9.4 |
| 8 | Canonical child order and child enumeration | §4.1, §4.3, §5.2, §5.3, §8.2 |
| 9 | Config schema: JSON flat node list, explicit fields, shallow templates | §2, §4.1, §5.2, §7.1, §10, §14.2 |
| 10 | Index layout for channel definitions and contributions; aggregation direction | §3.2, §4.3, §7 |
| 11 | Wire identity | §4.3, §8.2, §9.3, §9.4, §9.5 |
| 12 | Hierarchy build details | §2, §5.2–§5.4 |
| 13 | Benchmark host spec, fixed benchmark topology, and simulator configuration | §14 |
| 14 | Canonical reduction order and SIMD | §4.3, §5.5, §7, §9.2, §13, §14.3 |
| 15 | Transport encoding, keyframes, and egress | §2, §9.2–§9.4 |
| 16 | State-table concurrency: lock-free atomics, overwrite back-pressure | §2, §6 |
| 17 | Client rendering: InstancedMesh over a pre-allocated VBO | §2, §10.1 |
| 18 | Technology stack: Rust engine, simulator, and harness; Three.js client over WebGL; WebSocket binary transport; Vite/TypeScript demo tooling | §2, §3, §9.1, §15 |
| 19 | Engine configuration defaults: budget hysteresis Δ = 10% of `B`; camera-context timeout 1 s | §4.4, §8.1, §8.3 |
| 20 | Type templates and policy overrides: the type libraries supply a device's `metrics` and a group's `channels` with their `policy`; a node refines policy fields per key; expanded at boot before validation | §4.1, §14.2 |
| 21 | Group retention and degenerate LOD boxes: prune groups with no device descendant bottom-up; degenerate groups bypass the size threshold | §4.1, §4.3, §5.2, §8.2, §8.3, §13 |
| 22 | Declared group extents: optional per-group `aabb` with boot containment; geometry box (declared or derived) for culling and drawing, LOD box (derived) for projected height | §4.1, §4.3, §5.2, §5.3, §8.2, §10, §13 |
| 23 | Simulator operating parameters: rates and value models in a unit-keyed simulator profile — the simulator's configuration file — versioned with the report | §4.2 |

### Decision 1 — Frontier ordering

The frontier is a fixed-capacity **max-heap keyed on exact projected on-screen height**, with node id as the terminal tie-break.

- **Structure.** The hierarchy is the single structure driving visibility and aggregation; the state table is a separate flat value store aligned to leaf order.
- **Primary key.** Exact height is the primary key.
- **Ordering function.** Ordering is a pure function of `(topology, state, camera, selection)`; node id is a terminal tie-break for equal heights only. The index is immutable after boot, so the frontier holds node references and per-frame heights.
- **Bounds.** Heap operations are `O(log B)`; the frontier is bounded by `B` ≈ 5,000.
- **Budget-edge flicker.** Handled by the entry-count hysteresis in §8.3.
- **Alternatives rejected.** Bucketed ordering with a bucket size-error bound or a bucket-width parameter.

### Decision 2 — Accumulator propagation

One shared 60 Hz pass computes each group's local channels by accumulator propagation over its configured contributions (its own device readings and its child groups' contributed channels), with mergeable accumulators (`mean → (sum, count)`), exact over the subtree, published as a consistent per-frame snapshot.

**Depends on:** Decision 4.

- **Alternatives rejected.** Incremental push-up on ingest; per-viewer or per-request aggregation; event-driven recomputation; GPU/compute offload; alternative state layouts.

### Decision 3 — Stateful per-viewer change suppression

One persistent session per viewer; each frame reconciles the visible set against the previous frame (appeared → full value, removed → prune, otherwise changed slots as absolute values in their wire widths, ε-filtered in the wire domain); the selected object is streamed at full fidelity; a full keyframe is sent on (re)connect; per-session state is bounded with latest-wins on egress.

**Depends on:** Decision 15 — the baseline commits on an actual socket write; a dropped frame does not advance it.

- **Alternatives rejected.** Periodic keyframes on a fixed cadence; unbounded backlog for a slow client (FR-TR-04).

### Decision 4 — Group channel model

Each group node defines its own local channels — up to 16, in addition to its attention value — whose meaning is local to that group, and configures which of its channels contribute to which of its parent's channels.

**Implements:** PRD-001 FR-TD-05, FR-TD-06, FR-TD-08, FR-AG-01.

- **Local channel identity.** A channel's meaning is local to its group; there is no site-wide channel catalogue. The engine validates units per configured contribution; the client resolves a group entry's channel meanings per group.
- **Identity and policy.** A channel's `id`, `unit`, and `reduction` are its identity, declared only where the channel set is declared; `epsilon` and `limits` are its policy, carried in a companion `policy` map keyed by channel id — supplied in full where the channel set is declared on the node, and refinable per channel on a node that inherits it (Decision 20).
- **Explicit contributions, default none.** A group contributes nothing to its parent unless configured (FR-TD-06).
- **Same-reduction contributions.** A channel may contribute only to a parent channel of the same reduction (boot-validated), so a group's value is the canonical subtree reduction.
- **`mean` and `count`.** `mean` is carried as `(sum, count)` and merged as an accumulator, exact over the subtree's available readings; the wire carries the finalised scalar. `count` is the total number of available contributing readings in the subtree, merged from child `count` channels by summation (FR-AG-01).
- **Channel states.** A configured channel is active (has a value), inactive (no available contributor → NaN), or not-used (not configured). All configured channels are carried in a keyframe; steady state carries changes.
- **Terminology.** The canonical term is **aggregate channel**.
- **Reserved attention.** The reserved attention channel is not addressable in configured contributions; it propagates implicitly.
- **SIMD applicability.** 16 channels ≈ one 512-bit vector; permute/gather plus masked reduce, with effectiveness depending on wiring regularity.
- **Config volume.** Peer groups repeat identical channel definitions, which templating by group type removes; wiring is declared per node, so peers repeat it — one map per contributing device and one per group against a device-dominated file (§14.2). Metric definitions are the larger volume and are templated by device types (Decision 20); per-node wiring, and the optional `policy` overrides a site opts into on either kind of node, are the price of keeping both libraries free of context.
- **Alternatives rejected.** A `published` flag and a noop channel — all configured channels are intended aggregates, the per-frame sender logic governs transmission, and contributions are resolved at boot, so no target-resolution branch is required; a status/coverage channel — subsumed by per-metric absence.

### Decision 5 — Aggregation execution

One background pass at 60 Hz computes every group bottom-up from a **boot-built compiled plan** (seed ops: device metric → channel; merge ops: child channel → parent channel), with accumulator layout and reduction order fixed at boot. The pass runs continuously, independent of viewer count.

- **Freshness.** Freshness is re-evaluated every pass; aggregates change without ingest as metrics cross their freshness timeout (FR-IS-03).
- **Traversal.** Bottom-up is a reverse linear scan of the parent-first index; no recursion is required.
- **Snapshot.** Double-buffered accumulators with an atomic buffer swap give the per-frame consistent snapshot (FR-AG-03).
- **Frame boundary.** Atomic latest-wins reads may include a reading accepted during the pass or defer it to the next frame; a reading accepted before the pass cannot be missed (FR-AG-05).
- **Availability.** Unavailable contributions are skipped; `mean` accumulates `(sum, count)` over available inputs (FR-AG-04).
- **Alternatives rejected.** Event-driven recomputation and per-viewer or per-request aggregation, both recorded under Decision 2; a recursive walk — the reverse linear scan needs none.

### Decision 6 — Attention channel (severity and absence)

Each group carries a reserved unitless attention channel (reduction `max`) on the fixed severity scale (normal, advisory, warning, critical), computed by the aggregation pass and streamed per entry; a device's attention level is the greatest of its metrics' contributions.

**Implements:** PRD-001 FR-TD-11, FR-AG-06, FR-CD-03.

- **Value severity.** Derived from limits on metrics and channels, evaluated as the maximum over crossed limits; limits inherit the value's unit.
- **Absence severity.** Per metric: a metric contributes its configured absence level while it is unavailable; `0` is permitted.
- **Merge.** The attention channel merges as a `max` channel, evaluated in the same bottom-up pass as aggregation, and is always contributed to its parent's attention channel; it is additional to the ≤16 configured channels.
- **Availability.** Offline and NaN markers are streamed separately; a device is offline when all of its metrics are unavailable (FR-IS-04; staleness per FR-IS-03); a node with no data and no nonzero contribution reports normal.
- **Alternatives rejected.** Explicit out-of-service/offline signalling — out of scope (PRD-001 §10); fraction-based coverage policy — out of scope (PRD-001 §10), `max` does not distinguish the number of missing devices; a raise-only per-device absence field for attention — not adopted: a device's absence level now changes through the replace-whole `absence` field of its `policy` map (§4.1, Decision 20), and the raise-only form is left for a later decision.

### Decision 7 — Metric identity and state layout

Metric identity is the per-device instance `(device, label)`; labels are chosen freely by each device and need only be unique within it.

**Implements:** PRD-001 FR-TD-04, FR-TD-08.

- **Identity.** A reading names `(device_id, metric_label)`; the label binds at boot to a per-device `metric_slot`, and a slot is meaningful only within its device.
- **Typing.** Each metric instance declares its own unit and attributes (ε, freshness, absence, limits, contributions); no channel combines contributors of different units (FR-AG-02).
- **State layout.** The state is normalized to one slot per metric instance: a compact `values` array with one entry per `(device, metric)`, plus a per-device row-offset array (CSR) that groups a device's instances and orders them in leaf order and local-slot order. Rows are exact, so there are no unused slots.
- **Aggregation.** At boot each channel records the sorted state offsets of its contributing metric instances; the 60 Hz pass walks those offsets within the group's contiguous block of rows. K9 (≤ 5 µs over ≤ 1,000 contributors) is met by the bounded, prefetch-friendly gather rather than by a single sequential run.
- **Encode.** A device's whole value set is contiguous, matching the per-entry "all values" requirement (FR-TR-02, §9.3).
- **Ingest record.** The simulator emits the engine's ingest format directly — a compact little-endian struct `u32 device_index, u16 metric_slot, f32 value` (10 bytes), optionally followed by a `u64 sample_timestamp` — with no runtime middleware: both ends derive the bindings from the shared config at boot (§4.2).
- **Client.** Cross-object comparison uses the attention channel (FR-CD-03); per-metric selection is per-object by construction, so no global vocabulary is needed.
- **Alternatives rejected.** A site-wide metric vocabulary and a global kind slot; a global metric column; a separate `quantity` concept — the engine consumes **unit**, not the label; a label-keyed ingest record — strings on the 300k EPS path, when both ends bind device ids and labels to indices at boot (§4.2).

### Decision 8 — Canonical child order and child enumeration

Canonical child order is the order in which children are declared in the config, and the index carries a per-node child count for O(1) degree.

- **Child order.** Children are an ordered sequence; the preorder DFS visits them in declaration order. This order defines node indices (wire identity, Decision 11), leaf order (CSR row order and the per-channel offset lists), the canonical float reduction order (Decision 14), and the frontier tie-break. In the config only sibling order is significant: a parent need not precede its children, and subtrees may interleave; parent-first belongs to the derived index (§4.3). It is server-side only: the client does not replay the build.
- **Child enumeration.** Node shape: `subtree_size` (node count) and `child_count`. `first_child = index + 1` (parent-first preorder over emitted nodes; excluded groups never intervene) and `next_sibling = sibling + subtree_size` are derived, not stored. `deg(v)` is O(1), so the expansion-budget check `entries − 1 + deg(v) ≤ B` is O(1) even for a flat hierarchy with ~100k children; enumeration is O(deg) and only runs when `deg ≤ B`.
- **Subtree-size units.** `subtree_size` is a node count.
- **Invariant.** Boot checks that `child_count` equals the emitted direct children and that `first_child = index + 1`; the contiguity scan confirms `subtree_size`.
- **Alternatives rejected.** Order derived from ids, map iteration, or filesystem order; a stored per-node device-count extent — aggregation reads per-channel offset lists (Decision 7), and visibility uses projected height and the entry budget.

### Decision 9 — Config schema

The topology definition is three JSON documents — the group and device type libraries, and the topology document's flat list of nodes with parent references — loaded as one source (FR-TD-09); list order is canonical sibling order. The engine is one config-driven binary with no scale-specific code paths — scale is a property of the topology, not of the code (§2, G5).

- **Hierarchy.** A flat `nodes` array; each node has `kind` (`group`/`device`), `id` (topology-unique), and `parent` (null for the single root). A pre-pass resolves templates, buckets children by parent in list order, and validates the tree; the preorder DFS then runs in that order.
- **Metrics.** Per metric instance: `label` (device-local, unique), `unit`, `epsilon`, and `freshness_ms` are required; `absence` and `limits` are optional (default `normal` / `[]`); its contributions come from the device node's `contributes_to` map. A type entry splits these into `metrics` (identity) and a label-keyed `policy` map covering every label; a device node refines that map per label (§4.1, Decision 20).
- **Channels.** Per group: at most 16; per channel `id` (group-local, unique), `unit`, and `reduction` are required identity, with `epsilon` required and `limits` optional (default `[]`) in its `policy` entry — companion to the channel set wherever it is declared. `count` is unit-typed like any other channel — its unit names the readings tallied and constrains contributors only, while the channel's value, ε, and limits are counts and the client shows the tally with no unit (FR-CD-05).
- **Wiring.** Both directions are node-level `contributes_to` maps and neither type library carries them: a device node maps its metric labels to its containing group's channel ids (FR-TD-04), a group node maps its channel ids to its parent's channel ids — each map names another node's namespace. Both default none; fan-out and fan-in are allowed, and a channel may feed several parent channels (same reduction). A 17th channel is a boot error.
- **Limits and absence.** Fixed named scale `normal`/`advisory`/`warning`/`critical`; a limit is `{threshold, side, level}` with the threshold in the value's unit, fired by `>=` (high) or `<=` (low), and a value's level is the max over fired limits; `absence` is per metric, default normal. Attention is stored as one `u8` per group and one per device in the double-buffered snapshot.
- **Value validity.** `epsilon` is finite and ≥ 0 on every metric and channel; `freshness_ms` is a positive integer; violations abort boot (§4.1).
- **Units.** Opaque strings compared by exact match.
- **Format and templates.** JSON only, in three documents: the two type libraries and the topology document. Shallow single-level templates (group library for channels, device library for metrics): a node may reference one `type`; a group's declared `channels` array replaces the template's channels and `policy`, a `policy`-only group declaration refines it per channel, and a device declares no metric set of its own (Decision 20).
- **Validation order.** Template resolution precedes FR-TD-08 validation; diagnostics name the source (template or node).
- **Version contract.** Each type library carries its own `version`; a topology that declares a `type` carries a `requires` entry per library, so the three documents move as one versioned definition while each library versions independently, and a missing library or an entry mismatch aborts boot (FR-TD-09, FR-BR-05).
- **Alternatives rejected.** Type-extends-type inheritance and per-item merge of channel definitions — templates stay shallow and single-level; per-field policy refinement is the merge that remains (Decision 20).

### Decision 10 — Index layout for channels/contributions; aggregation direction

Variable-length per-node data is held in global structure-of-arrays addressed by ranges, and the aggregation plan is indexed by target channel and executed as a pull.

- **Arrays.** Node arrays carry `channel_base`/`channel_count`. A global **channel-def array** holds one entry per channel (`unit`, `reduction`, `epsilon`, `limit_base`, `limit_count`, `contrib_base`, `contrib_count`); a group's channels are contiguous. A global **limit array** holds `{threshold, side, level}`. A global **contribution array** holds one tagged entry per source: `seed` (source = state offset) or `merge` (source = contributing channel-def index).
- **Pull.** The plan is the contribution array indexed by target channel: each channel lists its incoming sources. The reverse scan — bottom-up, single background thread, writing the write buffer — reduces each channel's list (seeds in device-leaf order, then merges in child order) and finalises it. This vectorises the reduction, gives an explicit canonical order for exact `sum`/`mean` (Decision 14), needs no pre-clear, and leaves the seed reduction parallelisable if the pass is ever split.
- **Reserved attention.** One accumulator slot per group, outside the channel-def array; never a contribution target; computed from the group's channel limit levels, its direct devices' attention levels, and its children's attention channels.
- **Alternatives rejected.** Push execution — the background-thread/bottom-up design removes push's only advantage (no cross-source races), leaving its serial read-modify-write chain; pull also avoids a pre-pass buffer clear and yields the canonical order.

### Decision 11 — Wire identity

The wire key is the server's flat-array node index; the client maps it to config identities through a connect-time dictionary.

- **Per-frame.** Entries and the per-session diff are keyed by the node index (u32), as in §9.3.
- **Dictionary.** On (re)connect the server sends the dictionary covering every node in the index, in chunks cut at the 64 KB wire-message bound (§9.4), each chunk self-describing by node index; the client accepts frames only after the end chunk. The client resolves position, units, metric labels, and channel meanings from the shared config (FR-TD-09). The dictionary is delivered once per session as one chunk sequence, re-sent on reconnect, and may be delta/compressed.
- **Client selection.** The client keys rendered instances by node index, so the inspection target (FR-VS-02) is sent as the node index.
- **Restart / edit.** Indices are per-build; the topology is fixed while serving, so indices are stable within a session, and a fresh dictionary is sent across a restart or a config edit.
- **Alternatives rejected.** Cross-restart index stability and a client index cache — a fresh dictionary is sent instead, and no cache survives a reconnect.

### Decision 12 — Hierarchy build details

The hierarchy build is a single sequential pass over the resolved config, and validation runs fail-fast in seven stages before serving.

- **Leaf AABB.** A leaf's AABB is the device position (degenerate point); frustum tests are inclusive, and the brute-force reference uses the identical point test. There is no AABB inflation. A leaf's projected height is degenerate, which is harmless: a device leaf is terminal by kind, and the size threshold only decides group blending.
- **Positions.** Non-finite positions (NaN/Inf) are rejected at boot; coincident positions are legal and separated by the node-id tie-break.
- **Build parallelism.** The layout DFS is sequential — O(N) and not the K8 bottleneck. K8 is treated as a config-parse and validation budget.
- **Validation.** Checks run on the resolved config in seven stages — schema (FR-TD-08), structural (FR-TD-07), device/metric (FR-TD-04), channels (FR-TD-05), contributions (FR-TD-06/08), limits/absence (FR-TD-11/08), post-build invariants (§5.3) — and are fail-fast: the first fault aborts boot with a structured diagnostic `{code, node id, metric label / channel id, message}`.
- **Alternatives rejected.** A parallel layout DFS — the sequential walk is O(N) and not the K8 bottleneck; a pre-emptive streaming parse — parsing is optimised only if measurement demands it.

### Decision 13 — Benchmark host, topology, and simulator configuration

The backend host spec is a field template recorded when the architecture is frozen and filled before the first benchmark run; each tier's benchmark topology and simulator configuration are produced by a deterministic, seeded generator and published with the report.

**Implements:** PRD-001 FR-BR-05.

- **Host.** The backend host spec is recorded as a field template (§14.1); the K5 raw baseline is fixed in PRD-001 §7.1, and performance numbers are produced by the benchmark runs of §14.
- **Topology (FR-BR-05).** A deterministic, seeded generator (`bench-v1`) emits each tier's topology and the type libraries shared by all three tiers in the §4.1 schema, with fixed tier shapes (4,280 / 50,000 / 100,000 devices over 80 / 800 / 1,600 racks), a ~3-metric set, per-level channels with explicit contributions, and grid positions. The definition and its libraries are versioned and published with the report.
- **Configuration (FR-BR-05).** `bench-v1` also emits each tier's simulator profile — the tier's `reading_rate.global_eps`, the unit-keyed value models, and the seed (§4.2) — versioned and published with the report; the topology carries no rate field (§4.1).
- **Alternatives rejected.** A hand-authored or per-run topology — the spec plus seed reproduce the published file exactly.

### Decision 14 — Canonical reduction order and SIMD

Sums use a fixed blocked-lane order with `W = 8`, so the scalar engine and the brute-force reference are bit-exact; scalar is the default and the reference, and SIMD ships behind a feature flag.

- **Canonical order.** Source `i` accumulates into lane `i mod 8`, and the eight lanes fold in a fixed order. A channel's sources are already ordered (seeds by ascending state offset, then merges by child order, Decision 10), so the reduction is fully deterministic. `min`/`max`/`count` are order-free, and the encode pipeline (width convert, ε filter) is elementwise.
- **SIMD.** SIMD for aggregation and encoding ships behind a feature flag.
- **Alternatives rejected.** SIMD as the default path — the scalar path remains authoritative, and the ship decision is made from the K9 measurement.

### Decision 15 — Transport encoding, keyframes, and egress

The wire carries the absolute value in each slot's wire width — f16 for metric values and `mean`/`min`/`max` channels, f32 for `sum`/`count` channels — against a last-sent ε-suppression baseline, in a fixed little-endian frame layout, with a full keyframe on (re)connect and on-demand resync on a sequence gap.

- **Value encoding.** The last-sent baseline is only the ε-suppression reference; the displayed value is exactly the last-sent value in its wire width, within ε plus that width's rounding of current.
- **Frame layout.** Little-endian; a leading `u8 type`, then a header (sequence, flags, entry count), then entries keyed by node index (Decision 11), each with entry flags, a changed-slot mask, the included values in their wire widths, and attention only when changed. The canonical unavailable marker is the quiet NaN of the slot's own width (`0x7E00` for f16).
- **Keyframes.** Full visible keyframe on (re)connect; appeared entries sent in full on expansion or visibility change; on-demand resync when a client detects a sequence gap.
- **Baseline and slow clients.** The baseline advances only on an actual socket write. Egress is latest-wins: a stale pending frame is replaced, and a dropped frame does not advance the baseline, so a client never decodes against a value it did not receive. Per-session state stays bounded by the current visible set plus the selected object.
- **Alternatives rejected.** Numeric deltas — the wire carries absolute values, so there is no client-side accumulation drift; periodic keyframes — keyframes are sent only on (re)connect and on demand; f32 for every slot — it doubles every changed-slot and keyframe byte and puts K5's fixed 0.3 MB/s budget under test for the reductions f16 already covers; u32 for `count` — a third wire type for integers that f32 already represents exactly below 16,777,216.

### Decision 16 — State-table concurrency

Ingest writes each `(device, metric)` slot atomically and readers load the newest value — no lock on either path and no queue on ingest.

- **Back-pressure.** A saturated ingest overwrites stale values instead of buffering, which is what binds memory (FR-IS-05) and is the consequence K1 requires.
- **Alternatives rejected.** A mutex or sharded-lock table would plausibly meet K1 (300,000 atomic stores/s is well within reach of coarser schemes), so lock-freedom is a preference rather than a necessity; an MPSC queue with bounded backlog — queued readings would age out of the freshness window (FR-IS-03) under load; per-viewer or per-request state tables — duplicated state.

### Decision 17 — Client rendering

One `InstancedMesh` draws both device instances and blended group entries; per-frame instance transforms and colours are written into a pre-allocated VBO, so a frame performs no allocation and cannot stall on the heap (K6, K7, FR-CD-01).

- **Alternatives rejected.** Per-object `Mesh` instances — 5,000 draw-object updates per frame make main-thread overhead scale with the visible set rather than stay under 3 ms (K6); a GPU-side transform buffer — deferred: it moves the same writes off-thread but complicates picking (FR-CD-04), and the VBO path already meets K6/K7 at the building tier; per-frame mesh rebuilding — kept only as the R4 fallback, not the primary path.

### Decision 18 — Technology stack

The project uses one stack throughout: Rust server-side, Three.js over WebGL in the browser, WebSocket on the wire, and Vite/TypeScript for demo tooling.

- **Language/runtime.** The engine, the simulator, and the benchmark harness are Rust; the demo application and client build use Vite and TypeScript.
- **Client rendering.** Three.js over WebGL on the GPU-accelerated browser of PRD-001 §9, drawing through the `InstancedMesh` path of Decision 17.
- **Transport.** WebSocket carrying a compact binary frame format (§9.1), one persistent session per viewer (FR-TR-01).
- **Alternatives rejected.** A garbage-collected server runtime — pause-time risk against the K2 and K9 budgets; a native or non-browser client — out of scope (PRD-001 §10); HTTP request/response or raw TCP — no persistent bidirectional session, contrary to FR-TR-01.

### Decision 19 — Engine configuration defaults

The entry-budget hysteresis band defaults to 10% of `B`, and the camera-context timeout defaults to 1 s; both are engine configuration rather than topology (§4.4).

**Implements:** PRD-001 FR-VS-08.

- **Budget band.** Expansion is admitted while `entries < B`; a node already expanded is retained until the cut falls below `B − Δ` (§8.3), so the entry budget does not oscillate at its edge.
- **Camera timeout.** A last known camera context is reused until the 1 s timeout, after which that viewer's updates are held (§8.1).
- **Alternatives rejected.** No budget band — unrelated scene contents would flip a group between blended and detailed at the budget edge (§8.3); no camera-context timeout — a stale context would be reused indefinitely, against K11 freshness.

### Decision 20 — Type templates and policy overrides

A device references an entry in the device type library that supplies its `metrics` and its type-level `policy`, a group references an entry in the group type library that supplies its `channels` and its `policy`, and either node's own `policy` may refine its entry per key and per field; templates expand at boot, before validation, so the runtime still holds one metric instance per `(device, label)` (Decision 7).

- **Scope.** The group library names channel templates for groups; the device library names metric templates for devices. A group's `type` resolves only in the group library, a device's only in the device library; an unknown name, a missing library, an entry mismatch with `requires`, a mismatched container, or a device type entry that yields no metric after expansion is a boot error. A device type entry's `policy` covers every label in its `metrics`, a group type entry's every id in its `channels`; an entry that omits a key, or names one outside its identity set, is a boot error.
- **Expansion order.** Group templates resolve first — with each group node's own `channels` and `policy` resolved into its channel set — then device templates, so a device node's `contributes_to` resolves against its containing group's channels as they stand after the group template; a missing target is a boot error.
- **Override rule.** A group's declared `channels` array replaces the template's channels and its `policy` wholesale, with no per-item merge of definitions; a group node's `policy` alone refines the template's fields per channel. A device declares no metric set of its own, so the device library entry is the only source of its metric list and identity, while its `policy` map refines each of `epsilon`, `freshness_ms`, `absence`, `limits` per label. In either refinement a declared field replaces the type's value whole, an absent field inherits it, and a key naming none of the node's channels or metrics is a boot error (§4.1).
- **Identity vs policy.** Identity stays with the declaring source in both libraries: a metric's `label` binds device-local identity (Decision 7) and its `unit` gates every contribution (FR-AG-02); a channel's `id`, `unit`, and `reduction` define what the channel computes — no `policy` field reaches either kind. The policy fields — `epsilon`, `freshness_ms`, `absence`, `limits` for a metric, `epsilon`, `limits` for a channel — are deployment tuning: a site retunes a threshold on one device or one group without forking a type (FR-TD-04, FR-TD-05, FR-TD-11).
- **Sharing.** Expansion is deterministic from an unchanged topology, so the simulator, the engine, and the client derive the same identities (FR-TD-09).
- **Alternatives rejected.** A runtime kind lookup — expansion completes at boot and the engine consumes per-device `(device, label)` instances (Decision 7); type inheritance or composed types — templates stay shallow and single-level, mirroring the group library (Decision 9); template-provided positions — a device's position is its own (FR-TD-03); a per-item merge of template and node fields for metric sets and channel sets — whole-array replacement keeps every diagnostic attributable to one source, and a per-field policy refinement preserves that by naming the winning source per field; inline metric declarations on a device — every metric set lives in the device library, so the simulator configuration addresses types and no instance ids (§4.2).

### Decision 21 — Group retention and degenerate LOD boxes

A group is retained in the index only if it has at least one device descendant, decided bottom-up after device placement; a retained group whose LOD box is degenerate — a single device, or coincident devices — is exempt from the size threshold and expands when its parent expands.

- **Pruning.** A group with no device link — empty, or inherited from a type with nothing attached — is accepted (FR-TD-07) and pruned, so every retained group's LOD box is the union of a non-empty set.
- **Contribution sources.** Merge sources whose child group is pruned are dropped at build, so every contribution source resolves to a recorded channel definition; the pruned group's channels are valueless by construction (FR-AG-04), its accumulators are never allocated, and its attention is normal, so the parent's maximum is unchanged.
- **Degenerate sizing.** A group whose LOD box is degenerate projects to 0 px at every distance, so the size threshold would blend it forever; exempting it restores FR-CD-04's bring-into-view path, while the entry budget and the budget band (§8.3) still bound it.
- **Sharing.** Pruning and sizing are pure functions of the config and the camera, so the engine and the brute-force reference apply identical rules (§5.4).
- **Alternatives rejected.** Nominal AABB inflation — it requires a magic extent and a matching rule in the brute-force reference, and it contradicts the no-inflation rule for leaves (Decision 12); rejecting device-less groups — FR-TD-07 requires that groups may be empty; emitting pruned groups — their AABB is the union of an empty set.

### Decision 22 — Declared group extents

A group may declare an `aabb` — two corner points that contain every device position in its subtree. Each node keeps two boxes: a **geometry box** (the declared `aabb`, else the derived union) for frustum tests and for the transform the client draws, and an **LOD box** (always the union of the node's devices' positions) for projected on-screen height.

- **Why two boxes.** Geometry governs culling and rendering, so the drawn marker and the culled box are the same box and markers do not pop at frustum edges. LOD governs expansion, which follows the spread of a group's devices: that is what FR-VS-05 measures (`too small on screen to show its individual devices`) and what FR-VS-07's largest-on-screen ranking spends the entry budget on.
- **Validation.** The field is optional, so a topology that declares none boots unchanged. When present, a boot check places every descendant device position inside it, which makes culling on that box incapable of hiding an in-view device (FR-TD-07, §5.3).
- **Geometry is never templated.** `aabb` and `position` are declared on the node itself — absolute coordinates hold only for that instance, the same reason template-provided positions were rejected (Decision 20).
- **Cost.** One extra box per index node (6 × f32 ≈ 2.4 MB at 100k, boot-time and immutable), an O(devices) containment check, and no wire change: the client derives the geometry box from the shared config (FR-TD-09, §10).
- **Pruning unchanged.** A declared `aabb` exempts nothing from Decision 21: a group with no devices has no values to stream.
- **Alternatives rejected.** A single box (declared alone, or declared unioned with derived) — it moves the LOD decision onto empty volume, so an empty cabinet outranks a dense sensor cluster in FR-VS-07's ranking and expands groups whose devices stay indistinguishable; declared extents in the group library — absolute geometry cannot be shared by instances at different locations; sending boxes on the wire — the client already holds the config, and +24 B per node would double the dictionary (§9.5) or add ~117 KB per frame against K5.

### Decision 23 — Simulator operating parameters

Reporting rates are simulator configuration content: `reading_rate` sets the topology's target readings per second, divided evenly across metric instances, and a device type's `reporting_interval_ms` gives each of its devices one reading per metric per interval — replacing those devices' even shares, which are not redistributed. Rates and value models live in a **simulator profile** — the simulator's own configuration file — with value models keyed by `unit`, versioned with the benchmark report.

- **Rates in the configuration.** FR-BR-05 assigns each tier's reading rates to the benchmark simulator configuration, versioned and published with the report; the topology stays the single source of identities and geometry for all three consumers (FR-TD-09, §3.1), and the rate fields never enter it or the engine's contract.
- **Profile keying.** A model describes physics rather than identity, so metrics sharing a unit share a model — metric labels are device-local and cannot be a key (Decision 7). A unit's noise scale is set at or above the ε of its metrics, so within-noise jitter both exercises suppression and crosses it (§4.2).
- **Reproducibility.** The profile carries the seed for deterministic traces (FR-CD-07) and is versioned and published with the benchmark report (FR-BR-04, FR-BR-05), so a run's configuration is data rather than source.
- **Cost.** No engine change: rate fields live in the simulator's configuration, which the topology validator and the engine never parse; the simulator validates them at start-up.
- **Alternatives rejected.** Rates in prose only (§14.2) — the published artifact then fails FR-BR-05, which assigns each tier's reading rates to a versioned simulator configuration; rates in the topology — they are simulator inputs, not identities or geometry, so the topology schema and the engine's contract stay free of them; value models in the topology — PRD-001 §8's interface keeps models out of the engine's contract and Decision 7 forbids label-keyed lookup; model parameters as code constants — FR-BR-04 requires the configuration to be documented as data.
