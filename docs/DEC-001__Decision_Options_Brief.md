# DEC-001 — Design Comparison and Decision Brief

| Field | Value |
| --- | --- |
| Document ID | DEC-001 |
| Title | Design comparison for bounded per-consumer delivery of live telemetry |
| Version | 0.3.0 |
| Status | Draft — for discussion |
| Last Updated | 2026-10-19 |
| Related documents | DISCUSSION-SUMMARY (Parts I–IX, §1–§141), PRD-001, TDD-001, PLN-001 |

## 1. Purpose and question

### 1.1 Purpose

This document provides the design comparison requested during the review: whether the proposed server-side cut has an advantage over alternative designs. It defines an anchored reference scenario, compares the design dimensions one at a time, compares coherent design packages, and reports where the server-side cut wins, ties, or loses.

The document is written to be audited: every parameter has an external anchor or is swept, every estimate is labelled, and no weighted score is used.

### 1.2 The question

> Under an anchored reference scenario, which placement of selection and aggregation minimises delivery cost subject to a hard per-client bound, and under what conditions, if any, does the server-side cut outperform the alternatives?

### 1.3 Short answer

The comparison does not show an unconditional advantage.

1. Under the base scenario (S0), no design dominates. Fine-grained interest delivery and the server-side cut tie on egress (~13 KB/s per client); broadcast is approximately 5× worse (the ratio of devices to entry budget, `N/B`); the server-side cut pays approximately 8× the server CPU at a 60 Hz selection rate, narrowing to approximately 1.4× at 10 Hz.
2. The server-side cut uniquely provides a hard per-client bound. That bound is requirement K3, set by the project rather than sourced externally.
3. The cut's advantage is conditional on a thin client (S2) or scattered, metered consumers (S4). Neither condition is anchored at present.
4. No published demand was found for the conjunction of bounded, exact, consistent, per-consumer delivery under a live cut. Published demand exists for plain aggregation.
5. The comparison therefore supports a conditional claim in a narrow niche, not a general advantage.

### 1.4 What this document does not claim

- It does not claim novelty. COMP4299 requires a system-development project, not a novel result.
- It does not claim that the engine is necessary at the base scenario.
- It does not claim that a favourable scenario exists; it identifies the conditions one would have to satisfy.
- It does not present estimates as measurements.

### 1.5 How to read it

Section 1 is the answer. Sections 2–5 are the evidence: method, scenario, dimensions, packages. Sections 6–8 are the interpretation and the decisions requested.

## 2. Method

### 2.1 Three kinds of variable

The comparison separates variables that are often mixed. This separation prevents a scenario from being selected to favour a design.

| Kind | Examples | Treatment |
| --- | --- | --- |
| Scenario parameter | Devices (N), metrics per device (m), reading rate (r), viewers (V), entry budget (B), deployment, client class, link | Anchored externally, or swept; never chosen to make a design win |
| Design dimension | selection model, placement, implementation, substrate, aggregation placement, transport, encoding, memory strategy | Chosen; compared |
| Requirement | hard per-client bound, consistency, freshness | Sourced or stated; not a design option |

### 2.2 Metric vector

The same vector is used for every dimension and every package.

| Metric | Unit | Why it is included |
| --- | --- | --- |
| Egress | KB/s per client; $/hour per client | The metered resource in cloud deployment |
| Server CPU | cores per viewer; $/hour per client | The server-side cut's known ceiling |
| Client memory | MB for metadata and buffers | The cut's only unique claim |
| Client CPU | ms/frame on the main thread | Thin-client viability |
| End-to-end latency | p99 ms | Freshness requirement |
| Consistency | snapshot atomicity across viewers: yes/no | Structural value |
| Bound | hard per-client byte or entry cap: yes/no | Requirement K3 |
| Build cost | engineer-hours, marginal | The binding constraint on a one-term project |

**Cost convention.** Costs are stated as **rates**, not monthly totals. A month is taken as 730 h, and a total is a rate multiplied by the connected hours.

- **Compute:** `$/hour ≈ cores × 0.044`, from $32 per rented core-month (§15). This is a **utilization rate**, not a bill. A long-lived stream server cannot use per-request billing, since its sessions are stateful and long-lived, so it is billed as a **provisioned instance-hour**: `instances × instance rate × hours`, rounded up to available instance sizes and paid whether the CPU is busy or idle. The utilization rate is therefore a lower bound at low viewer counts; the two converge once instances are well loaded. Billing granularity is per second with a 60-second minimum on the major providers, so a one-second burst is billed as a minute; for a continuously running server the granularity does not affect the bill. Burstable instances cost $8–15 per core-month and committed or spot capacity costs less.
- **Egress:** `$/hour ≈ KB/s × 0.000324`, from $0.09/GB at 0.0036 GB per hour per KB/s. Egress is incurred only while a viewer is connected and the state is changing, so the total scales linearly with connected hours; a single-shift deployment (8 h/day) is one third of a continuous one. The duty cycle multiplies every package equally and does not change the ranking; it does change whether the egress cost is material. The convention excludes the provider's free allowance and volume tiers (which lower it further) and compression.

