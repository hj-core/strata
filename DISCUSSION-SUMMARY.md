# FYP Topic Review — Discussion Summary

| Field | Value |
| --- | --- |
| Document | DISCUSSION-SUMMARY |
| Date | 2026-10-13 (extended; first session 2026-10-12) |
| Subject | Review of a proposed final-year project: server-side spatial filtering engine for 3D digital twin visualization |
| Artifacts reviewed | `docs/PRD-001__Product_Requirements_Document.md`, `docs/TDD-001__Technical_Design_Document.md`, `docs/PLN-001__Master_Execution_Sequence.md` |
| Participants | Supervisor (critic) and student (author) — roles later swapped into a viva-style exchange |

This document records the discussion so it can be resumed later. **Part I (§1–§8)** captures the initial evaluation, the student's defence, the technical findings, and the reframe that emerged. **Part II (§9–§20)** records a second session that resumed from the open questions in §7 and carried the work forward into a concrete baseline, a cost model, and a deployment-scenario reframe. It is a record, not an instruction to change the design documents. All numerical results in Part II are **first-principles estimates conditioned on stated constants**, not measurements; the constants are named where they must be measured before use.

---

## 1. What was reviewed

Three design drafts by the student, said to be ~80% complete:

- **PRD-001** — product requirements: a server-side real-time spatial filtering and streaming engine; synthetic simulator, browser 3D client, demo app; benchmark harness across three tiers (4,280 / 50,000 / 100,000 devices); eleven KPIs (K1–K11).
- **TDD-001** — technical design: Rust engine, WebSocket binary transport, per-viewer visibility cut, exact hierarchical aggregation, 27 numbered design decisions.
- **PLN-001** — 25-week execution sequence (2026-09-14 → 2027-03-01), milestones M1–M10, a cut-down ladder.

---

## 2. Initial evaluation (supervisor's opening position)

### Verdict
**Do not dismiss the topic, but do not approve it as pitched.** The "simple filter" characterisation is inaccurate; however the student leads with the weakest claim and never confronts the alternative that could invalidate the premise.

### On "it's just a simple filter"
The engine is a genuine real-time view-dependent streaming server with four subsystems: lock-free latest-wins state table; exact hierarchical aggregation; best-first LOD cut under an entry budget; stateful per-viewer delta encoding — plus an independent brute-force correctness oracle. That is not a simple filter.

### On "just add more bandwidth"
Half right, for a different reason than bandwidth:
- The 30 MB/s baseline is full-key JSON (PRD §7.1), a **strawman**. The student's own ingest record is 10 bytes (TDD §12.1); the honest binary baseline is ~3 MB/s → the real cut is ~10×, not 99%.
- The real constraint is **where per-viewer work lives**, not bandwidth.
- The design principle "the browser draws; the server thinks" is **asserted, not justified**.

### Other findings
- No research question; no related work (plan item A1 has deliverable "—").
- KPIs are acceptance criteria, not findings; most are guaranteed by construction.
- Missing the two baselines that matter: a fat client (binary push + client cull) and a flat server-side filter.
- Demo runs at building scale where the entry budget **B = 5,000 can never bind** — the mechanism is never exercised.
- Scope far exceeds one undergraduate over 25 weeks.
- Plan is stale against the design (PLN v0.2.0 vs TDD v0.28.0) and behind its own milestone M1 (due 2026-10-05; benchmark host still TBD).
- No dissertation anywhere in the plan; P7 ends in a "v1.0 release / demo video", and PLN §1 calls it "a personal project".
- Integrity flag: document polish exceeds typical year-4 output; the student should be able to derive key decisions cold.

---

## 3. The student's defence, and where it landed

The student conceded the technical core and attempted successive reframes. Each was tested and narrowed:

1. **"More bandwidth" objection** → conceded the framing; accepted the honest ~10× figure.
2. **"The cut scales badly"** → conceded: per-viewer cut ≈ 1.8 ms/frame ≈ 0.11 cores per viewer; K10's "10 sessions" quietly admits a ceiling in the tens. Proposed mitigations: incremental cut refinement, decoupling cut rate from encode rate, multi-core viewer parallelism.
3. **"It's a sellable product feature"** → only conditionally valid. The thin-client scenario is the real justification (not "poor client performance is unreasonable"). But **every scenario that would make the feature matter is out of scope**: mobile/native clients, WAN, auth/governance, and the client class is fixed as a GPU laptop (PRD §9–§10). Sellability is not an academic criterion.
4. **"The algorithm is hard to implement"** → partially granted, but: hard-to-get-right ≠ hard-to-invent; the same cut is needed by the hybrid (it relocates, not differentiates); some hard parts are self-imposed and unnecessary (Decision 16 admits a mutex would suffice; Decision 27 optimises for a deployment the PRD excludes). Difficulty + lateness + solo = feasibility risk, not advocacy.

---

## 4. Key technical findings

### 4.1 The per-viewer cut is the architectural cost
Per-viewer cost is linear in viewer count with a non-trivial constant; the hybrid's per-viewer cost is a socket write. The server-side design's only clear win is per-client egress — which matters on WAN/mobile, both out of scope.

### 4.2 The client is O(N) anyway
The design ships a ~40 MB manifest to every client (TDD §10.5), so "the server thinks" saves nothing on client memory. The O(B) client requires a **lazy manifest** (node descriptors only for the current cut) — which was not designed. This is the technical addition that would make the server-side claim real (termed **A′**).

### 4.3 Existing libraries already solve the cut (student's question)
- **3D Tiles / CesiumJS** — hierarchical tile traversal, frustum + occlusion culling, screen-space-error budget, `skipLevelOfDetail` / `skipScreenSpaceErrorFactor`. Essentially the student's traversal model.
- **Potree** — octree point-cloud LOD, client-side adaptive traversal.
- **OpenSceneGraph / osgEarth `PagedLOD`** — paged scene-graph LOD.
- **Umbra** (culling), **UE5 World Partition / HLON / Nanite** (engine-integrated).
- **Server-side interest management** — SpatialOS, Photon, Mirror, FishNet, Unity NGO — "the server decides what the client sees" is standard netcode.

Conclusion: the **cut is not novel**, and the *product feature* is not novel either. What is *not* covered by any of these is **exact live aggregation over a semantic hierarchy under a view-dependent cut** — the aggregation/attention model. That is the student's real, narrow gap.

### 4.4 Collecting visible IDs (student's question)
In a client-cut design the client **computes** the cut, so it holds the visible set by construction — no library hook needed (Three.js: build the frustum and test; Cesium: `tileVisible` event / tile selection state; Potree: selected nodes; OSG: cull visitor; engines: `isVisible`/`OnBecameVisible`). The important implication: if the *server* needed the client's visible IDs, that would reintroduce the per-frame feedback loop **while** also paying the client cut — the worst of both.

### 4.5 The reframe to pub/sub (student's questions)
- Without a per-frame camera loop, the server does **not** send raw data: it can still send all viewer-independent reduction (aggregation, ε-suppression). The loop itself carries no bandwidth cost (~48 B/frame); the cost is *what the server computes per viewer*.
- **Regime 2 (interest/subscription)** is pub/sub with spatial/hierarchical topics. It sends **both** the continuous aggregate stream **and** device-level state for subscribed regions (on change) — not group-only, or it could never expand to devices (violating FR-VS-05).
- **All three regimes are pub/sub**, differing only in subscription granularity, refresh cadence, and who computes the subscription:

| Regime | Subscription | Cadence | Computed by | Server cost |
| --- | --- | --- | --- | --- |
| 1. Broadcast | everything | none | nobody | fanout only |
| 2. Interest | coarse region/subtree | on change | client | fanout + routing |
| 3. Original design | exact frustum | every frame | server | fanout + per-viewer cut |

### 4.6 The three axes
Egress vs N; server CPU vs V; client memory vs N. At most one is binding at the student's scales.

---

## 5. The reframe (emerging thesis)

