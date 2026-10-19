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

1. Under the base scenario (S0), no design dominates **on the quantified metrics among packages that meet the S0 consumer requirement**. Fine-grained interest delivery and the server-side cut tie on egress (~13 KB/s per client); broadcast is approximately 5× worse (the ratio of devices to entry budget, `N/B`); the server-side cut is expected to pay more server CPU at a 60 Hz selection rate, but the magnitude is unmeasured. At 10 Hz the model predicts a near-tie; the CPU comparison is unmeasured.
2. The server-side cut uniquely provides a hard per-client bound among the S0 packages. That bound is requirement K3, set by the project rather than sourced externally.
3. The cut's advantage is conditional on a thin client (S2) or scattered, metered consumers (S4). No condition is **fully** anchored at present: S2 lacks a client class; S4's metering is anchored, but its scatter is not (candidate: fleet/maritime).
4. No published demand was found for the conjunction of bounded, exact, consistent, per-consumer delivery under a live cut. Published demand exists for plain aggregation.
5. The comparison therefore supports a conditional claim in a narrow niche, not a general advantage.

### 1.4 What this document does not claim

- It does not claim novelty. COMP4299 requires a system-development project, not a novel result.
- It does not claim that the engine is necessary at the base scenario.
- It does not claim that a favourable scenario exists; it identifies the conditions one would have to satisfy.
- It does not present estimates as measurements.
- It does not claim that the S0 free-camera consumer profile is the modal one; it is the engine's niche case, and §3.1 sweeps it against guided and drill-down consumers.

### 1.5 How to read it

Section 1 is the answer. Sections 2–5 are the evidence: method, scenario, dimensions, packages. Sections 6–8 are the interpretation and the decisions requested.

## 2. Method

### 2.1 Three kinds of variable

The comparison separates variables that are often mixed. This separation prevents a scenario from being selected to favour a design.

| Kind | Examples | Treatment |
| --- | --- | --- |
| Scenario parameter | Devices (N), metrics per device (m), reading rate (r), viewers (V), entry budget (B), deployment, client class, consumer profile, link | Anchored externally, or swept; never chosen to make a design win |
| Design dimension | selection model, placement, implementation, substrate, aggregation placement, transport, encoding, memory strategy | Chosen; compared |
| Requirement | free-camera relevance, hard per-client bound, consistency, freshness | Sourced or stated; not a design option |

### 2.2 Metric vector

The same vector is used for every dimension and every package.

| Metric | Unit | Why it is included |
| --- | --- | --- |
| Egress | KB/s per client; $/hour per client | The metered resource in cloud deployment |
| Server CPU | cores per viewer; $/hour per client | The server-side cut's expected ceiling; unmeasured |
| Client memory | MB for metadata and buffers | The cut's client-side claim; binds only for a thin client |
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

**Requirement set.** Dominance is assessed among packages that meet the S0 consumer profile (Table 3.1): a free camera over the site. D6 serves only an explicitly named subtree; it is the baseline for the guided-view case, not a candidate under S0.

**Terms.** *Frontier* = the delivered set under B. *α* = the subscribed fraction under interest. *k* = the relay clustering factor. *Oracle parity* = exact agreement with the brute-force oracle. *Blended entries* = aggregate entries standing in for a subtree.

### 2.4 Baselines

The comparison uses best-of-breed designs, not strawmen.

| Baseline | Description | Role |
| --- | --- | --- |
| B0 | Full-state snapshot; client cuts and aggregates | The simplest design that meets the base case |
| B1 | Incremental view maintenance plus a broker plus an entitlement layer | The composition the engine must beat |
| B2 | Direct select: user-driven subtree subscription | The commodity inspection mechanism |
| B3 | Broadcast plus an edge relay | The egress rescue for clustered consumers |

**Baseline mapping.** B0 ≡ D1, B2 ≡ D6, B3 ≡ D5. B1 (IVM + broker + entitlement) is the composition baseline rather than a single package; it is the decisive failing-baseline experiment and is not yet quantified in Table 5.3.

### 2.5 Status of numbers

Every figure is labelled as one of:

- **measured** — obtained from a benchmark or an instrumented run;
- **derived** — computed from anchored parameters and stated assumptions;
- **assumed** — a modelling choice, disclosed and swept.

No figure in this document is measured today. All quantitative values are derived or assumed, and the measured constants that would convert them into results are identified in §6.4. TDD and PRD budgets are design targets, not measurements; they are not used as anchors.

### 2.6 Anti-tuning

The answer-shaping parameters are N, B, r, V, deployment, client class, consumer profile (free camera vs guided view), camera motion, duty cycle, and p (the change-pass fraction). Each is anchored, swept, or measured. A parameter is never set to make a design win. Where a value is not anchored, it is labelled and swept.

Two parameters have narrower roles than they appear to have:

- **r** changes absolute cost and the transport choice, but not the architecture ranking: broadcast and the cut scale with the same changed fraction, so their ratio is `N/B` (§5.3).
- **Camera motion** changes the CPU cost of a server-side cut by up to ~60× relative to a 1 Hz selection baseline, from ~free for a static, cache-hit overview to the 60 Hz worst case. It is stated per result.
- **Consumer profile** changes whether geometric delivery is required at all: a guided view or drill-down is served by D6 without a cut. It is swept in §3.1 and §6.1; the free camera is the engine's niche case, not the modal consumer.