Both are attributions, not a total bill; storage, load balancing, and support are excluded.

### 2.3 Comparison rule

> A design A **dominates** B under scenario S if A is no worse than B on every metric and strictly better on at least one, measured at S. Otherwise A and B are a **trade**, and the traded metrics are named.

No weighted score is used. A weighted score conceals the choice of weights, which is the tuning trap in another form.

### 2.4 Baselines

The comparison uses best-of-breed designs, not strawmen.

| Baseline | Description | Role |
| --- | --- | --- |
| B0 | Full-state snapshot; client cuts and aggregates | The simplest design that meets the base case |
| B1 | Incremental view maintenance plus a broker plus an entitlement layer | The composition the engine must beat |
| B2 | Direct select: user-driven subtree subscription | The commodity inspection mechanism |
| B3 | Broadcast plus an edge relay | The egress rescue for clustered consumers |

### 2.5 Status of numbers

Every figure is labelled as one of:

- **measured** — obtained from a benchmark or an instrumented run;
- **derived** — computed from anchored parameters and stated assumptions;
- **assumed** — a modelling choice, disclosed and swept.

No figure in this document is measured today. All quantitative values are derived or assumed, and the measured constants that would convert them into results are identified in §6.4.

### 2.6 Anti-tuning

The answer-shaping parameters are N, B, r, V, deployment, client class, camera motion, duty cycle, and p (the change-pass fraction). Each is anchored, swept, or measured. A parameter is never set to make a design win. Where a value is not anchored, it is labelled and swept.

Two parameters have narrower roles than they appear to have:

- **r** changes absolute cost and the transport choice, but not the architecture ranking: broadcast and the cut scale with the same changed fraction, so their ratio is `N/B` (§5.3).
- **Camera motion** changes the CPU cost of a server-side cut by up to ~60× through the cut cache, from effectively free for a static overview to the 60 Hz worst case for a free camera. It is stated per result.

## 3. Base scenario

### 3.1 S0 — modal reference deployment

A mid-size industrial site: approximately 10,000 tags, three metrics each, 1 Hz reading rate, ten operators, aggregation and visualization cloud-hosted, browsers over WAN. This is the reference scenario of DISCUSSION-SUMMARY §13 and the modal modern arrangement of §14.

#### Table 3.1 — Workload

| Parameter | Value | External anchor | Role |
| --- | --- | --- | --- |
| Devices (N) | 10,000 tags | Mid-size plant SCADA/OPC-UA tag counts (§13, §57) | Answer-shaping; sweep 1k–100k |
| Metrics per device (m) | 3 | Temperature, vibration, power | Fixed |
| Groups (G), channels per group (c) | ~180, ~3 | Hierarchy of §16; group library of §4.2 | Derived from topology |
| Viewers (V) | 10 | Control-room staffing (§13) | Answer-shaping; sweep 1–100 |
| Entry budget (B) | 2,000 | Marker legibility, ~32 px on 1080p (§13) | Answer-shaping; sweep 1k–8k |
| Consumer profile | Mostly dashboard subscriptions to a few aggregates; a minority use a free-camera 3D view | Operational practice: SCADA screens and observability dashboards show a subset (§42, §60, §73) | Answer-shaping |

**Note on the full snapshot.** A full-state snapshot is the worst-case delivery, not the modal one. Most operational consumption is a small, explicit subscription to a few aggregate values, which D6 serves directly. The snapshot is retained as a baseline, and as the substrate for clients that must recompute arbitrary aggregates locally; it is not assumed to be the base consumer.

#### Table 3.2 — Timing

| Parameter | Value | External anchor | Role |
| --- | --- | --- | --- |
| Reading rate (r) | 1 Hz per metric | OPC-UA publishing 0.5–2 s; SCADA 1–2 s; BMS 1–15 min (§74, §97). PRD K1 implies 1 Hz at 100k × 3 metrics | Answer-shaping; sweep 0.1–10 |
| Selection rate | 10 Hz slow pan to 60 Hz free camera | Decoupled from render; humans read at a few Hz (§74); camera-motion caveat (§12) | Answer-shaping; stated per result |
| Egress cadence | 1 Hz, tied to r | Cadence is the latency bound (§74) | Fixed at S0 |
| Viewer duty cycle | 24 h/day as an upper bound; 8 h/day for a single-shift variant | Shift patterns and control-room staffing | Answer-shaping; sweep |
| Render cadence | 60 Hz | Smooth panning; client-only | Fixed |
| Change-pass (p) | Measured; `p ≈ min(1, r/egress) × noise_pass` | Noise threshold is a simulator parameter | Measured |

#### Table 3.3 — Representation and environment

