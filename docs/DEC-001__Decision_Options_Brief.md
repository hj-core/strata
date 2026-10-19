# DEC-001 — Decision Options Brief

| Field | Value |
| --- | --- |
| Document ID | DEC-001 |
| Title | Decision Options Brief — base scenario and design comparison |
| Version | 0.2.0 |
| Status | Draft — for supervisor discussion |
| Last Updated | 2026-10-19 |
| Supersedes | v0.1.0 (2026-10-18, scoped to §1–§116) |
| Related documents | DISCUSSION-SUMMARY (Parts I–IX, §1–§141), PRD-001, TDD-001, PLN-001 |

## 1. Purpose and how to use

The supervisor asked for a comparison showing that the engine has an advantage over other designs. The review's own conclusion is that **no such advantage has been demonstrated**, and that the comparison had never been run properly. This revision is that comparison.

It has one job: **make the advantage question answerable.** It does so by fixing a base scenario, naming the variants in which a mechanism could bind, comparing coherent architecture packages under each, and reporting where — if anywhere — the engine wins.

**What changed from v0.1.0**

1. A **base scenario (S0)** with anchored parameters, plus four **binding variants** (§3). The v0.1.0 brief had decisions but no operating point, so "advantage" was undefined.
2. The design decisions are now compared **at that operating point**, and the comparison reports a verdict per decision (§6).
3. The result is stated as a **package comparison**, not a per-decision tally, because selection and aggregation are coupled (§4.3, §10 of the review).
4. A **sensitivity ranking** (§6.9) identifies which decisions can change the verdict at all.
5. **Three paths** for the supervisor (§9) replace the v0.1.0 "default package": keep and narrow, keep the comparison and drop the engine, or change topic.
6. **Appendix E** — a register of all 83 topics examined across the review, with a disposition for each, so that the breadth is visible and nothing settled is re-litigated.

**How to read it.** §2 is the summary. §3–§5 are the comparison. §6 is the decision-by-decision rationale. §9–§10 are the discussion material for the meeting.

**Two standing rules** (unchanged):

- **Anti-tuning.** A parameter is set from an external anchor or swept; it is never tuned to make a design win (§13).
- **Causality / binding.** A mechanism's claimed benefit must be caused by it and observable at an anchored operating point, and the study must include that point (§113). Tail value (caps, safety bounds) and structural value (correctness, consistency) are the stated exceptions — but they must still be *required*, not asserted.

## 2. Status at a glance

| # | Decision | Status | Comparison verdict under S0 |
| --- | --- | --- | --- |
| D-01 | Track | **decided** — system-development (COMP4299) | — |
| D-02 | Candidate problem | **open** — this document informs it | The engine is **not dominated**, but its advantage is conditional (§5.4) |
| D-03 | Justification anchor | **open** — no stakeholder | Demand exists for **aggregation**, not the coupling (§7.3) |
| D-04 | Selection model | **open** | Explicit selection suffices at S0; geometric only if a free camera is the product |
| D-05 | Cut placement | **open** | Client-side at S0; server-side only if a hard bound is required |
| D-06 | Cut implementation | **open** | **Does not affect the verdict** — build cost and oracle parity only |
| D-07 | Substrate | **open** | **Does not affect the verdict** — client memory and build cost only |
| D-08 | Transport | **open** | Snapshot-first at S0; delta changes magnitude, not ranking |
| D-09 | Aggregation placement | **coupled** | Server required for "aggregation of the unseen"; the requirement is the question |
| D-10 | View-dependence | **open** | Not required at S0; required only for a free-camera overview |
| D-11 | Deployment | **scenario variable** (moved to §3) | **The single largest lever** — it flips the ranking |
| D-12 | Client-memory bound | **open** | **The engine's only unique claim** — and its anchor is unproven |
| D-13 | AI / module fit | **open** | None recommended (COMP4299) |
| D-14 | Scope | **open** | Minimal; dissertation as deliverable |

> **Headline finding.** Under the base scenario, the server-cut engine (D3/D4) is **not the cheapest design** and is **not necessary**. It is a *bounded middle*: worse on cost than fine-grained interest (D2), better than broadcast (D1) only on egress, and uniquely offers a **hard per-client bound** — but that bound is a **self-set product requirement** (K3), not a sourced one. Its advantage therefore rests on two premises, **both currently unsupported**: an anchored thin-client class, and a requirement that the bound be hard.

## 3. The base scenario (S0) and the binding variants

### 3.1 S0 — the modal reference deployment

A mid-size industrial site: ~10,000 tags, 3 metrics each, 1 Hz, ten operators, aggregation and visualization cloud-hosted, browsers over WAN. This is the reference scenario of §13 and the modal modern arrangement of §14 (collection at the edge, aggregation and visualization in the cloud).

The rows run from **workload** (what the site produces, and who consumes it) through **timing** and **encoding** to the **runtime environment**. Each derived row follows the rows it depends on.

| Axis | Value | External anchor | Role |
| --- | --- | --- | --- |
| Devices (N) | 10,000 tags | mid-size plant SCADA/OPC-UA tag counts (§13, §57) | **answer-shaping** — sweep 1k–100k |
| Metrics per device (m) | 3 | temperature / vibration / power | fixed |
| Reading rate (r) | 1 Hz per metric (sweep 0.1–10) | OPC-UA publishing 0.5–2 s; SCADA 1–2 s; BMS 1–15 min (§74, §97). PRD K1 implies 1 Hz at 100k × 3 metrics | **answer-shaping** — sweep |
| Viewers (V) | 10 | control-room staffing (§13) | **answer-shaping** — sweep 1–100 |
| Entry budget (B) | 2,000 | marker legibility: ~32 px on 1080p (§13) | **answer-shaping** — sweep 1k–8k |
| Selection / cut rate | 10 Hz (slow pan) to 60 Hz (free camera, worst case); cut cache makes a static camera ~free | decoupled from render; humans read at a few Hz (§74); camera-motion caveat (§12) | **answer-shaping** — state per result |
| Egress cadence | 1 Hz, tied to r | cadence is the latency bound (§74) | fixed at S0 |
| Render cadence | 60 Hz | smooth panning; client-only, not a server cost | fixed |
| Change-pass (p) | **measured**; `p ≈ min(1, r/egress) × noise_pass` | noise ≈ ε is a simulator choice | measured |
| Value width | **f32 (4 B)** per metric; counts as integers, sums as f32/f64 | OPC UA `Float`, Sparkplug `Float`, Protobuf `float`; f16 is a graphics/ML format, not a telemetry one | fixed |
| Fixed-set encoding | flat value array over resolved slots — device metrics **and** group channels; no per-entry identity; attention derived or sent; the offset array is derivable from the manifest, not sent per tick | layout resolved once | derived |
| Dynamic-set encoding | two options: self-describing record (~16 B full / ~9 B changed at m=3) or slot table + flat (~0.125 B/entry bitmap + values); see §3.1.1 | required when set membership changes | derived |
| Full state per tick | 30,000 device values + ~540 group-channel values × 4 B ≈ **122 KB**; +10 KB attention if sent; ~65–67 KB delta | derived | fixed |
| Deployment | cloud/WAN; egress metered | $0.05–0.12/GB; hybrid edge+cloud is modal (§14) | **answer-shaping** |
| Server | 8 usable cores | typical cloud VM | fixed |
| Client | GPU laptop browser | PRD §9–§10 | **answer-shaping** |