Separate **infrastructure** from **contribution** from **design variable**:

| Feature | Nature | Action |
| --- | --- | --- |
| Hierarchy tree | infrastructure / substrate | keep, don't pitch |
| Aggregation | differentiated core | keep, pitch as a question |
| Camera cut | solved pattern + design variable | demote; implement the standard version, evaluate |

**Working title:** *Server-side versus client-side view-dependent streaming for large live-telemetry 3D twins: a measured crossover study.*

**Research question:** For a site of N devices and V concurrent viewers, where should view-dependent selection and level-of-detail live — server, client, or hybrid — and where should aggregation live, as a function of N, V, and link budget? What are the achievable exactness, consistency, and latency guarantees at each split, and where are the crossovers?

**Contribution:** the semantic aggregation model (per-unit exact rollups, `(sum,count)` merge, `count` tallies, attention/absence semantics, per-frame snapshot consistency) **plus** the placement/tradeoff characterization — not the transport, and not the cut.

**Candidate architectures:** A (per-viewer server cut), B (broadcast + client cut), A′ (A with a lazy manifest → O(B) client). Same cut crate compiled server-side (A) and to WASM (B); one shared oracle.

---

## 6. Process and integrity points raised

- Literature review must be restored and positioned against 3D Tiles, Potree, interest management.
- Replace self-set KPIs with crossover measurements; drop the nil-predicted sibling-order ablation.
- Bring the justifying scenario into scope (thin clients / constrained link / disclosure control) or drop the feature claim.
- Cut scope (100k tier, SIMD, ablations → stretch); resync PLN to TDD; acknowledge missed M1; replace the "v1.0 launch" phase with a dissertation.
- The student should be able to derive Decisions 16, 14, 24, and 22 cold, without the documents.

---

## 7. Open questions left on the table

1. **Research or engineering degree?** Unanswered — the student must commit definitively.
2. **Which axis is binding** — egress-vs-N, CPU-vs-V, or client-memory-vs-N?
3. **The crossovers** — at what N does broadcast become unacceptable; at what V does the per-viewer cut? If the cut saturates at V≈10 and broadcast at N≫100k, the client-side cut wins at the stated scales.
4. **Why not regime 2?** It may dominate both, keeping server CPU flat in V while bounding egress by subscription.
5. **Subscription-boundary behaviour** — boundaries inherit the flicker/hysteresis problem (R7), now on the data channel.
6. **Aggregation placement** — can aggregation be pushed to the client under subscription, and what breaks?
7. **Is the topic tree the group tree?** And is the semantic aggregation the true thesis, with the cut as a variable?

---

## 8. Bottom line at the time of writing

- As a **topic**: valid and above-average in technical depth, provided it is reframed from "a server-side filter" to a **placement/tradeoff study** anchored on the **semantic aggregation model**.
- As **pitched**: the student leads with the weakest claim (bandwidth) and never tests the premise against a client-side cut or a subscription model.
- The thesis, if there is one, was never the server-side cut — it is the **exact live aggregation over a semantic hierarchy**, with the **placement of selection** as the studied variable.

---

# Part II — Resumption and re-scoping (second session)

Part II picks up the open questions of §7. It records the taxonomy sharpening, the baseline and cost modelling, the deployment-scenario reframe that became the central result, the design of regime 2, and the position on client-side libraries. Every number is an estimate against stated constants; the constants that must be *measured* are called out.

## 9. Resolution of the open questions in §7