| Parameter | Value | Basis | Role |
| --- | --- | --- | --- |
| Value width | f32 (4 B) per value; counts as integers, sums as f32/f64 | OPC UA `Float`, Sparkplug `Float`, Protobuf `float`; f16 is a graphics/ML format | Fixed |
| Fixed-set encoding | Flat value array over resolved slots; layout from the manifest | Layout resolved once | Derived |
| Dynamic-set encoding | Slot table plus flat array; self-describing records as fallback | See Appendix B | Derived |
| Full state per tick | 30,000 device values + 540 group-channel values ≈ 122 KB; ~65 KB as a delta | Derived | Fixed |
| Attention | 1 B per node; ~10 KB if sent | Per-node severity | Derived |
| Availability | Quiet-NaN sentinel in the value slot | Per-metric staleness | Fixed |
| Deployment | Cloud/WAN; egress metered | $0.05–0.12/GB; hybrid edge-cloud is modal (§14) | Answer-shaping |
| Server | 8 usable cores at ~3 GHz nominal | Typical cloud VM; the per-entry cost coefficients in Appendix C assume this class | Fixed |

**Note on the CPU.** The per-entry cost coefficients are stated for a ~3 GHz class core. Sustained clock varies on shared cloud VMs and IPC varies across architectures, so the CPU figures are indicative. A clock change scales every package's CPU cost uniformly and does not change the ranking; a measured benchmark must fix the host (TDD §16.1).
| Client | GPU laptop browser | PRD §9–§10 | Answer-shaping |

**Note on 1 Hz.** 1 Hz is a reporting rate, not a scan rate. Field devices sample faster and report features; historians store at 1 s for critical tags and 10–60 s for the rest. The value is at the fast end of reporting and the slow end of control, which makes it conservative for a visualization. A heterogeneous mix (0.1–10 Hz) is the realistic variant and is swept.

**Note on the value width.** f16 is not used. It is a graphics and machine-learning format with no half-float type in any mainstream telemetry protocol, and its precision sits at the scale of an ε threshold, where it would manufacture or mask changes.

### 3.2 Anchored variants

Each variant changes one answer-shaping axis to a value with an external anchor.

| Variant | Change from S0 | Anchor | What it tests |
| --- | --- | --- | --- |
| S1 — Edge/LAN | 1 GbE, ~1 ms, unmetered, capable clients | Dominant Ethernet; LAN latency (§14) | Whether broadcast wins when bytes are unmetered |
| S2 — Thin client | Client with a hard memory/CPU bound | To be anchored: embedded HMI, kiosk, low-end panel (§112) | The cut's only unique claim |
| S3 — High fan-out | V = 100, r = 10 Hz | Plant tens of operators; process 1–10 Hz (§74) | Whether server CPU becomes the wall |
| S4 — Scattered metered | Many clients on metered links, no shared point of presence | Cloud egress pricing; no relay amortisation (§17) | Whether per-client bounding beats relay and broadcast |

### 3.3 Binding test

A mechanism's benefit is meaningful only when exercised at the operating point (§113).

| Variant | Entry bound binds? | Egress binds? | Client memory binds? | Server CPU binds? |
| --- | --- | --- | --- | --- |
| S0 | Only when the view exceeds ~1/5 of the site (§56) | Modestly; metered, 10 viewers | No; ~4 MB on a laptop | No; ~0.25 core at V = 10 |
| S1 | Same | No; unmetered LAN | No | No |
| S2 | Same | Yes | Yes, by definition | Maybe |
| S3 | Same | Yes | No | Not at V = 100; ~1/3 of the ~320-viewer wall at 60 Hz |
| S4 | Same | Yes | No | No |

The base scenario has no mechanism that binds hard. That is the central empirical point: at the modal operating point, the engine's guarantees are tail value, valuable only if the cap is required.

### 3.4 Anti-tuning declaration

The scenario was fixed before the comparison was run. The answer-shaping parameters are anchored above or swept. No parameter was selected to produce a favourable ranking. The measured constants that would convert estimates into results are `b` (cut cost per entry), `c` (encode cost per entry), `p`, and the actual cloud price; all are pending (§6.4).

## 4. Design dimensions

Each dimension is compared independently as rationale. The package comparison in §5 is the level at which a verdict is valid, because several dimensions are coupled.

### 4.1 Selection model and view-dependence

The question is whether relevance is determined by the camera, by the user, or by nobody.

| Criterion | Geometric (view-dependent) | Explicit (direct select) | None (broadcast) |
| --- | --- | --- | --- |
| Decides relevance | Camera | User | Nobody |
| Camera loop | Yes | No | No |
| Egress at S0 | Bounded only if server-side | Subtree-sized | O(N) |
| Server CPU | Per-viewer cut | Routing | Fan-out |
| Client memory | O(N), or O(B) with lazy manifest | O(subset) | O(N) |
| Bound | Only if server-side | Soft | None |
| Advantages | Relevance without asking; LOD and blending | Simple, deterministic; works off-screen | Simplest; client can verify |
| Costs | Camera loop; per-viewer CPU; hysteresis and oracle complexity | No free-camera relevance | Unbounded egress |
| Limits | Requires a bound to be justified | Caps at the root; no free camera | Fails metered or scattered consumers |
| Verdict at S0 | Not required | Preferred | Baseline |