## 3. Base scenario

### 3.1 S0 — reference deployment

A mid-size industrial site: approximately 10,000 devices (≈30,000 tags), three metrics each, 1 Hz reading rate, ten operators, aggregation and visualization cloud-hosted, browsers over WAN. This is the reference scenario of DISCUSSION-SUMMARY §13 and the modal modern cloud arrangement of §14.

#### Table 3.1 — Workload

| Parameter | Value | Anchor / status | Role |
| --- | --- | --- | --- |
| Devices (N) | 10,000 devices (≈30,000 tags) | Mid-size plant SCADA/OPC-UA tag counts (§13, §57); as physical devices this is dense (0.5/m²; §57) | Answer-shaping; sweep 1k–100k |
| Metrics per device (m) | 3 | Temperature, vibration, power | Fixed |
| Groups (G), channels per group (c) | ~180, ~3 | Hierarchy of §16; **c = 3 assumed** (sweep 1–5); channel types from TDD §4.2 | Derived from topology; c assumed |
| Viewers (V) | 10 | Control-room staffing (§13) | Answer-shaping; sweep 1–100 |
| Entry budget (B) | 2,000 | Marker legibility, ~32 px on 1080p (§13) | Answer-shaping; sweep 1k–8k |
| Consumer profile | Viewers of a live 3D twin: free camera and guided views over the site | **Assumed S0 profile**; the free camera is the engine's niche case, not the modal consumer; swept in §2.6 and §6.1 | Answer-shaping; swept |

**Note on the full snapshot.** The full state is required when the client must serve arbitrary views or recompute arbitrary aggregates locally — the free-camera case. Where a consumer names what it needs (a guided view, a drill-down), a smaller delivery suffices. Both are compared; the full state is the worse case for egress.

**Note on the consumer profile.** S0 assumes a free camera. That is the engine's niche case, not the modal consumer: the cadence evidence (§74) describes 1–10 Hz viewing, not a 60 Hz free camera, and the demand evidence (§6.3) is category-level. No external anchor establishes the free camera as modal; it is swept as the alternative profile, and D6 is the baseline for the guided-view case.

#### Table 3.2 — Timing

| Parameter | Value | Anchor / status | Role |
| --- | --- | --- | --- |
| Reading rate (r) | 1 Hz per metric | OPC-UA publishing 0.5–2 s; SCADA 1–2 s; BMS 1–15 min (§74, §97) | Answer-shaping; sweep 0.1–10 |
| Selection rate | 10 Hz slow pan to 60 Hz free camera | Decoupled from render; humans read at a few Hz (§74); camera-motion caveat (§12) | Answer-shaping; stated per result |
| Egress cadence | 1 Hz, tied to r | Cadence is the latency bound (§74) | Fixed at S0 |
| Viewer duty cycle | 24 h/day as an upper bound; 8 h/day for a single-shift variant | Shift patterns and control-room staffing | Answer-shaping; sweep |
| Render cadence | 60 Hz | Smooth panning; client-only | Fixed |
| Change-pass (p) | `p ≈ min(1, r/egress) × noise_pass`; **p = 0.5 assumed in §5.3** | Noise threshold is a simulator parameter | **Measured (pending)**; assumed at S0 |

**Note on 1 Hz.** 1 Hz is a reporting rate, not a scan rate. Field devices sample faster and report features; historians store at 1 s for critical tags and 10–60 s for the rest. The value is at the fast end of reporting and the slow end of control, which makes it conservative for a visualization. A heterogeneous mix (0.1–10 Hz) is the realistic variant and is swept.

**Note on freshness.** Freshness is bounded by the egress cadence, because a reading cannot be delivered before the next tick: `freshness ≈ 1 / egress cadence`. K11 specifies p99 ≤ 100 ms at a 60 Hz rate; at the S0 cadence of 1 Hz it is approximately one second. Freshness is therefore a property of the cadence rather than of a package. Meeting K11 exactly would require an egress rate of at least 10 Hz, multiplying egress and compute by ten without changing the ranking.

**Note on circular anchors.** PRD K1 is consistent with 1 Hz × 3 metrics at 100k devices, but the PRD is the artefact under evaluation; it is not used as an external anchor. Table 3.1's c = 3 and Table 3.3's client class are likewise project assumptions, labelled as such.

#### Table 3.3 — Representation and environment

| Parameter | Value | Basis / status | Role |
| --- | --- | --- | --- |
| Value width | f32 (4 B) per value; counts as integers, sums as f32/f64 | OPC UA `Float`, Sparkplug `Float`, Protobuf `float`; f16 is a graphics/ML format. f32 supersedes §13's f16; width cancels across packages | Fixed |
| Fixed-set encoding | Flat value array over resolved slots; layout from the manifest | Layout resolved once | Derived |
| Dynamic-set encoding | Slot table plus flat array; self-describing records as fallback | See Appendix B | Derived |
| Full state per tick | 30,000 device values + 540 group-channel values ≈ 122 KB; ~65 KB as a delta | Derived | Fixed |
| Attention | 1 B per node; ~10 KB if sent | Per-node severity | Derived |
| Availability | Quiet-NaN sentinel in the value slot | Per-metric staleness | Fixed |
| Deployment | Cloud/WAN; egress metered | $0.05–0.12/GB; hybrid edge-cloud is modal (§14) | Answer-shaping |
| Server | 8 usable cores at ~3 GHz nominal | Typical cloud VM; the per-entry cost coefficients in Appendix C assume this class | Fixed |
| Client | GPU laptop browser | **Assumed S0 client**; PRD §9–§10 is self-set; the thin-client class is swept as S2 | Answer-shaping |