> **Why four cadences.** The review distinguishes the reading rate (source volume), the frame cadence (bounds latency), and the egress cadence (per regime) (§11). S0 adds the **selection rate**, because a camera-driven cut runs per *view update* — not per reading, and not per rendered frame. The CPU figures in §5.2 and §12 are quoted at the **60 Hz worst case**; at a 10 Hz selection they divide by six, and a static camera hits the cut cache (Decision 26) and is effectively free. **Camera motion is an answer-shaping assumption, not a detail.**
>
> **On 1 Hz.** It is a *reporting* rate, not a scan rate: field devices sample faster (vibration at kHz) and report features; historians store at 1 s for critical tags and 10–60 s for the rest. It is at the fast end of reporting and the slow end of control, which makes it **conservative** for a visualization. It is also the rate implied by the PRD's own K1 (300,000 EPS at 100,000 devices × 3 metrics). A uniform 1 Hz flattens a heterogeneous reality (0.1–10 Hz), so it is a **base case to sweep**, not a constant. It does change absolute cost and the snapshot-vs-delta choice, but **not the architecture ranking** — see §5.2.

> **Flat array for fixed sets, records for dynamic sets.** If the client already knows the slot layout — the full state, a subscribed subtree, a direct-select region — the wire carries a flat array of values in canonical order. The offset array the client needs is derivable from the manifest, or sent once; it is not per-tick. This is why the full state is **122 KB of values** (132 KB if attention is sent), not 160 KB. A **dynamic** set (the per-viewer frontier) cannot do this without a slot table: membership changes, so either the server sends per-entry records (~9 B changed at m=3) or it maintains a `slot → node` map and sends it on change. The comparison uses the best encoding for each set type — flat for fixed sets, slot table + flat for the frontier — so the `N/B` ratio is not an artefact of encoding. For a dynamic set at m = 3 and f32, per-entry framing adds ~33% over a values-only model; grouping values under one header keeps it bounded, and a per-value framing would repeat the 3 B header m times.

> **Attention and availability are not free.** Attention is one `u8` per **node** — ~10 KB at 10k, ~8% of the values at m = 3 — not per value; it does not shrink with the value width. **In a frontier it cannot be derived client-side**: a blended group's attention is a rollup over descendants the client does not hold (the aggregation of the unseen), so the server sends it. **In a full state the client can derive it** from the values and the severity policy in the manifest; whether to do so is a design choice — deriving saves ~10 KB but duplicates the policy, while sending 1 B/node keeps one canonical computation. **The comparison credits the broadcast arm with derivation** (best-of-breed) and the frontier arm with receipt (it has no choice). **Availability is never derivable** (the client cannot know source staleness), but it costs no extra bytes as a quiet-NaN sentinel in the value slot, which also forces the slot to be sent when availability changes.

> **Aggregates are payload, not overhead.** The full-state value array is `N × m` device metrics **plus** `G × c` group-channel values — at 10k devices, ~180 groups × ~3 channels ≈ 540 slots (~2% of the device values). They are the *whole* payload of a blended group entry in the frontier, where the client cannot derive them (unseen children). In a full state the client could recompute `sum`/`mean`/`min`/`max`/`count` from the device values, but only non-canonically: the server's reduction order is what makes the aggregate bit-exact. Sending ~2% is cheaper than giving up canonicality, so **both arms send aggregate values**; only attention may be derived, and only in a full state.

> **Why identity is 1–2 B (dynamic sets).** Node indices are dense (`0…N−1`), and the frontier is sorted before encoding, so consecutive gaps average `N/B` — 5 at S0, 20 at 100k/B=5,000 — which fits a one-byte varint; only sparse outliers take two. The precondition is a sort by index (`O(B log B)`, trivial at B = 2,000). A bitmap over all N nodes is cheaper when `N/B` is small (0.6 B/entry at S0) but scales with N and loses at 100k, so 1–2 B is the N-independent choice. The model uses **2 B**, the conservative end.

#### 3.1.1 Dynamic-set encoding, worked

The dynamic set is the per-viewer frontier. **Membership** is the set of node indices in the frontier at a frame — not the values they carry. A **membership change** is an entry entering or leaving that set: a device crossing the frustum boundary, a group expanding into its children (or blending back), a threshold crossing, or an entry evicted by the budget. A value, attention, or availability change *within* an existing entry is not a membership change; neither is a metric drifting below ε. Because membership is server-determined, the client cannot infer which node a value belongs to. Two encodings are viable.

| | Self-describing record | Slot table + flat |
| --- | --- | --- |
| Identity cost | 1–2 B **per sent entry** | 1–2 B **per membership change** |
| Per tick | flags/mask + changed values + attention | bitmap `P/8` + mask + changed values + attention |
| S0 cost | ~15 KB/s | ~13 KB/s |
| Best when | few entries change, or membership churns | many entries change, membership stable |
| Failure mode | identity cost on every record | slot-table desync → keyframe |

**Membership, not delta, is what makes a set dynamic.** The encoding has two jobs: communicating *which* entries are in the set, and communicating *what changed*. Only the second is tied to the delta choice. A full snapshot of the frontier still needs the first — the client cannot infer the server's selection — so it still uses identity or a slot table, just without masks or omission. A full **global** state is a fixed set and needs neither. And when the **client** determines the set (fine interest, direct select), it already knows the membership, so a flat array in its own order suffices; the dynamic set collapses into the fixed-set encoding.

| Who determines membership | Stable? | Encoding |
| --- | --- | --- |
| Server | Yes — full global state | Fixed-set: flat array, no identity |
| Server | No — per-viewer frontier | Dynamic-set: identity or slot table; masks only if delta |
| Client | Yes — fixed subtree | Fixed-set: flat array, client order |
| Client | No — fine-interest subscription | Flat array in subscription order; no identity on the wire |

**Record layout** (m = 3, f32):

| Field | Bytes | Notes |
| --- | --- | --- |
| Node delta | 1–2 | LEB128, relative to the previous **sent** record in sorted order |
| Flags + mask | 1 | bit0 appeared, bit1 removed, bit2 attention-present, bits3–7 slot mask (m ≤ 5) |
| Values | 4k | changed slots in slot order; all slots on `appeared` |
| Attention | 1 | only if bit2 |

Sparse entries are omitted. Five worked cases:

1. **Appeared** — node 4,207, values 21.5 / 0.42 / 1180.0, attention 2, previous sent index 4,180: `delta 27 (1) + flags 0x05 (1) + 3×f32 (12) + attention (1) = 15 B`.
2. **Changed** — same node, slot 2 only: `delta 27 (1) + flags 0x20 (1) + value (4) = 6 B`.
3. **Removed** — node 109: `index (1) + flags 0x02 (1) = 2 B`.
4. **Frame with churn** — `{100,105,109,120} → {100,105,112,120}`, 109 removed, 112 new, 105 slot 2, 120 slot 0, 100 unchanged and omitted: `header 10 + removal 2 + 6 + 15 + 6 = 39 B`.
5. **Steady frame** — only 105 changes: `header 10 + record 6 = 16 B`.

**Slot-table variant.** The server assigns slots 0…P−1 and sends `slot → node` only on change (~2 B per changed slot). Each tick sends a bitmap over slots (P/8 = 250 B at P = 2,000) plus mask and values per changed entry. At S0: `250 B + 1,750 × 7 B ≈ 12.5 KB`, against ~15 KB for records. The bitmap wins when `0.125P < 1.5S`, i.e. when more than ~8% of entries change. Under continuous camera motion the mapping is resent and the advantage narrows; at very low churn the bitmap is wasted and records win.

**Reconciliation.** §5.2 uses the slot-table figure (~13 KB/s); the record fallback is ~15 KB/s. Both sit inside the model's range, and the verdict is unchanged.