The data and aggregation are view-independent; only the selection is view-dependent. A free camera is a presentation choice unless the product requires it.

### 4.2 Selection placement

| Criterion | Client | Server | Split |
| --- | --- | --- | --- |
| Egress at S0 | Unchanged (O(N) or subscribed) | ~13 KB/s | Bounded coarse |
| Server CPU | ~0 | 0.025 core/viewer at 60 Hz; ~0.004 at 10 Hz | Low |
| Client memory | O(N) tree plus metadata | O(B) only with the lazy manifest | O(coarse) |
| End-to-end latency | No camera round trip | Camera path | Mixed |
| Bound | No | Yes | Partial |
| Advantages | Cheap; no per-viewer CPU | Hard bound; exact rollup of the unseen | Balances both |
| Costs | Cannot reduce egress | Per-viewer CPU; camera loop | Two mechanisms to maintain |
| Limits | Client culling does not reduce egress | Justified only by a hard bound or thin client | Complexity without a clear claim |
| Verdict at S0 | Sufficient | Only if a hard bound is required | Not justified here |
| Changes ranking? | Yes, jointly with the client-memory bound | | |

### 4.3 Selection implementation

| Criterion | Library-owned | Library plus own policy | Own shared crate |
| --- | --- | --- | --- |
| Build cost | Lowest | Medium | Higher |
| Semantics control | Screen-space error only; no B, no Δ, no blended entries, no oracle parity | Fragile; the library has already pruned | Full control; oracle parity |
| Confounds a placement comparison | Yes | Yes | No |
| Changes the ranking? | No | No | No |
| Verdict | Not acceptable for a controlled comparison | Not recommended | Control configuration |

This dimension affects build cost and comparability, not the ranking. It cannot create an advantage.

### 4.4 Substrate

| Criterion | Manifest only | Tileset | Regular cells plus own check |
| --- | --- | --- | --- |
| Artifacts | One | Tileset plus content plus manifest; two sources of truth | Manifest plus cells |
| Client memory at 10k | ~4 MB, or O(B) with lazy loading | O(N) tree plus metadata | O(B) |
| Semantic control | Full | Library-defined; loses B and blended entries | Full |
| Build cost | Medium | Medium | Medium |
| Changes the ranking? | No | No | No |
| Verdict | Preferred | Cross-check only | Acceptable alternative |

The manifest is the source of truth for values. A tileset is a derived artifact and introduces a second encoding of the same hierarchy.

### 4.5 Aggregation placement

Selection and aggregation placement are coupled. Client-side aggregation requires the client to hold the subtree, which forces a broadcast or interest substrate and pushes device-level data onto the wire (§10, §109).

| Criterion | Server | Client |
| --- | --- | --- |
| Server CPU | One shared pass, amortised | ~0 |
| Client holds | Only the delivered slice | The whole subtree |
| Egress consequence | Bounded by the slice | Requires broadcast or interest |
| Aggregation of the unseen | Exact, server-side | Impossible without the unseen children |
| Cross-viewer consistency | Shared snapshot | Lost unless separately coordinated |
| Feasibility | Compatible with all selection models | Only with broadcast or coarse interest |
| Verdict | Required if aggregation of the unseen is a requirement | Not independent of §4.1–4.2 |

The requirement itself is the open question. The coupling constrains the design space but does not establish a need.

### 4.6 Transport

| Criterion | Full snapshot | Batched delta | Per-value delta |
| --- | --- | --- | --- |
| Egress at S0 (p = 0.5) | ~122 KB/s | ~65 KB/s | ~14× inflation from headers |
| Header overhead | Amortised (~0.09% of payload) | Amortised | ~52 B per message |
| Consistency | Trivial | Requires frame discipline | Requires frame discipline |
| Verdict | Baseline | Reduces magnitude, not ranking | Excluded |

Batching is a requirement, not an optimisation: per-value messages carry approximately 52 B of IP overhead (§11, §73). `p` is not free: `p ≈ min(1, r/egress) × noise_pass`, so at high r the delta approaches the snapshot.

### 4.7 Encoding

| Criterion | Flat fixed-set | Self-describing record | Slot table plus flat |
| --- | --- | --- | --- |
| Identity cost | None; layout from manifest | 1–2 B per sent entry | 1–2 B per membership change |
| Per-tick cost | Values only; bitmap for changes | Flags/mask plus changed values plus attention | Bitmap P/8 plus mask plus changed values |
| Best when | Set is fixed | Few entries change, or membership churns | Many entries change, membership stable |
| Limitation | Requires a fixed layout | Identity cost on every record | Slot-table desync requires a keyframe |
| S0 cost | ~65 KB/s (delta) for the full state | ~15 KB/s for the frontier | ~13 KB/s for the frontier |
| Changes the ranking? | No | No | No |