**Note on the value width.** f16 is not used. It is a graphics and machine-learning format with no half-float type in any mainstream telemetry protocol, and its precision sits at the scale of an ε threshold, where it would manufacture or mask changes. The choice supersedes the f16 anchor in DISCUSSION-SUMMARY §13; because every package uses the same width, the ratio and ranking are unchanged.

**Note on the CPU.** The per-entry cost coefficients are **assumptions**, not measurements; TDD budgets are design targets, not anchors. They are stated for a ~3 GHz class core. Sustained clock varies on shared cloud VMs and IPC varies across architectures, so the CPU figures are indicative model outputs. A clock change scales every package's CPU cost uniformly and does not change the ranking; a microbenchmark on a fixed host is required before the CPU figures are used as results (TDD §16.1 is a plan, not a source).

### 3.2 Anchored variants

Each variant changes one answer-shaping axis to a value with an external anchor, or states the missing anchor where none exists.

| Variant | Change from S0 | Anchor / status | What it tests |
| --- | --- | --- | --- |
| S1 — Edge/LAN | 1 GbE, ~1 ms, unmetered, capable clients | Dominant Ethernet; LAN latency (§14) | Whether broadcast wins when bytes are unmetered |
| S2 — Thin client | Client with a hard memory/CPU bound | **To be anchored**; candidate classes include embedded HMI, kiosk, low-end panel (§112), but none is sourced | The cut's client-memory claim; the hard bound is the surviving unique claim |
| S3 — High fan-out | V = 100, r = 10 Hz | Plant tens of operators; process 1–10 Hz (§74) | Whether server CPU becomes the wall |
| S4 — Scattered metered | Many clients on metered links, no shared point of presence | Cloud egress pricing (§14); no relay amortisation (§17); scatter **not yet anchored** (candidate: fleet/maritime, §42 #3) | Whether per-client bounding beats relay and broadcast |

**Note on S3.** Because egress cadence is tied to r (Table 3.2), S3 raises every package's egress by 10×; the `N/B` ratio and the ranking are unchanged.

### 3.3 Binding test

A mechanism's benefit is meaningful only when exercised at the operating point (§113).

| Variant | Entry bound binds? | Egress binds? | Client memory binds? | Server CPU binds? |
| --- | --- | --- | --- | --- |
| S0 | Only when the view exceeds ~1/5 of the site (§56) | Modestly; metered, 10 viewers | No; ~4 MB on a laptop | No; ~0.25 core at V = 10 |
| S1 | Same | No; unmetered LAN | No | No |
| S2 | Same | Yes | Yes, by definition | Maybe |
| S3 | Same | Yes | No | Not at V = 100; ~1/3 of a model wall (~235–570 viewers, unmeasured) at 60 Hz |
| S4 | Same | Yes | No | No |

The base scenario has no mechanism that binds hard. That is the central analytical point: at the S0 operating point, the engine's guarantees are tail value, valuable only if the cap is required.

### 3.4 Anti-tuning declaration

The scenario was fixed before the comparison was run. The answer-shaping parameters are anchored above or swept; the consumer profile is swept, and `p` is assumed at 0.5 for the package table. No parameter was selected to produce a favourable ranking. The measured constants that would convert estimates into results are `b` (cut cost per entry), `c` (encode cost per entry), `p`, and the actual cloud price; all are pending (§6.4).

## 4. Design dimensions

Each dimension is compared as rationale. The package comparison in §5 is the level at which a verdict is valid, because several dimensions are coupled.

The dimensions follow the delivery pipeline:

| Layer | Question | Dimensions |
| --- | --- | --- |
| Selection | Which entries are delivered, who decides, and where? | §4.1 model and placement (§4.2 notes) |
| Implementation | How is the selection built? | §4.3 library or own, §4.4 substrate |
| Aggregation | What is computed alongside the selection? | §4.5 (coupled to selection) |
| Delivery | How are the selected entries framed and encoded? | §4.6 transport and encoding (§4.7 notes) |
| Retention | What does the client hold? | §4.8 memory strategy |

Dimensions in **different** layers are orthogonal: any selection model can be paired with any transport and any encoding. Within a dimension, the columns are alternatives; dimensions may also be orthogonal within a layer. §4.1 compares the selection model and its placement together; §4.6 compares transport and encoding together. The layering is why selection and delivery remain separate: one chooses the set, the other frames its bytes.

Figures in §4–§5 are derived or assumed at S0; none are measured.

### 4.1 Selection: model and placement

The delivered set is determined by the camera, the consumer, or nobody; placement decides where that selection runs. Rendering selection — what to draw — is always the client's and always present; this section concerns delivery selection — what is sent.

| Model + placement | Delivered set | Membership | Supported transports | Server CPU (unmeasured) | Client memory | Latency | Bound | Verdict / role |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Geometric + client (broadcast source) | all state; client computes the frontier | static (delivery); dynamic (render) | full-state snapshot; full-state batched delta | fanout only | O(N) | local | none | client cut; no egress bound |
| Geometric + client (subscription) | frontier (client-selected) | dynamic | frontier snapshot; frontier batched delta; subscription | routing | O(N) manifest + O(B) frontier | subscription round trip | soft (α) | D2f; ties D3 on egress; CPU advantage expected but unmeasured |
| Geometric + server | frontier | dynamic | frontier snapshot; frontier batched delta | cut+encode; model 0.014–0.034 core/viewer at 60 Hz (unmeasured) | O(B) with lazy manifest; O(N) otherwise | camera path | hard | only under a hard bound or thin client |
| Explicit + client | subscribed subtree | dynamic on subscription | subtree snapshot; subtree batched delta; subscription | routing | O(subtree) | local | soft | inspection |
| Explicit + server | subscribed subtree | dynamic on subscription | subtree snapshot; subtree batched delta; subscription | routing | O(subset) | round trip | soft | D6; drill-down |
| None (broadcast) | all state | static | full-state snapshot; full-state batched delta; single broadcast topic | fanout only | O(N) | local / network | none | baseline at S0; wins S1 |

**Reading the table.** The model and placement are orthogonal: geometric can run on the client or server; explicit is a subscription, not a cut; none has no placement decision. The table therefore lists coherent combinations, not alternatives among model, placement, and transport. Server CPU is qualitative and unmeasured: fanout/routing for client/explicit, cut+encode for server. The server-cut range is a model hypothesis, not a measurement.

**Transport compatibility.** Transport is a separate dimension (§4.6). This column lists compatible transports, not a ranking; per-value delta is excluded throughout. Actual egress at S0 is in §5.3, where the set size and transport are stated.

**Model notes.**
- Geometric buys relevance without asking and LOD/blending; server-side it costs the camera loop and per-viewer CPU, and it needs hysteresis and oracle parity either way.
- Explicit is simple, deterministic, and works off-screen; it caps at the root and has no free-camera relevance.
- None is simplest and client-verifiable; its egress is unbounded.
- **Geometric + client (broadcast source) and None have the same delivered set.** The difference is that the client performs a geometric cut for rendering; neither reduces egress.
- Only geometric and explicit define a per-consumer set that can be used as a subscription. None degenerates to a single broadcast topic.

**Membership and transport.** Membership belongs to the delivered set, not to the model alone. None and geometric + client broadcast are static for delivery (the rendered frontier is dynamic). Geometric + client (subscription, D2f), geometric + server, and explicit are dynamic. A static set can use a flat array; a dynamic set needs identity or a slot table, `appeared`/`removed`, and keyframes. This is why transport remains a separate dimension in §4.6: it composes with every row.

**Client placement has two delivery variants.** The broadcast-source row sends all state. The D2f row has the client compute the frontier and subscribe to it, so the delivered set is the frontier with a soft bound; the client still holds the manifest unless a lazy structure is used. Both are client placement; the difference is delivery, not selection.

### 4.2 Selection placement: conditional effects

The placement columns are part of the combined table in §4.1. The conditional effect is that client placement cannot reduce egress, while server placement can bound it but pays per-viewer CPU. A server-coarse/client-fine split is a hybrid variant, not a separate placement; it is not evaluated here.

- **Client placement** does all selection locally; it cannot reduce egress, because culling happens after the bytes leave the server.
- **Server placement** produces a hard bound but pays per-viewer CPU and the camera path.

**Two criteria from the separate tables are now implicit.** A camera loop exists only when geometric selection runs server-side — geometric + server; client placement does not need one, and explicit/none never do. Client cull per frame follows the delivered set: O(B) for geometric + server, O(N) for geometric + client and none, O(subtree) for explicit.

The placement choice changes the ranking only jointly with deployment, the client-memory bound, and the consumer profile. At S0 no mechanism binds hard, so neither placement is forced by load.

**Rendering selection is always present.** Whatever the delivery model, the client frustum-culls and levels the scene for drawing, using library math or its own code (§4.3, §4.4). The models differ only in whether the client holds the data to perform it: under geometric delivery the server performs the selection and sends the resulting set, so the client draws it directly; under explicit delivery the client receives the named subtree and culls it locally; under broadcast it culls the whole state locally. The choice is therefore not whether selection happens, but whether it is duplicated on the server, and which side pays — the trade recorded above as server CPU against client memory and client CPU.

The data and aggregation are view-independent; only the selection is view-dependent, and whether its delivery is geometric depends on the bound rather than on the visualization.

### 4.3 Selection implementation

| Criterion | Library-owned | Library plus own policy | Own shared crate |
| --- | --- | --- | --- |
| Build cost | Lowest | Medium | Higher |
| Semantics control | Screen-space error only; no B, no Δ, no blended entries, no oracle parity | Fragile; the library has already pruned | Full control; oracle parity |
| Confounds a placement comparison | Yes | Yes | No |
| Changes the ranking? | No | No | No |
| Verdict | Not acceptable for a controlled comparison | Not recommended | Control configuration |

This dimension affects build cost and comparability, not the ranking. It cannot create an advantage. It applies to both selections: for the client's rendering selection, "library" means the library's cull and LOD math (for example three.js `Frustum`, Cesium `CullingVolume`) and "own" means hand-written traversal, with the same trade-offs, though no oracle is required because nothing leaves the client.

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
| Verdict | Required if aggregation of the unseen is a requirement | Not independent of §4.1 |

The requirement itself is the open question. The coupling constrains the design space but does not establish a need.

### 4.6 Delivery: transport and encoding

Transport and encoding are separate choices but are coupled through membership. The rows are transports; the columns are encodings.

| Transport | Flat fixed-set | Self-describing record | Slot table + flat |
| --- | --- | --- | --- |
| Full snapshot | best for full state (~122 KB/s) | fallback | possible; ~26 KB/s for a frontier snapshot |
| Batched delta | fixed set only (~65 KB/s full state) | workable | best for a dynamic frontier (~13 KB/s) |
| Per-value delta | excluded (~52 B header per value) | excluded | excluded |

**Header overhead.** Batching is a requirement, not an optimisation: per-value messages carry approximately 52 B of IP overhead (WS 2 + TCP ~30 with options + IPv4 20; §11, §73). One full snapshot amortises the header to ~0.04% of the full state; a per-value delta inflates a 4 B value to ~14×. `p` is not free: `p ≈ min(1, r/egress) × noise_pass`, so at high r the delta approaches the snapshot.

**Consistency and verdict.** A full snapshot is trivially consistent; batched and per-value deltas require frame discipline. Full snapshot is the baseline, batched delta reduces magnitude without changing the ranking, and per-value delta is excluded.

**Membership, not delta, makes a set dynamic.** A full frontier snapshot still requires identity or a slot table; a full global state does not. A fixed set can use a flat value array; a dynamic set needs a slot table or self-describing records, and a keyframe on desync.

### 4.7 Encoding: alternatives and costs

The encoding alternatives are the columns of the matrix in §4.6. Flat fixed-set has no identity cost and requires a fixed layout. Self-describing records pay 1–2 B of identity per sent entry and are best when few entries change or membership churns. Slot table plus flat pays identity only on membership change, is best when many entries change but membership is stable, and requires a keyframe on desync. For the frontier delta at S0, self-describing records cost ~15 KB/s and the slot table ~13 KB/s. Neither choice changes the ranking; it changes absolute bytes. Appendix B gives worked byte-level examples and the S0 costs.

### 4.8 Client-memory strategy

| Criterion | Full manifest | Lazy manifest (A′) |
| --- | --- | --- |
| Client memory | O(N); ~4 MB at 10k, ~40 MB at 100k | O(B); ~0.05–0.1 MB at B = 2,000 (~120–200 KB at B = 5,000) |
| Requires | Nothing | Server-side selection |
| Build cost | Baseline | Additional ~15 h |
| Binds at S0 | No; a laptop holds 4 MB | No; nothing to bound |
| Verdict | Sufficient at S0 | Necessary only if server-side selection is claimed and a client bound exists |

The lazy manifest is the mechanism by which server-side selection produces a client-memory advantage. Without it, the client is O(N) regardless of where the cut runs. At B = 2,000 the descriptors are ~0.05–0.1 MB; total footprint is ~0.1–0.4 MB.

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

**B1 is not a row in Table 5.3.** It is the composition baseline; the failing-baseline spike is its decisive test. B0/B2/B3 map to D1/D6/D5.

### 5.3 Comparison at S0

Assumptions: f32 values, m = 3, c = 3, G = 180, p = 0.5 assumed, 1 Hz egress. Set sizes: D1 broadcasts the full 30,000-value state as a p = 0.5 delta; D2f/D3/D4 serve the B = 2,000 frontier; D2c serves a ~2,750-device region as a full f32 state (~33 KB/s); D6 serves the benchmark room (~625 devices) as a p = 0.5 delta (~3.75 KB + overhead ≈ 4 KB/s). Costs are rates per §2.2; a continuous month is 730 h. Build hours are marginal over a shared substrate of state table, topology, simulator, manifest, and client (~100 h). All entries are derived or assumed at S0; none are measured.

**Dominance set.** D6 serves an explicitly named subtree, so it is outside the S0 dominance set: it is the baseline for the guided-view case, not a free-camera package. D1–D5 meet the S0 consumer requirement.

| Package | Egress per client (scope) | Egress $/hour | Server CPU (unmeasured) | Client CPU (ms/frame) | Client memory | p99 latency (ms) | Bound | Consistency | Build (h, marginal) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| D1 | site-wide; 65 KB/s | 0.021 | fanout only | ~0.5–2 | ~4 MB | ~1,020–1,080 | No | Yes | ~10 |
| D2f | frontier (B = 2,000); 13 KB/s | 0.0042 | routing | ~0.3–1 | ~4 MB + frontier (~0.1 MB) | ~1,050–1,150 | Soft | Yes, with keyframe | ~20 |
| D2c | region (~2,750 devices); 33 KB/s | 0.011 | routing | ~0.5–2 | ~4 MB + region (~0.1–0.2 MB) | ~1,050–1,150 | Soft | No unless snapshotted | ~15 |
| D3 | frontier (B = 2,000); 13 KB/s | 0.0042 | cut+encode; model 0.014–0.034 at 60 Hz | ~0.1–0.3 | ~4 MB | ~1,020–1,100 | Yes | Yes | ~70 |
| D4 | frontier (B = 2,000); 13 KB/s | 0.0042 | cut+encode; model 0.014–0.034 at 60 Hz | ~0.1–0.3 | ~0.05–0.1 MB descriptors; ~0.1–0.4 MB total | ~1,050–1,200 | Yes | Yes | ~85 |
| D5 | site-wide; 65 KB/s divided by k | 0.021 divided by k | fanout only | ~0.5–2 | ~4 MB | ~1,030–1,100 | No | Yes | ~20 |
| D6 | room (~625 devices); 4 KB/s | 0.0013 | routing | ~0.1–0.5 | ~subset (~0.1–0.2 MB) | ~1,020–1,080 | Soft | Yes | ~10 |

**Labels for the new columns.** Server CPU is **unmeasured**; the server-cut range is a model hypothesis, not a result. Client CPU and p99 latency are also assumed order-of-magnitude ranges, not measurements; they complete the §2.2 vector and are swept. At S0, latency is dominated by the 1 Hz egress cadence (Table 3.2), so the ranges overlap and do not change the ranking; the ~1 s p99 follows from the cadence, not from package latency. D6's lower figures reflect its narrower delivered set, not a free-camera capability.

**D2c's build cost.** D2c avoids the shared cut crate (no WASM frontier cut), so its marginal build is ~15 h rather than D2f's ~20 h; that is its surviving trade against D2f.

**Egress scope and transport.** The scope column states the set size: D1/D5 site-wide, D2f/D3/D4 frontier, D2c region, D6 room. Figures are batched delta at p = 0.5 unless stated; snapshot alternatives at S0 are ~122 KB/s for full state, ~26 KB/s for the frontier, and ~7.5 KB/s for the room. At the same room, a geometric server cut culls and blends and should be ≤ D6, unless it duplicates aggregate and raw data or carries a large guard band.

The egress ratio is `N/B`, not a bandwidth saving. Broadcast sends the whole state; the cut sends the frontier. Both scale with the changed fraction, so the ratio cancels `p` and `r`:

```
D1 / D3 = (N × m) / (B × m) = N / B
```

At N = 10,000 and B = 2,000 the ratio is 5×. At 100,000 and B = 5,000 it is 20×. The lever is B, and B is a perceptual cap, not a physical one.

At S0:

- On egress, D2f, D3 and D4 tie at approximately $0.0042 per client-hour (about $3.1 per continuous month). D1 is 5× worse. D6 is cheapest but serves only an explicitly selected subtree.
- On server CPU, the server cut is expected to cost more than routing; the magnitude is unmeasured. The model range is 0.014–0.034 core/viewer at 60 Hz; at 10 Hz the model overlaps routing (a model tie).
- On client memory, nothing binds: 4 MB is trivial for a laptop. D4's O(B) advantage is real but irrelevant at S0.
- On the new columns, the ranges overlap. D1/D5 carry O(N) client culling; D2f carries the WASM cut; D2c carries client aggregation; D3/D4 draw only the delivered slice.
- **No package dominates among D1–D5 on the quantified metrics.** CPU is unmeasured, so this is conditional. D6 dominates D2f on the printed metrics, but it does not meet S0's free-camera requirement; that is why the requirement is stated formally above.

**CPU cost is not computed.** `b` and `c` are unmeasured, so per-package CPU dollars are not results. The model form is `cores ≈ (T_cut + T_encode) × selection_rate`; a microbenchmark on a fixed host is required. The instance-hour caveat still applies: at low V, the smallest available instance sets a floor; above roughly V = 100, the marginal rates are more meaningful. A rented core is usually a hyperthread; if physical cores are meant, the rate doubles. Totals scale with connected hours: an 8 h/day duty cycle is one third of continuous operation, and the provider free allowance may absorb most of the remainder.

### 5.4 Comparison under the variants

**Terms.** "Preferred" means lowest cost among packages that meet the variant's requirement; it is not dominance — the other metric differences remain named trades.

| Variant | Preferred | Not preferred | Reason |
| --- | --- | --- | --- |
| S1 edge/LAN | D1 on cost and latency | D3, D4 (expected CPU; unmeasured), D5 (no benefit) | Bytes are unmetered and latency is ~1 ms |
| S2 thin client | D4 on memory; D2f approaches O(B) only via a coarse-top lazy descent, at round-trip cost | D1, D2c, D3 | Strict O(B) memory requires server-side selection; D2f's approximation adds latency |
| S3 high fan-out | D2f on CPU (model; unmeasured) | D3, D4 (rising CPU, still ~1/3 of the model wall at V = 100) | Per-viewer cut cost is linear in V; routing is not |
| S4 scattered metered | D3, D4 on egress; D2f ties | D1, D5 | A relay cannot amortise without a shared point of presence |

### 5.5 Verdict

The engine's advantage is conditional, and **no condition is fully anchored**.

- It never dominates on the quantified metrics. D2f ties it on egress in every variant and is expected to be cheaper on CPU at 60 Hz (model; unmeasured); at 10 Hz the model is a tie. D1 is cheaper to build and wins in S1.
- It uniquely supplies a **hard per-client bound among the S0 packages**. Exact aggregation of the unseen and a shared cross-viewer snapshot accompany it but are not unique: D2f and D6 also aggregate server-side, and a broadcast snapshot is shared. None of the three is sourced externally: the bound is self-set (K3), and consistency and exactness are properties of a centralised design rather than demands (§39, §64).
- The strongest surviving claim is therefore the hard bound, and it is a self-set product requirement rather than a sourced one.
- The demand test fails for the conjunction. Published demand exists for aggregation, not for bounded, exact, consistent, per-consumer delivery under a live cut.

## 6. What the comparison shows

### 6.1 Result

Under S0 the server-side cut is a bounded middle: expected to be more expensive on CPU than fine interest, less expensive on egress than broadcast, and the only package with a hard bound. It is not the cheapest design and is not necessary at the base scenario.

If the consumer instead names what it needs — a guided view, a drill-down — D6 serves it at ~4 KB/s with commodity aggregation and requires no cut. The engine's niche is therefore the free-camera case under a hard bound or a thin client, not visualization as such: in the natural visualization deployment (S1: GPU laptops in a control room on a LAN), broadcast with a client cut is the cheapest design.

**Consumer-profile sweep.** The two ends are a free camera, where D1–D5 compete as in §5.3, and a named subtree or guided view, where D6 serves the need at ~4 KB/s and the cut is unnecessary. The engine's case exists only in the first; the second is the modal deployment described in §74.

The advantage appears only when a variant is anchored: a thin client with a hard memory bound (S2), or scattered consumers on metered links (S4). In both cases, fine interest ties it on egress; the remaining difference is the hard bound. Exact aggregation of the unseen is also available under fine interest's server-side aggregation.

### 6.2 Sensitivity ranking

| Rank | Factor | Effect on the verdict | Evidence |
| --- | --- | --- | --- |
| 1 | Deployment (scenario) | Flips the ranking: broadcast on edge/LAN; metering makes egress scarce | §14–§15 |
| 2 | Client state bound (scenario) | Creates the cut's only unique claim; without it D3 and D4 are unnecessary | §112 |
| 3 | Consumer/selection model | Determines whether a cut exists; geometric delivery is justified only under a bound and a free-camera consumer | §3.1, §108–§109 |
| 4 | Aggregation placement | Coupled; defines what "unseen" means | §10 |
| 5 | Selection placement | Matters only after the first two are decided | §107 |
| 6 | Transport | Changes magnitude, not ranking | §11 |
| 7 | Implementation and substrate | No effect on ranking; build cost and credibility only | §111 |
| 8 | Encoding | No effect on ranking; absolute bytes only | Appendix B |

Only three of the eight rows change the verdict: deployment, the client bound, and the consumer/selection model. The comparison is therefore a three-variable question. That is the substance of the answer.

### 6.3 Demand status

| Element | Evidence | Status |
| --- | --- | --- |
| Digital-twin visualization (category) | Industrial-facility twins ~$5.1B (2021) → ~$13.9B (2026), ~18% CAGR as stated by Verdantix (endpoints imply ~22%); vendor investment (Siemens Digital Twin Composer, 2026; NVIDIA Omniverse; Azure 3D Scenes Studio) | Real and growing |
| Live, large-scale twin visualization | Documented as a technical challenge (Maiterth et al., *Visualizing an Exascale Data Center Digital Twin*, VIS 2024; telemetry-rendering bottlenecks in industry sources) | Active; addressed on the rendering side by incumbents |
| Aggregation over a hierarchy | Eclipse Ditto #1525 (2022, open); FIWARE Orion #3816 (2021, open); tippecanoe #324 and #830; deck.gl #10795 | Real, dated demand for aggregation (Ditto, Orion); solved for shared content (tippecanoe, deck.gl) |
| Per-consumer selection | Mirror #3197; interest management | Solved |
| Bounded, per-consumer telemetry delivery under a live cut (the conjunction) | None found | Unrequested |

A universal gap is evidence of absent demand more often than of unmet need. The demand is for aggregation, not for the coupling. Market figures are vendor-produced and vary widely — roughly $10–40B for 2026 across reports — so they indicate a growing category, not a specific mechanism; the category is served by client-side rendering in the incumbent platforms, and the delivery mechanism studied here remains unrequested.

### 6.4 What would change the verdict

| Unknown | If resolved as | Consequence |
| --- | --- | --- |
| A thin-client class with a hard bound | An embedded HMI or kiosk with a small metadata budget | D4 wins S2 outright; the claim becomes anchored |
| The hard bound is required | A sourced requirement rather than K3 | The bound moves from tail value to requirement |
| Demand for the conjunction | A published complaint or a standard | The justification strengthens |
| Selection rate and camera motion | 10 Hz slow pan rather than 60 Hz free camera | The model's CPU gap narrows; the CPU comparison is unmeasured |
| Measured constants b, c, p | Any values | Converts estimates into results; does not change the ranking |
| Real cloud prices | $0.05–0.12/GB | Confirms or weakens the egress case; the ranking is robust |

### 6.5 Limits of the analysis

- All numbers are derived or assumed; none are measured.
- S2, the decisive variant, has no anchored client class.
- The demand evidence is a sample of public issue trackers, not an exhaustive search.
- No stakeholder is available; a host laboratory is the realistic source of data or hardware.
- The comparison assumes the hierarchy is spatial and immutable at boot.
- Build estimates are planning figures, not measured effort.
- CPU is unmeasured; `b` and `c` are assumptions, and TDD/PRD budgets are design targets, not measurements.
- The S0 consumer profile is assumed, not anchored; the free camera is the engine's niche case, not the modal consumer.
- D2f's O(B) approach is a coarse-top lazy descent with a round-trip cost, not strict O(B) memory.
- B1, the composition baseline, is not quantified in Table 5.3.

## 7. Options for the project

| Path | Description | Requirements | Cost | Risk |
| --- | --- | --- | --- | --- |
| A — Keep and narrow | Retain the engine, scope it to S2 or S4 | An anchored thin-client class and a sourced hard bound; evaluate D4 against D2f (coarse-top lazy descent, with its round-trip cost) and D1 | ~200 h | Demand unproven; if the composition passes, the dissertation is a build report |
| B — Comparison as contribution | Make the characterisation the contribution, with one reference implementation of the preferred package | The comparison, one implementation, anchored variants | ~150–180 h | Not novel; admissible for system-development, weak as a research claim |
| C — Change topic | Use the comparison as rationale and related work; move to an existing faculty research line or the disclosure direction | A named and reachable anchor | ~120–160 h | Starting over; the analysis transfers |

Path A is the only path on which the engine survives, and it depends on anchors that do not yet exist. Path B answers the question and preserves the work. Path C is the honest move if neither the bound nor the demand can be anchored.

## 8. Open questions

1. Is a conditional advantage, true only in an anchored thin-client or metered scenario, sufficient for a COMP4299 system-development project?
2. If the comparison shows no unconditional advantage, is Path B (comparison as contribution) or Path C (change topic) preferred?
3. What client class would be accepted as thin and hard-bounded for S2?
4. Is the hard per-client bound (K3) acceptable as a self-set engineering requirement, or must it be sourced from a standard or a deployment?
5. Which line has data or hardware available this term?
6. Should any design that touches a model be assessed under COMP4299 or CSAI4299?
7. Are the expected deliverables a dissertation only, or also a demo and benchmarks?
8. Is the next artefact a full proposal, a design-space report, or a prototype spike?
9. Is the free-camera profile the intended S0 requirement, or is the modal consumer a guided view or dashboard subscription?

## Appendix A — Evidence: competitor feature-request hunt

| Competitor | Issue | Age and state | What it shows |
| --- | --- | --- | --- |
| Eclipse Ditto | #1525 | 2022, open | Dated demand for aggregation over a fleet |
| FIWARE Orion | #3816 | 2021, open | Same shape; workaround is client-side aggregation |
| Mapbox tippecanoe | #324, #830 | 2016–2020 | Aggregation coupled to LOD exists, precomputed and shared |
| deck.gl | #10795 | 2026, open | View-dependent aggregation is solved (collaborator tracker, not user demand) |
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

**Frame header.** The 10-B header used in the worked cases is `frame_id u32 (4) + entry_count u16 (2) + payload_bytes u16 (2) + flags u16 (2)`.

Worked cases:

1. Appeared, node 4,207, previous sent index 4,180: `delta 27 (1) + flags 0x05 (1) + 3×f32 (12) + attention (1) = 15 B`.
2. Changed, same node, slot 2 only: `delta 27 (1) + flags 0x20 (1) + value (4) = 6 B`.
3. Removed, node 109: `index (1) + flags 0x02 (1) = 2 B`.
4. Frame with churn, `{100,105,109,120} → {100,105,112,120}` with 100 unchanged: `header 10 + removal 2 + 6 + 15 + 6 = 39 B`.
5. Steady frame, only 105 changes: `header 10 + record 6 = 16 B`.

**Slot-table variant.** The server assigns slots 0…P−1 and sends the mapping only on change. Each tick sends a bitmap over slots (P/8 = 250 B at P = 2,000) plus mask and values per changed entry. At S0: `250 B + 1,750 × ~7 B average ≈ 12.5 KB`, against ~15 KB for records; the average is 1 B mask plus 1.5 changed slots × 4 B. The bitmap wins when `0.125P < 1.5S`, approximately 8% of entries changing.

**State-based sync.** A client can retain its last set and diff an incoming full set. This is robust (idempotent, no keyframe protocol, no desync) but does not save bytes: unchanged entries are resent. At S0 a full frontier is ~26 KB/tick against ~13 KB for the delta. The client copy is necessary to apply a delta but not sufficient to produce one; producing one requires the sender to know the receiver's baseline.

## Appendix C — Anchors and sources

| Parameter | Anchor | Source |
| --- | --- | --- |
| Tag counts | 1k–100k per site | §13, §57 |
| Update rates | 0.1–10 Hz heterogeneous; process 1–10 Hz; 15 s Prometheus | §13, §74, §97 |
| Viewer counts | 5–20 control room; tens per plant; 10–50 field fleet | §74 |
| Marker legibility | ~32 px on 1080p, B ≈ 2,000 | §13 |
| Cloud egress | $0.05–0.12/GB | §14 |
| IP overhead | ~52 B per message (WS 2 + TCP ~30 with options + IPv4 20) | §11, §73 |
| Cut and encode cost | Assumed ranges b ∈ [0.06, 0.19] µs/entry; c ∈ [0.03, 0.07] µs/entry, at ~3 GHz (not measured) | §12; TDD budgets are targets |
| LAN and WAN latency | ~1 ms; 20–80 ms (WAN 10–100 ms in §14) | §13, §14 |
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
