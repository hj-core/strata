# FYP Topic Review — Discussion Summary

| Field | Value |
| --- | --- |
| Document | DISCUSSION-SUMMARY |
| Date | 2026-10-15 (four sessions: 2026-10-12, 2026-10-13, 2026-10-14, 2026-10-15) |
| Subject | Review of a proposed final-year project: server-side spatial filtering engine for 3D digital twin visualization |
| Artifacts reviewed | `docs/PRD-001__Product_Requirements_Document.md`, `docs/TDD-001__Technical_Design_Document.md`, `docs/PLN-001__Master_Execution_Sequence.md` |
| Participants | Supervisor (critic) and student (author) — roles later swapped into a viva-style exchange |

This document records the discussion so it can be resumed later. **Part I (§1–§8)** captures the initial evaluation, the student's defence, the technical findings, and the reframe that emerged. **Part II (§9–§20)** records a second session that resumed from the open questions in §7 and carried the work forward into a concrete baseline, a cost model, and a deployment-scenario reframe. **Part III (§21–§33)** records a third session that examined a domain pivot (to "IoT"), pinned down what "the cut is solved" means and which libraries implement it, worked through a client-side cut mechanised with a tiles library and its pub/sub variant, and ended in a viva-style examination that rejected the crossover framing and produced a proposed reposition. **Part IV (§34–§47)** records a fourth session: a continued viva in which the student conceded the original proposal, established that COMP4299 permits a system-development project, and then explored the problem-first discipline — re-skinning versus anchoring, the failing-baseline test, candidate pain points, the uplink-versus-egress and edge-versus-cloud distinctions, the field-technician scenario, and the Google-Maps analogy — arriving at a working framing. It is a record, not an instruction to change the design documents. All numerical results in Part II, Part III, and Part IV are **first-principles estimates conditioned on stated constants**, not measurements; the constants are named where they must be measured before use.

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

---

# Part III — Domain pivot, libraries, and viva (third session)

Part III records a third session (2026-10-14). It covers: whether the topic should pivot to "semantic filtering and aggregation for IoT devices"; a precise account of what "the cut is solved" means and which libraries implement it; the mechanics and costs of implementing a client-side cut with a tiles library, and its pub/sub variant; how attention rendering and client instance mapping work in the server cut; and a viva-style examination that ended in a **rejection** of the crossover framing and a **proposed reposition**. Section numbers continue from §20. As in Part II, all figures are first-principles estimates against stated constants, not measurements.

## 21. The IoT pivot: is it defensible?

The proposal examined was to turn the project into "a semantic filtering and aggregation implementation for IoT devices." The student clarified that **"semantic" means a structural/semantic hierarchy** — the tree of §4–§8 with per-unit configured channels — **not** RDF/OWL/SPARQL reasoning.

**Verdict: not defensible as stated**, on three grounds:

1. **It is a more crowded field than the digital-twin setting**, not a less crowded one.
2. **It removes the coupling that made aggregation non-trivial** (§10). Drop the view-dependent selection and the problem reduces to "roll up a static tree," which is solved.
3. **"Implementation for IoT devices" reads as engineering**, not research; adding hardware adds systems scope without thesis content.

| Already does hierarchical IoT aggregation | Coverage |
| --- | --- |
| TimescaleDB continuous aggregates, InfluxDB downsampling | Hierarchical rollups over time-series |
| Flink / Kafka Streams / Spark Structured Streaming | Windowed + hierarchical aggregation |
| ThingsBoard, AWS IoT Analytics, Azure Stream Analytics | Device hierarchies, rule-based aggregation |
| **MQTT Sparkplug B** | A *semantic* industrial namespace + payload model |
| Eclipse Ditto, Azure Digital Twins (DTDL), FIWARE NGSI-LD | Twin abstractions with subscriptions and aggregation |
| C-SPARQL, CQELS, RSP-QL, Streaming MASSIF | Semantic stream processing |
| TAG/TinyDB and sketch-based summarisation | In-network aggregation |

**The gate is the scarce resource.** A pivot is defensible only if a real scarce resource is retained **and** selection stays coupled to aggregation; the natural one is **edge/cloud placement under a metered or constrained link**. Plain aggregation with no named scarce resource is engineering-only.

Decisions were deferred pending the professor's approval; the degree gate (§19.1) remains open.

## 22. What "the cut is solved" means

"The cut" is the per-viewer, per-frame visibility + LOD selection that produces a bounded set of entries under the entry budget `B` (§9.2). "Solved" is a claim about **novelty**, not about effort:

- **The cut as an algorithm — solved.** The mechanism is prior art, deployed at scale.
- **The cut as a design variable — open.** The same standard cut compiles server-side (R3) and to WASM (R2), so the interesting question is *where it runs*, not how to write it.