Membership, not delta, makes a set dynamic. A full frontier snapshot still requires identity or a slot table; a full global state does not. Appendix B gives worked byte-level examples.

### 4.8 Client-memory strategy

| Criterion | Full manifest | Lazy manifest (A′) |
| --- | --- | --- |
| Client memory | O(N); ~4 MB at 10k, ~40 MB at 100k | O(B); ~0.1–0.3 MB at B = 2,000 |
| Requires | Nothing | Server-side selection |
| Build cost | Baseline | Additional ~15 h |
| Binds at S0 | No; a laptop holds 4 MB | No; nothing to bound |
| Verdict | Sufficient at S0 | Necessary only if server-side selection is claimed and a client bound exists |

The lazy manifest is the mechanism by which server-side selection produces a client-memory advantage. Without it, the client is O(N) regardless of where the cut runs.

## 5. Coherent packages

### 5.1 Feasibility constraints

Not every combination of the options in §4 exists.

1. **Client aggregation requires a broadcast or interest substrate.** It cannot coexist with a server-side cut that delivers only a slice.
2. **A client-side cut cannot bound egress.** Culling happens after the bytes have left the server.
3. **A lazy manifest requires server-side selection.** The client cannot cut what it does not hold.
4. **Per-value delta is excluded.** Header overhead dominates.
5. **A hard bound requires a server-side or explicit subscription mechanism.** Broadcast and client cut provide none.

### 5.2 Packages

| ID | Package | Who selects | Who aggregates | Bound |
| --- | --- | --- | --- | --- |
| D1 | Broadcast plus client cut | Client | Either | None |
| D2f | Fine interest (frontier subscription) | Client, via the shared cut crate compiled to WASM | Server | Soft (α) |
| D2c | Coarse interest (region or subtree) | Client | Client | Soft (α) |
| D3 | Server cut | Server | Server | Hard |
| D4 | Server cut plus lazy manifest | Server | Server | Hard; O(B) client |
| D5 | Broadcast plus edge relay | Client | Either | None |
| D6 | Direct select / drill-down | User | Server | Soft unless capped |

### 5.3 Comparison at S0

Assumptions: f32 values, m = 3, c = 3, G = 180, p = 0.5, 1 Hz egress. Costs are rates per §2.2; a continuous month is 730 h. Build hours are marginal over a shared substrate of state table, topology, simulator, manifest, and client (~100 h).

| Package | Egress per client | Egress $/hour | Server cores per viewer | Client memory | Bound | Consistency | Build (h, marginal) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| D1 | 65 KB/s | 0.021 | ~0.002 | ~4 MB | No | Yes | ~10 |
| D2f | 13 KB/s | 0.0042 | ~0.003 | ~4 MB plus subtree | Soft | Yes, with keyframe | ~20 |
| D2c | 33 KB/s | 0.011 | ~0.003 | ~4 MB | Soft | No unless snapshotted | ~20 |
| D3 | 13 KB/s | 0.0042 | 0.025 at 60 Hz | ~4 MB | Yes | Yes | ~70 |
| D4 | 13 KB/s | 0.0042 | 0.025 at 60 Hz | ~0.1–0.3 MB | Yes | Yes | ~85 |
| D5 | 65 KB/s divided by cluster | 0.021 divided by k | ~0.002 | ~4 MB | No | Yes | ~20 |
| D6 | 4 KB/s | 0.0013 | Routing | ~subset | Soft | Yes | ~10 |

The egress ratio is `N/B`, not a bandwidth saving. Broadcast sends the whole state; the cut sends the frontier. Both scale with the changed fraction, so the ratio cancels `p` and `r`:

```
D1 / D3 = (N × m) / (B × m) = N / B
```

At N = 10,000 and B = 2,000 the ratio is 5×. At 100,000 and B = 5,000 it is 20×. The lever is B, and B is a perceptual cap, not a physical one.

At S0:

- On egress, D2f, D3 and D4 tie at approximately $0.0042 per client-hour (about $3.1 per continuous month). D1 is 5× worse. D6 is cheapest but serves only an explicitly selected subtree.
- On server CPU, D3 and D4 cost approximately 8× D2f at 60 Hz, narrowing to approximately 1.4× at 10 Hz.
- On client memory, nothing binds: 4 MB is trivial for a laptop. D4's O(B) advantage is real but irrelevant at S0.
- No package dominates.

**Compute cost.** At $0.044 per core-hour (§2.2), with the selection rates of Table 3.2:

| Package | cores per viewer | $/hour at 60 Hz | at 10 Hz |
| --- | --- | --- | --- |
| D1 | ~0.002 | 0.00009 | 0.00009 |
| D2f, D2c | ~0.003 | 0.00013 | 0.00013 |
| D3, D4 | 0.025 | 0.0011 | 0.00018 |
| D5 | ~0.002 | 0.00009 | 0.00009 |
| D6 | ~0.003 | 0.00013 | 0.00013 |

At 60 Hz the cut's compute is about 27% of its egress; at 10 Hz it is about 4%. Two caveats apply. First, cloud billing is per instance-hour, so at V = 10 the cut's 0.25-core load is served by a 2-vCPU instance costing $0.021–0.041 per hour, an attributable $0.0021–0.0041 per client-hour, comparable to the egress rate; the marginal rates apply above roughly V = 100. Second, a rented core is usually a hyperthread; if physical cores are meant, the rate doubles and the 320-viewer wall halves. Neither changes the ranking, because compute and egress both scale with a package's work. Totals scale with connected hours: an 8 h/day duty cycle is one third of continuous operation, and the provider free allowance may absorb most of the remainder.

### 5.4 Comparison under the variants

| Variant | Winner | Loser | Reason |
| --- | --- | --- | --- |
| S1 edge/LAN | D1 on cost and latency | D3, D4 (CPU), D5 (no benefit) | Bytes are unmetered and latency is ~1 ms |
| S2 thin client | D4 on memory; D2f with lazy manifest ties | D1, D2c, D3 | O(B) memory is mandatory; only the hard bound remains unique to D4 |
| S3 high fan-out | D2f on CPU | D3, D4 (rising CPU, still ~1/3 of the wall at V = 100) | Per-viewer cut cost is linear in V; routing is not |
| S4 scattered metered | D3, D4 on egress; D2f ties | D1, D5 | A relay cannot amortise without a shared point of presence |

### 5.5 Verdict

The engine's advantage is conditional, and the condition is not anchored.

- It never dominates. D2f ties it on egress in every variant and is cheaper on CPU. D1 is cheaper to build and wins in S1.
- It uniquely supplies a hard per-client bound, exact aggregation of the unseen, and a shared cross-viewer snapshot. The snapshot is also available under broadcast; aggregation of the unseen matters only where the client cannot hold the state.
- The strongest surviving claim is the hard bound. The bound is K3, a project requirement, not a sourced one.
- The demand test fails for the conjunction. Published demand exists for aggregation, not for bounded, exact, consistent, per-consumer delivery under a live cut.

## 6. What the comparison shows

### 6.1 Result

Under S0 the server-side cut is a bounded middle: more expensive on CPU than fine interest, less expensive on egress than broadcast, and the only package with a hard bound. It is not the cheapest design and is not necessary at the base scenario.

The modal consumer — a dashboard subscribing to a few aggregates — is served by D6 at ~4 KB/s with commodity aggregation, and requires no cut. The engine's niche requires the free-camera or large-selection case, which the consumer profile treats as a minority.

The advantage appears only when a variant is anchored: a thin client with a hard memory bound (S2), or scattered consumers on metered links (S4). In both cases, fine interest ties it on egress; the remaining difference is the hard bound and the exact aggregation of the unseen.

### 6.2 Sensitivity ranking

| Rank | Factor | Effect on the verdict | Evidence |
| --- | --- | --- | --- |
| 1 | Deployment (scenario) | Flips the ranking: broadcast on edge/LAN; metering makes egress scarce | §14–§15 |
| 2 | Client-memory bound (requirement) | Creates the cut's only unique claim; without it D3 and D4 are unnecessary | §112 |
| 3 | Selection model and view-dependence | Determines whether a cut exists at all | §108–§109 |
| 4 | Aggregation placement | Coupled; defines what "unseen" means | §10 |
| 5 | Selection placement | Matters only after the first two are decided | §107 |
| 6 | Transport | Changes magnitude, not ranking | §11 |
| 7 | Implementation and substrate | No effect on ranking; build cost and credibility only | §111 |
| 8 | Encoding | No effect on ranking; absolute bytes only | Appendix B |

Only two of the eight rows change the verdict: deployment and the client bound. The comparison is therefore a two-variable question. That is the substance of the answer.

### 6.3 Demand status

| Element | Evidence | Status |
| --- | --- | --- |
| Aggregation over a hierarchy | Eclipse Ditto #1525 (2022, open); FIWARE Orion #3816 (2021, open); tippecanoe #324 and #830; deck.gl #10795 | Real, dated, unmet |
| Per-consumer selection | Mirror #3197; interest management | Solved |
| The conjunction | None found | Unrequested |
| Cross-consumer consistency | None found | Not a stated pain |

A universal gap is evidence of absent demand more often than of unmet need. The demand is for aggregation, not for the coupling.

### 6.4 What would change the verdict