| §7 question | Status after Part II |
| --- | --- |
| 1. Research or engineering degree | **Still open.** The student remained undecided through the session; it remains the gate on whether Part II's results are a thesis or an evaluation chapter. |
| 2. Which axis is binding | **Resolved, with a twist:** the binding resource *differs per regime and per scenario* — regime 1 binds on egress (and, in cloud, on egress **cost**); regime 3 binds on server CPU; regime 2 binds on subscription size α. "Which axis binds" has no single answer; it is a function of the scheme. |
| 3. The crossovers | **Computed.** The link crossover is approximately `V_cpu(regime 3) × egress(regime 1)`, landing near **1 GbE-equivalent** under the PRD's original parameters — i.e. inside the LAN environment the PRD assumes. It moves with N, B and r. |
| 4. Why not regime 2 | **Analysed.** Regime 2 can dominate, but only under coarse subscriptions (see §16). A *fine* (frontier) subscription collapses regime 2's egress onto regime 3's, leaving only the cut placement as the difference. |
| 5. Subscription-boundary behaviour | **Confirmed and designed for.** The hysteresis problem (R7) moves from the renderer to the subscription channel; §16 specifies banded, rate-limited subscription updates. |
| 6. Aggregation placement | **Resolved structurally.** Client-side aggregation is *not* an independent axis: it requires the client to hold the subtree, which forces a broadcast/interest substrate. Pushing aggregation out therefore drags more data out. Selection and aggregation placement are coupled. |
| 7. Is the topic tree the group tree | **Yes.** Regime 2's topic namespace is the resolved manifest's node tree, keyed by node index; subtree subscriptions are contiguous index ranges (see §16). |

## 10. The three regimes, sharpened

The student's three architectures were confirmed to map onto §4.5's regimes:

| Student's label | Regime | Subscription | Computed by |
| --- | --- | --- | --- |
| client receives all | 1 — broadcast | everything | nobody |
| client cull + LOD, pub/sub | 2 — interest | region/subtree | client |
| original design | 3 — server cut | exact frustum | server |

Two sharpenings were recorded:

- **All three are pub/sub.** The distinguishing variables are **subscription granularity, refresh cadence, and who computes the subscription** — not "pub/sub vs not." Regime 2 is not special for being pub/sub.
- **The original taxonomy holds aggregation on the server in all three variants.** Aggregation placement is therefore a separate (and coupled) axis, not a fourth regime. This is what makes aggregation the likely genuine contribution and selection placement the studied variable.

**The coupling (formally):** client-side aggregation requires the client to hold the subtree, which only regime 1/2 substrates provide; moving aggregation to the client therefore forces selection toward the client and forces more device-level data onto the wire. Selection and aggregation placement cannot be varied independently.

## 11. Egress, cadence and headers

The student's first egress calculation (`20,000 × 3 × 20 B × 0.4 suppression × 60 snapshots × 8 = 219 MB/s`) was corrected on three grounds:

1. **Unit slip (~8×).** Multiplying bytes/s by 8 yields **bits/s**; the correct figure is **28.8 MB/s = ~230 Mbit/s** (219 was the mebibyte-derived count mislabelled).
2. **The 60 appeared twice.** The formula asserts both a 60 Hz *snapshot* cadence and a 60 Hz *reading* rate, and never names either. `0.4 × 60` means each slot changes 24 times per second — a telemetry rate, not a snapshot effect.
3. **The reading rate was missing**, and it is the parameter that sets volume.

**Corrected decomposition:**

```
egress = (values sent / s) × (bytes / value) + (messages / s) × (header bytes / message)
```

**Three independent cadences** must not be conflated: the **reading rate** (source volume), the **frame cadence** (60 Hz, bounds latency), and the **egress cadence** (per regime). In a frame loop with correct per-value suppression, the per-frame changed fraction is `r / cadence × p`, not an assumed constant.

**Header overhead matters and is not small.** A 10 B payload carries ~2 B WebSocket + 20 B TCP + 20 B IPv4 ≈ **52 B at the IP layer** (~70 B with Ethernet framing). At high message rates this is a **5–7× inflation**; frame-paced batching amortises it to near-zero. The 60 Hz frame is therefore not overhead — it is the **header-amortisation mechanism**. Payload-only byte counts hide this entirely.

**Server egress capacity** was characterised across four ceilings: link (1 GbE ≈ 110 MB/s; 10 GbE ≈ 1.1 GB/s), packets-per-second (~1–2 Mpps, the wall for small messages), process CPU, and path. Capacity is **shared across V viewers**, so `per-client budget = capacity / V`. At the PRD's own K10 (10 sessions × 0.3 MB/s) egress uses ~3% of a 1 GbE link — i.e. **egress is not the binding axis in the PRD's stated environment.**