| Piece of the cut | Prior art that already does it |
| --- | --- |
| Frustum culling | Umbra, every engine, three.js/Cesium frustum math |
| Hierarchical LOD under an SSE / projected-size budget | **3D Tiles / CesiumJS** (`skipLevelOfDetail`), **Potree**, OSG/osgEarth `PagedLOD`, UE5 World Partition / HLOD / Nanite |
| Server decides what each client is sent (interest) | SpatialOS, Photon, Mirror, FishNet, Unity NGO |

"Solved" does **not** mean unimportant, trivial, or unnecessary: the cut is still infrastructure, and getting it exact, deterministic, hysteretic, and oracle-comparable is hard — but **hard-to-get-right ≠ hard-to-invent**. What is *not* covered is **exact live aggregation over the semantic hierarchy under the cut** and the **placement characterization**. That is the seam Part III explored repeatedly.

## 23. Reusable libraries for the cut

| Library | Gives | Lock-in | Exposes visible set? | Aggregation? |
| --- | --- | --- | --- | --- |
| **CesiumJS** (3D Tiles) | Tile traversal, culling, SSE budget, `skipLevelOfDetail` | 3D Tiles; globe engine | Yes — `tileVisible` | No |
| **`loaders.gl` `Tileset3D`** | Headless 3D Tiles traversal (Node + browser) | 3D Tiles | Yes — selected tiles | No |
| **Potree** | Octree point-cloud LOD | Potree octree | Yes — `visibleNodes` | No |
| **OpenSceneGraph / osgEarth** | `PagedLOD`, cull visitor | OSG (C++) | Yes (cull visitor) | No |
| **Babylon.js** | Culling + LOD | Babylon scene graph | Partially — `scene.getActiveMeshes()` | No |
| **deck.gl** | `TileLayer` / `Tile3DLayer` | deck.gl | Yes (events) | No |
| **three.js** | Frustum/`Box3` math, `Object3D.visible` flag | — | **No** — one `InstancedMesh` collapses to one test | No |
| Umbra / UE5 Nanite / World Partition | Occlusion / LOD | Commercial / engine | Engine-specific | No |

**Three caveats.** (1) They cut **their own** hierarchy/format, forcing a mapping to the semantic manifest. (2) Adopting one in only one arm **confounds the placement comparison** (§18). (3) **None performs the aggregation** — the student's actual contribution.

**Recommendation:** build the cut as **one shared crate** (native + WASM) for experimental control; use Cesium / `loaders.gl` as a **cross-check and related-work anchor**; reuse only the math (`glam`/`nalgebra`, `Frustum`/`Box3`).

## 24. Tiles, visibility, and the data seam

**What a tile is.** A **spatial streaming node**: bounding volume + `geometricError` + children (`REPLACE`/`ADD`) + optional content (b3dm/i3dm/pnts/glTF) + transform + (3D Tiles 1.1) metadata. It is defined by the tileset author and is **not** a semantic object — a tiler merges, splits, and re-levels geometry, so a rack may be merged into a building tile, split across tiles, or present at several LODs. It is "a rack" only if **authored 1:1**.

**The LOD criterion is the same idea.** Cesium's screen-space error — `sse = geometricError · H / (distance · 2 · tan(fovY/2))` — is **proportional to projected size**, i.e. to the §9.2 fraction `Δy / z`. This is *why* the cut is commodity: the criterion is standard.

**Can a library expose rack visibility?** Yes, if rack = tile: Cesium `tileVisible`, `loaders.gl` selected tiles, Potree `visibleNodes`, Babylon `getActiveMeshes`, deck.gl tile events, OSG cull visitor (three.js is the exception).

**Can you "subscribe the drawn rack to a value"?** **No — not from the tiles library.** 3D Tiles has no pub/sub topics and no live per-object values; metadata is static. The tile layer answers "what is drawn"; the value/attention layer is the student's. Colouring is therefore either (a) a per-feature property + `Cesium3DTileStyle` (feasible, but off the intended path and scaling with the number of features/tiles per rack), or (b) **separate** — stream values, render your own markers/instances, and never touch tile properties.

## 25. Client-cut mechanised with a tiles library

The flow the student proposed, and its corrections:

1. Engine **resolves the manifest** (§4.6); the client reads it (FR-TD-09).
2. **Author a tileset** from the manifest: one tile per node, `boundingVolume.box` = geometry box, `geometricError` from extent, `refine: "REPLACE"`, `node_index` in metadata — a **build/boot step**.
3. The library **traverses** and selects the visible frontier.
4. **Query** visible tiles → tile → `node_index`.

**Corrections recorded.**
- **The library's cut ≠ the student's cut.** It refines by SSE — no entry budget `B` (`K3`), no budget hysteresis `Δ`, no first-class blended entries, no oracle parity. This is fine **only if the client arm is defined as "the standard client-side cut."**
- **Renderer vs traversal-only** must be chosen deliberately: library draws (overlay values) or library traverses only (draw your own instances).
- **Do not upload visibility every frame** — that recreates the per-frame feedback loop (§4.4). Regime 2 sends **subscription deltas on change**, rate-limited and hysteretic (§16).
- **Still missing:** the values (broadcast or subscription) and the identity/metric metadata. A tileset gives **lazy geometry** (an A′-like benefit), but **not** lazy metric/channel definitions — those remain O(N).