| Unknown | If resolved as | Consequence |
| --- | --- | --- |
| A thin-client class with a hard bound | An embedded HMI or kiosk with a small metadata budget | D4 wins S2 outright; the claim becomes anchored |
| The hard bound is required | A sourced requirement rather than K3 | The bound moves from tail value to requirement |
| Demand for the conjunction | A published complaint or a standard | The justification strengthens |
| Selection rate and camera motion | 10 Hz slow pan rather than 60 Hz free camera | D3's CPU disadvantage narrows to ~1.4× |
| Measured constants b, c, p | Any values | Converts estimates into results; does not change the ranking |
| Real cloud prices | $0.05–0.12/GB | Confirms or weakens the egress case; the ranking is robust |

### 6.5 Limits of the analysis

- All numbers are derived or assumed; none are measured.
- S2, the decisive variant, has no anchored client class.
- The demand evidence is a sample of public issue trackers, not an exhaustive search.
- No stakeholder is available; a host laboratory is the realistic source of data or hardware.
- The comparison assumes the hierarchy is spatial and immutable at boot.
- Build estimates are planning figures, not measured effort.

## 7. Options for the project

| Path | Description | Requirements | Cost | Risk |
| --- | --- | --- | --- | --- |
| A — Keep and narrow | Retain the engine, scope it to S2 or S4 | An anchored thin-client class and a sourced hard bound; evaluate D4 against a lazy D2 and D1 | ~200 h | Demand unproven; if the composition passes, the dissertation is a build report |
| B — Comparison as contribution | Make the characterisation the contribution, with one reference implementation of the preferred package | The comparison, one implementation, anchored variants | ~150–180 h | Not novel; admissible for system-development, weak as a research claim |
| C — Change topic | Use the comparison as rationale and related work; move to an existing faculty research line or the disclosure direction | A named and reachable anchor | ~120–160 h | Starting over; the analysis transfers |

Path A is the only path on which the engine survives, and it depends on an anchor that does not yet exist. Path B answers the question and preserves the work. Path C is the honest move if neither the bound nor the demand can be anchored.

## 8. Open questions

1. Is a conditional advantage, true only in an anchored thin-client or metered scenario, sufficient for a COMP4299 system-development project?
2. If the comparison shows no unconditional advantage, is Path B (comparison as contribution) or Path C (change topic) preferred?
3. What client class would be accepted as thin and hard-bounded for S2?
4. Is the hard per-client bound (K3) acceptable as a self-set engineering requirement, or must it be sourced from a standard or a deployment?
5. Which line has data or hardware available this term?
6. Should any design that touches a model be assessed under COMP4299 or CSAI4299?
7. Are the expected deliverables a dissertation only, or also a demo and benchmarks?
8. Is the next artefact a full proposal, a design-space report, or a prototype spike?

## Appendix A — Evidence: competitor feature-request hunt

| Competitor | Issue | Age and state | What it shows |
| --- | --- | --- | --- |
| Eclipse Ditto | #1525 | 2022, open | Dated demand for aggregation over a fleet |
| FIWARE Orion | #3816 | 2021, open | Same shape; workaround is client-side aggregation |
| Mapbox tippecanoe | #324, #830 | 2016–2020 | Aggregation coupled to LOD exists, precomputed and shared |
| deck.gl | #10795 | 2026, open | View-dependent aggregation is solved |
| Cesium | #6182 | 2018, closed | User-built workaround; redirected to tile pipelines |
| Mirror | #3197 | 2022, closed | Per-consumer selection is solved |

Method caveat: a sample, not an exhaustive search; absence of a request is weak evidence.

## Appendix B — Dynamic-set encoding, worked

The dynamic set is the per-viewer frontier. Membership is the set of node indices in the frontier at a frame, not the values they carry. A membership change is an entry entering or leaving that set: a frustum crossing, an LOD transition, a threshold crossing, or budget eviction. A value, attention, or availability change within an existing entry is not a membership change, and neither is a slot move caused by compaction.

Two encodings are viable.

| Criterion | Self-describing record | Slot table plus flat |
| --- | --- | --- |
| Identity cost | 1–2 B per sent entry | 1–2 B per membership change |
| Per tick | Flags/mask plus changed values plus attention | Bitmap P/8 plus mask plus changed values |
| S0 cost | ~15 KB/s | ~13 KB/s |
| Best when | Few entries change, or membership churns | Many entries change, membership stable |
| Limitation | Identity cost on every record | Slot-table desync requires a keyframe |

**Record layout** (m = 3, f32):

| Field | Bytes | Notes |
| --- | --- | --- |
| Node delta | 1–2 | LEB128, relative to the previous sent record in sorted order |
| Flags plus mask | 1 | bit0 appeared, bit1 removed, bit2 attention-present, bits3–7 slot mask (m ≤ 5) |
| Values | 4k | Changed slots in slot order; all slots on appeared |
| Attention | 1 | Only if bit2 |

Worked cases:

1. Appeared, node 4,207, previous sent index 4,180: `delta 27 (1) + flags 0x05 (1) + 3×f32 (12) + attention (1) = 15 B`.
2. Changed, same node, slot 2 only: `delta 27 (1) + flags 0x20 (1) + value (4) = 6 B`.
3. Removed, node 109: `index (1) + flags 0x02 (1) = 2 B`.
4. Frame with churn, `{100,105,109,120} → {100,105,112,120}` with 100 unchanged: `header 10 + removal 2 + 6 + 15 + 6 = 39 B`.
5. Steady frame, only 105 changes: `header 10 + record 6 = 16 B`.

**Slot-table variant.** The server assigns slots 0…P−1 and sends the mapping only on change. Each tick sends a bitmap over slots (P/8 = 250 B at P = 2,000) plus mask and values per changed entry. At S0: `250 B + 1,750 × 7 B ≈ 12.5 KB`, against ~15 KB for records. The bitmap wins when `0.125P < 1.5S`, approximately 8% of entries changing.

**State-based sync.** A client can retain its last set and diff an incoming full set. This is robust (idempotent, no keyframe protocol, no desync) but does not save bytes: unchanged entries are resent. At S0 a full frontier is ~26 KB/tick against ~13 KB for the delta. The client copy is necessary to apply a delta but not sufficient to produce one; producing one requires the sender to know the receiver's baseline.

## Appendix C — Anchors and sources

| Parameter | Anchor | Source |
| --- | --- | --- |
| Tag counts | 1k–100k per site | §13, §57 |
| Update rates | 0.1–10 Hz process; 15 s Prometheus | §74 |
| Viewer counts | 5–20 control room; tens per plant; 10–50 field fleet | §74 |
| Marker legibility | ~32 px on 1080p, B ≈ 2,000 | §13 |
| Cloud egress | $0.05–0.12/GB | §14 |
| IP overhead | ~52 B per message (WS 2 + TCP 20 + IPv4 20) | §11 |
| Cut and encode cost | b ∈ [0.06, 0.19] µs/entry; c ∈ [0.03, 0.07] µs/entry, at ~3 GHz | §12 |
| LAN and WAN latency | ~1 ms; 20–80 ms | §14 |
| Egress ceiling | 1 GbE ≈ 110 MB/s; ~1–2 Mpps | §11 |

## Appendix D — Register of topics examined

The review examined the following topics. Each disposition is recorded so settled questions are not reopened.

| Group | Topics and dispositions |
| --- | --- |
| Design and architecture | Server-side cut (rejected as pitched); frustum and LOD (baseline); per-instance selection (finding); exact aggregation (baseline); selection-aggregation coupling (adopted); placement study (rejected); client cut (set aside); fine and coarse interest (baseline); edge relay (baseline); lazy manifest (adopted if a cut is kept); direct select (baseline); attention channel (set aside); compaction (set aside); snapshot and delta (adopted); tileset authoring (rejected per-node); i3dm and tile lifecycle (finding); hysteresis (conditional); ingress (set aside) |
| Deployment and economics | Cloud versus edge (adopted); metered egress (adopted); header overhead (finding); client-memory bound (open); binding rule (adopted); performance and product arena (finding) |
| Domain pivots | IoT semantic filtering (rejected); dashboards (set aside); smart city (rejected); data-centre twin (rejected); server room (rejected); field technician (set aside); remote LPWAN (rejected); fleet and satellite (set aside); substation PMU (set aside); observability cardinality (set aside); Google Maps analogy (finding); AR and robot knowledge (rejected); campus twin (rejected) |
| Candidate topics | Drift-aware twin (set aside, formerly primary); accessibility (rejected on access); the six offered topics (last resort); QR attendance (rejected); salon booking (set aside); extend a faculty research line (open); standards or dataset (open); disclosure delivery (open); live aggregation service (open); AI integration (rejected as decoration); proxy-resistant attendance (set aside) |
| Method and defensibility | Problem-first versus hunting (adopted); sourced and provisional constraints (adopted); failing-baseline method (adopted); reference scenario and anti-tuning (adopted); composition test (adopted); CRUD test (adopted); prior art as baseline (adopted); citation specificity (adopted); "sounds trivial" diagnosis (finding); three-worlds result (finding); consistency as constraint (finding); integrity (standing rule) |
| Literature | SHIP (related work); edge-IoT security monitoring (related work); seven-paper catalogue (related work); semantic drift repository (related work); competitor hunt (evidence); IVM and stream processing (baseline); 3D Tiles and Cesium (baseline) |
| Programme and practicalities | COMP4299 versus CSAI4299 (gate); track decision (closed); author-position analysis (adopted); engine fit (finding); portfolio value (finding); feasibility (constraint); demo expectation (constraint); IoT device survey (evidence); upload and trust (framework); privacy without demand (finding) |
| Unresolved | Demand for the conjunction; thin-client anchor; exactness versus DP; measured constants; reachable stakeholder; outstanding answers |
