# FYP Topic Review — Discussion Summary

| Field | Value |
| --- | --- |
| Document | DISCUSSION-SUMMARY |
| Date | 2026-10-19 (nine sessions: 2026-10-12, 2026-10-13, 2026-10-14, 2026-10-15, 2026-10-16 ×2, 2026-10-17 ×2, 2026-10-18→19) |
| Subject | Review of a proposed final-year project: server-side spatial filtering engine for 3D digital twin visualization |
| Artifacts reviewed | `docs/PRD-001__Product_Requirements_Document.md`, `docs/TDD-001__Technical_Design_Document.md`, `docs/PLN-001__Master_Execution_Sequence.md` |
| Participants | Supervisor (critic) and student (author) — roles later swapped into a viva-style exchange |

This document records the discussion so it can be resumed later. **Part I (§1–§8)** captures the initial evaluation, the student's defence, the technical findings, and the reframe that emerged. **Part II (§9–§20)** records a second session that resumed from the open questions in §7 and carried the work forward into a concrete baseline, a cost model, and a deployment-scenario reframe. **Part III (§21–§33)** records a third session that examined a domain pivot (to "IoT"), pinned down what "the cut is solved" means and which libraries implement it, worked through a client-side cut mechanised with a tiles library and its pub/sub variant, and ended in a viva-style examination that rejected the crossover framing and produced a proposed reposition. **Part IV (§34–§47)** records a fourth session: a continued viva in which the student conceded the original proposal, established that COMP4299 permits a system-development project, and then explored the problem-first discipline — re-skinning versus anchoring, the failing-baseline test, candidate pain points, the uplink-versus-egress and edge-versus-cloud distinctions, the field-technician scenario, and the Google-Maps analogy — arriving at a working framing. **Part V (§48–§78)** records a fifth session (2026-10-16): the testing of the framing's load-bearing claims (all of which fail in the natural setting), the audit of the best-of-breed libraries and their universal gaps, the assessment of two domain pivots (dashboards and smart city), the supervisor's challenges, the recalibration of the bar for COMP4299, the reframed system-development project, the 19-week feasibility audit, and the concrete mechanics of the cut and of delivery — arriving at a provisional candidate and an explicit set of gates. **Part VI (§79–§92)** records a sixth session (2026-10-16, continuing): the engine's status review, the catalogue of six arXiv papers and one repository read during the session and the pattern they reveal, the recommendation on the engine's disposition, and the turn to finding a *system-development* topic — the module's criteria, the sources and methods for locating one, the four-part filter, and the strongest candidates (a drift-aware twin; a DT-drift-management project). **Part VII (§93–§98)** records a seventh session (2026-10-17): the drift topic subjected to the same critique that killed the engine; the resolution of the recurring **"it sounds trivial"** reaction (separating *novelty* from the module's bar); the **track decision** (system-development vs research) as the gate; a **calibration survey of IoT scenarios** (device counts, rates, EPS, links); and an explicit **state and gates** section so a new Plan agent can resume. It is a record, not an instruction to change the design documents. **Part VIII (§99–§116)** records an eighth session (2026-10-17, continuing): the closure of the **track gate** (system-development); the exploration of candidates beyond the drift topic under the student's declared **absence of stakeholder access** (laptop-only); the assessment of a **campus digital-twin-with-dashboard** proposal and of **AI-model integration**; a **feature-request hunt across the competitor repositories** (the demand test the review had lacked); a **design-space comparison** of the candidate architectures for the professor; and a sharpening of the **cut** taxonomy — **placement versus implementation**, the **shared-crate control**, **direct select**, **view-independence**, the **delivery/rendering separation**, **compaction**, and **client memory as the surviving justification** — closing on the shared **"does it bind?"** value and the **nice-to-have** question. **Part IX (§117–§141)** records a ninth session (2026-10-17 → 2026-10-19): the **defensibility** questions (the data-centre twin, CRUD, mature solutions, the salon-booking case, FYP versus product, what one citation buys); the **privacy/disclosure** direction worked to a concrete problem, fourteen requirements, and a key extension (online hierarchical statistical disclosure control + a reconstruction adversary); the reading and positioning of **SHIP** and the **hierarchical edge-IoT security-monitoring** paper; the **supervisor's profile and authorship**; whether the **engine fits** those applications or has **résumé value**; and **IoT telemetry upload and trust**. All numerical results in Part II through Part IX are **first-principles estimates conditioned on stated constants**, not measurements; the constants are named where they must be measured before use.

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

---

# Part V — Candidate testing, the supervisor's challenge, and the mechanics of the cut (fifth session)

Part V records a fifth session (2026-10-16). It continues from the working framing of §46–§47. It tests the framing's load-bearing claims one by one; examines the Google-Maps tile-selection mechanism and the pub/sub adaptation; resolves the lazy-manifest question; calibrates device density and the bound; explores the dashboard and smart-city pivots; audits the best-of-breed libraries and their universal gaps; assesses the supervisor's offered topics; recalibrates the bar for COMP4299; reframes the engine as a defensible system-development project; audits the 19-week × 16 h/week feasibility; and works through the concrete mechanics of the cut (culling + LOD, i3dm, tile lifecycle, per-node vs per-instance visibility, per-node coupling vs regular tiles + own check) and of delivery (full-site snapshot vs interest-based delta, headers, cadence, consumers, aggregation placement). Section numbers continue from §47. As before, all figures are first-principles estimates against stated constants, not measurements.

## 48. Orientation

The session reopened from the handover (`~/.opencode/plan/fyp-topic-handover.md`) and the open gates of §47. The method was reaffirmed: **problem-first**; the engine is not to be re-skinned into a domain; every claim is to be tested against the world.

## 49. The candidate, stated

The current candidate was stated: **a hierarchical telemetry aggregation-and-delivery engine that serves each consumer a bounded, exact, consistent slice, with the selection↔aggregation coupling as the studied contribution** — the Google-Maps framing of §46. A **one-paragraph problem statement** was drafted (world-first, model-free): large live systems produce far more current data than any one viewer can examine; consumers each care about a different part, often over constrained or metered links; today's two options fail them (broadcast-and-trim exceeds the link; a shared central aggregate discards per-consumer relevance and exactness); the unmet need is a per-consumer, current, correct, internally consistent view under a fixed delivery budget, while still accounting exactly for what is not viewed. Its **three load-bearing claims** were then enumerated and tested.

## 50. The three load-bearing claims, tested

| Claim | Verdict |
| --- | --- |
| **1. The constrained link is the *delivery* link** | **Holds only in a specific topology.** The canonical LPWAN/satellite constraint is the **uplink** (site→network), which aggregate-and-forward solves; it is *not* a delivery constraint. It becomes true only when consumers and data share the constrained link (fleet/maritime, isolated site). Candidate #2's "remote industrial" fails it by default. |
| **2. Consumers are numerous / heterogeneous / scattered** | **The weakest.** Human consumers of an industrial system are **few** (10 operators, one technician). It holds only if "consumer" = query / tenant / dashboard / subscriber (platform scale) — which pushes away from industrial telemetry. |
| **3. Exactness / consistency is genuinely required** | **Fails for viewing.** Situational awareness runs on approximate and stale (Prometheus/Grafana sample precisely because that is fine). Genuine exactness lives in **control** (local, real-time), **billing/audit** (offline), and **disclosure** (adversarial) — none of which is "live viewing." |

**The joint finding:** the three claims live in **three different worlds** — (1) remote/fleet, (2) platform/observability, (3) control/accounting — and their **intersection is thin and must be deliberately constructed**, which is the signature of a problem assembled around a solution (§36). Stated plainly: **claims 2 and 3 do not hold in the natural setting, claim 1 holds only in a topology not chosen, and their overlap looks contrived.**

## 51. Do we need the aggregation? (the deduction)

The three claims define the only cell in which the engine is a project:

| | Exactness required? | |
| --- | --- | --- |
| Consumers homogeneous | No | One shared summary → broadcast + rollup + cache |
| Consumers heterogeneous | No | Personalised *approximate* slices → caching/CDN (solved) |
| Consumers heterogeneous | **Yes** | **The coupling — the only cell that is a project** |

So **aggregation alone always collapses into a commodity** (SQL, IVM, stream processors; §39); it becomes a contribution only when forced to be **exact and per-consumer under a bound at once** — i.e. only via the coupling. **Claims 2 and 3 are not decorations on the problem; they *are* the problem.** With both gone, the stated problem is descriptive dashboards behind a CDN — solved.

## 52. Google Maps tile selection, and where the analogy breaks

The client computes its tile set from the camera: project the viewport corners through Web Mercator into *world pixels* at zoom `z`, convert to `tileX = floor(worldX/256)`, `tileY = floor(worldY/256)`, and fetch the uncached rectangle plus a prefetch ring. The server is a **dumb CDN cache keyed by `(z,x,y)`**; it never sees the camera and computes nothing per viewer.

**The analogy breaks on two properties:** Maps tiles are **shared** (the same `(z,x,y)` is byte-identical for everyone, so it is cacheable and computed once) and **attribute-carrying** (vector tiles let the client style locally). The engine's layer is **live and personalised**, so there is no shared key to cache and nothing the client can verify alone. **Maps proves the *skeleton* (pyramid + client viewport selection + lazy fetch + edge cache + live layer joined by id), but only for shared, view-independent content.** That is the seam.

## 53. Everything is pub/sub; topics altered by selection

The observation was made that the design has been adapted to **pub/sub whose topic set is altered by the selection** (the frontier becomes the subscription set). Recorded:

- The formulation is **coherent, and the crispest statement of the mechanism yet**: the topic namespace = the node tree; the frontier = the subscription set; the payload of each topic = the exact rollup of the subtree it stands in for; the grain of publication = the grain of selection.
- But **who alters the topics decides the regime**: client-altered = fine pub/sub = R3 with the cut moved to the client; server-altered = R3 plus a broker. **There is no third option.**
- **Dynamic topics delete the Maps free lunch** (the sets are no longer shared) and impose **churn**: a topic set that tracks the camera is a per-frame feedback channel, requiring coalescing + hysteresis, moving R7 onto the data channel (§16).
- **It does not change the requirement.** It renames the mechanism; the requirement is still "a consumer needs a view that is both tailored (claim 2) and exact/bounded/consistent (claim 3)." It is a better *description of the contribution*, not new evidence that the contribution is *needed* (§36's test).

## 54. The lazy manifest reconsidered (A′) and the analytic/irregular distinction

The lazy manifest (ship descriptors only for the cut → client memory O(B), not O(N); §4.2, §38) was re-examined and elevated:

- **Without it, the server-side arm has no client-side advantage** over broadcast (the client is O(N) regardless). With it, the server-side arm is genuinely differentiated. So if server-side placement is argued at all, **the lazy manifest is mandatory**.
- **The decisive structural distinction:** client-side selection is cheap only when the space is **regularly addressable** — Maps from a closed-form projection (O(1)), 3D Tiles from a regular quadtree/octree (**implicit tiling**). The semantic tree is **irregular** (§30: implicit tiling "not applicable"), so it must be shipped explicitly (40–60 MB at 100k). **This is *why* the Maps skeleton does not transfer.**
- **The coupling this exposes (candidate invariant):** for an irregular hierarchy of N nodes and a view selecting B, one cannot simultaneously have (i) **O(B) client memory**, (ii) **purely client-computed selection**, and (iii) **no extra round trips**. Stated honestly as a **trade among memory, decision locus, and latency** — *not* an impossibility (a small coarse top structure plus lazy descent approaches O(B), at round-trip cost).
- **It is necessary, not novel** (ordinary on-demand loading; 3D Tiles **external tilesets** already lazy-load *the tree*, not only content), so it is a buildable **extension**, not a contribution.
- It also supplies a **better justification axis than exactness**: **bounded client memory/CPU** — a physical, citable constraint — i.e. the thin-client niche (§31).

## 55. Consumer topology: single roaming consumer

Question: is a **single consumer whose position varies** "all consumers share the same cut"? **No.**

- "Shared cut" means **concurrent coincidence** across consumers (compute once, serve many). A single consumer varying over time is **temporal variation**, not sharing; at every instant one cut exists and nobody shares it.
- Two currencies were separated: **concurrent fan-out** (compute once, serve V) and **temporal reuse** (cache on revisit). A single roaming consumer has **no fan-out** and only revisit-cache.
- Consequence: **V = 1 kills the CPU-scaling argument** (§12) — the main reason to put the cut on the server — while keeping the view-dependence that client-side lazy loading handles naturally. So a single, intermittently-moving consumer **tilts toward the client**, not the server.
- The lazy manifest's O(B) bound needs an **eviction policy**; over a long roaming session the working set drifts toward O(N) absent eviction, and re-entry costs a re-fetch — a concrete instance of the memory↔latency↔locus trade.

## 56. The tight view: the bound never binds

Scenario: a **5 m × 10 m room** within a **100 m × 200 m site** (20,000 m²). The room is **0.25%** of the site area → at 10,000 uniform devices it holds **~25 devices**, against a budget `B ≈ 2,000` — **two orders of magnitude under**. The entire mechanism (frontier, budget, hysteresis, blending) is **unexercised** — the same criticism that sank the original demo (§2).

- **`B` is a *cap*, not an every-frame mechanism.** Its job is to guarantee bounded work/bytes, not to fire constantly. `B` binds **iff the view's device count exceeds `B`** — i.e. a **wide/dense** view. At 10k devices and `B = 2,000`, that is a view covering **more than ~1/5 of the site**.
- **The coupling presupposes the bound binds:** no bound → no selection → no "rollup of the rest" → no coupling. So the contribution lives only in a **middle band** — dense *and* per-consumer *and* constrained — while tight views (commodity: just send it) and shared wide views (commodity: the same aggregate for all) both fail.
- `B` is a **perceptual** bound (marker legibility, §13), not physics, so an examiner can argue it away by tolerating clutter.

## 57. Device density calibration

The reference scenario's **N = 10,000 on a 2 ha (100×200 m) site** was calibrated: that is **0.5 devices/m² (1 per 2 m²)** — **dense**, an upper-end, instrument-heavy figure, not a site average. As *tags* it is a plausible mid-size plant; as *physical devices* it is high.

| Facility (2 ha) | Typical points/tags | Rough devices |
| --- | --- | --- |
| Building BMS/BAS | ~1k–10k | hundreds–~2k |
| Mid-size plant / unit | ~10k–50k | a few hundred–few thousand |
| Large refinery / power plant | 100k–1M | tens of thousands |

So a plausible average is **~1,000 devices** (0.05/m²). The binding quantity is not N alone but the ratio of **budget to on-screen device density**; the mechanism lives at **zoom-out**, always. (N must also be externally anchored — the anti-tuning rule, §13.)

## 58. The server-room case

A **server room** was examined: ~**40 racks** × ~**20–30 servers** ≈ **~1,200 servers** (≈1,240 nodes incl. groups). Notable:

- **Servers per rack:** standard 42U; **~20–30** for 1U, ~10–20 for 2U, **~64–100+** for blades/high-density. **Tens**, not hundreds.
- **Is a server a device?** A modelling choice: **server = device** (leaf with metrics) or **server = group** with sensor leaves. A server is a *telemetry source* either way.
- **The device count is *low*:** ~1,200 servers → at `B ≈ 2,000` the **bound may not bind** → the same "mechanism unexercised" trap. The data-centre scenario is *smaller* than the industrial one, not larger.
- **The tree is strongly spatial** (rows → racks → U), so it nests cleanly as a tileset — the "spatial by design" property holds strongly here.
- Per-server metrics make the *data* realistic (BMC/Redfish/IPMI + OS agents) but inflate **channels per entry, not entries per consumer** — so they do not make the bound bind. And they push the interface toward **time-series monitoring** (Grafana/DCIM), a crowded field.

## 59. The bound and EPS

It was noted that a site **easily exceeds 100k EPS** (a large hall: 10k servers × 10 metrics at 1 Hz; 2–3 rooms at higher metric counts). Recorded: **EPS is the wrong axis** for the contribution — it is the **source update rate** and stresses **ingest/aggregation** (commodity, §39), not the **selection/delivery bound** (which is on *entries*). High EPS *tilts* toward bounding (churn, fan-out, consistency pressure) but does not make the coupling mandatory; the composition handles high EPS too. **"100k devices" and "100k EPS" must not be conflated** (same EPS, different entry counts).

## 60. Dropping the visualisation → dashboards

Dropping the spatial panel changes "the view" from a camera to a **dashboard/query spec** — many, concurrent, heterogeneous consumers (this *helps* claim 2) — and sharpens claim 3 from "exact" to "**consistent across consumers and drill-downs**." Candidate pains were graded:

| Pain | Real? | Anchor | Incumbent | Coupling survives? |
| --- | --- | --- | --- | --- |
| Cross-consumer disagreement | Yes | ops/SLO practice | none guarantees it | **Yes** |
| Drill-down vs overview mismatch | Yes | TSDB bucketing | continuous aggregates | **Yes if exact** |
| Unbounded per-tenant cost | Yes | SaaS scaling | query limits | partial |
| Absence / partial availability | Yes | Prometheus staleness | partial | partial |
| Cardinality explosion | Yes | cardinality literature | sketching/labels | weak |
| Rollup / retention cost | Yes | downsampling | TSDB | No (commodity) |
| Dashboard latency | Yes | caching | cache/CDN | No |

The residue is rows 1–3 — **all the coupling again** — and all contested by strong incumbents (Grafana, Materialize, Timescale, Sparkplug B). The move is legitimate only if it **starts from one of these pains** (with a specific cited complaint), not from "dashboards have many consumers."

## 61. The smart-city pivot

Examined: smart city has real unsolved problems (interoperability; privacy-preserving open data; cross-agency consistency; entitlement-bounded viewing; twin fusion; edge latency; cost; data quality) — but **it is more crowded than industrial** (FIWARE, NGSI-LD, oneM2M, OGC, funded platforms, active DP research), its problems are **the wrong size or the wrong kind** (standards, research, huge scope), and the ones that fit the engine are **the same coupling**. The closest fit, **entitlement / disclosure-bounded aggregate release**, usefully converts the bound into a **hard authorization bound** (not perceptual) and supplies per-consumer heterogeneity from the domain — but it is partly commodity (ACL/query filtering) and partly a research field (differential privacy, which contradicts the engine's exactness). Also noted: smart city **re-introduces the spatial views** just dropped.

## 62. The abstraction test: "hierarchical knowledge management + selective publishing"

The student proposed the engine's abstract identity. Assessment: **partly right, with a correction and a consequence.**

- "Hierarchical" ✓ and "selective publishing" ✓ describe the **substrate**.
- **"Knowledge management" over-reaches** — the engine maintains numeric state in a tree and rolls it up; it does no semantics, inference, querying, or retrieval. Better: *hierarchical state aggregation with per-consumer bounded delivery.*
- **The consequence:** the abstraction **names the two commodity halves and omits the only non-commodity part** — the coupling. Taken as a definition, it describes a **commodity composition**. And **abstraction does not help grounding** (a pain lives at a concrete level; generality implies commodity). Its legitimate use is **related-work positioning**: the two literatures and the gap at their intersection.

## 63. Best-of-breed libraries and their universal gaps

The layers and their best-of-breed were enumerated:

- **Aggregation / IVM:** Materialize, Feldera/DBSP, RisingWave, Flink, Kafka Streams, Timescale continuous aggregates, ClickHouse/Druid/Pinot, Redis.
- **Pub/sub:** NATS/JetStream, MQTT (EMQX/HiveMQ), Kafka, Pulsar, RabbitMQ; WS gateways (Centrifugo).
- **Selection / entitlement / twin:** Hasura, PostgREST+RLS, FIWARE Orion/NGSI-LD, Eclipse Ditto, Azure Digital Twins, AWS TwinMaker, ThingsBoard, Sparkplug B.

**Universal gaps per category** (what *no* member provides): aggregation is **view-independent** (by definition); brokers are **compute-free** (by definition); query/entitlement layers are **unbounded and refetch-based**; twin platforms **route events, not rollups**; TSDBs are **bucketed/sampled**; renderers **own no values**.

**Cross-cutting universal gaps:** (1) no per-consumer bound anywhere; (2) no aggregation coupled to selection anywhere; (3) no cross-consumer snapshot; (4) hierarchy is *either* a namespace *or* a computation, never both; (5) view-dependence exists only in rendering; (6) cost scales with consumers, unbounded.

**Conclusion 1:** each single-category gap is a **separation of concerns** filled by the *composition*; the only gap **no combination fills** is the **coupling** — reassuring about its reality.

**Conclusion 2 (the caution):** a **universal gap is evidence of *absent demand* far more often than of unmet need** — the categories are mature and well-funded; "nobody built X" usually means X is a composition someone assembles when needed (glue), or nobody needs it. A universal gap is a **negative** observation about **supply**; it says nothing about **demand**. To distinguish a *needed* gap from an *unneeded* one requires a **published workaround** or a **published complaint** — neither of which was found.

## 64. Cross-consumer consistency: a constraint, not a pain

The engine's cross-consumer consistency (a shared per-frame snapshot; §28) was examined. It is **real**, and better than change-based brokers (late-joiner divergence) and refetch-based layers (per-subscriber staleness). But:

- **It is an artefact of centralisation, not a technique.** Broadcast of a snapshot is *also* cross-consumer consistent; you get it by centralising, whose cost is the CPU ceiling.
- It is **fragile** (per-viewer deltas and baselines can break it), holds **at frame boundaries only**, and matters **only if consistency is required** (claim 3).
- **Consistency is a *constraint*, not a *pain*:** a pain is observable and citable; a constraint is an engineering preference. Where a user-observable version exists ("our dashboards disagree"), it resolves to **"use a single source of truth"** — a data-governance fix, not a delivery-engine problem.

**Cumulative signal:** this was the **fourth claim in a row** to fail to find a real pain (link, consumers, exactness, consistency). The honest read: **the engine's requirements are self-generated**, and a sequence of load-bearing claims that all dissolve on contact with a user is not a search problem — it is evidence that the requirements have no natural home.

## 65. Composition and the defensibility of a composition project

The question "is a real problem solved by **compositing existing libraries** a defensible FYP topic?" was answered **yes — with the right framing**:

- **Composition is a *method*, not disqualifying.** COMP4299 requires "integrate **and extend**"; pure assembly fails, but composition-plus-reconciliation passes.
- A project is defensible **iff**: the problem is real and cited; the composition is a **method** not the **contribution**; the parts conflict and the seam is **built**; there is a **baseline** and a **failing demonstration**; verification and critical evaluation exist.
- It fails if the problem is invented to fit, the composition alone suffices (glue), the extension is configuration, or there is no baseline/evaluation.
- **The failing-baseline method is the machine that produces the defensibility**: compose → show the failure → build the reconciliation → evaluate.

## 66. The professor's offered topics (assessed, held as last resort)

The supervisor offered **six topics**: RFID attendance, a smart mailbox, smart plant watering, smart parking, server-room monitoring, and home-security motion detection. All six are **instantiations of one template** — `commodity sensor → Pi/ESP32 → threshold → dashboard → Telegram/email` — with the architecture identical and the depth identical. Assessed at the COMP4299 bar: **none is defensible as pitched** (pure assembly of a known solution; no extension; no algorithm beyond a threshold→relay; no evaluation; the blurbs self-describe as "introductions"). The RFID topic additionally **resolves away its own hard parts** (RFID does not stop proxying — it relocates it; MIFARE Classic is clonable; privacy persists). **Best anchor among them:** smart parking (the ~30% cruise-for-parking statistic). These were recorded as a **last resort**, not the objective.

## 67. The bar recalibrated

Assessed honestly: **the bar had been set too high on *sourcing*.** The discussion required a **sourced, externally-cited pain with an anchoring consumer** — a **research** standard. COMP4299, as a **system-development** project, requires only a **stated problem in a relevant area of computing** plus novelty-free execution. So:

- **Relaxed:** a problem statement may be a **stated engineering need with a justified scenario**, not a market complaint.
- **Not relaxed:** (1) **extension beyond assembly** ("integrate *and extend*"); (2) **rigorous evaluation** (baselines, measurement, limits); (3) **integrity** — the scenario must not be tuned so the design wins (§13).

## 68. The engine reframed as a defensible system-development project

With the bar recalibrated, the engine survives — not as a claim about the world, but as an **engineering project with a stated scenario**:

> **Design, build, and rigorously evaluate a hierarchical telemetry delivery engine that provides bounded, exact, consistent per-consumer slices — measured against best-of-breed compositions, with the selection↔aggregation coupling as the extension.**

Applicability to the adjusted criteria was tabulated (stated problem; the coupling as the extension; the composition as the baseline; the oracle; the lifecycle). Noted: this is a *passable, well-evaluated engineering project*, not a novel one — exactly what §9/§35 permits. Residual risks: the **failing baseline** is now an *engineering* test (if the composition passes, it is glue and the dissertation is thin); and **supervisor fit** (the engine is server-side streaming; the supervisor's domain is embedded/IoT).

## 69. Feasibility: 19 weeks × 16 h/week

Budgeted: **16 h/week × 19 weeks = 304 h**, of which a dissertation is ~70–100 h, leaving **~200 h of code**. The minimal build (state + index; aggregation + oracle; bounded selection + snapshot; socket-free encoding; simulator; one composition baseline; evaluation) was estimated at **~225 h** — **over** the code budget before any visual demo. Conclusions:

- **The socket gap is avoidable:** the evaluation measures bytes/consistency/latency, so the **transport can be evaluated socket-free** (serialise the slice, count bytes, model the ~52 B network overhead analytically). Sockets are needed only for an **optional demo**, and then **thread-per-connection**, not async.
- **Rust experience covers the core**; the binding constraint is **hours, not skills**.
- The project fits **only** as a **minimal, socket-free, one-baseline** engineering project with heavy library reuse, no richness (attention/absence trimmed), one scale, and **continuous dissertation writing**.
- **Ambition beyond that (visual demo, second composition, 100k, async) pushes it over budget.**

## 70. The supervisor's challenges

Two standing challenges were recorded, plus two positions:

- **Simulator realism.** The supervisor wants a simulator "close to the real scenario" — which is the **anti-tuning/grounding discipline** (§13) in his words. Satisfy it by naming one **reference scenario** with **cited** parameters (device counts, update rates, noise, drift, dropout, layout) and **sweeping** (not setting) the answer-shaping parameters; ideally ingesting a public dataset.
- **Frontend demo.** The supervisor expects a demo. The cheapest credible one is a **2D Leaflet map** whose viewport drives the selection (~20–30 h), not 3D/Cesium. Both requirements **add ~35–55 h**, pushing the build further over budget; satisfying both is what tips the project to its budget edge.
- **"The engine is trivial."** The supervisor independently reached the §37 conclusion that the **cut is commodity**.
- **"Frontend libraries can do the cutting."** He is **right** for this (spatial, by-design) tree: the library's traversal can serve as the semantic cut — so the cut is genuinely not the contribution.

## 71. The cut, concretely

- **The cut = culling *and* LOD** — visibility + resolution selection — producing a **bounded frontier under `B`** with entries at mixed depths (leaf entries and blend/aggregate entries). The engine's LOD is **aggregate/semantic** (show the rollup), distinct from a library's **geometric** LOD.
- **i3dm** = **Instanced 3D Model**: one geometry + N per-instance transforms, **one draw call** (batching, not parallelism; the GPU SIMT parallelism is inherent). It deliberately trades **per-instance visibility** for scalability: it is the **content of one tile**, so `tileVisible` fires **once per tile**, and all N instances are submitted when the tile is drawn (off-screen ones are **clipped**, not culled — you pay for all N).
- **Tile lifecycle:** `tileLoad` (once, on content load) → `tileVisible` (**per tile per frame**, during the render loop) → `tileUnload` (once, on eviction); `tileFailed` on error. Because `tileVisible` is per frame, you **accumulate a frame's firings** to get the visible set, then **coalesce + hysteresis** before acting.
- **A tile is a *spatial streaming node*** (bounding volume + `geometricError` + children + transform + optional content), **not an art asset**. Its content may be geometry (b3dm/glTF), instances (i3dm), points (pnts) — or **nothing** (a grouping/metadata node).
- **Tiles form a parent–child tree**; **external tilesets** let a tile's content be another `tileset.json`, loaded lazily — so **a child can be a tileset**, and the **tree itself can be loaded lazily** (the tile-world "lazy manifest"). One listener on the root `Cesium3DTileset` covers external tilesets too (filter by subtree); separate `Cesium3DTileset` objects each have their own events.
- **Visibility exposure:** `tileVisible` is **per tile**; a tile's features can be **enumerated** (`featuresLength` / `getFeature(i)`), but there is **no per-feature (per-instance) visibility callback or visible list**. Per-feature visibility (occlusion, off-screen within a visible tile) is **computed by the user**.
- **Picking / colouring:** the identity is a **batch-table property** (`node_index`); `scene.pick` → `Cesium3DTileFeature` → `node_index`; colour a node with `feature.color`. Keep a `node → features` index if a node's geometry is split across features (author 1:1 to avoid it).

## 72. Per-node coupling vs regular cells + own check

Two approaches were contrasted:

- **A — per-node coupled tile** (tile = semantic node): free per-node ids, simple logic, fits the (spatial) structure — **but** O(N) tiles at leaf level (~10,180 at 10k devices), **no tile optimisation** (no merging/splitting/regular-grid/implicit tiling/batching), poor occlusion handling, re-authoring on change, and **no bound or aggregation**.
- **B — regular cells + own check**: efficient tiles (balanced, batched, possibly implicit), scalable, semantic-agnostic, **imposes the bound** — at the cost of writing the **traversal/policy** (a few dozen lines; the intersection **math** is library-provided: Cesium `CullingVolume`/`computeVisibility`, three.js `Frustum`, Rust `glam`/`parry3d`). **Occlusion** is the only part with no off-the-shelf answer (and it is optional).

**At the server-room scale (~1,240 vs a few hundred), the performance difference is modest and concentrated in draw calls** (per-node: ~one call per visible node; regular+instanced: ~one per visible cell) — real but not disqualifying. So **performance is not the deciding factor at this scale**; the deciding factors are the bound, the aggregation, and maintenance.

**A strong concession was recorded:** because the semantic tree is **spatial by design** (nodes carry AABBs; the cut is a frustum test), the tileset **can be coupled to it** and the library's cut can be **delegated** — the strong "spatiality" objection does not apply. What remains anyway: the **bound** (SSE ≠ `B`; no byte cap), the **aggregation** (libraries don't roll up), and **consistency**.

## 73. Snapshot vs delta: transport and the recommendation

For **10,000 devices × 3 metrics = 30,000 values = 60 KB**:

- **Full-site snapshot:** 60 KB/tick × V (one message → headers amortised).
- **Interest-based delta:** ≈ 6,000 values × `p` × 4 B = **24 KB × p** per tick.

**TCP/IP overhead is decisive, and batching is the answer.** Per-message overhead is ~**52 B** at the IP layer (WS ~2 + TCP 20 + IPv4 20; ~70 B with Ethernet). One big snapshot → header ≈ **0.09%**. **Per-value delta** (4 B payload + 52 B header = 56 B) → **14× inflation**. **Batched delta** (one frame/tick) → near-zero. Crossover: batched delta beats the snapshot while `24p < 60` (essentially always); per-value delta only while `p < 0.18`. So **batching is a requirement, not an optimisation** — that is the design's frame pacing (§11).

**Aggregation is shared across viewers** (one tree state, one computation), so it is **amortised to ~nothing per viewer**; the per-viewer costs are only the **selection** (pub/sub) and the **delivery**.

**Clarification recorded:** a stateful-twin server **also aggregates on the snapshot path** (it maintains the tree and ships leaves + rollups), so server-side aggregation is **not** unique to pub/sub. The real differentiator is **delivery scope**: the snapshot leaves **nothing unseen** (so the client can always recompute/verify, and "the rollup of the unseen" is vacuous), whereas pub/sub **creates an unseen** that only the server can roll up. **The snapshot approach may have client *or* server aggregation; the pub/sub approach forces server-side aggregation.**

**Recommendation for the scenario: the full-site snapshot** — the state is small (60 KB), consistency is trivial, the client can **cut and aggregate exactly** (dissolving the coupling), there is no feedback loop, and there is far less machinery. The delta/pub-sub wins only at **larger V × cadence** or **metered/constrained egress**. Directly: **at this scenario, the engine is not needed.**

## 74. Cadence and consumers

Reasonable values, anchored:

- **Cadence:** **1–10 Hz** for viewing (Prometheus 15 s; Grafana 1–5 s; SCADA 1–2 s; process 1–10 Hz; humans read at a few Hz). **60 Hz is an animation holdover**, not a data need; cadence **is** the latency bound.
- **Consumers:** **10–100** for personalised delivery (control room 5–20; plant tens; field fleet 10–50); above ~100 you would share/cache rather than compute per consumer.

The resulting load (`B = 2,000`, 3 metrics, ~12 KB/tick, `T_cut ≈ 0.35 ms`): at 2 Hz/100 consumers ≈ **~2.4 MB/s and ~7% of a core**; at 10 Hz/1,000 ≈ 3.5 cores. **At reasonable values the server-side cut and the egress are trivial** — so the engine's necessity must come from a **non-performance** constraint (a **hard cost cap**, a **thin client**, a **disclosure** obligation, a **consistency** requirement), not from load. The "unsane" cadence (60 Hz) is the 3D premise already conceded.

## 75. The design-space + implementation framing (current candidate shape)

The session's provisional landing: the topic can be a **systems study** whose **contribution is the characterisation + the implementation + the evaluation**, with the design space as **rationale**:

> **A design-space characterisation and reference implementation for bounded per-consumer delivery of live hierarchical telemetry** — criteria derived from external anchors (link budgets, metered egress, latency), an implementation of the preferred combination, and an evaluation against best-of-breed baselines.

Recorded as defensible **only if**: the criteria are **anchored** (not chosen to make the preferred combination win); the **requirement is real, or at least stated and justified**; the baselines are **best-of-breed, not strawmen**; and the **scope is one combination** (analyse the space analytically, implement one, evaluate against a couple of baselines). It is **not** defensible as *only* a design-space report. It fits COMP4299's "integrate and extend" if the **coupling** supplies the extension.

## 76. Ingress scope

Recorded: **minimal ingress, not a subsystem.** Ingress (protocol adapters, brokers, time-series storage, retention) is **commodity** and off-contribution. The ingress needed is the **grounded simulator** (optionally via a thin MQTT/OPC-UA adapter for realism). Storage/persistence, if needed for a demo, is **off-the-shelf** (SQLite/Timescale). The test to apply to every component: **does it carry the contribution? If not, use off-the-shelf or skip.** A thin ingress is legitimately the "integrate" half; the coupling is the "extend" half.

## 77. Open questions carried forward (Part V)

1. **The requirement** — still the gate: is there (or can a stated scenario supply) a consumer for whom the **bounded, consistent, per-consumer** slice is **mandatory**? If not, §73's snapshot verdict stands and the engine is unnecessary.
2. **Supervisor fit and bar** — does the supervisor accept a **student-proposed, server-side** topic, and will he relax the "trivial / libraries do the cut" objection given the coupling concession (§70)? This decides whether the engine is viable here at all.
3. **The failing baseline** — wire `IVM + broker` (or `Timescale + Hasura`) and demonstrate **whether the coupling fails**; the decisive, cheap experiment.
4. **Scenario and scale** — industrial (~10k tags) vs **server room** (~1,200 servers); the server room is **small** (the bound weak). The scenario must make the bound bind, or supply a non-performance constraint.
5. **Snapshot vs delta** — provisionally **snapshot** at the stated scale; delta only under metered/constrained egress or large V × cadence.
6. **Cadence and V** — provisionally **1–10 Hz** and **10–100**.
7. **Cut delegation** — couple the tileset to the (spatial) semantic tree and delegate the cut to the library; then the residue is the **bound + aggregation + consistency**.
8. **Scope and budget** — 16 h/week, ~304 h, ~200 h of code; minimal build, socket-free transport, one baseline, no richness; the frontend demo is the cheapest credible (Leaflet).
9. **The supervisor's topics** — held as **last resort**; if used, extend one with depth (an anti-proxy study, an empirical comparison) rather than submit as pitched.

## 78. Bottom line after Part V

- **The original three load-bearing claims all fail** in the natural setting, and **cross-consumer consistency** is a *constraint*, not a pain — a **sequence of four** self-generated requirements. As a claim about the world, the engine's requirements have **no natural home**.
- **But the bar had been set too high on sourcing.** For COMP4299's **system-development** project, a **stated requirement suffices**, and the engine is a defensible **engineering** project — with the **coupling** as its extension, the **composition** as its baseline, and a **rigorous evaluation** as its content.
- **The cut is conceded** (commodity; the spatial tree makes delegation feasible). The residue is **exactly the coupling**: bounded, exact, consistent, **per-consumer** delivery — where the **bound**, the **aggregation of the unseen**, and the **consistency** are inseparable (§10: aggregation cannot be placed without selection).
- **At reasonable scale the problem is small** (cadence 1–10 Hz, V 10–100, ~60 KB state): the **full-site snapshot** covers the scenario and the engine is unnecessary. The engine's necessity must come from a **non-performance constraint** — a hard cost cap, a thin client, disclosure, or consistency.
- **Feasibility is tight:** 304 h, ~200 h of code, socket-free transport, one baseline, no richness; the supervisor's simulator + demo requirements push it to its edge.
- **Current candidate (provisional):** a **systems study** — a design-space characterisation plus a **reference implementation of the preferred combination**, evaluated against best-of-breed baselines, with the **coupling** as the extension and the **snapshot vs interest-based selection** as the studied axis. **Status: not yet approved; the requirement, and the supervisor's acceptance, remain the gates.**

---

# Part VI — The engine's disposition and the search for a topic (sixth session)

Part VI records a sixth session (2026-10-16, continuing). It reviews the engine's status; catalogues **six arXiv papers and one repository** read during the session and abstracts the **pattern** they reveal; reaches a **recommendation on the engine's disposition**; and turns to finding a **system-development topic** — the module's criteria, the sources and methods for locating one, the **four-part filter**, and the strongest **candidates** (a drift-aware twin; a DT-drift-management project), including the finding that drift detection is **not inherently ML**. Section numbers continue from §78. As before, all figures are first-principles estimates against stated constants, not measurements.

## 79. The engine's status review

The engine was re-assessed in one place:

- **The four load-bearing claims all fail** in the natural setting (§50, §64): the constrained link is the uplink (§43); consumers are few (§50.2); exactness is not required for viewing (§50.3); consistency is a *constraint*, not a *pain*.
- **Every component is commodity** (§39, §63): the tree, the topology, the aggregation, the broker, the cut, the transport.
- **The supervisor independently** reached "the engine is trivial" (§70).
- **The build is heavy**: ~225 h of code + ~90 h dissertation against a **304 h** budget (§69).

**The characterisation recorded:** the project is **trivial in *novelty*, substantial in *engineering*** — there is no hard algorithm to invent anywhere; the work is volume, correctness, and consistency. That profile is admissible for COMP4299's *system-development* track (**novelty not required**), but it shifts the entire burden onto the **evaluation** (§65): *if the components are standard, the build is not the contribution — the measurement is.*

## 80. What novelty is (and is not) open

The open seams discussed were graded by **feasibility in ~19 weeks**:

| Open seam | Nature | Feasible? |
| --- | --- | --- |
| The **coupling** (selection ⊗ aggregation) | **characterisation** (invariant) | ✅ cheap |
| **Consistency vs bounded delivery** (what is *achievable*) | characterisation | ✅ cheap |
| **Memory ↔ latency ↔ locus** for irregular hierarchies (§54) | characterisation (trilemma) | ✅ cheap |
| **Absence / partial availability** (exact rollups with gaps) | technique + semantics | ✅ moderate |
| **Frontier stability** (bounded selection with hysteresis) | technique + evaluation | ✅ moderate |
| **Incremental maintenance of *only the frontier's* rollups** | algorithm | ⚠️ risky |
| **Overlapping-hierarchy aggregation** (a tree cannot express it) | algorithm | ⚠️ risky |
| **Bit-exactness under SIMD** | technique | ⚠️ known-ish |
| **DP + exactness** | research | ❌ too big |
| **Cost-optimal placement** (the crossover) | study | ❌ rejected (§31) |

**Recorded recommendation:** the **characterisations first** (cheap, **invariant** — exactly what survives the "implementation-dependent" objection that killed the crossover, §32), plus **one small technique** (absence semantics) to demonstrate extension beyond assembly. **The caution stands:** novelty is **not required**, and *hunting* for a novelty is the design-first trap (§36).

## 81. The papers read (catalogue)

Six arXiv papers and one repository were read during the session:

| Paper / artifact | Venue · domain | What it addresses | What it misses |
| --- | --- | --- | --- |
| **Decision-Making in Digital Twins** | OR/ML/control | optimisation, RL, risk-aware autonomy, human-in-the-loop | **delivery** (streaming assumed) |
| **Semantic Edge-Cloud Communication for Real-Time Urban Traffic DTs** | networking · DT | **uplink** compression: send task-relevant *semantics*, not raw video (99% saving; 0.12 m error) | egress, per-consumer |
| **TwinGuard-Lite** | **cs.CR** · HealthSec'26 | a **state-admission gateway** (ingress integrity); properties **GSC** (grounded state consistency) and **CPN** (cross-patient noninterference) | egress delivery |
| **S²GS: Structured Sparse Gaussian Streaming** | **cs.CV** | sparse **Gaussian residual** selection over an octree for FVV streaming on edge-IoT | delivery, exactness (photometric) |
| **Mixed-Reality Feedback Control** | **cs.RO** | fusing control signals across **realities**; clock sync + sample alignment; fusion/defusion operators | delivery, per-consumer |
| **Missing Dimensions: Integrating Human and Social Systems into DT Engineering** | **cs.SE** | DT **dimensions** (Engineered/Natural/Human/Social); four open challenges | delivery |
| **`dt-semantic-drift-management`** (repo) | MDE · semantic web | **semantic/model drift** across *modelling layers* (JSMF, Brick ontology, SSM-OM, OAEI benchmarks) | twin↔reality drift; delivery |

Each was assessed against the **delivery/coupling** residue and found to be a **neighbour**, not the seam.

## 82. The pattern: neighbours, not the seam

Across all seven:

> **Every one touches a *piece* of the pattern — hierarchy, selection, streaming, consistency, integration, fusion — and *none* names the *combination*: bounded, exact, consistent, per-consumer rollup delivery.**

Two readings, per §63's caution: the gap is **either genuinely open, or genuinely not a felt problem**. Notably, **no paper states it as a limitation** — which leans toward "not a named problem." And the field's *attention* (decision-making, uplink compression, ingress security, rendering, control, human/social scope) is **upstream** of the view. **Useful output:** a strong **related-work chapter** and a positioning sentence — *"the field optimises decisions, uplinks, ingress, rendering, and scope; the bounded, exact, consistent per-consumer delivery of state is assumed, not solved."*

## 83. The engine's disposition

Two questions were separated:

- **Is the engine a good topic?** The evidence is one-directional: **no sourced need** (seven artifacts), **commodity parts**, a **skeptical supervisor**, a **heavy build**.
- **Is building it worth it?** **Admissibility does not depend on need** (§67) — but the *effort is the binding constraint*, and the payoff is a **coin-flip** (if the composition passes, the dissertation is a *build report*).

**Recommendation recorded:** *unless "building the engine" is itself the goal*, the rational move is to **let it go** — the engine is the **most expensive option with the weakest motivation**. If it is kept, it must be **shrunk** to the **coupling + one baseline composition + the evaluation** (§65, §68). **Sunk-cost caution:** the PRD/TDD being ~80% done is *not* a reason to continue; but the **analysis is not wasted** — the coupling, the commodity audit, the failing-baseline method, and the seven related-work anchors carry into *whatever* is done next.

## 84. Finding a topic: the module's criteria

The module text supplied the **filter**:

> *"students are responsible for identifying a problem, analysing stakeholder needs, designing an effective solution using appropriate technologies and tools, and ensuring system quality through verification processes."*

So every candidate must have **problem + stakeholder + buildable solution + verification**. **What the engine failed:** it had *solution* and *verification* but **no problem and no stakeholder**. Hence the operative question for any new topic is **"who is the stakeholder, and can you reach them?"**

## 85. Where to look: sources and methods

**Sources (nearest first):** your own environment (department, labs, societies, IT); local organisations (charities, clinics, small businesses, schools — manual processes, no budget); public sector / open data; **standards and compliance** (the *mandated* requirement — the "citation of need"); **open-source issue trackers** (stated needs in the users' words); the supervisor's domain; and your own workflow.

**Methods:** observation (find *manual, repetitive, error-prone* steps — "where there's a spreadsheet and a copy-paste, there's a project"); interviews (*"what's the most annoying part of your week?"*); complaint mining (reviews, forums, trackers); gap analysis (process vs standard); root cause (5 whys).

**The filter, plus a fifth criterion drawn from the whole discussion:** the pain must be **reachable and observable** — *a citable standard is good; a citable standard plus a person you can interview is a project.*

## 86. The "Missing Dimensions" paper: the four dimensions and the strongest candidate

The paper's **Table 1** (Smart Building) enumerates four dimensions — **Engineered, Natural/Biological, Human, Social** — each with *entities, sensors, models, services, actuation, and challenges*. Filtered:

| Candidate (dimension) | Stakeholder | Buildable? | Verified? | Verdict |
| --- | --- | --- | --- | --- |
| **Neurodivergent sensory accessibility** (Human) | **occupants + the accessibility office** | ✅ software | ✅ user testing | **strongest** |
| Personalised comfort (Human) | occupants, FM | ✅ | ✅ face validity | strong |
| Space allocation (Social) | facilities | ✅ | ✅ utilisation | medium — *planning* |
| Crowd flow (Social) | facilities | ⚠️ infra | ✅ | weak |
| Air quality / daylight (Natural) | occupants | ⚠️ hardware | ✅ | weak |
| Predictive maintenance (Engineered) | FM | ⚠️ access | ✅ | weak |

**Why the Human dimension dominates:** **named, reachable stakeholders** (the asset the engine lacked); a **mandated requirement** (accessibility law — the citation of need); a **software** solution (preference profiles, recommendations, accessible routing, notifications — no hardware required); **measurable verification** (comfort/overload, task performance); and **ethics built in** (the table's own *"privacy… ethical handling of sensitive disability/neurodiversity data… occupant agency and consent"* — the module's reflection/ethical component, *genuinely* motivated). Recorded candidate:

> **A decision-support system helping a building adapt to (or advise) neurodivergent / sensory-sensitive occupants** — real users, a lawful requirement, a buildable software solution, a measured evaluation; ethics approval is the plan-ahead hurdle.

## 87. The four research challenges, filtered

The paper's §5 names four **open** challenges (so each is a *gap statement*). Filtered on **problem + stakeholder + buildable + verifiable**:

| # | Challenge | Nature | Buildable / verifiable | Verdict |
| --- | --- | --- | --- | --- |
| 1 | Cross-dimensional collaboration & **tooling** (DSLs for non-SE experts) | tools/SE | ✅ / ⚠️ (experts unreachable) | plausible; **hard stakeholder** |
| 2 | Multi-dimensional **reference architecture** | architecture | ⚠️ / ✅ (locality-of-change measurable) | large & vague |
| 3 | **Heterogeneous data & model integration** (consistency + traceability) | data/systems | ✅ / ✅ (consistency, provenance) | **strong** |
| 4 | **DT drift detection & management** | monitoring/detection | ✅ / ✅ (inject drift, measure) | **strongest** |

**Caveats recorded:** they are **research-shaped** (need narrowing: two dimensions, one building, one stakeholder); their stakeholders are **abstract**; and **none is the engine**. Two nonetheless **touch the discussion's themes**: #3 = *consistency* at the **source** (rather than the consumer); #4 = *freshness/staleness* at the **twin** (rather than the view) — once more, the field names consistency/sync/integration/drift **upstream of the view**.

## 88. The DT-drift topic (#4), detailed

**Title (working):** *A drift-aware digital twin for a building: detecting and managing divergence between the twin and reality.*

- **Problem (stakeholder's words):** *"we can't tell when the twin has quietly stopped matching the real building — and then we make decisions on wrong numbers."* Drift arises as **model drift**, **data drift**, or **scope drift**.
- **Stakeholder:** someone who **trusts the twin** — a facility manager, a lab operator, or (Human dimension) the **accessibility office**; reachable.
- **Solution:** (1) a **simulated twin** (temperature/CO₂/humidity/occupancy) — *the simulator is central, satisfying the supervisor's §70 requirement*; (2) a **ground-truth reference**; (3) a **statistical/sequential detector** (residual monitoring; PSI/KS; ADWIN/Page-Hinkley) with a **threshold baseline**; (4) a **management loop** (alert → recalibrate → re-scope → fall back) with **provenance/traceability**; (5) an **evaluation harness**.
- **Verification:** **inject drift** of known types and measure **detection latency, precision/recall, false-alarm rate (alarm fatigue), and recovery time** — against the threshold baseline (§40's method).
- **Ethics:** if human/social data is carried, drift management touches **privacy, consent, fairness** (the module's reflection component).
- **Feasibility:** ≈ **140–160 h** — *half* the engine: simulator ~40 h, detector ~20 h, management + provenance ~40 h, evaluation ~40 h, demo optional.
- **Two cautions:** frame it as a **system, not ML**; **do not imitate** the repository (§90) — that is *model* drift across *modelling layers*, research-shaped and much bigger.

## 89. Drift detection is not inherently ML

The decomposition recorded:

| Sub-problem | Methods | ML? |
| --- | --- | --- |
| **Data drift** (covariate shift) | statistical tests (**KS, PSI, χ², MMD**) | No |
| **Concept/model drift** | sequential detectors (**ADWIN, DDM, Page-Hinkley**) or **residual monitoring** | No |
| **Scope drift** | rules + anomaly heuristics | No |
| **Diagnosis / attribution** | sometimes ML | *maybe* |
| **Management / remediation** | **systems** (a control loop) | No |

**Key facts:** the classic **concept-drift literature is largely *statistical*** (not deep learning), and the **management half is pure systems** — and, per the paper, the **under-studied** half (*"detection must trigger appropriate DT evolution"*). So the **open** part is **management/evolution**, which is a **systems** problem. Frame the project as a **monitoring + management *system*** with a **statistical** detector; the only way it becomes "an ML project" is if the student frames it so.

## 90. The repository (`dt-semantic-drift-management`)

Read at README level: a **research artifact** (conference submission; arXiv 2512.15281) for **"semantic drift management in a multi-layered, model-driven DT"** — three steps **identification → evaluation → propagation**; air-quality use case; layers = **data / model-metamodel (JSMF) / ontology (Brick via SSM-OM)**; experiments on **OAEI** ontology-matching benchmarks, ablations, interoperability/traceability.

**`"Semantic drift"` here is *not* the paper's "DT drift":** it is **model/metamodel/ontology drift against the data** ("the data outgrows the model"), detected by **model alignment / ontology matching** — an **MDE + semantic-web** problem, **research-shaped and much bigger** than a 150 h system-development project. **Assessment:** it **occupies** a *variant* of the "DT drift" space (so "semantic drift management" would duplicate it), and its skill set is heavy. **Use it as:** a **related-work anchor** (model↔data drift is addressed by MDE; twin↔reality **state** drift management with a stakeholder-facing evaluation is less so), a **scope warning**, and a source of a **concrete open dataset and domain** (room air quality).

## 91. Open questions carried forward (Part VI)

1. **The engine's disposition** — *drop it, or shrink it to the coupling + baseline + evaluation* (§83). **Unresolved; the recommendation leans to drop unless building it is the goal.**
2. **The supervisor conversation** — ask what bar he holds and whether he accepts a **student-proposed** topic; bring **candidates**, not questions.
3. **The topic candidates** — (a) the **drift-aware twin** (§88); (b) the **accessibility / neurodiversity** system (§86); (c) **extend one supervisor topic** with a real stakeholder + evaluation (§66, §84 — the pragmatic choice).
4. **The stakeholder** — the **missing asset**: name one and reach one.
5. **The four-part filter** — apply to each candidate: **problem + stakeholder + solution + verification** (+ *reachable & observable*).
6. **Integrity** — anchor the criteria; no tuned parameters; the candidate must be derived, not fitted.

## 92. Bottom line after Part VI

- **The engine is a solution without a problem** — trivial in novelty, heavy in build, unsourced in need — and **six papers across six domains** confirm that the field's attention is **not** on per-consumer bounded delivery. **The rational move is to drop it** (or shrink it), and the analysis becomes the **rationale** and the **related-work chapter** for whatever follows.
- **The module supplies a better filter than the engine ever did:** a **problem**, **stakeholders**, a **buildable solution**, and **verification** — with the *stakeholder* as the operative asset the engine lacked.
- **The strongest candidate sources** are the **Human dimension** of DT engineering (**accessibility / neurodiversity** — named stakeholders, a mandated requirement, software-only, measurable, ethics built in) and the **DT-drift** challenge (**#4** — smallest, most measurable, with the simulator central); and the **pragmatic** route is to **extend a supervisor topic** with a **reachable stakeholder** and a **real evaluation**.
- **Drift detection is not inherently ML** — its **management** half is systems and is the paper's actual gap.
- **Next step:** bring the supervisor **2–3 candidates** written as *problem → stakeholder (and how you'll reach them) → minimal solution → verification → ethics*, and **talk to a stakeholder** — because that is what turns a candidate into a project.
- **Status: the engine is provisionally abandoned; no new topic is yet adopted. The gate is a *sourced, reachable stakeholder*.**

---

# Part VII — The drift topic's sufficiency, the "trivial" question, and device calibration (seventh session)

Part VII records a seventh session (2026-10-17). It continues from Part VI's candidates. It subjects the **drift topic** to the same critique that killed the engine; resolves the recurring **"it sounds trivial"** reaction by separating *novelty* from the *module's* bar; records the **track decision** (system-development vs research) as the gate; and adds a **calibration survey** of IoT scenarios (device counts, rates, EPS, links) so candidates can be sized realistically. It closes with an explicit **state and gates** section so a new Plan agent can resume. Section numbers continue from §92. Figures in §97 are indicative ranges, not sourced measurements.

## 93. Orientation

The session returned to *topic sufficiency*. The method was unchanged: **apply the same critical lens to every candidate** — the one that killed the engine (§50, §64, §83). Two questions dominated: *"is the drift topic just *inject → detect*?"* and *"does it sound trivial?"* A calibration survey was then added so future candidates can be sized against real deployments.

## 94. The drift topic under critique: "inject → detect"?

Question: **is the whole system simply *inject a drift* and then *run a statistical detector*?** Answer: **yes, framed that way — and that framing is too thin.**

- **The injection is the *harness*, not the system.** Injecting drift creates the scenario to test, like the simulator in the engine's case (§70). It is not a deliverable.
- **A detector alone is *commodity*.** ADWIN, Page-Hinkley, PSI, KS, residual monitoring exist and are simple to apply — the paper's *starting point*, not its gap. If the deliverable is "we ran ADWIN on the twin," that is the **engine's failure mode**: a commodity component dressed as a project.
- **The *system* must be the response**: a **closed loop** — detect → **decide** → **act** → **verify** — with the **management policy** (when to *recalibrate*, *re-scope*, or *fall back*) and **provenance/attribution** (*which* source or dimension drifted). That is the section the paper explicitly frames as the open gap: *"detection must trigger appropriate DT evolution."*
- **The evaluation must be more than "did it detect":** recovery time, false-alarm rate (alarm fatigue), detection latency, and cost, over **several drift types**, **against a baseline**, and **grounded**.
- **The validity risk (recorded):** injecting *your own* drift into a *simulator* means detecting *drift you planted* — easier than real drift, and coupled to the injector. Mitigation: inject into a **real dataset** (e.g. the air-quality dataset used in the repository, §90) so the *baseline behaviour* is real even if the drift is synthetic.
- **The advantage over the engine:** the paper **names the management gap** (§5.4) — a *cited open problem*, which the engine never had.
- **The framing that fixes it:** *"a closed loop that keeps the twin trustworthy: it detects divergence, decides the right response, executes it, and verifies recovery — evaluated on recovery time, false alarms, and cost against a naive threshold."* The subject is the **loop and its policy**; the detector is a component; the injection is the harness; the **evaluation is the evidence**.

**The test recorded (same as the engine):** *is the residue — the management policy — a real problem, or a line of code?* If a competent engineer would just write "threshold + recalibrate and move on," it is thin; if *when and how* to evolve the twin without thrashing is genuinely non-obvious, it is a project.

## 95. "It sounds trivial": novelty vs the module's bar

The recurring reaction — *every* candidate "sounds trivial" — was diagnosed:

| Considered | Verdict |
|---|---|
| The engine | commodity parts + thin residue |
| The supervisor's six topics | commodity parts (one template) |
| Dashboard / smart-city / robot pivots | commodity or crowded |
| The four DT research challenges | research-shaped (need narrowing) |
| The drift topic | commodity detector + thin residue |
| Every paper read | a neighbour, not a seam |

**The diagnosis:** a **research bar** was being applied to a **system-development module** where **novelty is not required** (§35, §67). By the research bar, almost every systems project is trivial; by the module's bar, they are precisely what is asked.

**So the question changes:** not *"is it novel?"* but *"does it satisfy the module's four criteria — problem, stakeholder, solution, verification — and can it be executed and evaluated well?"* By that bar the drift topic is **not disqualified**: it has a **named open problem**, a **potential stakeholder**, a **buildable solution**, and a **measurable verification**.

**Where depth must come from (it is not novelty):** (1) a **real stakeholder** with a lived pain; (2) a **rigorous, grounded evaluation** (multiple drift types, recovery, false alarms, latency, cost, against a baseline); (3) a **genuine constraint** (a hard bound, a safety/ethics obligation, a scale that breaks the composition); (4) for *research* only, a real gap.

**The engine comparison recorded:** the engine died **not because it was trivial** but because it was **trivial *and* unmotivated *and* heavy**. The drift topic is **trivial, better-motivated (a cited gap), and lighter** — so it is *strictly better*, **provided** the stakeholder and the rigorous evaluation are brought.

## 96. The track decision

The conversation's central unresolved fork was sharpened:

- **System-development** (COMP4299's named option): **trivial-in-novelty is fine**; depth comes from **stakeholder + evaluation**; the drift topic, the accessibility topic, and an extended supervisor topic are all **admissible**.
- **Research** (the other permitted option): **novelty is required**; the seven papers across six domains found **no named seam**; the crossover was **rejected** (§31); the hunt is **harder** and **riskier**.

**Recorded:** the topic's quality is decided by the **stakeholder** and the **evaluation**, not by whether the components are novel. Until the track is chosen, every candidate will keep "sounding trivial" — because the bar is unstated.

## 97. IoT device survey (calibration)

Indicative ranges (not sourced; anchor to a specific deployment before use):

| Scenario | Devices / points | Update rate | Metrics/device | EPS (order) | Link / protocol | Notes |
|---|---|---|---|---|---|---|
| Smart home | 10–100 | event + 1/min–1 Hz | 1–5 | 10¹–10² | Wi-Fi, Zigbee, Matter, BLE | privacy; low criticality |
| Commercial building (BMS) | 1k–50k points | 1–15 min (critical 1–60 s) | 1–3 | 10²–10³ | BACnet, Modbus, KNX | energy/HVAC; history |
| Industrial plant (SCADA/DCS) | 10k–1,000k **tags** | 0.1–10 Hz | 1–10 | 10⁴–10⁵ | OPC-UA, Modbus, PROFINET | safety; exactness |
| Manufacturing line (IIoT) | 10²–10⁴ sensors | 1–1,000 Hz | 1–10 | 10³–10⁵ | OPC-UA, TSN, Sparkplug B | vibration; maintenance |
| Smart grid — meters | 10⁵–10⁷ per utility | 15 min–1 h | 2–5 | 10²–10³ | DLMS/COSEM, PLC, cellular | **billing → exactness**; privacy |
| Smart grid — substation/PMU | 10–10² per substation | **30–120 Hz** (C37.118) | 10+ phasors | 10⁴–10⁵ | IEC 61850, DNP3, C37.118 | **hard latency** |
| Smart city — traffic | 10²–10⁴ | 1–30 Hz | 2–10 | 10²–10⁴ | cellular, fibre, LoRaWAN | latency-sensitive |
| Smart city — air quality | 10²–10³ stations | 1 min–1 h | 3–6 | 10⁰–10² | LoRaWAN, NB-IoT | calibration drift |
| Smart city — lighting/parking | 10³–10⁵ | event / daily | 1–3 | 10⁰–10² | LoRaWAN, NB-IoT | constrained; low rate |
| Agriculture (soil/weather) | 10²–10³ per farm | 5–60 min | 2–5 | 10⁰–10¹ | **LoRaWAN, NB-IoT** | constrained; battery |
| Livestock / asset tracking | 10²–10⁵ | minutes–hours | 1–3 | 10⁰–10² | BLE, LoRaWAN | battery; geofencing |
| Fleet / logistics | 10²–10⁵ | 1–60 s | 5–20 | 10³–10⁵ | **cellular, satellite** | metered; intermittent |
| Maritime / aviation | 10²–10⁴ | 1 s–1 min | 5–20 | 10²–10⁴ | **satellite (metered)** | cost/latency |
| Healthcare — vitals | 10²–10⁴ per hospital | 1 s–1 min | 3–8 | 10²–10³ | Wi-Fi, BLE, FHIR | privacy; safety |
| Wearables | 1–10 per person | 1–100 Hz | 3–10 | 10⁰–10² per person | BLE, cellular | battery; high rate |
| Retail — ESL / shelves | 10³–10⁵ | event / daily | 1–3 | 10⁰–10² | BLE, Zigbee, Wi-Fi | huge count, low rate |
| Water / utilities | 10⁴–10⁶ meters | 15 min–1 day | 2–5 | 10¹–10³ | LoRaWAN, NB-IoT, PLC | constrained; billing |
| Data centre (DCIM) | 10³–10⁵ servers | 1–10 s | **10–100** | **10⁵–10⁶** | Redfish, IPMI, SNMP | high cardinality |

**Three observations recorded:**

1. **"Big data" ≠ "many consumers."** High EPS lives in **plants, grids, data centres, fleets** — whose consumers are **few** (operators, planners). Many consumers live in **home / city / SaaS** — where rates are **low** and content is often **shared**.
2. **Constrained-link scenarios are low-rate.** Agriculture, meters, lighting, parking, maritime — **LPWAN/satellite**, but **minutes-to-hours** and **few metrics**. So "constrained link" and "high rate" **rarely co-occur**.
3. **Exactness and privacy cluster by domain.** Billing (meters, water) and protection (grid) need **exactness**; health, home, meters need **privacy**; industrial viewing needs **neither** urgently.

**The structural finding (restated in numbers):** the properties are each common but **live in different scenarios** — the same "three different worlds" result as §50 and §63.

## 98. State and gates for a new Plan agent

**Purpose:** this section is a handover for continuing in Plan mode.

**The engine.** Provisionally **abandoned** (§83, §92) — trivial in novelty, heavy in build, unsourced in need, and the supervisor calls it trivial. If it is ever revived, it must be **shrunk** to the **coupling + one composition baseline + the evaluation** (not the full engine), and framed as a **stated-requirement system-development project**. The **analysis is the asset**: the coupling, the commodity audit, the failing-baseline method, and the **seven related-work anchors** (§81) — usable for any topic's related-work chapter.

**The candidates on the table.**

| Candidate | Source | Status |
|---|---|---|
| **Drift-aware twin** (DT drift detection & management) | §87–§89 | **primary** — a *cited* open challenge, a reachable stakeholder, a measurable evaluation, ~140–160 h; **risks thinness** if framed as "inject → detect" (§94) |
| **Accessibility / neurodiversity** support system | §86 | **strong** — named stakeholders, a *mandated* requirement, software-only, measurable, ethics built in |
| **Extend a supervisor topic** with a real stakeholder + evaluation | §66, §84 | **pragmatic** — his domain, his approval, a small build (~120 h) |

**The critiques that must be answered for any candidate.**

- **Not "is it novel?"** — novelty is not required for system-development (§95). The question is the **module's four criteria**: problem, stakeholder, solution, verification.
- **Is the *residue* real, or a line of code?** (The engine's test.) For drift: is the **management policy** non-trivial? For accessibility: is the **decision support** more than a form?
- **Is it *grounded*?** A synthetic injection/testbed must rest on **real data** or a **real stakeholder** (§94).
- **Is the build right-sized?** 16 h/week × ~19 weeks ≈ **304 h**, ~**90 h** of which is the dissertation (§69). Prefer **~120–160 h** builds.

**The gates, in order.**

1. **Choose the track** — **system-development** (trivial-is-fine; depth from stakeholder + evaluation) or **research** (novelty required; the seam is elusive). *Everything else depends on this* (§96).
2. **Choose the track's candidate** — drift-aware twin (primary), accessibility, or an extended supervisor topic.
3. **Name a *reachable* stakeholder** — the asset the engine never had. If none can be reached, the candidate is a hypothetical (§84–§85).
4. **Talk to the supervisor** — bring **2–3 candidates** as *problem → stakeholder (how you'll reach them) → minimal solution → verification → ethics*, and ask **what bar he holds** and whether he accepts **student-proposed** topics (§91).
5. **Run the four-part filter** on the chosen candidate: **problem + stakeholder + buildable solution + verification** (+ *reachable and observable*).
6. **Integrity** — anchor the criteria; no tuned parameters; the candidate must be **derived**, not fitted.

**What a new Plan agent should do.** Ask **which track** and **which candidate** the student is on; then help **name and reach a stakeholder** and **draft the candidate paragraph** for the supervisor. Do not re-litigate the engine unless asked; do not let the conversation drift back into "find a domain for the engine" (§36). The recurring trap is now well documented: **every candidate will "sound trivial" until the track and the stakeholder are fixed** — so fix those first, and judge the topic on *stakeholder + evaluation*, not novelty.

**Bottom line:** the discussion has moved from **"can the engine be defended?"** (answer: not as a problem; only as a stated-requirement engineering project, and at high cost) to **"which system-development topic has a real stakeholder and a measurable evaluation?"** — with the **drift-aware twin** as the primary candidate, **accessibility** as the strongest human-centred alternative, and **extending a supervisor topic** as the pragmatic fallback.

---

# Part VIII — Track closure, the competitor hunt, the design space, and the cut (eighth session)

Part VIII records an eighth session (2026-10-17, continuing). It closes the **track gate** (system-development); records the candidate exploration beyond the drift topic and the student's declared **absence of stakeholder access**; assesses a **campus digital-twin-with-dashboard** proposal and the question of **AI-model integration**; reports a **feature-request hunt across the competitor repositories** — the demand test the review had lacked — and its verdict; and works through the **design space** and the **cut** in depth: placement versus implementation, the shared-crate control, **direct select**, **view-independence**, the **delivery/rendering separation**, **compaction**, and **client memory as the surviving justification**. It closes on the shared **"does it bind?"** value and on whether a **nice-to-have** can justify a project or a product. Section numbers continue from §98. As in Parts II–VII, all figures are first-principles estimates against stated constants, not measurements.

## 99. Orientation

The session resumed from §98's state-and-gates. Two things changed the picture. First, the student **closed the track gate**. Second, the engine was re-examined with a new instrument — **the competitor repositories' feature requests** — and with a sharper account of the **cut's design space**. The method was unchanged: apply the critique that killed the engine to every candidate, and do not re-skin.

## 100. The track decision (the gate of §96 closes)

The student committed to the **system-development** track (COMP4299). Consequences recorded: **novelty is not required**; depth must come from a **stakeholder** and a **rigorous evaluation**; the engine remains admissible as a **stated-requirement engineering project**, not as a research contribution. The gate of §96/§98.1 is closed. Caution recorded: choosing the track fixes the *bar*, not the *topic* — it does not by itself make any candidate defensible.

## 101. Candidate exploration and the access constraint

Asked to develop a candidate beyond the drift topic, the student explored alternatives rather than committing, and declared two hard constraints: **no reachable stakeholder** and **laptop-only** (no hardware, no institutional data). The lanes were triaged by what the constraint kills:

| Lane | Verdict under "laptop-only, no access" |
| --- | --- |
| Accessibility / assistive | **blocked** — verification *is* user testing |
| Learning / education | **blocked / weak** — same reason |
| Campus / dept tooling | **weakened** — needs a champion and institutional data |
| Embedded / sensing | **blocked** — no host site, no devices |
| Security / privacy | partly open — public corpora, no participants |
| Public / open data | open — the dataset substitutes for access |
| Developer / ops tooling | open — OSS issue trackers are reachable online |
| Simulation / benchmark systems | open — but this is the engine's shape |

Recorded: **the stakeholder is the scarce resource**; with none, the **external anchor** — a public dataset, a standard, a published complaint — must carry the justification (§41). The student **declined to test the no-access assumption** (declined to spend a short, cheap effort asking around), so the anchor-only route was adopted.

## 102. The campus digital-twin proposal

The student proposed a **campus digital twin with a dashboard**. Verdict: **as stated, not defensible.** It is a *means* statement — an artifact (a twin) and a presentation (a dashboard) — with no problem, stakeholder, or definition of success; twins and dashboards are commodity; and the "digital twin" claim must be **earned** (bidirectional sync / what-if), or the artifact is a map with charts. But the **campus is the nearest stakeholder** (estates, sustainability, IT, accessibility office), so the direction is promising **iff** anchored to a specific, measured pain (energy/carbon, ventilation/CO₂, space utilisation, maintenance, accessibility) and a reachable person; the **extension must be a mechanism**, not a screen. Caution: with no access, a campus twin is a **synthetic twin of the campus** — the engine's invented-scenario trap in a smaller box.

## 103. AI-model integration

The student asked whether **integrating an AI model** into a system like the engine would be defensible. Verdict: **not as "engine + a model."** It touches none of the engine's failures (problem, stakeholder, need); "integrate a model" is a *means*; it adds a second contribution with its own evaluation burden to an already over-budget build; and an AI-centric project risks the **CSAI4299** bar ("clear AI focus") rather than COMP4299's. The test recorded: *is there a requirement only inference can satisfy; can the model's contribution be isolated and measured against a baseline; does the problem survive removing the model; and which module does the result belong to?* Roles graded — learned selection, value imputation (contradicts exactness), anomaly detection (crowded, CSAI-shaped), threshold tuning (thin), forecasting (shifts the topic) — **none rescues the engine**; each either shifts the topic or adds an unrelated contribution. Distinction recorded: **AI as decoration** (a bolt-on) versus **AI as the mechanism for a stated problem** (then the AI problem *is* the topic).

## 104. The competitor feature-request hunt

Method: GitHub issue search scoped to the major competitors, plus direct reads of the salient issues (3D Tiles/Cesium, deck.gl, loaders.gl, Potree, three.js, Eclipse Ditto, FIWARE Orion, ThingsBoard, Materialize/RisingWave/Feldera, Mirror/FishNet, Mapbox tippecanoe, Grafana). Rate-limited, so a **sample, not exhaustive**; **absence of a request is weak evidence**. The target was the engine's **conjunction** — bounded + exact + consistent + per-consumer hierarchical rollups under a **live, view-dependent** selection.

| Competitor | Issue | Age / state | What it asks | What it proves |
| --- | --- | --- | --- | --- |
| **Eclipse Ditto** | [#1525](https://github.com/eclipse-ditto/ditto/issues/1525) | 2022-10, **open** (`outlook`) | Aggregation (`mean`/`sum`/`count`) over "hundreds/thousands of selected things" | **Real, dated, unmet demand** — *"We have a clear need to have aggregations … to manage a large fleet of devices."* But **view-independent**; 4 years unbuilt |
| **Eclipse Ditto** | [#2291](https://github.com/eclipse-ditto/ditto/issues/2291) | 2025-12, open | Timeseries facade | Workaround is **manual TSDB integration** |
| **FIWARE Orion** | [#3816](https://github.com/telefonicaid/fiware-orion/issues/3816) | 2021-03, **open** | `?aggregate=category&operation=avg:score` | Same shape; workaround: **fetch all entities, aggregate client-side** |
| **ThingsBoard** | rule-engine `RELATED_ENTITIES_AGGREGATION` | productized | Hierarchical aggregation over the entity tree | The aggregation half is **already productized** |
| **Mapbox tippecanoe** | [#324](https://github.com/mapbox/tippecanoe/issues/324), [#354](https://github.com/mapbox/tippecanoe/issues/354), [#830](https://github.com/mapbox/tippecanoe/issues/830) | 2016–2020, closed | "Combine features, **aggregate properties** to fit tiles" | **The coupling exists** — `--accumulate-attribute`; **precomputed, shared, offline** |
| **deck.gl** | [#10795](https://github.com/visgl/deck.gl/issues/10795) | 2026-10, open | "aggregation layers that refine into raw points as you zoom in" | View-dependent aggregation is **solved — a documentation task** |
| **Cesium** | [#6182](https://github.com/CesiumGS/cesium/issues/6182) | 2018, closed | Aggregate geometries "for different zoom levels" | User built the **workaround himself**; redirected to tile pipelines |
| **Mirror** | [#3197](https://github.com/MirrorNetworking/Mirror/issues/3197) | 2022, closed | Send state to a **subset of observers** | Per-consumer selection is **solved** (Team Interest Management) |
| **Grafana** | [#347](https://github.com/grafana/grafana/issues/347) | 2014, closed | Tree explorer of metric keys | The **topic-namespace = tree** idea exists as navigation |
| **misc** | `autheris` #112 | 2026, open | GraphQL sums/counts per group "without reading every row" | Another live **aggregation-over-hierarchy** request |

## 105. What the hunt establishes

1. **The aggregation half has real, dated, open demand** — the **first published complaints and workarounds** in the entire review, in the users' own words (§63's missing evidence).
2. **The view-dependent coupling is already solved wherever content is *shared*** — tippecanoe `--accumulate-attribute`, Cesium clustering, deck.gl zoom aggregation; **precomputed/offline**, i.e. the Google-Maps case.
3. **Per-consumer selection is solved** — interest management.
4. **No published demand for the conjunction** — bounded + exact + consistent + per-consumer, coupled to a live cut; nor for cross-consumer snapshot consistency; nor for a hard per-consumer bound.

**Net:** the demand is for **aggregation, not the coupling** — the engine's residue is **unrequested**. This **corroborates and sharpens** §83/§92 rather than overturning them. Assets produced: citable **related work** (stated needs and workarounds); a narrower **live-aggregation-service** framing; and the material for the **"why not"** argument every write-up will need. Caution (§63) stands: a gap's **age is not demand** — the Ditto request is four years old and labelled `outlook`; the platforms have declined to pay for it.

## 106. The design-space comparison (for the professor)

The student needed to report **different designs and their pros and cons**. The comparison produced five architectures, plus three layered choices:

- **D1 — Broadcast + client cut** (Regime 1): server fanout only; egress **O(N)/client**; no bound.
- **D2 — Interest / subscription** (Regime 2): fine (frontier) → egress **≈ D3**; coarse (region) → **> D3**; server CPU ≈ routing; binds on α.
- **D3 — Server cut** (Regime 3): the only **hard per-client byte bound**; exact server-side aggregation of the unseen; deterministic, oracle-comparable, consistent snapshot; **CPU-bound** (`V_cpu ≈ 74–260`).
- **D4 — Server cut + lazy manifest** (A′): client **O(B)** instead of O(N); ordinary lazy loading, not novel.
- **D5 — Broadcast + edge relay**: cuts wide-area egress by the clustering factor; clustered clients only.

Layered choices: **transport** (full snapshot vs batched delta — ~52 B IP overhead makes **batching a requirement**); **aggregation placement** (coupled to selection, not independent); and the **deployment axis** (edge LAN vs cloud/WAN **flips the ranking**; at cloud prices bytes are ~40× the CPU needed to avoid them). The master comparison and the conclusions were recorded: **three designs bind on three different resources**; **deployment decides the winner**; **D3 is a bounded niche**; **the cut is commodity**; and **at reasonable scale a snapshot with a client cut suffices**, so the engine's necessity must come from a **non-performance** constraint. Recorded: this is a **design-space characterisation** — legitimate, but **not novel**.

## 107. The cut, sharpened: placement × implementation, and the substrate

The student proposed **three choices** — client-side library; client-side + own check; server-side + own check. Corrected: these **mix two axes** — **placement** (client / server / split) and **implementation** (library-owned / library + own policy / own) — so the space is closer to a **2×2** than a three-way choice, with the missing cell **server + library** (`loaders.gl` headless; possible, not sensible). Clarified **"own check"**: *own traversal* versus *own policy on library output*; the latter is fragile, because a library selects by **screen-space error** and has already pruned — so the honest client option is **own traversal using library math**. Recorded: **pure client-library is weaker than it sounds** — three.js exposes **no visible set** (one `InstancedMesh` collapses to one test), Cesium `i3dm` is **tile-level**, and **per-node tiles are impractical at 100k**; the realistic client route is **regular cells + own check** (§72's B). The **control** is a **single cut crate compiled native (server) and to WASM (client)**, so **placement is the only variable** and a third-party cut does not confound the comparison.

## 108. Direct select (D6): explicit subtree subscription

The student proposed clicking a room in the hierarchy and receiving all its readings. Classified: an **explicit subtree subscription** — the simplest **Regime 2**, with the subscription chosen by the **user**, not the camera. It is **not a cut**: no view-dependence, no `B` unless imposed. Mechanics: a subtree is a **contiguous preorder range**, so membership is O(1); routing is a **per-node subscriber bitset**; `appeared`/`removed` + keyframe on subscription change; **no camera loop, no frontier, no hysteresis**. Two variants: **flat** (all devices in the subtree) and **progressive drill-down** (aggregates + child summaries; expand on demand — i.e. **manual LOD**). Egress: a benchmark room (~625 devices × 3) ≈ **3.75 KB/snapshot** (~3.75 KB/s at 1 Hz); the **root = broadcast** (~60 KB/snapshot). Pros: trivial, deterministic, works **off-screen**, maps to picking (a blended group's node index → expand), generalises FR-VS-02. Cons: **commodity** (MQTT/Sparkplug/OPC-UA/Ditto/ThingsBoard all do it); **unbounded at the root**; no free-camera relevance; churn on rapid clicking. Consequence: it covers the common **drill-down / inspection** case, so **the cut is unnecessary for inspection**; it is **the honest baseline that narrows D3's niche**; and **cut + direct select compose** (overview + inspection).

## 109. View-dependence: only the selection is view-dependent

Clarified: the **properties are view-independent** — a group's aggregate, its attention/severity, and its device values are the exact rollup of the subtree **regardless of the camera** (which is why aggregation is shared across viewers); only the **selection** is view-dependent, and that is a **delivery/presentation** decision. What view-dependence buys (relevance without asking; a **server-side** egress bound — **client-side culling does not reduce egress**; LOD/blending; "just works") and costs (the camera loop; per-viewer CPU; hysteresis/oracle complexity; latency sensitivity; commodity). The three **selection models** — geometric (camera) / explicit (user/query) / none (broadcast or precomputed) — and the conditions under which geometric selection is genuinely necessary (exploration; scene too large to enumerate; continuous pan/zoom; thin client; metered link; many viewers) versus not (dashboards, drill-down, inspection, alerting, field technicians). Dropping view-dependence leaves **state + hierarchy + aggregation + attention/severity + a subscription/delivery layer** — a smaller, more commodity system, and the shape with published demand.

## 110. Delivery versus rendering selection; colouring and compaction

**Colouring does not require the cut.** Colouring needs (1) the value, (2) a **node → instance mapping**, and (3) the instance in the draw buffer — none of which is a cut. The engine **conflated** **delivery selection** (what to send) with **rendering selection** (what to draw / at what LOD), because the client only held what the server sent. With direct select, delivery is the subscription and rendering is client-side; and if you **render all instances**, `slot = node index` is **static** — **no compaction, no slot table, no per-frame diff**, and picking is `instanceId → node index` directly (the engine needed `slotToNode` *because* the cut compacted). The engine's "40 MB client" is the **full manifest**; a compact render buffer is ≈ **1.6 MB at 100k**. The cut is therefore a **delivery** mechanism; rendering (culling, LOD, picking, colouring) does not require it.

When the visible set is **compacted** (slots `0…count-1`, `mesh.count = count ≤ B`), the mapping is maintained by the diff: `nodeToSlot` / `slotToNode`, with **swap-remove** on removal. Worked micro-example: `{A,B,C} → A↦0,B↦1,C↦2`; after `A` leaves and `D` enters: `C↦0,B↦1,D↦2` — *C moved slots without anyone deciding it*, so its matrix is copied and picking resolves slot 0 to C. Because transforms are **static**, matrices are written only on `appeared` and on the swap inside `removed`; colours are the only per-frame write. The mapping is **stable only under an unchanged cut** (the cut cache, Decision 26). The alternative — **identity mapping** (render all, slot = node index, hide via a per-instance visibility flag) — trades the churn for an **O(N) buffer** and drawing everything.

## 111. Tilesets, traversal, draw calls, and the control configuration

Clarified **tile versus tileset** for a library cut: to obtain the visible semantic nodes, use **per-node *tile*, inside *one* tileset** — **not** per-node *tilesets* (which multiply traversals, add N roots, and destroy hierarchy pruning). The library's traversal is **pruned** (frustum + screen-space error + `skipLevelOfDetail`), not "every tile," but a per-node tileset is **~40–60 MB at 100k**, **O(N)** client memory, and roughly **one draw call per visible tile** (batching lost). **Draw calls are issued by the library, not the developer**: the developer triggers a frame and controls scene membership, selection parameters, and content authoring; the per-tile draws are the library's. The **scaleable** structure is therefore **coarse group tiles + `i3dm` batches + own per-instance check** (§72's B). The **control configuration** was confirmed: **own the cut, write an instanced buffer, let the renderer issue one instanced draw** — the library is a **renderer, not a culler** (disable its own `frustumCulled`, preallocate `max = B`, set `mesh.count`). Recorded: **the manifest and a tileset are two encodings of one hierarchy**; a tileset is a **derived** artifact that must coexist with the manifest for value semantics — **two sources of truth**, roughly double the client metadata, and a lossy **two-box → one-box** mapping; the own-cut route needs **the manifest alone**.

## 112. Client memory as the surviving justification

If the goal is **minimal client memory**, the answer is the **lazy manifest** — ship descriptors only for the current cut — turning the dominant **~40 MB** metadata into **≈ 120–200 KB** at `B = 5,000` (O(B) rather than O(N)), with total client footprint ≈ 200 KB – 1 MB. But it **forces server-side selection**: the client cannot cut what it does not hold — §54's finding that **bounded client memory is a physical constraint and a better justification axis than exactness**. The **trilemma** (§54): O(B) memory + purely client-side selection + no extra round trips — **pick two**. Other levers, in order: own cut + renderer-only (avoid the O(N) tileset tree); compaction; **server-computed attention** (no per-node severity metadata client-side); compact binary encoding; progressive drill-down; remote/pixel streaming (the extreme). **Verdict:** the memory bound is the engine's **strongest justification** — physical, citable, non-performance — but it justifies **server-side selection**, not the cut uniquely (direct select and drill-down meet it too); it requires an **anchored device class**, an **O(N) baseline that genuinely exceeds** the bound, and the **lazy manifest (A′)** (without which the server cut gives no memory advantage); and in the PRD's **own scenario — a GPU laptop browser — it does not bind**. So it makes the engine **defensible in a thin-client, free-camera, hard-bound niche**, not necessary.

## 113. The shared value: "if it does not bind, is it worth it?"

The student observed that the supervisor and the agent share the value that **a mechanism that does not bind is not worth it**. Refined: the value is not *binding* but **causality / falsifiability under the studied conditions** — the **ablation test**. A mechanism's claimed benefit must be **caused** by it, **observable** at the operating point, and the study must **include** that point; otherwise the claim is vacuous. Where the crude form is wrong: **tail value** (caps, rate limits, safety bounds are valuable *precisely when they do not fire*), **structural value** (correctness, consistency, privacy constrain rather than bind), and **latent value**. Its **dark twin**: requiring the bound to bind tempts the **tuning trap** (choosing the regime so it binds), so the value must be paired with the **anchored-operating-point** rule (§13). **Constructive rule recorded:** *name the operating point where the mechanism is load-bearing, anchor it externally, and study it there; if no such point can be named honestly, the mechanism is decoration.*

## 114. Performance, must-have, and the product arena

**Performance improvement is unnecessary unless needed** — optimise against an **anchored requirement at a stated operating point**, measure first; an unneeded optimisation adds complexity and can **mask the real problem** (the engine's problem was an ungrounded premise, not speed). Corrections recorded: "needed" includes the **tail** and the **metered costs** (cloud egress, energy); and **improvement ≠ study** (a design-space characterisation is a contribution with no improvement at all — and COMP4299 requires evaluation, not improvement). Need is **regime-relative** ("unnecessary *at point P*," never in the abstract); and the right response to a requirement is sometimes to **avoid** it rather than optimise. On **nice-to-haves**: a nice-to-have is a *preference with no owner and no measure* and fails the anchor/measure/baseline tests, so it **cannot justify a project** — but the must/nice boundary is set by the **anchor**, not the property (60 FPS is a must in VR, a nicety in a dashboard), and **even a self-set "must"** (K3/K6) does *not* save a project if it is unsourced and met trivially. A nice-to-have **can justify a product — by demand**: markets rank among options that already meet the musts, and differentiation lives in the nice-to-haves, converted to justification by an **owner** and a **measure** (revealed preference). Recorded distinctions: **feature versus quality**; and the **arena** distinction — **product** justified by demand, **FYP** by problem + method + contribution + evaluation, **research** by a gap. Noted: the **GitHub hunt was the product-style demand test run inside a project** — it found demand for **aggregation**, none for the **coupling**. The rule did not change; the **currency** did.

## 115. State and gates

- **Track:** **decided — system-development** (COMP4299). Gate closed.
- **Candidate:** **unresolved.** The drift topic is no longer the leading default (the student explored beyond it).
- **The engine:** **not revived.** The competitor hunt supplies the **demand test** the review had lacked, and its verdict is **negative for the coupling** and **positive for plain aggregation**. The analysis remains the asset: the coupling, the commodity audit, the failing-baseline method, the related-work anchors, the design-space comparison, the cut taxonomy, and the memory justification.
- **Stakeholder:** declared **none**; the **external anchor** must carry the justification.
- **Open gates:** choose a candidate and anchor (an aggregation service; the drift topic; a standards/dataset project; an extended supervisor topic); **talk to the supervisor** (his bar; whether he accepts student-proposed topics); run the four-part filter (problem + stakeholder + buildable solution + verification, + reachable and observable); **integrity** (anchored criteria; no tuned constants presented as findings); and a **module-fit check** if any AI/ML is involved (COMP4299 vs CSAI4299).
- **What a new agent should do:** do not re-litigate the engine unless asked; ask which **candidate** and which **anchor** the student is on; help **name the anchor** and **draft the candidate paragraph** (*problem → anchor (how it substitutes for a user) → minimal solution → verification → ethics*).

## 116. Bottom line after Part VIII

- The session **closed the track gate** and applied two new instruments. The **feature-request hunt** produced the review's **first real demand evidence** — and it points at **aggregation**, not the **coupling**: the engine's residue is **unrequested**.
- The **design-space comparison** and the **cut taxonomy** — placement × implementation, the shared-crate control, **direct select**, **view-independence**, the **delivery/rendering separation**, and **compaction** — make the trade-offs reportable to the supervisor.
- **Client memory** emerged as the engine's **only surviving justification**: physical and citable, but **narrow** — it justifies **server-side selection** generally, binds only for a **thin-client, free-camera** consumer, requires the **lazy manifest**, and **does not bind in the PRD's own scenario**.
- The student, the supervisor, and the agent **share one value**: a mechanism must be **causal at an anchored operating point**. Applied precisely, it separates a project from a demo; applied crudely, it kills good engineering and invites the **tuning trap**. A **nice-to-have** can justify a **product by demand**, but not a project without an owner and a measure.
- **Status:** track fixed; **candidate and anchor open.**

---

# Part IX — Defensibility, disclosure, supervisor fit, and trust (ninth session)

Part IX records the ninth session (2026-10-17 → 2026-10-19). It closes the loop on **defensibility** — what makes a topic admissible, and what "mature solutions" and "one citation" actually buy; works the **privacy/disclosure** direction to a concrete problem, requirements, and a key extension; reads and positions two recent papers (**SHIP**; **hierarchical edge-IoT security monitoring**); examines the **supervisor's profile and authorship**; assesses whether the **engine fits** the supervisor's applications and whether it has **résumé value**; and closes on **IoT telemetry upload and trust**. Section numbers continue from §116. Figures remain estimates unless cited.

## 117. Orientation

The session moved from "which topic?" to "what makes *any* topic defensible?" — and stress-tested the disclosure direction against the literature and the supervisor's actual work.

## 118. The data-centre digital-twin proposal

"View-dependent data-centre telemetry visualization" assessed: **as pitched, not defensible**, and the domain weakens rather than strengthens the premise. Reasons: (i) a server room is ~**1,200 servers** (§58), so the entry budget never binds; (ii) the real pains — hot spots, PUE, capacity, blast radius — are **severity- and correlation-driven, not view-driven**; (iii) DCIM runs on a **LAN with capable clients**, the edge regime where broadcast wins; (iv) crowded (Vertiv, Sunbird, Nlyte, Cadence/6SigmaDC, Grafana, NVIDIA Omniverse). "100k EPS ≠ 100k devices" (§59).

## 119. Do data-centre twins use visualization?

Yes — prominently; 3D visualization is close to the signature of a DC twin (Sunbird: "a virtual 3D replica"; Raritan: "a 3D, software-based model"; Cadence Reality; NVIDIA Omniverse DSX). **But visualization is the interface, not the twin**: the industry itself says a true twin "goes far beyond 3D visualization" (Modius) — the substance is the synchronized model plus simulation (CFD/thermal, capacity, what-if). Visualization is **table stakes**, not a seam; 3D earns its place for **airflow/CFD**, not for telemetry.

## 120. Does an FYP rule out simple CRUD?

No — it rules out **simple**, not **CRUD**. The test: **delete the framework, the ORM, and the UI kit — is there a paper left?** A project passes on **problem + mechanism + evaluation** (plus substantial software and "integrate *and extend*"), not on the technology label. The failing pattern is assembly — the supervisor's six topics are one template: `sensor → threshold → dashboard → notify`.

## 121. Mature solutions and defensibility

Prior art does **not** disqualify (novelty is not required for COMP4299); it is the **baseline**. Four shapes: (a) build-where-exists → **assembly**; (b) build-for-a-context-the-incumbents-fail → defensible; (c) **study/evaluate** → defensible; (d) integrate-X+Y-where-the-composition-fails → defensible (the failing-baseline method). The fatal case is only (a). In a mature area the realistic routes are a **wanted residual** (needs access), a **study**, or an **engineering fit** judged on execution.

## 122. The salon booking example

A local hair salon wanting a booking system clears the gate the engine never did — a **real, reachable stakeholder** (and reopens the access constraint). But as pitched it is **assembly**: booking is commodity (Fresha, Booksy, Square, Vagaro, Google Reserve). The problem–solution mismatch: booking does **not** eliminate waiting; the real pains are **no-shows, service-time variability, walk-in mix, and skill assignment**. Reframes: no-show-aware overbooking, variable-duration scheduling, hybrid queues — each a mechanism plus an evaluation against the incumbent/manual baseline.

## 123. FYP versus product

Two objective functions: the FYP is an **assessment** (judge = examiner; success = a defensible contribution); the product is **value capture** (judge = market; success = users/revenue). The FYP deliberately **excludes** distribution, support, maintenance, monetisation. They share **one rung**: **problem legitimacy** (someone has the pain, or a citation). The product needs the whole ladder — **existence → salience → willingness to pay → viability**. The FYP's stakeholder is an **epistemic device** (prove the problem is real, enable evaluation), not a customer. A nice-to-have can justify a **product by demand**, not a project without an owner and a measure.

## 124. One citation and specificity

A generic citation ("someone wants a view-dependent visualization") is **the wrong citation** — it validates the **commodity half** and would be satisfied by Google Maps/Cesium. The citation must match the **contribution's specificity**. Evidence ladder: a paper's "future work" (weak) < one user (weak–moderate) < a **published complaint/workaround** (strong) < a **standard/regulation** (strong). One strong citation is **necessary, not sufficient**: extension, evaluation, feasibility, and integrity are still owed.

## 125. QR-code attendance

Public repos already do IP restriction, geofencing, and short-lived codes — commodity heuristics. A QR code is a **data carrier, not a presence proof** (trivially shared), so the design **resolves away its own hard part** (proxy attendance), exactly as §66 found for RFID. Residual seams: **relay/"mafia-fraud" resistance** (distance bounding, UWB, acoustic — mostly hardware), **privacy-preserving presence** (anonymous credentials / ZK: Semaphore, BBS+, eIDAS 2.0), a **rigorous comparative evaluation**, **collusion detection**, and **offline integrity**. The failing-baseline method applies: build the naive QR system, red-team it, build the reconciliation, evaluate (proxy rate, false accept/reject, usability, privacy).

## 126. Justifying privacy without demand

"Students want privacy" is weak as **demand** and probably false for the median student (privacy paradox; attendance is mandatory). Privacy survives on a different anchor: **regulation** (GDPR data minimisation/proportionality), **harm** (visa students, disabled or mentally-ill students, the stalked, marginalised groups), and **design obligation**. The "wanter" is the regulator/DPO, not the student — a mandated requirement without an eager user. The design tension to resolve: **verifiable accountability without central surveillance**.

## 127. The disclosure problem

Concrete problem: a campus operator shares a **live, aggregate view** of a building twin with an **external FM contractor** (one building; no other tenants; no identifiable individuals). Fourteen requirements across **access/scope, disclosure, delivery/correctness, accountability**, evaluated against the **engine** and the **best-of-breed composition**:

- The engine **supplies** aggregation, exactness plus a consistent snapshot, freshness, and bounded delivery — the **delivery half**.
- It **lacks** policy, suppression, temporal control, audit, revocation, and reconstruction evaluation.
- It **conflicts** where exactness/consistency **enables subtraction**, where **delta encoding hands over the differencing signal**, and where camera-driven selection conflicts with a fixed policy scope.
- The composition supplies policy/audit/static SDC but is **weak on live consistency, freshness, and bounds**.
- **Both fail** on live suppression, temporal privacy, and a reconstruction adversary — the seam.

## 128. Visualization not needed

The disclosure problem is a **delivery** problem; rendering is orthogonal. Dropping the 3D client removes the camera loop, the per-viewer cut, the roaming problem, and a large build cost; the scope becomes a **policy-defined subtree**. Keep at most a minimal demo (dashboard/table) and an **operator-side audit view** (which supports accountability and the evaluation). State the project as **disclosure-bounded delivery**, not visualization.

## 129. The key extension

**Online statistical disclosure control over a live hierarchical model, coupled to per-consumer bounded delivery, verified by a reconstruction adversary.** Components: **E1** live hierarchical disclosure control (minimum group size + **complementary suppression** across levels; the reduction-type nuance — `min`/`max` leak a unit); **E2** temporal control (differencing, the timing channel, a release budget); **E3** suppression stability (transitions leak; hysteresis); **E4** absence confidentiality; **E5** the reconstruction-adversary harness; **E6** the coupling (shared aggregation, per-consumer disclosure, bounded delivery). Why it is an extension, not assembly: the composition **fails** the requirement (exact aggregates leak), and the reconciliation is the engineering.

## 130. Existing solutions

Yes, in parts: **data clean rooms** (AWS Clean Rooms minimum aggregation, BigQuery aggregation thresholds) do scope + aggregate-only + minimum group size, commercially, **query-based**; **SDC** (sdcTable, tau-ARGUS, complementary suppression) is **static** and known-vulnerable (Cox 2010); **DP under continual observation** (Dwork 2010; Jain 2023; SPAS `w`-event; ToPS) is **noisy**; **hierarchical DP** (Fioretto/Census) is noisy. The **combination** — live, hierarchical, suppression-based, bounded per consumer — is **not productized**, but the space is **well studied**: research-shaped, modest novelty. The "live" crux: if periodic suffices, a clean room plus a scheduler is close to the solution.

## 131. Live versus periodic

In **rate**, live ≈ periodic with Δ → 0; but **event-driven** adds a **timing channel** that periodic does not. The real problem is **repetition over a horizon** (≈ T/Δ releases), because differencing accumulates — and static per-release SDC is known unsafe under repetition. So drop "live" as the framing: it is **online/repeated release**, and the controlling variables are **freshness Δ, horizon T, and whether timing is published**.

## 132. Why it is important

Three legs: the setting is real and growing (data sharing, clean rooms, data spaces); the obligation is real and citable (GDPR plus sectoral confidentiality); and the **current practice is unsafe and the safe alternative missing** — the **silent failure** of repeated thresholded release. The strongest leg is the third. Honest counterpoints: demand may be weak or unspoken; mature fields may be "good enough"; the scenario may be contrived; the problem can be avoided. Importance is **conditional** on a documented deployment or standard.

## 133. How to address, why nobody, how to prove

Two coherent routes: **DP under continual release** (formal, noisy) versus **suppression-based online SDC** (exactness, no general proof). "Why nobody does it" — the §63 prior: most likely **exactness is the idiosyncratic requirement** (drop it and DP solves it), or demand is weak, or it is designed away. **Proving no leak**: DP is the only general proof (composition handles repetition); complementary-suppression correctness proves **single-release** bounds, not sequences; otherwise you get **measured** resistance. The honest claim: **exactness where released + no single-release reconstruction (proved) + measured sequence resistance (evaluated)** — not a universal no-leak proof.

## 134. SHIP (arXiv 2610.21048)

**SHIP: Structured and Hierarchical Insight with Privacy** (UVA + Google Research; cs.CL; 15 Oct 2026): DP insight extraction from **textual corpora**; a **hierarchical hybrid taxonomy** (open-set DP partition selection + closed-set supplementation) and a **DP synthetic table** (AIM) preserving cross-attribute marginals; a **zCDP + composition** guarantee. Relevance: a **related-work anchor**, not a competitor — hierarchical DP release is established, and DP-with-composition is the **proof** answer. It is **textual, one-shot, and noisy**: it does not address repeated release, per-consumer delivery, or exactness. Applying it = joining the DP route (and conceding exactness).

## 135. The supervisor's profile and authorship

**Su-Kit Tang**, Associate Professor, Faculty of Applied Sciences, **Macao Polytechnic University**. Five recent lines: **embedded/edge AI** (review; zero-copy instrumentation); **lightweight vision for conservation** (eBirdNet-Nano); **LLM + built environment** (floor-plan analysis/compliance); **air quality** (feature screening); **auditability/privacy** (auditable blockchain confidentiality). Authorship check (Crossref): he is **never first author**; **last/senior** on the edge-AI/floor-plan/embedded-AI papers (his students lead); **middle** on the air-quality, blockchain, employment, and sentiment collaborations. Implication: the last-author lines are where he **supervises students** with data/hardware — the best bets; the middle-author collaborations are less clearly his.

## 136. Fitting the engine to those applications

The engine's **parts** can appear (hierarchy, aggregation, filtering), but the **engine as contribution** does not fit: the bird monitor's hard part is detection; the floor-plan's is extraction/rules; the embedded benchmark has no hierarchy; air quality's is calibration. In every case the **distinguishing mechanism — live, bounded, per-consumer, view-dependent — does not bind**. Diagnosis: "hierarchy + aggregation + filtering" is **infrastructure** — ubiquitous as a component, never a product alone. The search direction is inverted: not "where does my solution fit?" but **"whose requirement fails the existing tools?"**

## 137. Résumé value and engineering depth

The engine has **portfolio value** — high for systems/backend/infrastructure/real-time roles — **conditional on building and measuring** (docs-only is weak). Present **engineering metrics**, not users; frame it as an exploration. **"Engineering depth" ≠ from scratch**: refusing libraries is a yellow flag (NIH); depth = justified dependencies + understanding + measured trade-offs + correctness discipline. Reuse transport/math/rendering; build the **state table, aggregation, cut, delta protocol, oracle, harness**.

## 138. The edge-IoT security-monitoring paper (arXiv 2610.09817)

**Hierarchical Security Monitoring for Edge-IoT: A Formal Methods Approach** (IEEE CSR 2026; Cyprus/Greece): two-level runtime verification — **L1 TeSSLa** per-device predicates (size, fuzz, monotonic time, rate) aggregated to **four-valued verdicts** (⊤/⊤p/⊥p/⊥); **L2 MonPoly MFOTL** at the gateway correlating per-device verdicts (three nodes with ⊥p within 60 s) to catch **coordinated APT**; a lattice-meet composition with **witness sets**; a shield (block/confirm/log/allow); uplink **one Boolean per window per node**. Evaluated on a Docker testbed: L1 catches single-node classes; APT 12/16 → 14/16 with L2; FPR 24% (timestamp artifact) → 4% on a cleaner replay; ~0.52 µs/event, ~1 µs/verdict; ~200 bit/s uplink at N = 1000. Limits: synthetic, offline, single gateway, **non-adaptive adversary**. **Lesson: start from the cost asymmetry (edge vs cloud vs attention), then derive the hierarchy** — the inversion the engine never made.

## 139. IoT telemetry: upload and trust

**Upload**: link (Wi-Fi/Ethernet/cellular/LPWAN/BLE/Zigbee/Modbus/CAN/satellite); protocol (MQTT via broker; HTTP(S); CoAP; AMQP; OPC UA; Modbus/DNP3/IEC 61850; Sparkplug B; LoRaWAN; WebSocket); patterns (gateway-mediated, store-and-forward, periodic/delta/event, JSON/CBOR/Protobuf); **time** (device vs gateway-observed vs server). **Trust** is six separable questions: authenticity (mTLS/certs/PSK), integrity (TLS/MAC/signature), freshness (nonce/sequence/timestamp), device integrity (secure boot, **remote attestation**), **data truth** (range/rate/plausibility/redundancy/calibration), and **behaviour** (rate/size/time predicates, cross-device correlation). **Cryptography gives authenticity and integrity, not truth**; a compromised sensor signs a wrong value perfectly. Governance adds provisioning, revocation, access control, and **audit/provenance**. This maps onto the papers: the edge-IoT paper is the **behavioural** layer (and assumes identity/attestation); the drift topic is the **data-truth** layer; the disclosure topic is the **governance/audit** layer.

## 140. State and gates

- **Track:** system-development (COMP4299) — **decided**.
- **Candidate:** **open**, but the landscape has contracted. The engine is **not revived**; its parts are reusable and its analysis is the transferable asset.
- **Access:** no stakeholder declared; **the supervisor's lab is the realistic source** (data/hardware), and the **last-author lines** are the strongest bets.
- **The disclosure direction:** the most intellectually coherent, but **research-shaped** and dependent on the exactness/DP decision and the "repetition over a horizon" framing.
- **Gates:** (1) choose the candidate/anchor; (2) confirm **COMP4299 vs CSAI4299**; (3) establish **data/hardware availability** with the supervisor; (4) decide the **exactness vs DP** route if the disclosure direction is taken; (5) integrity (no tuned constants; derivable decisions).
- **Supervisor questions:** which line has data/hardware this term; does he accept a student-proposed topic; which module; what bar.

## 141. Bottom line after Part IX

- The session produced the review's **most complete account of defensibility**: prior art is the **baseline**, not a disqualifier; the fatal shape is only "build the solved thing again, for nobody, measured by nothing"; a **study** or a **failing-baseline extension** is the honest escape.
- The **disclosure** direction reached a concrete problem, requirements, and a **key extension** (online hierarchical SDC + a reconstruction adversary) — but the literature shows it is **well studied**, the "live" framing collapses to **repetition over a horizon**, and the **exactness** constraint is the idiosyncratic bet.
- The **supervisor's profile** is the strongest practical asset: his **last-author** lines (edge AI, bird monitoring, floor plans) provide data, hardware, and supervision; the middle-author collaborations are weaker bets.
- The **engine** retains value as a **portfolio** piece (conditional on building/measuring) and as **analysis** (coupling, design space, commodity audit) — not as the FYP contribution.
- **Status:** track fixed; candidate, anchor, and module fit open; the next move is a **supervisor conversation** with 2–3 one-paragraph candidates and two questions (data/hardware; COMP4299 vs CSAI4299).