## 26. Pros and cons: library client-cut vs server-side cut

| Dimension | Library client-cut (regime 2/B) | Server cut (original, R3) |
| --- | --- | --- |
| Cut implementation | Solved; author tileset + traversal — low effort | Build index + frontier + cache + oracle — high effort |
| Cut semantics | No `B`, no `Δ`, no blend entries, no oracle parity | Full control (`B`, `Δ`, blend, determinism, oracle) |
| Egress | Unchanged by culling; broadcast = full egress | Bounded, hard per-client byte bound |
| Server CPU/viewer | ≈ 0 (fanout) → scales to many V | `T_cut+T_encode` → `V_cpu ≈ 74–570`, CPU-bound |
| Client CPU/mem | Geometry lazy; values + O(N) metadata unless A′ | Thin client; geometry O(B) if lazy manifest |
| Latency | No camera round trip | Camera loop |
| Thin clients | Bad | **Good** (hard client CPU bound) |
| Deployment fit | LAN / capable clients | **Cloud/WAN, metered/constrained links** |
| Novelty | None on the cut | Only on aggregation + placement |
| Fit to PRD/TDD | Inverts the premise | Native |
| Build risk | Low | High |
| Correctness | Not comparable to the oracle | Oracle, bit-exact |

**Four inversions.** Build cost ↔ control; server CPU ↔ egress; deployment decides; novelty deletes if the cut is the claim.