## 12. Per-viewer cost model and the regime-3 bottleneck

Regime 3's bottleneck is computed as the first resource to saturate as V grows:

```
V_max(ρ) = capacity(ρ) / per_viewer_demand(ρ);   bottleneck = argmin over ρ
```

**Per-viewer cost** = `T_cut(B) + T_encode(B)`, both approximately linear in B. Calibrating to TDD §13's own budgets (K2: 1.5 ms at B = 5,000; encode 0.3 ms at B = 5,000) and re-deriving from first principles:

| | First-principles (hot) | First-principles (cold/p99) | TDD budget |
| --- | --- | --- | --- |
| `T_cut(5000)` | ~0.35 ms | ~0.6–1.0 ms | 1.5 ms |
| `T_encode(5000)` | ~0.17 ms | ~0.25–0.35 ms | 0.3 ms |
| per-entry | 0.06–0.19 µs | | |

Fitted forms: `T_cut(B) ≈ 0.05 ms + b·B`, `b ∈ [0.06, 0.19] µs`; `T_encode(B) ≈ c·B`, `c ∈ [0.03, 0.07] µs`.

**Findings:**

- **T_encode is stable** (sequential, prefetchable); **T_cut is not** (dependent, branchy, scattered) — it ranges ~3× on cache/branch behaviour alone.
- Converting to cores (`× 60`) and dividing an 8-core budget gives **V_cpu ≈ 74 (TDD budget) to ~260 (hot estimate)** at B = 5,000 — a point estimate was replaced by a range. The unsigned constant is `b`, and it is the single number a micro-benchmark should pin down first.
- **Regime 3 is CPU-bound at every B examined**; its NIC wall sits ~5× further out.
- Amortised caveats: the cut cache (Decision 26) makes a static viewer's cut free, so **camera-motion assumptions** set the average; p99 (not mean) is the KPI; the CPU budget (50%) is policy, not physics.

## 13. A defensible baseline (reference scenario)

The student challenged the earlier habit of citing **PRD values as justification** — correctly, since the PRD is the artefact under evaluation and citing it is circular. The agreed method is a **reference scenario**: describe a plausible deployment in prose, derive every parameter from *external* anchors (standards, physics, ergonomics, typical deployments), fix them **before** knowing which architecture wins, then sweep the studied axis.

**Reference scenario:** *a mid-size industrial site (~10k tags), heterogeneous 0.1–10 Hz state updates, 10 operators in a control room, cloud-hosted service, browsers over WAN.*

| Axis | Defensible default | External anchor | Role |
| --- | --- | --- | --- |
| N devices | 10,000 | mid-size plant SCADA/OPC-UA tag counts | axis (1k–100k) |
| m metrics/device | 3 | temp / vibration / power | fixed |
| r state-update | 1 Hz/metric (heterogeneous 0.1–10 as realistic variant) | monitoring cadence; Prometheus 15 s; ISO 10816 | axis / sweep |
| B entries | 2,000 | marker **legibility**: ~32 px on 1080p | axis (1k–8k) |
| V viewers | 10 | control-room staffing | axis (1–1,000) |
| link | 1 GbE (or cloud VM ceiling) | dominant Ethernet | axis |
| latency | ~1 ms (edge) / 20–80 ms (cloud) | LAN vs regional WAN | axis |
| p (change-pass) | **none — measure it** | noise ≈ ε is a simulator choice | measured |
| value width | f16, **symmetric across arms** | sensor ranges fit f16 | fixed |

**The tuning trap (recorded as a warning).** Across the session the student proposed adjusting N down, B down, B up, r up (three times), and f16 off — five parameters, each defensible alone, several of them moving the result toward the server cut. The rule agreed: **a parameter is set from an external anchor or swept; it is never tuned to make the crossover appear**, because an examiner's "why 2,000 and 5, not 5,000 and 1?" has no safe answer if it was tuned. In particular, **N and B change the crossover ratio (~N/B) and r shifts it via the CPU/NIC asymmetry**, so all three are answer-shaping and must be reported as ranges/sweeps.

