# FYP Topic Review — Discussion Summary

| Field | Value |
| --- | --- |
| Document | DISCUSSION-SUMMARY |
| Date | 2026-10-12 |
| Subject | Review of a proposed final-year project: server-side spatial filtering engine for 3D digital twin visualization |
| Artifacts reviewed | `docs/PRD-001__Product_Requirements_Document.md`, `docs/TDD-001__Technical_Design_Document.md`, `docs/PLN-001__Master_Execution_Sequence.md` |
| Participants | Supervisor (critic) and student (author) — roles later swapped into a viva-style exchange |

This document records the discussion so it can be resumed later. It captures the initial evaluation, the student's defence, the technical findings, and the reframe that emerged. It is a record, not an instruction to change the design documents.

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