**Two non-obvious costs.** (1) It **drags aggregation to the client** (or forces duplicate aggregate+raw), because values must reach the client to be overlaid (§10's coupling). (2) `B` is a **product requirement, not an implementation detail** — without it the bounded-workload guarantee (`G2`, `K3`) disappears.

## 27. Adding pub/sub to the client-cut

Adding interest-based pub/sub removes the client-cut's **worst** weakness (broadcast egress). It turns on **subscription granularity** (§16):

| | Fine (frontier subscription) | Coarse (region/subtree) |
| --- | --- | --- |
| Server sends | frontier values | **raw device state** for the region |
| Egress | **≈ R3** | **> R3** when zoomed out |
| Consequence | R2 ≡ *R3 with the cut moved to the client* | the genuinely distinct third architecture |

- **Egress ordering:** `R1 > R2(coarse) > R3 ≈ R2(fine)`. **CPU ordering:** `R1 < R2 < R3` (§16).
- **Improves:** egress (from worst to ≈ R3), server CPU (routing ~0.003 vs cut ~0.025 core/viewer), cloud fit (R2 cheapest in cloud, §15).
- **Costs:** aggregation moves to the client (or duplicates); the **feedback channel returns** (subscription deltas need rate-limiting + hysteresis, or R7 flicker moves to the data channel); binds on **α** (as α→1 egress → broadcast); client holds subscription state (and possibly the subtree); per-frame consistency harder.
- **Consequence:** **fine pub/sub makes R2 and R3 differ only in where the cut runs** — the cleanest experimental control, i.e. the R2 arm of the study, not an alternative to it.

## 28. Attention rendering in the server-cut design

1. **Server computes** one `u8` per node (reserved attention channel / device level) in the same bottom-up aggregation pass — shared across viewers (FR-AG-06, §8.1, Decision 6).
2. **Streams per entry, on change:** frame entry flag `bit2 attention-present` + trailing `[u8 attention]` (§10.4, Decision 15); an attention baseline per session suppresses unchanged values.
3. **Client maps to colour:** keeps a per-node attention map; each frame writes `palette[level]` into the instance colour VBO (§11.1/§11.4, Decision 17). Default display = attention; switching to a metric/channel is client-only.
4. **Attention vs availability are orthogonal** (severity `u8` vs separate NaN/offline markers, §8.1).
5. **Cost:** server-side compare-and-selects folded into the existing pass; client O(entries) colour writes; egress one `u8` per changed entry (inside K5).

**Contrast:** with a static tileset the attention value cannot ride the tile — it must be overlaid (property+style or your own markers). The server cut is cleaner here because the value stream and the selection are the same frame.

## 29. Client instance mapping (node index → instance slot)

The wire key is the node index; the instance slot in the `InstancedMesh` is **compacted** (≤ `B`), so the client owns a `nodeToSlot` table built from the **`appeared`/`removed` diff** (§10.3, Decision 11):

- `appeared` → allocate a slot, write the **static** transform from `manifest[node_index]` (position or geometry box), write colour.
- `removed` → swap-remove and clear the mapping.
- otherwise → look up the slot; update colour (attention/value) only.
- `mesh.count = activeCount`; two meshes (marker for devices, box for groups); picking reuses `slotToNode` (§11.3); the table is rebuilt on reconnect with the keyframe (per-build indices).

Transforms are static (topology fixed at boot), so only colours change per frame; the frontier order is unstable, which is why the diff — not entry position — defines the mapping.

## 30. Tileset authoring: limitation and performance

Authoring is a **mechanical transform of the manifest**, not research-hard, but **scale-sensitive**:

| | Estimate |
| --- | --- |
| `tileset.json` per tile | ~400–600 B (box + transform + geometricError + metadata) |
| Building tier (~4,460 tiles) | ~2 MB — fine |
| Full tier (~101,781 tiles) | ~40–60 MB, explicit (implicit tiling needs a regular octree — not applicable) |

**Performance implications:** client startup parse O(N); client memory O(N) for the tile tree (content lazy, tree not); per-frame traversal moves to the client (plus occlusion culling, which PRD excludes); on-demand content loading causes **pop-in** (can hurt freshness/consistency); steady-state value egress unchanged; no `B`/`K3`; not oracle-comparable.

**Alternative:** compile the **shared cut crate to WASM** — no authoring, no second artifact, same cut semantics across arms, client memory as designed. Use tiles only as a cross-check.

## 31. The viva: challenges and landings

| Challenge | Landing |
| --- | --- |
| "Why server-side cut? It reinvents wheels and bottlenecks CPU. Use client-cut." | Conceded the cut is solved and server CPU is a real ceiling (`V_cpu ≈ 74–570`); defended only that client-side **culling does not reduce egress** — culling happens after the bytes left |
| "Client-cut need not broadcast; it can use pub/sub." | Conceded broadcast is not necessary. Fine pub/sub → egress ≈ R3; coarse → > R3 and aggregation moves client-side; R2 binds on α. Retracted "only the server cut bounds egress" |
| "R1 can use a relay so co-located clients share the egress." | Conceded — the **edge relay** (§17); cloud→relay is one stream, divided by the clustering factor. Adds a scenario axis (**client topology**) and a fourth architecture; doesn't reduce client work |
| "Name scenarios where R3 beats R2/R1." | Thin/embedded clients (hard client CPU bound); scattered metered links; coarse-subscription overshoot / exact interest; disclosure control; exact server-side aggregation; locked-down deployments. Conceded **R3 is a bounded niche**, not a general winner |
| "Defend the topic if it doesn't beat R2." | Reframed: the contribution is the aggregation model + the placement/coupling characterization; the cut is a controlled variable |
| "Is the problem worth study? Is the answer unclear?" | Partly. The *direction* is guessable (weak); the **crossover locations** (unknown constants), the **coupling**, the **guarantee differences**, and the **counterintuitive** result are not obvious |
| "Will you implement all four regimes + benchmarks? How long?" | No. Two measured arms (R1, R3) off one shared crate + one oracle; R4 a relay process; R2 modelled with a spot-check; surface obtained by a **calibrated cost model + coarse measured grid**, not exhaustive sweeps. ~19 weeks remain; full four-regime study is 40+ weeks |
| **Rejection:** too risky; can't finish on time; doubtful research value; the surface is only a ranking; results depend on implementation and don't generalise | Conceded. The surface is overbuilt for a ranking; the crossover is implementation-specific (`b`, `c`, `p`, prices) and generalises poorly. Proposed a reposition to **invariant** results: the aggregation guarantees and the structural **coupling** result |

## 32. Rejection and proposed reposition

The session ended with the professor **rejecting the crossover proposal** as too risky, likely uncompletable in time, of doubtful value (a surface that only yields a ranking), and non-generalisable (results depend on the implementation).

**Student's concession.** The research value cannot rest on measured constants. It must rest on what is **invariant**:

1. **The aggregation model** — exact rollups over a semantic hierarchy under bounded selection, with provable properties (mergeable accumulators, canonical reduction order for bit-exactness, attention/absence semantics orthogonal to availability, per-frame snapshot consistency), validated by an oracle.
2. **The coupling result** — client-side aggregation requires the client to hold the subtree, forcing a broadcast/interest substrate and dragging selection client-side. **Structural and implementation-independent**; it constrains the design space regardless of constants.

**Proposed reposition.** Keep the engine, drop the crossover as the headline:

> A **hierarchical telemetry aggregation and bounded-delivery engine**: exact live rollups with severity and absence over a semantic tree, delivered per consumer as a bounded slice — evaluated on observability-style rollups and on a constrained-link scenario, with the 3D twin as one demo client.

**Main component** identified as the **hierarchical semantic aggregation engine** (tree + per-unit channels + exact rollups + attention + snapshot consistency, coupled to bounded selection); index, cut, transport, client, simulator, oracle are supporting. **Reusable** toward: observability rollups with severity; constrained-link/remote-site telemetry; disclosure control / multi-tenant delivery; a general "worst-state-in-subtree" severity primitive; and the original digital twin. **Reuse broadens usefulness, not novelty** — the differentiator remains the exactness/consistency guarantees under bounded selection.

**Status: unresolved.** The reposition had not been approved at the time of writing; the degree gate remains open.

## 33. Open questions carried forward

1. **Degree framing** (§19.1) — still the gate.
2. **Approve the reposition or not** — engine contribution evaluated on observability rollups + constrained-link, versus something narrower the professor names.
3. **The anchor application** — observability rollups or constrained-link telemetry.
4. **The coupling result** — is it strong enough to carry a research thesis, and can it be stated as a bound rather than measured?
5. **Guarantee ordering** — do exactness/consistency requirements flip the regime ranking, independent of implementation constants?
6. **Integrity** — no tuned parameters presented as findings; key decisions derivable cold.

---

# Part IV — Viva continuation, problem-first discipline, and the Google-Maps framing (fourth session)

Part IV records a fourth session (2026-10-15). It continues the viva-style examination begun in Part III, then moves into the methodology of finding a problem: the distinction between re-skinning a design and anchoring it, the failing-baseline test, candidate pain points and domains, the uplink-versus-egress and edge-versus-cloud distinctions, the relay, the field-technician scenario, and finally the Google-Maps analogy that produced a working framing. Section numbers continue from §33. As before, all figures are first-principles estimates against stated constants, not measurements.

## 34. The viva continuation: the defence narrows

The examination resumed with the supervisor pressing on the stated design; the student conceded that the original proposal "does not stand at the scenario it describes" and that better solutions exist for it. Successive challenges and their landings:

| Challenge | Landing |
| --- | --- |
| "The cut is solved and trivial — use a library." | Conceded: hierarchical LOD/tile traversal is commodity (3D Tiles/Cesium, Potree, OSG, interest management); build nothing bespoke to *prove* a point |
| "It reinvents wheels and bottlenecks the server CPU." | Conceded the CPU ceiling; only the egress argument survives, and only where a client-side cut cannot touch it |
| "Nothing requires server-side selection." | Conceded again; the cut is a design variable, not a contribution |
| "Why not regime 2 / pub/sub?" | Conceded broadcast is not a must; fine pub/sub ≈ R3 on egress (via the frontier) |
| "Why not an edge relay?" | Conceded the relay amortises cloud egress for clustered consumers |
| "Name a case where R3 beats R2/R1." | Only thin clients, disclosure control, scattered metered links, and server-side exactness; R3 is a bounded niche, not a winner |
| "Is the topic defensible if it does not beat R2?" | Reframed: the contribution is the aggregation model plus the selection↔aggregation **coupling**, not the cut |
| "Is the problem worth study — is the answer unclear?" | Partly: the direction is guessable; the crossover locations, the coupling, and the guarantee differences are not |
| "Will you build all four regimes and the surface?" | No: two arms plus a modelled regime; a calibrated model, not exhaustive sweeps |
| **Rejection:** too risky, uncompletable, doubtful value, the surface is only a ranking, results are implementation-dependent | Conceded; the research value cannot rest on measured constants — only on **invariant** results (the coupling and the guarantees) |

## 35. Programme scope: COMP4299 permits a system-development project

The student supplied the FYP requirements. The decisive text: the module supports **either a system development project or a research-oriented project with a software development component**, requiring students to "integrate and extend knowledge," and covering the full lifecycle — problem formulation, background and related works, methodology/system design, development and verification, critical evaluation, project and risk management, formal communication, reflection, and societal/ethical implications. **All projects must include a substantial software development component.**

Two programme codes matter:
- **COMP4299 (BSc Computing):** topics must address a problem in a relevant area of computing — the topic qualifies.
- **CSAI4299 (BSc AI):** topics must have a **clear AI focus** — the topic would **not** qualify without change.

The student is on **COMP4299**. Consequently: **a system-development project is admissible, and novelty is not required** — the requirements are a real problem, substantial software, integration *and extension*, verification, critical evaluation, and the lifecycle. The supervisor's standing caution, however, is that "integrate **and extend**" still rules out pure assembly, and the missing apparatus (related work, evaluation, reflection/ethics, a dissertation — not a "v1.0 launch") remains owed.

## 36. Re-skinning versus anchoring: the pattern named

Across the session the same engine was re-presented as a digital twin, generic IoT, AR-on-smart-glasses, a robot knowledge service, and spatial knowledge for robots. The supervisor named this the **solution-looking-for-a-problem** pattern and drew the line:

| | Domain-hunting | Re-anchoring |
| --- | --- | --- |
| Direction | design → domain | **problem → design** |
| Held fixed | the design | **the problem/requirement** |
| Variable | the domain label | **the design's scope/features** |
| Requirement from | invented to fit | **external** (user, standard, physics) |
| Output | a demo/label | **a solution + a yardstick** |
| Test | "does the design fit?" | **"does it meet the need?"** |
| Falsifiable? | no | **yes — "wrong tool when…"** |
| Failure mode | contrivance | scope creep |

**Three tells.** (1) If moving to a new domain changes **nothing** about the design or the requirements — only the label — it is hunting; if it imposes a requirement you did not choose and forces the design to change, it is anchoring. (2) Hunting has a **demo**; anchoring has a **yardstick**. (3) Anchoring lets you **abandon parts of the design**; never being willing to cut a feature means defending, not anchoring.

The student's proposed framing — "find a problem that needs real-time, massive EPS, and selection" — was identified as **hunting**, because those are the design's *inputs*, not a pain. A problem-first start begins from a pain and **derives** the requirements (which may include some, all, or none of those inputs). When challenged on how to start without constraints, the distinction was drawn between **sourced constraints** (external facts that anchor) and **provisional constraints** (working hypotheses that must be tested); the sin is not having constraints but refusing to test them. A **minimal pain hypothesis** — one sentence about the world, no solution properties — was proposed as the correct starting point.

## 37. What "the cut is solved" means (restated), and where it is not

- **Hierarchical tile/LOD traversal — solved:** 3D Tiles/Cesium, Potree, OSG/osgEarth, interest management. The cut is a commodity and a **design variable**, not a contribution.
- **Per-instance selection — not provided by libraries.** Established in detail:
  - Cesium `i3dm` renders instances efficiently but gives **tile-level** visibility, not per-instance ids; batch-table properties can carry a node id for `pick`.
  - three.js/Babylon `InstancedMesh` render instances with **no per-instance visible-list API**.
  - Potree exposes **node-level** visibility (`visibleNodes`); deck.gl offers picking, not a visible list.
  - **Consequence:** for per-instance entries with values, **you compute the cut yourself** (frustum tests over your boxes/positions — cheap), and maintain the `node↔instance` mapping for colouring and picking.
- **The per-node-tile option** (one Cesium tile per node, no instancing) would give per-node ids for free, but is **impractical at scale** (≈100k tiles → huge tileset JSON, O(N) client memory/traversal), yields **no control** over cut semantics (`B`, hysteresis, determinism, oracle), and is **client-side only** — ruining the server-side visibility needed for bounded delivery.
- **Conclusion:** build the selection; use Cesium/three.js purely as the **renderer** (`i3dm`/`InstancedMesh`). Regime choice (server- vs client-cut) becomes a **placement** variable.

## 38. Transport and metadata mechanisms

- **Cesium picking:** `scene.pick` returns a `Cesium3DTileFeature`; the node identity must be stored as a **batch-table property** to recover the node index. Clicking a blended rack yields the **group node's** index. The engine never sees the click — the **client** raycasts, maps `instanceId → slot → node index`, and publishes the index as the inspection target (FR-VS-02).
- **Lazy manifest:** necessary, not novel. Shipping only the descriptors for nodes the client has seen makes client memory **O(B)** rather than O(N) (≈40 MB at 100k). It is ordinary lazy/on-demand loading (cf. 3D Tiles streaming content); in the **server-cut** regime it lazifies the *dictionary*, while the **client-cut** regime additionally needs a **coarse culling structure** (a different problem).
- **Whole-tileset reuse:** reusing a tileset **across clients** is good (static geometry is cacheable, unlike per-client values); reusing it as **multiple placements** in one scene is inefficient (each copy is traversed/cached separately — use `i3dm`). Reuse covers **geometry only**; values, attention, and the node↔instance mapping remain the engine's.

## 39. Exactness is necessary but not scarce

The supervisor's earlier emphasis on "exact" rollups was corrected. Exactness (correct, and in the stronger sense bit-reproducible against a reference) has value for trust, cross-consumer consistency, composability, delta stability, and verifiability — but:

- **SQL** aggregates (`sum/mean/min/max/count`) are exact (though not bit-reproducible across plans);
- **incremental view maintenance** (Differential Dataflow, DBSP, Materialize, RisingWave) is exact and incremental;
- **CRDTs** give exact, convergent, mergeable aggregation;
- **stream processors** compute exact operators (subject to FP-order under parallelism).

So **exactness alone is commodity.** Achieving it requires only the mergeable reductions, mergeable accumulators, a canonical reduction order for bit-exactness, availability semantics, a per-frame snapshot, and an oracle — all known disciplines. The genuine difficulties are **bit-exactness under parallelism/SIMD**, **exactness under partial availability**, and **exactness coupled to a bounded, consistent, per-consumer selection**. The last is the residue.

## 40. Competitors, and the glue question

The competitor landscape was set out by capability (see §23–§24) with the **closest threats** being: **IVM engines** (Materialize/DBSP/Differential Dataflow) for the aggregation; **interest management + a broker** for the delivery; **integrated twin platforms** (Azure Digital Twins, AWS TwinMaker, Eclipse Ditto, NGSI-LD) for the product angle. The gap — hypothesised, to be verified — is the **bottom row** of the matrix: **all of exact hierarchical rollup, consistent snapshot, per-consumer selection, hard per-consumer bound, and constrained delivery, coupled.**

**Will it fall to the glue criticism?** It does **iff** the naive composition of best-of-breed components satisfies the requirement. The test is empirical, not rhetorical: **wire the components the obvious way and see whether the coupling fails.** It likely does — the contracts conflict (event-time windows vs per-frame snapshots; relevance vs exactness; routing vs bounding) — so the reconciliation is engineering, not glue. **The way to defeat the criticism is to build the baseline and show the failure**, not to argue.

**Baselines are cheap and are not a second system:** modes of the engine (broadcast, static/manual selection, no bound, no snapshot) and one configured off-the-shelf stack. The comparison is the *evaluation*; the system plus the coupling is the *contribution*. Two cautions: the alternatives must be **best-of-breed, not strawmen**, and building them must not crowd out the system.

## 41. No access: use public anchors

The student has no industry/lab/dataset access. **External anchors are still available:**

| Kind | Available? | Gives |
| --- | --- | --- |
| User/site/contact | No | lived problem, validation |
| **Standards/specs** (OPC-UA, ISA-95, LoRaWAN, IEC 61850, C37.118) | Yes | citable requirements |
| **Literature** (latency budgets, scale, failure reports) | Yes | anchors, related work |
| **Public datasets/benchmarks** | Yes | external validity |
| **Simulators** (Gazebo, `netem`, synthetic generators) | Yes | a testbed |

Without a user, the project's structure becomes **a cited requirement → a best-of-breed baseline that fails it → the composition → an evaluation** — a legitimate systems/benchmark shape. The burden shifts onto the **citation** and the **baseline**, and the temptation to fabricate a scenario must be resisted.

## 42. Candidate pains and domains

The telemetry domain is **not** exhausted — real open pains include cardinality explosion/cost, anomaly-preserving reduction, energy/carbon ("green observability"), late/out-of-order correctness, cost-optimal placement, semantic interoperability, and root-cause across hierarchies. **But the pains that are genuinely unsolved mostly do not fit the engine; the pain that fits (bounded, consistent, exact rollups) is largely solved by IVM/brokers.** That is why "everything feels solved": the domain was being judged through the engine's lens.

Candidate pains with a physical, citable constraint:

| # | Pain | Anchor | Fit |
| --- | --- | --- | --- |
| 1 | Substation link cannot carry full PMU/synchrophasor telemetry, but operators need wide-area aggregates | IEEE C37.118; utility telecom limits | strong |
| 2 | Remote industrial sites over LPWAN/satellite cannot carry the tag stream | LoRaWAN (0.3–50 kbps), NB-IoT (~100 kbps), satellite cost | **strongest, lowest effort** |
| 3 | Vessels/aircraft report over metered satellite; fleet aggregates needed | Inmarsat/Iridium cost | strong |
| 4 | High-cardinality observability: costly, stale, unbounded per-tenant delivery | cardinality/cost literature | strong on scale, weak on physical constraint |
| 5 | Renewable-farm SCADA over constrained comms | IEC 61400-25 | moderate |
| 6 | Water/wastewater remote sites | LPWAN limits | low (same shape as #2) |

**#2 (remote industrial/LPWAN)** was ranked lowest effort: the constraint is physical and citable, the engine fits with minimal change (keep state, hierarchy, aggregation, selection, transport, simulator, oracle; cut real-time and LOD), the data is synthetic, and the evaluation is a straight byte-budget comparison.

## 43. Uplink vs egress; edge vs cloud; the relay

- **"Remote industrial" ≠ LPWAN.** Connectivity is a scenario choice — LPWAN (bandwidth/duty cycle), satellite (metered cost/latency), cellular (budget/coverage), private radio, leased line — and the constraint differs by technology. The pain exists only if the chosen link is **constrained**.
- **Sending to the cloud does not remove the constraint — it relocates it.** The site→cloud **uplink** is still the site's link; the cloud→consumer **egress** may be constrained or **metered** (egress is billed). Which link is the bottleneck decides **where the engine sits**:
  - **Uplink-constrained → engine at the edge** (aggregate before transmitting);
  - **Egress-constrained → engine in the cloud** (bound per-consumer delivery);
  - **Neither → no pain** (broadcast suffices).
- **The uplink is solved by aggregation, and selection does not help it** — the uplink carries the *union* of consumer interests, so per-consumer selection reduces only the **downlink/egress**. For the uplink, a gateway's aggregate-and-forward is sufficient and commodity.
- **The office relay** amortises **cloud egress** for **clustered** consumers (one stream to the relay, fanned out locally). It fails for **scattered** consumers (no amortisation), **thin** consumers (they still receive the full stream), and **heterogeneous** interests (the relay sends the union). So the pain requires consumers that are **scattered**, **thin**, or **heterogeneous**.
- **"Local aggregate + upload"** therefore fails **not** on the uplink but on consumer-side requirements: per-consumer bounded delivery, cross-consumer consistency, exactness under partial availability, mixed per-field freshness, dynamic selection under a bound, severity propagation, and fleet-wide consistency — i.e., the coupling.

## 44. The field-technician scenario

A concrete human anchor was examined: a field technician servicing an asset, using a tablet to see telemetry **near the point of issue** to grasp the **cause** and the **effect of the fix**. The twin gives: asset location, status at a glance (colour by severity), surrounding context (adjacent equipment, zone aggregates), per-asset readings/history, fault extent, and safety context. The technician's **position** drives a **bounded, relevant slice** — the engine's shape.

Two corrections were recorded:
- The need is **deliberative, not real-time.** Safety-critical reactive information comes from **onboard sensors**, not the twin; the twin supplies status/history/spatial context at **seconds** scale.
- Usage is **intermittent**, not continuous — the technician opens the twin at an asset, so the "many moving viewers at 60 Hz" premise still fails.

The derived requirements were graded: **position-dependent selection holds** (intrinsic to the task); **aggregation holds** but **exactness is weakly motivated** for human diagnosis; **"real-time" is over-claimed** — the honest requirement is **responsive** (sub-second to ~1 s), justified by watching a fix take effect. The scenario also has only **one** consumer (a weak bound pain) and does not yet test the alternatives.

## 45. The Google-Maps analogy

The strongest structural analogy found: Google Maps is a **tiled, view-dependent, lazily streamed, edge-cached, bounded** delivery system for **static substrate + a live layer**. Its **selection is client-side** (the client requests the tiles for its viewport), and it knows each object's properties because **vector tiles carry features with attributes** that the client styles; genuinely live layers (traffic) are a **separate channel joined by id**.

| | Google Maps | The engine |
| --- | --- | --- |
| Substrate | map tiles (geometry + identity) | manifest/tiles (geometry + node id) |
| Live layer | traffic (joined by id, styled) | value stream (joined by id) |
| Selection locus | **client** (viewport) | **server** (for bounded/per-consumer delivery) |
| Personalization | **shared** layer | **per-consumer** bounded slice |
| Aggregation | pre-aggregated, approximate, **shared** | **exact, hierarchical, per-consumer, consistent** |

The key divergence: Maps can be **client-selection because its content is shared**; **personalized content forces server-side selection**. So the skeleton (static substrate + live channel + join + view-dependent selection + LOD + lazy + bounded) is **solved and proven by Maps**, while the residue — **personalized, exact, per-consumer, consistent hierarchical rollup** — is what Maps does **not** do.

## 46. Working framing at the end of the session

> **A Google-Maps-shaped service for live telemetry**: static substrate (geometry + node id) plus a live value channel joined by id, **view/position-dependent**, **lazily streamed**, and delivered as a **bounded, exact, consistent, per-consumer slice** — where the tile/streaming half is a solved substrate (Maps, 3D Tiles) and the **selection↔aggregation coupling** is the contribution.

Evaluated as a **COMP4299 system-development project**: reuse the engine (state, hierarchy, aggregation, selection, transport, simulator, oracle), cut real-time/LOD/3D as the pain does not demand them, **build the selection** (cheap; needed for control, placement, determinism), use a library renderer (`i3dm`/`InstancedMesh`) for drawing only, and **prove the coupling by building the baselines** (broadcast, static selection, relay, naive IVM+broker) and measuring where they fail.

**Status:** a working framing, not yet an approved problem. The unresolved gates are a **sourced pain and requirement** (candidate #2 is the current lead) and whether the hierarchy of open decisions can be closed in the remaining time.

## 47. Open questions carried forward (Part IV)

1. **The sourced requirement** — which pain, from which citable anchor (candidate #2 leads)?
2. **The constraint** — is it the **uplink** (then aggregation suffices; no project) or **egress/consumer-side** (then the coupling is the project)? Must be the latter.
3. **Consumer topology** — clustered (relay wins), scattered, thin, or heterogeneous? The pain lives only in the latter three.
4. **Placement** — engine at the **edge** or in the **cloud**, driven by the bottleneck link.
5. **The failing baseline** — does the naive stack (IVM/stream processor + broker) actually fail the coupling? This is the make-or-break experiment and should run first.
6. **One extension, one evaluation, one consumer class** — the scope that fits the remaining time.
7. **The missing apparatus** — related work (the competitor matrix), critical evaluation, reflection/ethics, a dissertation, a resynced PLN.
8. **Integrity** — derive the key decisions cold; no tuned parameters presented as findings.