**Two soft parameters** were flagged for special handling: **B** (perceptual — ground it in marker legibility and sweep it) and **p** (empirical — measure from the simulator). The student's intuition that "1000 visible devices is rare" was refuted from the benchmark topology: a room is ~630 devices, so **two rooms cross B = 1,000**, and 1,000 markers at ~45 px exactly fills a 1080p screen — i.e. 1,000 is the *design case*, not the rare case.

## 14. Deployment-scenario reframe (the central new result)

The wired-LAN assumption of PRD §9 was examined and found **permissible but unrepresentative**, and — more importantly — *load-bearing*: it removes both possible reasons for the feature (cheap bytes and ~1 ms latency), so it selects the one world in which broadcast wins.

**The typical modern arrangement is hybrid edge + cloud:** collection/control at the edge (cannot traverse the cloud: latency, safety, cybersecurity); aggregation, storage and **visualization** in the cloud; browsers over WAN. For a visualization/streaming service, cloud is the modal placement.

**What cloud hosting changes — three things, not one:**

1. WAN latency/jitter (10–100 ms) — threatens K11.
2. Constrained client last hops.
3. **Egress is metered** (~$0.05–0.12/GB).

**The economic axis is the strongest justification the feature has.** Egress in cloud is money, and it scales with `per-client bytes × V × time`. Illustrative cost (100k tier, $0.09/GB, per client/month): broadcast ~$350; server cut ~$70; raw-binary baseline ~$700; JSON baseline ~$7,000. **The PRD's own 30 MB/s framing only matters if egress is billed — i.e. if the server is in the cloud. The PRD describes a cloud-scale problem and then assumes a LAN; that is the internal inconsistency at the root of the whole review.**

**Hosting a Rust engine on cloud is straightforward:** static musl binary → tiny distroless container; ~20 MB + 1.3 MB/viewer memory; 2–4 vCPU covers the reference load. A **VM or container** (Cloud Run / App Runner / ECS) fits; **serverless functions do not** (stateful streaming, per-session tables, timeouts). Load balancer must support long-lived WebSocket connections.

**Recommended study frame:** primary scenario **cloud-WAN**; contrast scenario **edge-LAN**; each with a declared reference scenario. Use **`tc netem`** for deterministic delay/loss rather than the live internet, and **price** instrumented bytes rather than transferring terabytes. This requires a **versioned PRD amendment** (PRD §10 excludes WAN/cloud).

**The scenario-vs-regime trade (recorded for time budgeting):** *scenarios are cheap; regimes are expensive.* Cloud vs edge is a parameter change, not code. So if time is short, cut the **third regime** (model regime 2 analytically), not the contrast scenario.

## 15. Bottleneck comparison at the latest settings (cloud vs edge)

Settings: N = 10k, m = 3 (S = 30,000 slots), r = 1 Hz, p = 0.5, B = 2,000, f16 values, 8 usable cores, cloud egress $0.09/GB, cloud CPU ~$32/core/month.

**Per client:**

| | Egress | GB/month | $/month | CPU/viewer |
| --- | --- | --- | --- | --- |
| R1 broadcast | 120 KB/s | 311 | **$28** | ~0.002 core (fanout) |
| R2 interest (α = 0.1) | 12 KB/s | 31 | **$2.8** | ~0.003 core (routing) |
| R3 server cut | 24 KB/s | 62 | **$5.6** | **~0.025 core** |

**Binding resource per regime:**

| Regime | Scarce resource | Wall |
| --- | --- | --- |
| R1 | egress **cost** / client link | ~36 clients per $1,000/month |
| R2 | subscription α (routing CPU / cost) | ~357 per $1,000/month |
| R3 | **server CPU** | ~320 viewers (range 235–570) |

At cloud prices, bytes are **~40× more expensive than the CPU needed to avoid them**, so the ranking inverts relative to the edge:

| Scenario | Cheapest | Middle | Most expensive |
| --- | --- | --- | --- |
| **Cloud/WAN** | R2 | R3 | R1 |
| **Edge/LAN** | R1 | R2 | R3 (CPU-bound) |

**The central finding:** *the three regimes bind on three different resources — R1 on egress/client-link, R2 on subscription size, R3 on server CPU — and the deployment model (not parameter tuning) decides the winner.* In cloud the server cut sits defensibly in the middle: more expensive than pub/sub, but the only design giving a **hard per-client CPU bound** independent of scene and subscription. In an edge LAN, broadcast wins while the cut is CPU-bound at a few hundred viewers.

## 16. Regime 2 design: topics and subscriptions

**Topic namespace = the resolved manifest's node tree.** No new structure is needed:

| Manifest field | R2 role |
| --- | --- |
| node index | topic id (same key the wire uses, Decision 11) |
| `kind` | which kind of update the topic carries |
| preorder / `parent_idx` | topic hierarchy |
| `subtree_size` + contiguity | a subtree is a **contiguous index range** — subscription is a range, membership is O(1) |
| `metrics` / `channels` / boxes | what each topic publishes; what the client needs to compute subscriptions |

At 10k devices the namespace is ~**10,180** nodes (~10,000 devices + ~180 groups). A viewer holds ~200 **subscription ranges**, not 10,000 topics — so the topic count is *not* the binding metric; **α, the subscribed fraction, is**.

**What R2 adds on top of the (immutable) manifest:** a per-session **node → subscriber map**; a **publish path** (the shared aggregate stream plus device-level values, routed rather than pulled); and a **subscription lifecycle**. Recommended implementation: a **per-node subscriber bitset** (`N × V/8` bytes — ~13 KB at V = 10), with routing as a bit test and fanout as a popcount iteration.

**Subscription updates:**

- **Computed by the client** (the shared cut crate in WASM), sent as **deltas on change — not every frame**, coalesced/rate-limited (e.g. 10 Hz) to avoid rebuilding the per-frame feedback loop.
- **Hysteresis is mandatory** (the enter/exit size bands and budget band): without it, a boundary viewer churns subscriptions every frame, moving the R7 flicker problem onto the data channel and causing keyframe spikes.
- **Newly added nodes = "appeared"** (keyframe), removed = pruned — reusing Decision 15/§10.3.

**The granularity fork (must be fixed before comparing regimes):**

| | Fine (frontier subscription) | Coarse (region/subtree) |
| --- | --- | --- |
| Server sends | frontier values (device metrics + group channels) | **raw device state** for the region |
| Egress | **= R3** | **> R3** when zoomed out |
| Consequence | R2 ≡ "R3 with the cut moved to the client" | the genuinely distinct third architecture |

If R2 subscribes to *exactly the frontier*, it has regime 3's egress and differs only in **where the cut runs**; if it subscribes to *regions* and receives raw devices to blend locally, it sends **more** than R3 when zoomed out. The earlier claim that R2 is "cheapest on egress" was corrected accordingly: **R1 > R2(coarse) > R3 ≈ R2(fine)** on egress, and **R1 < R2 < R3** on CPU.

## 17. Sharing broadcast egress

Question: can regime 1 send once and share bytes across clients? **The computation, yes (already shared via one frame buffer); the bytes, no.**

- WebSocket/TCP is **unicast**, one connection per viewer (FR-TR-01); frameworks' `broadcast()` loops the sockets. The bytes cross the wire V times.
- **IP multicast** would replicate once, but browsers cannot subscribe to it and **cloud VPCs disable multicast**; CDNs cannot cache live personalised streams; P2P relay moves but does not remove the send.
- **Cloud egress is billed per byte leaving the cloud**, so sharing a buffer inside the datacentre reduces nothing.

**The one real mechanism: an edge relay for co-located clients.** If viewers share a network point of presence (one control room / site), the cloud sends **one** stream to a relay, which fans out locally — cutting *wide-area* egress by the **clustering factor**. This makes **"broadcast + edge relay" a fourth architecture**, whose value depends on **client topology (scattered vs clustered)** — a new scenario characteristic, cheap to model (one division), and the one lever that rescues broadcast in cloud.