**State-based sync vs server-side delta.** A client can keep its last set and diff an incoming **full** set — state-based sync. It is robust: idempotent, no keyframe protocol, no desync. But it does **not** save bytes: to diff, the client must receive the full set, so unchanged entries are resent. At S0 a full frontier is ~26 KB/tick (24 KB values + 2 KB attention) against ~13 KB for the delta — **~2×**. The client's copy is necessary to *apply* a delta but not sufficient to *produce* one; producing one requires the sender to know the receiver's baseline. The server either tracks it (per-session baseline, the engine's choice) or sends full state. The choice is **orthogonal to the architecture comparison** — both arms pay the same ~2×, so the `N/B` ratio survives — and it is a robustness-versus-bytes trade, not a correctness one. A middle ground is a shared version history plus a per-viewer ack; it shares the state instead of removing it, at the cost of a ring buffer.

### 3.2 Anchored variants

Each variant changes **one** answer-shaping axis to a value with an external anchor. None is chosen to favour a design.

| Variant | Change from S0 | Anchor | What it tests |
| --- | --- | --- | --- |
| **S1 — Edge/LAN contrast** | 1 GbE, ~1 ms, unmetered, capable clients | dominant Ethernet; LAN latency (§14) | Whether broadcast wins when bytes are cheap |
| **S2 — Thin client** | client with a **hard memory/CPU bound** | **to be anchored** — embedded HMI, kiosk, low-end tablet (§112) | The engine's only unique claim (D-12) |
| **S3 — High fan-out** | V = 100, r = 10 Hz | plant tens of operators; process 1–10 Hz (§74) | Whether server CPU becomes the wall |
| **S4 — Scattered consumers** | many clients on metered links, no shared POP | cloud egress pricing; no relay amortisation (§17) | Whether per-client bounding beats relay/broadcast |

### 3.3 The binding test

A mechanism's claimed benefit is only meaningful if the benefit is exercised at the operating point (§113). For each variant:

| Variant | Does the entry bound B bind? | Does egress bind? | Does client memory bind? | Does server CPU bind? |
| --- | --- | --- | --- | --- |
| S0 | Only when zoomed out past ~1/5 of the site (§56) | Modestly — metered, but 10 viewers | No — ~4 MB on a laptop | No — ~0.25 core at V=10 |
| S1 | Same | **No** — unmetered LAN | No | No |
| S2 | Same | Yes | **Yes — by definition** | Maybe |
| S3 | Same | Yes | No | **Not yet** — at V = 100 it is ~1/3 of the ~320-viewer wall (60 Hz) |
| S4 | Same | **Yes** | No | No |

**Consequence.** In S0, **no mechanism binds hard**. That is the review's central empirical point (§112): at the modal operating point the engine's guarantees are **tail value** — valuable only if the cap is required. The comparison must therefore be read as *conditional on a variant*, not as a general win.

### 3.4 Anti-tuning declaration

The answer-shaping parameters are **N, B, r, V, deployment, client class, camera motion, and p**. Each is either anchored above or swept. No parameter was set to make a design win. Where a figure is an estimate rather than a measurement, it is labelled. The measured constants that would convert estimates into results are `b` (cut cost per entry), `c` (encode cost per entry), `p` (change-pass), and the actual cloud price — all pending (§7.2).

Two of these have narrower roles than they appear to:

- **`r`** changes **absolute cost and the transport choice**, but **not the architecture ranking**: broadcast and the cut scale by the same `p`, so their ratio is `N/B` (§5.2).
- **Camera motion** changes the **CPU cost of a server-side cut** by up to ~60× through the cut cache — from effectively free for a static overview to the 60 Hz worst case for a free camera. It must be stated per result, not left implicit.

## 4. The comparison method

### 4.1 Metric vector

Six runtime metrics and one project metric. "Advantage" is undefined without all seven.

| Metric | Unit | Why it is in the vector |
| --- | --- | --- |
| Egress | KB/s per client; $/month per client | The metered resource in cloud |
| Server CPU | cores per viewer; total cores at V | The engine's known ceiling (§12) |
| Client memory | MB resident for metadata + buffers | The engine's only unique claim (§112) |
| Client CPU | ms/frame main thread | Thin-client viability |
| End-to-end latency | p99 ms | Freshness requirement |
| Consistency | snapshot atomicity across viewers: yes/no | Structural value (§64) |
| Build cost | engineer-hours | A student project's binding constraint (§69) |

### 4.2 Dominance and trade

> **Dominance.** Design A dominates B under scenario S iff A is **no worse on every metric** and **strictly better on at least one**, measured at S. Otherwise A and B are a **trade**, and the document names the traded metrics.

No weighted score is used. A weighted score hides the choice of weights, which is exactly the tuning trap. Where the comparison produces a trade, the decision is stated as a trade and left to the requirement.

### 4.3 Packages, not toggles

The review established that **selection and aggregation placement cannot be varied independently** (§10, §109): client-side aggregation requires the client to hold the subtree, which forces a broadcast/interest substrate and pushes device-level data onto the wire. Therefore:

- **§6 compares decisions** as rationale — why each option was chosen.
- **§5 compares coherent packages** — the only level at which a verdict is valid.

A "best of each column" combination is not a design and is not reported as one.

### 4.4 The baselines

The comparison is against best-of-breed, not strawmen.

| Baseline | What it is | Why it is included |
| --- | --- | --- |
| **B0 — Snapshot + client cut** | Full state each tick; client cuts and aggregates exactly | The simplest design that meets the modal case (§73) |
| **B1 — IVM + broker** | Materialize/Feldera + NATS/MQTT + an entitlement layer | The composition the engine would have to beat (§63) |
| **B2 — Direct select** | User-driven subtree subscription / drill-down | The commodity alternative for inspection (§108) |
| **B3 — Broadcast + edge relay** | One cloud stream per POP, fanned out locally | The rescue for broadcast under metering (§17) |

## 5. The package comparison

### 5.1 The packages

| ID | Package | Who selects | Who aggregates | Bound |
| --- | --- | --- | --- | --- |
| D1 | Broadcast + client cut | client | either | none |
| D2f | Fine interest (frontier subscription) | client (cut crate in WASM) | server | soft (α) |
| D2c | Coarse interest (region/subtree) | client | client | soft (α) |
| D3 | Server cut | server | server | **hard** |
| D4 | Server cut + lazy manifest | server | server | **hard** + O(B) client |
| D5 | Broadcast + edge relay | client | either | none |
| D6 | Direct select / drill-down | user | server | soft unless capped |

### 5.2 Under S0 (cloud/WAN, N=10k, r=1 Hz, V=10, B=2,000, f32, p=0.5)

| Package | Egress/client | $/month | Server cores/viewer | Client memory | Bound | Consistency | Build (h) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| D1 broadcast | 65 KB/s (flat values incl. groups + bitmaps; attention derived) | ~$15 | ~0.002 | ~4 MB | ✗ | ✓ | ~10 |
| D2f fine interest | ~13 KB/s (flat; the subscription defines the order) | ~$3.0 | ~0.003 | ~4 MB + subtree | soft | ✓ (w/ keyframe) | ~20 |
| D2c coarse | ~33 KB/s (flat values + attention over region) | ~$7.6 | ~0.003 | ~4 MB | soft | ✗ unless snapshotted | ~20 |
| D3 server cut | ~13 KB/s (slot table + flat) | ~$3.0 | ~0.025* | ~4 MB | **✓** | ✓ | ~70 |
| D4 D3 + lazy manifest | ~13 KB/s (slot table + flat) | ~$3.0 | ~0.025* | ~0.1–0.8 MB | **✓** | ✓ | ~85 |
| D5 relay | 65 KB/s ÷ cluster | ~$15 ÷ k | ~0.002 | ~4 MB | ✗ | ✓ | ~20 |
| D6 direct select | ~4 KB/s (flat over subtree) | ~$0.9 | routing | ~subset | soft | ✓ | ~10 |

\* At the **60 Hz free-camera worst case**; divide by six at a 10 Hz selection and by ~60 with a static camera (cut cache). Egress is at 1 Hz.

*Derived at f32, m = 3, c = 3, G = 180, p = 0.5, 30 days at $0.09/GB. Fixed sets use a flat value array (device metrics + group channels) + changed-slot bitmaps and **derive** attention (sending it adds ~10 KB snapshot / ~2 KB delta); the frontier uses a slot table + flat array and **receives** attention (1 B/entry), assumed stable (cut cache). Availability is a NaN sentinel (0 B). Under continuous camera motion the frontier rises to ~17 KB/s. Client memory from §112's descriptor sizes. Build estimates are planning figures, not measurements.*

**The egress ratio is `N / B`, not a bandwidth saving.** A broadcast sends the whole state; the cut sends the frontier. Both scale with the changed fraction, so the ratio cancels `p` and `r`:

```
D1 / D3 = (N × m) / (B × m) = N / B
```

At N = 10,000 and B = 2,000 that is **5×**. At 100k and B = 5,000 it is **20×**. This is the honest form of the engine's egress advantage — the earlier 10× compared an **unsuppressed** broadcast against a **suppressed** cut, which is not like-for-like. `B` is the lever, and `B` is a perceptual cap, not a physical one.

Both arms use the best encoding for their set type (flat for fixed sets, slot table + flat for the dynamic frontier), so the ratio is not an encoding artefact. Under continuous camera motion the dynamic arm pays the slot-table update and the ratio falls to ~4×.

**Reading of S0**

- On **cost**, D2f, D3 and D4 **tie** (~$3.0/month/client); D1 is **5×** worse (`N/B`); D6 is cheapest per client but only serves an explicitly selected subtree.
- On **server CPU**, D3/D4 are ~8× D2f at the 60 Hz worst case — the engine pays for the bound. At a 10 Hz selection the gap narrows to ~1.2×, because the cut runs less often while routing is unchanged.
- On **client memory**, nothing binds: ~4 MB is trivial for a laptop. D4's O(B) win is real but **irrelevant here**.
- **No package dominates.** D2f is the cheapest continuous overview; D6 is cheapest per client for inspection; D3/D4 are the only ones with a hard bound.

### 5.3 Under the variants

| Variant | Winner(s) | Loser(s) | Why |
| --- | --- | --- | --- |
| **S1 edge/LAN** | **D1** (wins on cost and latency) | D3/D4 (CPU-bound), D5 (no benefit) | Bytes are unmetered and latency is ~1 ms; the relay and the cut buy nothing |
| **S2 thin client** | **D4** on memory; **D2f + lazy manifest** ties | D1, D2c, D3 | O(B) client memory is mandatory; but a lazy D2 closes the gap — D4's remaining unique claim is the **hard bound** |
| **S3 high fan-out** | **D2f** on CPU | D3/D4 (CPU cost rises, but at V = 100 it is still ~1/3 of the ~320-viewer wall at 60 Hz) | Per-viewer cut cost is linear in V; routing is not |
| **S4 scattered metered** | **D3/D4** on egress; **D2f** ties | D1, D5 (relay cannot amortise) | Per-client bounding is the only lever when no POP is shared |

### 5.4 Verdict

**The engine's advantage is conditional, and the condition is not yet anchored.**

- **It never dominates.** D2f ties it on egress in every variant; D2f beats it on CPU everywhere; D1 beats it on build cost and in S1.
- **It uniquely supplies** a hard per-client byte/entry bound, exact aggregation of the **unseen**, and a shared cross-viewer snapshot. Of these, the snapshot is also available to broadcast (§64), and "aggregation of the unseen" is only meaningful when the client cannot hold the whole state — i.e. in S2/S4.
- **Its strongest surviving claim** is the hard bound. But the bound is **K3**, a requirement the project set for itself, and §114 established that a self-set "must" does not save a project.
- **The demand test fails for the conjunction.** Published, dated demand exists for aggregation (Ditto #1525, Orion #3816) but **not** for bounded + exact + consistent + per-consumer delivery under a live cut (§104–§105).

## 6. Decision-by-decision comparison

Format: the question, the options, the comparison at S0, whether it binds, and the verdict. **"Does it change the verdict?"** is the column that matters — it separates decisions that create advantage from decisions that only consume effort.

### 6.1 D-04 — Selection model

| Option | Egress | Camera loop | Binds at S0? | Changes the verdict? |
| --- | --- | --- | --- | --- |
| Geometric (cut) | bounded only if server-side | yes | Only when zoomed out | **Yes** — creates the engine's premise |
| Explicit (direct select) | subtree-sized | no | No | **Yes** — the commodity baseline |
| None (broadcast/precomputed) | O(N) | no | No | **Yes** — the null hypothesis |

**Verdict.** At S0 the operators inspect and drill down; a free camera is a *presentation* choice, not a requirement. Explicit selection or none suffices. Geometric selection is justified only if the product is a free-camera overview over a scene too large to enumerate (§109).

### 6.2 D-05 — Cut placement

| Option | Egress at S0 | Server CPU | Client memory | Bound |
| --- | --- | --- | --- | --- |
| Client | unchanged (O(N) or subscribed) | ~0 | O(N) tree + metadata | ✗ |
| Server | ~13 KB/s | ~0.025 core/viewer at 60 Hz (V_cpu ≈ 74–260, §12) | O(B) **only with the lazy manifest** | ✓ |
| Split | bounded coarse | low | fine refinement | partial |

**Verdict.** Client-side placement is sufficient at S0; **client culling does not reduce egress** (§109). Server placement is justified **only** by a hard bound or a thin client — i.e. only in S2/S4. It is the most expensive choice in build hours and the only one that creates a CPU ceiling.

### 6.3 D-06 — Cut implementation

| Option | Build | Semantics | Confounds a placement comparison? |
| --- | --- | --- | --- |
| Library-owned | lowest | SSE only; no B, no Δ, no blended entries, no oracle | yes |
| Library + own policy | medium | fragile — the library has already pruned | yes |
| Own shared crate (native + WASM) | higher | full control, oracle parity | no |

**Verdict. Does not change the verdict.** It changes build cost and comparability only. If a cut is kept, the shared crate is the control configuration (§107); if the topic changes, this decision disappears. **It cannot create an advantage** — it can only make the comparison credible.

### 6.4 D-07 — Substrate

| Option | Artifacts | Client memory at 10k | Loses |
| --- | --- | --- | --- |
| Manifest-only (own cut) | one | ~4 MB (O(N)); ~0.1–0.8 MB lazy (O(B)) | library traversal |
| Tileset (library cut) | two sources of truth | O(N) tree + metadata | the two-box distinction; B |
| Regular cells + own check | manifest + cells | O(B) | nothing relevant |

**Verdict. Does not change the verdict.** It affects client memory and maintenance. The manifest is the source of truth for values; a tileset is a derived artifact (§111).

### 6.5 D-08 — Transport

| Option | Egress at S0 (p = 0.5) | Consistency | Binds? |
| --- | --- | --- | --- |
| Full snapshot | 122 KB/s (device + group values; attention derived) | trivial | No |
| Batched delta | ~65 KB/s (bitmaps + changed values) | needs frame discipline | No |
| Per-value delta | ~14× inflation (§11) | — | **excluded** |

**Verdict.** Snapshot-first at S0. Batching is a **requirement**, not an optimisation: per-value messages carry ~52 B of IP overhead (§11, §73). Delta changes the **magnitude** on the egress axis, not the **ranking** — every package can use it.

**`p` is not free.** `p ≈ min(1, r / egress_cadence) × noise_pass`. At r = egress = 1 Hz, `p` is the noise-pass (0.5 means half the analog values move more than ε in a second — a simulator choice to be measured). At r = 10 Hz with 1 Hz egress, `p → 1` and a delta costs the same as a snapshot. This is why `r` is answer-shaping for **transport**, but not for the **architecture ranking**: broadcast and the cut scale by the same `p`.

### 6.6 D-09 — Aggregation placement (coupled)

| Option | Server CPU | Client holds | Consequence |
| --- | --- | --- | --- |
| Server | one shared pass, amortised | only the slice | exact aggregation of the unseen; consistent snapshot |
| Client | ~0 | the whole subtree | forces broadcast/interest; drags device data out; loses per-consumer scoping |

**Verdict.** Server is required **if** "aggregation of the unseen" is a requirement. That requirement is precisely what is in question. The coupling is real and is the review's one structural invariant; it constrains the design space but does not by itself constitute a need.

### 6.7 D-10 — View-dependence

| Option | Buys | Costs | Binds at S0? |
| --- | --- | --- | --- |
| Required | relevance without asking; server-side egress bound; LOD | camera loop; per-viewer CPU; hysteresis/oracle complexity | No |
| Not required | no camera loop; no per-viewer ceiling; simpler | needs explicit selection or broadcast | — |

**Verdict.** Not required at S0. The data and aggregation are **view-independent**; only selection is view-dependent, and that is a delivery decision (§109).

### 6.8 D-12 — Client-memory bound

| Option | Consequence |
| --- | --- |
| Anchored thin-client bound | Server cut + lazy manifest justified; a physical, citable, non-performance constraint |
| No bound | The cut's necessity is unmet; a snapshot + client cut suffices |

**Verdict. This is the load-bearing external premise.** It requires (i) an anchored device class, (ii) an O(N) baseline that genuinely exceeds the bound, and (iii) the lazy manifest. At S0 (a GPU laptop) it **does not bind**. **The anchor is currently missing** — S2 is named but not sourced. Until it is, the engine's unique claim is a hypothesis.

### 6.9 Sensitivity ranking

Which decisions can change the verdict at all:

| Rank | Decision | Effect on the verdict | Evidence |
| --- | --- | --- | --- |
| 1 | **D-11 deployment** | **Flips the ranking** — broadcast wins on edge/LAN; metering makes egress scarce | §14–§15 |
| 2 | **D-12 client bound** | **Creates the engine's only unique claim**; without it, D3/D4 are unnecessary | §112 |
| 3 | D-04 selection / D-10 view-dependence | Determines whether a cut exists at all | §108–§109 |
| 4 | D-09 aggregation placement | Coupled; changes what "unseen" means | §10 |
| 5 | D-05 cut placement | Matters only after D-04 and D-12 are decided | §107 |
| 6 | D-08 transport | Changes magnitude, not ranking | §11 |
| 7 | D-06 implementation / D-07 substrate | **No effect on ranking** — build cost and credibility only | §111 |

**The finding:** fourteen decisions reduce to **two that create advantage** (deployment, client bound), two that define the problem (selection, view-dependence), and the rest that are engineering. The comparison the supervisor asked for is therefore not a fourteen-row table — it is **a two-variable question**.

One **scenario parameter outside the decision list** also moves the verdict: **camera motion**, via the cut cache. `r` does not move the ranking (§3.4).

## 7. What the comparison does and does not show

### 7.1 It shows

- The engine is **not dominated** — it holds a bounded middle in S0 and wins on egress in S4.
- Its advantage is **conditional**, and the conditions are **named and testable**.
- **Six of eight design decisions do not change the verdict.** Effort spent there is engineering, not argument.
- The **base case does not justify the engine**: at S0 a snapshot plus a client cut suffices (§73).

### 7.2 What would change the verdict (the decisive unknowns)

| Unknown | If it resolves as… | Consequence |
| --- | --- | --- |
| **A thin-client class with a hard bound** | embedded HMI/kiosk with, say, ≤ 2 MB metadata | D4 wins S2 outright; the engine's claim becomes anchored |
| **The hard bound is required** | a sourced requirement, not K3 | The bound moves from tail value to requirement |
| **The selection rate / camera motion** | 10 Hz slow pan vs 60 Hz free camera | Changes D3's CPU by 6×; at 10 Hz its CPU disadvantage nearly vanishes (~1.2× D2f) |
| **Demand for the conjunction** | a published complaint or a standard | The project's justification strengthens |
| **p, b, c measured** | any values | Converts estimates into results; does not change the ranking |
| **Real cloud prices** | $0.05–0.12/GB | Confirms or weakens the egress case; the ranking is robust |

Until the first three resolve, the comparison supports a **conditional claim**, not an advantage.

### 7.3 The demand status

| Half | Evidence | Status |
| --- | --- | --- |
| **Aggregation over a hierarchy** | Ditto #1525 (2022, open); Orion #3816 (2021, open); tippecanoe #324/#830; deck.gl #10795 | **Real, dated, unmet** |
| **Per-consumer selection** | Mirror #3197; interest management | **Solved** |
| **The conjunction** (bounded + exact + consistent + per-consumer, live) | none found | **Unrequested** |
| **Cross-consumer consistency** | none found | **Not a stated pain** |

A universal gap is evidence of absent demand more often than of unmet need (§63). **The demand is for aggregation, not the coupling.**

## 8. Project-frame decisions

| # | Decision | Options | Status / lean |
| --- | --- | --- | --- |
| D-01 | Track | system-development · research | **Decided** — system-development (COMP4299). Novelty not required; depth from anchor + evaluation. |
| D-02 | Candidate | engine · aggregation service · drift twin · accessibility · standards/dataset · extend supervisor topic | **Open** — see §9. The comparison is the instrument for deciding. |
| D-03 | Anchor | stakeholder · dataset · standard · OSS issue · published complaint | **Open** — no stakeholder; the realistic source is the supervisor's lab. Dataset/standard plus a published complaint where one exists. |
| D-13 | AI / module fit | none · decoration · mechanism | **Lean: none.** Any model touched raises the CSAI4299 question (§103, §135). |
| D-14 | Scope | minimal · fuller · dissertation vs v1.0 | **Lean: minimal.** ~304 h total, ~90 h dissertation, ~200 h code; builds of 120–160 h preferred (§69). |

## 9. Three paths for the supervisor

| Path | What it is | Requires | Cost | Risk |
| --- | --- | --- | --- | --- |
| **A — Keep and narrow** | Keep the engine, scope it to the binding region (S2/S4) | An anchored thin-client class **and** a sourced hard bound; evaluate D4 vs lazy-D2 vs D1 | ~200 h | Demand unproven; if the composition passes, the dissertation is a build report |
| **B — Keep the comparison, drop the engine** | The contribution becomes the **characterisation**: which design wins where, with one reference implementation of the preferred package | The comparison itself (this document, extended); one implementation; anchored variants | ~150–180 h | "Not novel" — admissible for system-development, but weak as a research claim |
| **C — Change topic** | Use the comparison as the rationale and related work; move to a supervisor line or the disclosure direction | Name and reach an anchor; keep the analysis | ~120–160 h | Starting over; but the analysis transfers |

**Assessment.** Path A is the only one in which the engine survives, and it depends on an anchor that does not yet exist. Path B answers the supervisor's question honestly and salvages the work; it is the comparison *as* the contribution. Path C is the honest move if neither the bound nor the demand can be anchored.

## 10. Questions for the professor

1. **The bar.** Is a *conditional* advantage — true only in an anchored thin-client/metered region — sufficient for a COMP4299 system-development project, given that the demand for the conjunction is unproven?
2. **The verdict.** If the comparison shows no unconditional advantage, does he prefer Path B (the comparison as the contribution) or Path C (change topic)?
3. **The anchor.** What client class would he accept as "thin" and hard-bounded (S2)? This is the decisive external premise.
4. **The bound.** Is the hard per-client bound (K3) acceptable as a self-set engineering requirement, or must it be sourced from a standard or a deployment?
5. **Data/hardware.** Which of his lines has data or hardware available this term? (For Path C.)
6. **Module.** COMP4299 or CSAI4299 for any design that touches a model?
7. **Deliverables.** Dissertation only, or a demo and benchmarks too? (This decides D-14.)
8. **Next artefact.** Does he want the comparison as a table in the proposal, or a prototype spike first?

## Appendix A — Evidence: competitor feature-request hunt (§104–§105)

| Competitor | Issue | Age / state | Proves |
| --- | --- | --- | --- |
| Eclipse Ditto | [#1525](https://github.com/eclipse-ditto/ditto/issues/1525) | 2022, **open** | Real, dated demand for aggregation over a fleet |
| FIWARE Orion | [#3816](https://github.com/telefonicaid/fiware-orion/issues/3816) | 2021, **open** | Same; workaround = aggregate client-side |
| Mapbox tippecanoe | [#324](https://github.com/mapbox/tippecanoe/issues/324), [#830](https://github.com/mapbox/tippecanoe/issues/830) | 2016–2020 | Aggregation coupled to LOD exists — **precomputed, shared** |
| deck.gl | [#10795](https://github.com/visgl/deck.gl/issues/10795) | 2026, open | View-dependent aggregation is **solved** |
| Cesium | [#6182](https://github.com/CesiumGS/cesium/issues/6182) | 2018, closed | User built the workaround; redirected to tile pipelines |
| Mirror | [#3197](https://github.com/MirrorNetworking/Mirror/issues/3197) | 2022, closed | Per-consumer selection **solved** |

**Method caveat:** a sample, not exhaustive; absence of a request is weak evidence.

## Appendix B — Design comparison (§106)

| Design | Who selects | Who aggregates | Egress | Server CPU | Bound | Best deployment |
| --- | --- | --- | --- | --- | --- | --- |
| D1 Broadcast + client cut | client | either | O(N) | ~0 | ✗ | edge |
| D2 Interest / subscription | client | client | ≈/ > D3 | low | α | cloud |
| D3 Server cut | server | server | bounded | **high** | **✓** | cloud/metered |
| D4 Server cut + lazy manifest | server | server | bounded | high | ✓ (O(B) client) | thin clients |
| D5 Broadcast + edge relay | client | either | O(N)/cluster | ~0 | ✗ | clustered cloud |
| D6 Direct select | **user** | server | subtree | routing | only if capped | inspection |

## Appendix C — Client-memory budget (§112)

| Component | Full manifest (N) | Lazy (B = 5,000) |
| --- | --- | --- |
| Metadata (descriptors) | ~40 MB at 100k | ~120–200 KB |
| Transforms + colours | O(N) | O(B) |
| Total (order) | tens of MB | ~200 KB – 1 MB |

## Appendix D — Anchors and sources

| Parameter | Anchor | Source |
| --- | --- | --- |
| Tag counts | 1k–100k per site | §13, §57 (SCADA/OPC-UA practice) |
| Update rates | 0.1–10 Hz process; 15 s Prometheus | §74 |
| Viewer counts | 5–20 control room; tens per plant; 10–50 field fleet | §74 |
| Marker legibility | ~32 px on 1080p → B ≈ 2,000 | §13 |
| Cloud egress | $0.05–0.12/GB | §14 |
| IP overhead | ~52 B/message (WS 2 + TCP 20 + IPv4 20) | §11 |
| Cut/encode cost | b ∈ [0.06, 0.19] µs/entry; c ∈ [0.03, 0.07] µs/entry | §12 |
| LAN latency | ~1 ms; WAN 20–80 ms | §14 |
| Egress ceiling | 1 GbE ≈ 110 MB/s; ~1–2 Mpps | §11 |

## Appendix E — Register of topics examined

The review ranged well beyond the engine. This appendix records every topic that was examined and its disposition, for three reasons: (i) the breadth of the work is visible to the supervisor; (ii) settled questions are not re-litigated; and (iii) per §83, the analysis is the **transferable asset** if the topic changes.

**Disposition legend**

| Mark | Meaning |
| --- | --- |
| **Adopted** | Used in the current proposal or as a standing method |
| **Baseline** | Retained as a comparison or control, not claimed as a contribution |
| **Finding** | An established result that is not itself a design choice |
| **Evidence** | Supports or refutes a requirement |
| **Constraint** | A limit on scope or feasibility |
| **Gate** | A programme-level decision point |
| **Framework** | A structure used for analysis |
| **Related work** | Used for positioning; not claimed as a contribution |
| **Standing rule** | A method applied throughout |
| **Set aside** | Examined, not pursued (viable elsewhere) |
| **Rejected** | Examined and ruled out |
| **Open** | Unresolved; needs an anchor or a measurement |
| **Closed** | Decided; no longer open |

A parenthetical qualifies the mark — e.g. *Set aside (was primary)* means examined as the leading candidate and later displaced, not ruled out on merit.

### E.1 Design and architecture — the engine's interior

| # | Topic | What was established | Where | Disposition |
| --- | --- | --- | --- | --- |
| 1 | Server-side per-viewer cut (the original engine) | Not defensible as pitched; admissible only as a stated-requirement project | §2–§4, §31, §83 | Rejected |
| 2 | Frustum culling and hierarchical LOD traversal | Commodity: 3D Tiles/Cesium, Potree, OSG, interest management | §22–§23 | Baseline |
| 3 | Per-instance selection | No library exposes it; you compute the cut and keep a node↔instance map | §37, §71 | Finding |
| 4 | Exact hierarchical aggregation | Commodity: SQL, IVM, CRDTs, stream processors | §39 | Baseline |
| 5 | The selection ⊗ aggregation coupling | Structural and implementation-independent; constrains the design space | §10, §109 | Adopted |
| 6 | The placement/crossover study | Rejected as too risky, uncompletable, and implementation-specific | §31–§32 | Rejected |
| 7 | Client-side cut using a library | Viable; loses `B`, hysteresis, blended entries, and oracle parity | §18, §26 | Set aside |
| 8 | Fine vs coarse interest (pub/sub) | Fine ≈ server cut on egress; coarse > server cut when zoomed out | §16, §27 | Baseline |
| 9 | Broadcast + edge relay | Amortises cloud egress only for clustered consumers | §17 | Baseline |
| 10 | Lazy manifest (A′) | Necessary for O(B) client memory; ordinary lazy loading, not novel | §54, §112 | Adopted (if a cut is kept) |
| 11 | Direct select / drill-down | Commodity; covers inspection without a cut; narrows the engine's niche | §108 | Baseline |
| 12 | Attention/severity channel | Orthogonal to availability; a design detail, trimmed for scope | §28 | Set aside |
| 13 | Compaction and node↔slot mapping | Needed only because the cut compacts; identity mapping removes it | §29, §110 | Set aside |
| 14 | Snapshot vs batched delta transport | Snapshot-first at the reference scale; batching is a requirement | §73 | Adopted |
| 15 | Tileset authoring (per-node vs regular cells) | Per-node tiles impractical at 100k; regular cells + own check preferred | §30, §72, §111 | Rejected (per-node) |
| 16 | i3dm, tile lifecycle, visible-id exposure | Tile-level events, not per-instance; you build the per-instance check | §71 | Finding |
| 17 | Hysteresis / anti-flicker | Mandatory if a subscription tracks the camera; otherwise churn | §16, §19 | Adopted (conditional) |
| 18 | Ingress and protocol adapters | Commodity and off-contribution; minimal ingress only | §76 | Set aside |

### E.2 Deployment and economics

| # | Topic | What was established | Where | Disposition |
| --- | --- | --- | --- | --- |
| 19 | Cloud vs edge deployment | Flips the ranking: broadcast on LAN, interest in cloud | §14–§15 | Adopted (S0/S1) |
| 20 | Metered cloud egress | The strongest economic lever for bounding delivery | §14 | Adopted |
| 21 | Header overhead and batching | ~52 B/message; per-value delta inflates ~14× | §11 | Finding |
| 22 | Client-memory bound | The engine's only unique claim; requires an anchored thin-client class | §112 | Open |
| 23 | "Does it bind?" / causality rule | A mechanism must be exercised at an anchored operating point | §113 | Adopted |
| 24 | Performance, nice-to-have, and the product arena | Improvement ≠ study; a nice-to-have needs an owner and a measure | §114 | Finding |

### E.3 Domain pivots considered and set aside

| # | Topic | What was established | Where | Disposition |
| --- | --- | --- | --- | --- |
| 25 | Generic IoT semantic filtering | More crowded than the twin setting; removes the coupling | §21 | Rejected |
| 26 | Dashboards | Coupling survives in three rows; all contested by incumbents | §60 | Set aside |
| 27 | Smart city | Crowded, wrong size, re-introduces spatial views | §61 | Rejected |
| 28 | Data-centre twin | ~1,200 servers; the bound never binds; LAN deployment | §118–§119 | Rejected |
| 29 | Server-room monitoring | Smaller than industrial; mechanism unexercised | §58 | Rejected |
| 30 | Field-technician tablet | Deliberative and intermittent, not real-time; one consumer | §44 | Set aside |
| 31 | Remote industrial / LPWAN | The constraint is the uplink, which aggregation already solves | §42–§43 | Rejected |
| 32 | Fleet / maritime / satellite | Metered links, but few consumers; relay or aggregation suffices | §42, §97 | Set aside |
| 33 | Substation / PMU telemetry | Strong physical constraint; access and domain depth are the barriers | §42 | Set aside |
| 34 | Observability cardinality | Real pain, but it belongs to TSDB/sketching, not this engine | §42, §60 | Set aside |
| 35 | The Google-Maps analogy | The skeleton is solved for shared content; personalised live content is the seam | §45, §52 | Finding |
| 36 | AR / smart glasses; robot knowledge service | Re-skins of the engine; no new requirement | §36 | Rejected |
| 37 | Campus twin with dashboard | A means statement, not a problem; commodity | §102 | Rejected |

### E.4 Candidate topics beyond the engine

| # | Topic | What was established | Where | Disposition |
| --- | --- | --- | --- | --- |
| 38 | Drift-aware digital twin | A cited open challenge; measurable; risks "inject → detect" thinness | §88–§89, §94 | Set aside (was primary) |
| 39 | Accessibility / neurodiversity support | Strongest human-centred candidate; blocked by no participant access | §86 | Rejected (access) |
| 40 | The supervisor's six topics | One template: sensor → Pi/ESP32 → threshold → dashboard → notify | §66 | Set aside (last resort) |
| 41 | QR-code attendance | A QR code is a data carrier, not a presence proof | §125 | Rejected |
| 42 | Salon booking | Assembly as pitched; mechanism reframes exist (no-shows, variable duration) | §122 | Set aside |
| 43 | Extend a supervisor line | Best access via his last-author lines; needs a systems slice | §91, §135 | Open |
| 44 | Standards / dataset project | No stakeholder needed; can be thin without a documented divergence | §41, D-02 | Open |
| 45 | Disclosure-bounded delivery | Most coherent argument; research-shaped; exactness vs DP unresolved | §127–§133 | Open |
| 46 | Live aggregation service | The only half with published, dated demand; commodity-adjacent | §105 | Open |
| 47 | AI-model integration | Not defensible as a bolt-on; raises the CSAI4299 question | §103 | Rejected (as decoration) |
| 48 | Proxy-resistant attendance | Distance bounding, attestation; mostly hardware | §66, §125 | Set aside |

### E.5 Method and defensibility

| # | Topic | What was established | Where | Disposition |
| --- | --- | --- | --- | --- |
| 49 | Problem-first vs domain hunting | Re-skinning changes nothing; anchoring forces the design to change | §36 | Adopted |
| 50 | Sourced vs provisional constraints | The sin is not having constraints but refusing to test them | §36 | Adopted |
| 51 | The failing-baseline method | Compose the best-of-breed, show the failure, build the reconciliation | §40, §65 | Adopted |
| 52 | Reference scenario and anti-tuning | Parameters are anchored or swept, never tuned to make a design win | §13 | Adopted |
| 53 | The composition-project test | Admissible iff the parts conflict and the seam is built and evaluated | §65 | Adopted |
| 54 | The CRUD test | Delete the framework, ORM, and UI kit — is there a paper left? | §120 | Adopted |
| 55 | Prior art as baseline | Maturity is a warning, not a disqualifier | §121 | Adopted |
| 56 | One citation and specificity | The citation must match the contribution's specificity | §124 | Adopted |
| 57 | The "sounds trivial" diagnosis | A research bar applied to a system-development module | §95 | Finding |
| 58 | The "three worlds" result | The engine's claims live in three different scenarios | §50, §97 | Finding |
| 59 | Consistency as constraint, not pain | An artefact of centralisation; resolves to "single source of truth" | §64 | Finding |
| 60 | Integrity | Key decisions derivable cold; no tuned constants as findings | §6, §47 | Standing rule |

### E.6 Literature and related work

| # | Topic | What was established | Where | Disposition |
| --- | --- | --- | --- | --- |
| 61 | SHIP — hierarchical DP over text | DP release is established; textual, one-shot, noisy | §134 | Related work |
| 62 | Edge-IoT hierarchical security monitoring | Two-level TeSSLa/MonPoly; the template is "start from the cost asymmetry" | §138 | Related work |
| 63 | Seven-paper catalogue | Every paper touches a piece; none names the combination | §81 | Related work |
| 64 | `dt-semantic-drift-management` | Model/metamodel drift, not twin↔reality; scope warning | §90 | Related work |
| 65 | Competitor feature-request hunt | Demand for aggregation, none for the coupling | §104–§105 | Evidence |
| 66 | IVM / stream processing / CRDT | Exactness and incremental aggregation are commodity | §39, §63 | Baseline |
| 67 | 3D Tiles / Cesium / Potree / OSG | The cut is commodity and library-provided | §22–§23 | Baseline |

### E.7 Programme, supervisor, and practicalities

| # | Topic | What was established | Where | Disposition |
| --- | --- | --- | --- | --- |
| 68 | COMP4299 vs CSAI4299 | System-development permits novelty-free work; AI focus moves it to CSAI4299 | §35, §103 | Gate |
| 69 | The track decision | System-development, decided 2026-10-19 | §100 | Closed |
| 70 | Supervisor profile and authorship | Never first author; senior on the edge-AI/floor-plan lines | §135 | Adopted |
| 71 | Engine fit to his applications | Parts fit; the engine as contribution does not | §136 | Finding |
| 72 | Résumé / portfolio value | High for systems roles, conditional on building and measuring | §137 | Finding |
| 73 | Feasibility | 304 h total; ~200 h code; the minimal engine is ~225 h | §69 | Constraint |
| 74 | Demo expectation | Cheapest credible demo is a 2D Leaflet viewport (~20–30 h) | §70 | Constraint |
| 75 | IoT device survey | Big data ≠ many consumers; constrained links are low-rate | §97 | Evidence |
| 76 | IoT upload and trust | Six separable questions; cryptography gives integrity, not truth | §139 | Framework |
| 77 | Privacy without demand | The regulator/DPO is the anchor, not the median user | §126 | Finding |

### E.8 Explicitly unresolved

| # | Topic | Status | Where |
| --- | --- | --- | --- |
| 78 | Demand for the conjunction | No published complaint or workaround found | §105 |
| 79 | The thin-client anchor (S2) | Named but not sourced — the decisive missing premise | §112, §3.2 |
| 80 | Exactness vs DP (if disclosure) | Unresolved; the exactness constraint is the idiosyncratic bet | §133 |
| 81 | The measured constants (b, c, p, prices) | Estimates only; each converts an estimate into a result | §19, §7.2 |
| 82 | A reachable stakeholder | None declared; the supervisor's lab is the realistic source | §101, §140 |
| 83 | The supervisor's answers | Bar, data/hardware, module, deliverables — pending the meeting | §10 |