## 18. Client-side cut and libraries

**Three.js does not expose the visible set.** Its frustum culling is **per-`Object3D`**; for the design's single `InstancedMesh` that degenerates to **one aggregate test** covering all instances. There is no per-instance visibility, no visible-list API, and no visibility event (`onBecameVisible` is Unity). It provides culling *math* (`Frustum`, `Box3`) and a *flag* (`Object3D.visible`), which the renderer reads but never sets.

**This does not cost anything:** in the client-cut regimes the client computes the frontier anyway, and **the frontier is the visible set** — the ids come from the student's own cut crate, not the library. Practical notes: reuse the shared crate; test the manifest geometry boxes; compact visible instances and set `InstancedMesh.count`; beware a stale instance bounding sphere (`frustumCulled = false` is simpler).

**Libraries that do expose visible ids** — **CesiumJS** (3D Tiles: `tileVisible`, per-tile visibility), **`loaders.gl` `Tileset3D`** (headless 3D Tiles traversal, runs in Node and browser), **Potree** (octree, `visibleNodes`), **Babylon.js** (`scene.getActiveMeshes()`), **deck.gl** `TileLayer`, **OpenSceneGraph/osgEarth** — all cut **their own** hierarchies and expose **their own** ids. **None performs the blend/aggregation** (the student's actual contribution), and all force a format mapping (tiles ↔ manifest nodes ↔ topics).

**Position agreed:** keep the cut as **one shared crate** compiled server-side (R3) and to WASM (R2), so the regimes differ only in placement — otherwise a third-party cut **confounds the comparison**. Use Cesium/`loaders.gl` as a **cross-check** and as the **related-work anchor** (they prove the cut is solved; aggregation remains the gap). Do not outsource the cut.

## 19. Open questions remaining

1. **Research or engineering degree** — still the gate; unresolved.
2. **R2 subscription granularity** — fine (frontier) or coarse (region) — decides whether R2 is distinct from R3; must be fixed before comparing.
3. **Client topology** — scattered vs clustered — decides whether the edge relay (fourth architecture) applies; enables the egress-cost rescue of broadcast.
4. **Two scenarios or one** — adopt the cloud-WAN + edge-LAN pair (requires a PRD amendment), or lead with cloud alone (weaker, no contrast).
5. **The measured constants** — `b` (cut), `c` (encode), `p` (change-pass), real cloud prices — each converts an estimate into a result.
6. **The integrity question** — the student must still be able to derive the key decisions cold and must not present tuned parameters as findings.

## 20. Bottom line after Part II

- The **method** changed from "propose a design and hit self-set KPIs" to "characterise where selection/aggregation placement is optimal, as a function of deployment." The **reference scenario** (external anchors) replaced the PRD as the source of parameters.
- The **central result** is a crossover driven by **deployment model**, not by tuning: on an **edge LAN**, broadcast wins and the server cut is CPU-bound at a few hundred viewers; in a **cloud/WAN** deployment, egress becomes the scarce *and metered* resource, interest-based pub/sub is cheapest, and the server cut occupies a defensible middle by giving the only **hard per-client CPU bound**.
- The **honest headline does not flatter the original design.** The server cut is not the winner in general; it is the *bounded* option, and its advantage appears precisely in the region the PRD excludes.
- The **contribution** remains the **exact live aggregation over a semantic hierarchy** (per-unit `(sum,count)` rollups, `count` tallies, attention/absence, per-frame snapshot consistency), with **placement of selection as the studied variable** — now given a deployment axis and a cost metric that make the tradeoff measurable and non-trivial.
- **Cheapest feasible path:** two measured arms (R1, R3) off one shared cut crate and one oracle; R2 modelled analytically; cloud-WAN measured with `netem`; edge-LAN modelled or lightly measured; N, V, r, link, B swept; every constant cited to an external anchor and every tuned value disclosed.
