# FYP Topic Review — Discussion Summary

| Field | Value |
| --- | --- |
| Document | DISCUSSION-SUMMARY |
| Date | 2026-10-19 (nine sessions: 2026-10-12 → 2026-10-19) |
| Subject | Review of a proposed FYP: server-side spatial filtering engine for 3D digital-twin visualization |
| Artifacts reviewed | `docs/PRD-001`, `docs/TDD-001`, `docs/PLN-001` |
| Participants | Supervisor (critic) and student (author); later a viva-style exchange |

**Purpose.** A resumable record of the review. Section numbers are stable and are cited by `DEC-001` and the build handover. Unless labelled otherwise, figures in Parts I–VIII are first-principles estimates against stated constants, not measurements; Part IX adds cited literature.

**Editorial note.** Condensed 2026-10-19: repeated restatements and superseded status sections were collapsed; all 141 section numbers and the evidence tables are retained. The pre-condensation text remains in git history.

| Part | Sections | Session | Focus |
| --- | --- | --- | --- |
| I | §1–§8 | 10-12 | Initial evaluation, defence, reframe |
| II | §9–§20 | 10-13 | Cost model, deployment reframe, regime 2 |
| III | §21–§33 | 10-14 | IoT pivot, libraries, viva, reposition |
| IV | §34–§47 | 10-15 | Problem-first discipline, Google-Maps framing |
| V | §48–§78 | 10-16 | Claims tested, mechanics, feasibility |
| VI | §79–§92 | 10-16 | Engine disposition, paper catalogue, topic search |
| VII | §93–§98 | 10-17 | Drift critique, "trivial" question, calibration |
| VIII | §99–§116 | 10-17 | Track closure, competitor hunt, design space |
| IX | §117–§141 | 10-17→19 | Defensibility, disclosure, supervisor fit, trust |

---

## 1. What was reviewed

Three drafts by the student, said to be ~80% complete:

- **PRD-001** — server-side real-time spatial filtering/streaming engine; synthetic simulator, browser 3D client, demo; 4,280 / 50,000 / 100,000-device tiers; KPIs K1–K11.
- **TDD-001** — Rust engine, WebSocket binary transport, per-viewer visibility cut, exact hierarchical aggregation, 27 design decisions.
- **PLN-001** — 25-week sequence (2026-09-14 → 2027-03-01), M1–M10, cut-down ladder.

## 2. Initial evaluation (supervisor's opening position)

**Verdict:** do not dismiss, but do not approve as pitched. The "simple filter" characterisation is inaccurate; the student leads with the weakest claim and never confronts the alternative that could invalidate the premise.

**"It's just a simple filter":** wrong. The engine has four subsystems — a lock-free latest-wins state table, exact hierarchical aggregation, a best-first LOD cut under an entry budget, and stateful per-viewer delta encoding — plus a brute-force oracle.

**"Just add bandwidth":** half right, for a different reason.

- The 30 MB/s baseline is full-key JSON (PRD §7.1), a **strawman**. The ingest record is 10 B (TDD §12.1); the honest binary baseline is ~3 MB/s → the real cut is ~10×, not 99%.
- The real constraint is **where per-viewer work lives**, not bandwidth.
- "The browser draws; the server thinks" is **asserted, not justified**.

**Other findings:** no research question or related work (plan item A1 has deliverable "—"); KPIs are acceptance criteria, mostly guaranteed by construction; missing baselines are a fat client (binary push + client cull) and a flat server-side filter; the demo runs at building scale where `B = 5,000` can never bind; scope far exceeds one undergraduate over 25 weeks; the plan is stale against the design and behind M1 (due 2026-10-05); no dissertation — P7 ends in a "v1.0 release / demo video", and PLN §1 calls it "a personal project"; document polish exceeds typical year-4 output, so decisions must be derivable cold.

## 3. The student's defence, and where it landed

1. **"More bandwidth"** → conceded; accepted the honest ~10× figure.
2. **"The cut scales badly"** → conceded: ≈1.8 ms/frame ≈ 0.11 cores/viewer; K10's "10 sessions" admits a ceiling in the tens. Mitigations proposed: incremental cut refinement, decoupling cut from encode rate, multi-core parallelism.
3. **"It's a sellable product feature"** → conditional only. The thin client is the real justification, but every scenario that would make the feature matter (mobile/native, WAN, auth/governance) is out of scope; the client is fixed as a GPU laptop (PRD §9–10). Sellability is not an academic criterion.
4. **"The algorithm is hard to implement"** → hard-to-get-right ≠ hard-to-invent; the hybrid needs the same cut (it relocates, not differentiates); some complexity is self-imposed (Decision 16 admits a mutex would suffice; Decision 27 optimises a deployment the PRD excludes). Difficulty + lateness + solo = feasibility risk, not advocacy.

## 4. Key technical findings

**4.1 The per-viewer cut is the architectural cost.** Per-viewer cost is linear in V with a non-trivial constant; the hybrid's per-viewer cost is a socket write. The only clear win is per-client egress — which matters on WAN/mobile, both out of scope.

**4.2 The client is O(N) anyway.** The design ships a ~40 MB manifest to every client (TDD §10.5), so "the server thinks" saves nothing on client memory. O(B) requires a **lazy manifest** (descriptors only for the current cut) — not designed. This addition (termed **A′**) is what would make the server-side claim real.

**4.3 Existing libraries already solve the cut.** 3D Tiles/CesiumJS (hierarchical traversal, frustum + occlusion culling, SSE budget, `skipLevelOfDetail`) is essentially the student's traversal model; also Potree (octree LOD), OSG/osgEarth `PagedLOD`, Umbra, UE5 World Partition/HLOD/Nanite, and interest management in SpatialOS/Photon/Mirror/FishNet/Unity NGO. The **cut is not novel**, and the product feature is not novel. Not covered: **exact live aggregation over a semantic hierarchy under a view-dependent cut** — the real, narrow gap.

**4.4 Collecting visible IDs.** In a client-cut design the client **computes** the cut, so it holds the visible set by construction (three.js frustum/`Box3`; Cesium `tileVisible`; Potree selected nodes; OSG cull visitor; engines `isVisible`). If the *server* needed the client's visible IDs, that would reintroduce the per-frame feedback loop **while** paying the client cut — the worst of both.

**4.5 The reframe to pub/sub.** Without a camera loop the server still sends viewer-independent reduction (aggregation, ε-suppression); the loop carries ~48 B/frame, so the cost is what the server *computes* per viewer. Regime 2 sends both the aggregate stream and device-level state for subscribed regions (on change), or it could never expand to devices (FR-VS-05). **All three regimes are pub/sub**, differing in subscription granularity, refresh cadence, and who computes the subscription:

| Regime | Subscription | Cadence | Computed by | Server cost |
| --- | --- | --- | --- | --- |
| 1. Broadcast | everything | none | nobody | fanout only |
| 2. Interest | coarse region/subtree | on change | client | fanout + routing |
| 3. Original design | exact frustum | every frame | server | fanout + per-viewer cut |

**4.6 The three axes.** Egress vs N; server CPU vs V; client memory vs N. At most one binds at the student's scales.

## 5. The reframe (emerging thesis)

| Feature | Nature | Action |
| --- | --- | --- |
| Hierarchy tree | infrastructure | keep, don't pitch |
| Aggregation | differentiated core | keep, pitch as a question |
| Camera cut | solved pattern + design variable | demote; evaluate the standard version |

**Working title:** *Server-side versus client-side view-dependent streaming for large live-telemetry 3D twins: a measured crossover study.*

**Research question:** For N devices and V viewers, where should selection/LOD and aggregation live — server, client, or hybrid — as a function of N, V, and link budget; what exactness, consistency, and latency guarantees are achievable at each split; and where are the crossovers?

**Contribution:** the semantic aggregation model (per-unit exact rollups, `(sum,count)` merge, `count` tallies, attention/absence semantics, per-frame snapshot consistency) **plus** the placement/tradeoff characterisation — not transport, not the cut.

**Architectures:** A (per-viewer server cut), B (broadcast + client cut), A′ (A + lazy manifest → O(B) client). One cut crate compiled server-side (A) and to WASM (B); one oracle.

## 6. Process and integrity points raised

- Restore the literature review and position it against 3D Tiles, Potree, interest management.
- Replace self-set KPIs with crossover measurements; drop the nil-predicted sibling-order ablation.
- Bring the justifying scenario into scope (thin client / constrained link / disclosure) or drop the feature claim.
- Cut scope (100k, SIMD, ablations → stretch); resync PLN to TDD; acknowledge missed M1; replace "v1.0 launch" with a dissertation.
- Derive Decisions 16, 14, 24, and 22 cold.

## 7. Open questions left on the table

1. **Research or engineering degree?** Unanswered — the gate.
2. **Which axis binds** — egress-vs-N, CPU-vs-V, or client-memory-vs-N?
3. **The crossovers** — at what N does broadcast fail; at what V does the per-viewer cut? If the cut saturates at V≈10 and broadcast at N≫100k, the client cut wins at the stated scales.
4. **Why not regime 2?** It may dominate both, keeping CPU flat in V while bounding egress by subscription.
5. **Subscription boundaries** — they inherit the flicker/hysteresis problem (R7) on the data channel.
6. **Aggregation placement** — can it move to the client under subscription, and what breaks?
7. **Topic tree = group tree?** And is aggregation the true thesis, with the cut as a variable?

## 8. Bottom line at the time of writing

- As a **topic**: valid and above-average, if reframed from "a server-side filter" to a **placement/tradeoff study** anchored on the **semantic aggregation model**.
- As **pitched**: it leads with the weakest claim (bandwidth) and never tests the premise against a client cut or subscription.
- The thesis was never the server-side cut — it is **exact live aggregation over a semantic hierarchy**, with the **placement of selection** as the variable.
---

# Part II — Resumption and re-scoping (second session)

Part II resumes from §7: the taxonomy, baseline and cost model, deployment reframe, regime-2 design, and the position on client-side libraries.

## 9. Resolution of the open questions in §7

| §7 question | Status after Part II |
| --- | --- |
| 1. Research or engineering degree | **Still open**; it remains the gate. |
| 2. Which axis binds | **Per regime and scenario:** regime 1 on egress (and cloud egress *cost*); regime 3 on server CPU; regime 2 on subscription size α. |
| 3. The crossovers | **Computed.** Link crossover ≈ `V_cpu(R3) × egress(R1)`, near **1 GbE-equivalent** under the PRD's parameters — inside its assumed LAN. It moves with N, B, r. |
| 4. Why not regime 2 | **Analysed.** It can dominate under coarse subscriptions (§16); a *fine* frontier subscription collapses its egress onto R3's. |
| 5. Subscription boundaries | **Designed for.** Hysteresis moves to the subscription channel; §16 specifies banded, rate-limited updates. |
| 6. Aggregation placement | **Resolved structurally:** client aggregation needs the subtree, forcing broadcast/interest and more data onto the wire. Selection and aggregation placement are coupled. |
| 7. Topic tree = group tree | **Yes.** The manifest node tree is the topic namespace, keyed by node index; subtrees are contiguous ranges (§16). |

## 10. The three regimes, sharpened

| Student's label | Regime | Subscription | Computed by |
| --- | --- | --- | --- |
| client receives all | 1 — broadcast | everything | nobody |
| client cull + LOD, pub/sub | 2 — interest | region/subtree | client |
| original design | 3 — server cut | exact frustum | server |

- **All three are pub/sub.** The variables are **granularity, refresh cadence, and who computes the subscription**.
- **Aggregation stays server-side in all three.** It is a separate, coupled axis, not a fourth regime.

**The coupling (formally):** client-side aggregation requires the client to hold the subtree, which only regime 1/2 substrates provide; moving aggregation client-side therefore forces selection client-side and more device data onto the wire. The two placements cannot vary independently.

## 11. Egress, cadence and headers

The student's first calculation (`20,000 × 3 × 20 B × 0.4 × 60 × 8 = 219 MB/s`) was wrong three ways:

1. **Unit slip (~8×).** Bytes/s × 8 gives bits/s; the correct figure is **28.8 MB/s ≈ 230 Mbit/s** (219 was a mebibyte count mislabelled).
2. **The 60 appeared twice** — as both a snapshot cadence and a reading rate, neither named.
3. **The reading rate was missing**; it sets volume.

```
egress = (values sent / s) × (bytes / value) + (messages / s) × (header bytes / message)
```

**Three cadences must not be conflated:** reading rate (source), frame cadence (60 Hz, latency), egress cadence (per regime). Per-frame changed fraction is `r / cadence × p`, not a constant.

**Headers matter.** ~2 B WS + 20 B TCP + 20 B IPv4 ≈ **52 B per message** (~70 B with Ethernet); at high rates that is **5–7× inflation**. Frame-paced batching amortises it to ~0.09%. The 60 Hz frame is therefore the **header-amortisation mechanism**; payload-only counts hide this.

**Server egress capacity** has four ceilings: link (1 GbE ≈ 110 MB/s; 10 GbE ≈ 1.1 GB/s), ~1–2 Mpps, process CPU, and path. Capacity is shared, so `per-client budget = capacity / V`. At K10 (10 × 0.3 MB/s), egress is ~3% of a 1 GbE link — **not the binding axis in the PRD's environment**.

## 12. Per-viewer cost model and the regime-3 bottleneck

```
V_max(ρ) = capacity(ρ) / per_viewer_demand(ρ);   bottleneck = argmin over ρ
```

Per-viewer cost = `T_cut(B) + T_encode(B)`, both ~linear in B. Calibrated to TDD §13 (K2: 1.5 ms at B = 5,000; encode 0.3 ms) and re-derived:

| | First-principles (hot) | Cold / p99 | TDD budget |
| --- | --- | --- | --- |
| `T_cut(5000)` | ~0.35 ms | ~0.6–1.0 ms | 1.5 ms |
| `T_encode(5000)` | ~0.17 ms | ~0.25–0.35 ms | 0.3 ms |
| per-entry | 0.06–0.19 µs | | |

Fitted: `T_cut(B) ≈ 0.05 ms + b·B`, `b ∈ [0.06, 0.19] µs`; `T_encode(B) ≈ c·B`, `c ∈ [0.03, 0.07] µs`.

- **T_encode is stable** (sequential, prefetchable); **T_cut is not** (dependent, branchy, scattered), varying ~3× on cache/branch behaviour.
- Cores (`×60`) over an 8-core budget gives **V_cpu ≈ 74 (TDD) to ~260 (hot)** at B = 5,000. The unsigned constant is `b`; benchmark it first.
- **Regime 3 is CPU-bound at every B examined**; its NIC wall is ~5× further out.
- The cut cache (Decision 26) makes a static viewer free, so **camera motion** sets the average; p99 is the KPI; the 50% CPU budget is policy.

## 13. A defensible baseline (reference scenario)

Citing PRD values is circular. The method: describe a plausible deployment, derive every parameter from **external** anchors, fix them **before** knowing the winner, then sweep the studied axis.

**Reference scenario:** a mid-size industrial site (~10k tags), heterogeneous 0.1–10 Hz updates, 10 operators, cloud-hosted, browsers over WAN.

| Axis | Default | External anchor | Role |
| --- | --- | --- | --- |
| N devices | 10,000 | mid-size plant SCADA/OPC-UA tag counts | axis (1k–100k) |
| m metrics/device | 3 | temp / vibration / power | fixed |
| r | 1 Hz (0.1–10 variant) | monitoring cadence; Prometheus 15 s; ISO 10816 | axis / sweep |
| B entries | 2,000 | marker **legibility**: ~32 px on 1080p | axis (1k–8k) |
| V viewers | 10 | control-room staffing | axis (1–1,000) |
| link | 1 GbE / cloud VM ceiling | dominant Ethernet | axis |
| latency | ~1 ms edge / 20–80 ms cloud | LAN vs regional WAN | axis |
| p | **measure it** | noise ≈ ε is a simulator choice | measured |
| value width | f16, symmetric across arms | sensor ranges fit f16 | fixed |

**The tuning trap.** The student proposed adjusting N down, B down, B up, r up (three times), and f16 off — five parameters, each defensible alone, several moving the result toward the server cut. Rule: **anchor or sweep; never tune to make the crossover appear.** N and B change the ratio (~N/B); r shifts it via CPU/NIC asymmetry.

**Soft parameters:** **B** is perceptual (marker legibility; sweep it) and **p** is empirical (measure it). "1,000 visible devices is rare" was refuted: a room is ~630 devices, so two rooms cross B = 1,000; 1,000 markers at ~45 px fills a 1080p screen. 1,000 is the design case.

## 14. Deployment-scenario reframe (the central new result)

The PRD's wired-LAN assumption is **permissible but unrepresentative**, and *load-bearing*: it removes both reasons for the feature (cheap bytes and ~1 ms latency), selecting the one world where broadcast wins.

**The modal modern arrangement is hybrid edge + cloud:** collection/control at the edge (latency, safety, cybersecurity); aggregation, storage and **visualization** in the cloud; browsers over WAN.

**Cloud changes three things:** WAN latency/jitter (10–100 ms, threatening K11); constrained client last hops; and **metered egress** ($0.05–0.12/GB).

**The economic axis is the strongest justification.** Egress scales with `per-client bytes × V × time`. Illustrative (100k tier, $0.09/GB, per client-month): broadcast ~$350; server cut ~$70; raw binary ~$700; JSON ~$7,000. **The PRD's 30 MB/s framing only matters if egress is billed — i.e. in the cloud. The PRD describes a cloud-scale problem and assumes a LAN; that is the internal inconsistency at the root of the review.**

**Hosting:** static musl binary → tiny distroless container; ~20 MB + 1.3 MB/viewer; 2–4 vCPU covers the reference load. VM/container fits; **serverless does not** (stateful streams, sessions, timeouts). The load balancer must support long-lived WebSockets.

**Study frame:** primary **cloud-WAN**, contrast **edge-LAN**; use **`tc netem`** for deterministic delay/loss and **price** instrumented bytes. Requires a **versioned PRD amendment** (PRD §10 excludes WAN/cloud).

**Scenario-vs-regime trade:** *scenarios are cheap; regimes are expensive.* If time is short, cut the **third regime** (model regime 2 analytically), not the contrast scenario.

## 15. Bottleneck comparison at the latest settings

Settings: N = 10k, m = 3, r = 1 Hz, p = 0.5, B = 2,000, f16, 8 cores, egress $0.09/GB, CPU ~$32/core-month.

| | Egress | GB/month | $/month | CPU/viewer |
| --- | --- | --- | --- | --- |
| R1 broadcast | 120 KB/s | 311 | **$28** | ~0.002 (fanout) |
| R2 interest (α = 0.1) | 12 KB/s | 31 | **$2.8** | ~0.003 (routing) |
| R3 server cut | 24 KB/s | 62 | **$5.6** | **~0.025** |

| Regime | Scarce resource | Wall |
| --- | --- | --- |
| R1 | egress **cost** / client link | ~36 clients per $1,000/month |
| R2 | subscription α | ~357 per $1,000/month |
| R3 | **server CPU** | ~320 viewers (235–570) |

At cloud prices, bytes are **~40× more expensive than the CPU needed to avoid them**, so the ranking inverts:

| Scenario | Cheapest | Middle | Most expensive |
| --- | --- | --- | --- |
| **Cloud/WAN** | R2 | R3 | R1 |
| **Edge/LAN** | R1 | R2 | R3 (CPU-bound) |

**Central finding:** the regimes bind on three different resources — R1 on egress/client-link, R2 on subscription size, R3 on server CPU — and **deployment, not tuning, decides the winner.** In cloud the server cut is a defensible middle: dearer than pub/sub but the only **hard per-client CPU bound** independent of scene and subscription. In an edge LAN, broadcast wins while the cut is CPU-bound at a few hundred viewers.

## 16. Regime 2 design: topics and subscriptions

**Topic namespace = the resolved manifest's node tree.** Node index is the topic id (Decision 11); `kind` selects update kind; preorder/`parent_idx` gives the hierarchy; `subtree_size` + contiguity makes a subtree a **contiguous range** (membership O(1)); metrics/channels/boxes define what each topic publishes.

At 10k devices the namespace is ~**10,180** nodes. A viewer holds ~200 **ranges**, not 10,000 topics; **α, the subscribed fraction, is the binding metric**.

**R2 adds:** a per-session **node → subscriber map**; a **publish path**; and a **subscription lifecycle**. Recommended: a **per-node subscriber bitset** (`N × V/8` ≈ 13 KB at V = 10), routing as a bit test, fanout by popcount.

**Subscription updates:** computed by the client (shared cut crate in WASM), sent as **deltas on change — not per frame**, coalesced/rate-limited (~10 Hz); **hysteresis is mandatory** (size and budget bands), or boundary viewers churn every frame and move R7 onto the data channel; new nodes are "appeared" keyframes, removed nodes are pruned.

**The granularity fork (fix before comparing):**

| | Fine (frontier) | Coarse (region/subtree) |
| --- | --- | --- |
| Server sends | frontier values | **raw device state** |
| Egress | **= R3** | **> R3** when zoomed out |
| Consequence | R2 ≡ R3 with the cut moved to the client | the genuinely distinct third architecture |

Corrected ordering: **R1 > R2(coarse) > R3 ≈ R2(fine)** on egress; **R1 < R2 < R3** on CPU.

## 17. Sharing broadcast egress

Can regime 1 send once and share bytes? **Computation yes, bytes no.**

- WebSocket/TCP is **unicast**; `broadcast()` loops sockets, so bytes cross the wire V times.
- **IP multicast** would replicate once, but browsers cannot subscribe and cloud VPCs disable it; CDNs cannot cache live personalised streams; P2P moves but does not remove the send.
- **Cloud egress is billed per byte leaving the cloud**, so an internal shared buffer saves nothing.

**The one real mechanism: an edge relay for co-located clients.** The cloud sends **one** stream to a relay, which fans out locally — cutting *wide-area* egress by the **clustering factor**. This makes **"broadcast + edge relay" a fourth architecture**, dependent on **client topology**, cheap to model (one division), and the one lever that rescues broadcast in cloud.

## 18. Client-side cut and libraries

**three.js does not expose the visible set.** Culling is per-`Object3D`; with one `InstancedMesh` it degenerates to **one aggregate test**. There is no per-instance visibility, visible-list API, or visibility event (`onBecameVisible` is Unity). It gives culling *math* (`Frustum`, `Box3`) and a *flag* the renderer reads but never sets.

**This costs nothing:** in client-cut regimes the client computes the frontier anyway, and **the frontier is the visible set**. Practical: reuse the shared crate; test manifest boxes; compact instances and set `InstancedMesh.count`; beware stale bounding spheres (`frustumCulled = false` is simpler).

**Libraries that expose visible ids** — CesiumJS (`tileVisible`), `loaders.gl` `Tileset3D`, Potree (`visibleNodes`), Babylon (`getActiveMeshes`), deck.gl, OSG/osgEarth — cut **their own** hierarchies and expose **their own** ids. **None aggregates**, and all force a format mapping.

**Position:** one **shared cut crate** compiled server-side and to WASM, so placement is the only variable; Cesium/`loaders.gl` as cross-check and related-work anchor; reuse only the math. Do not outsource the cut.

## 19. Open questions remaining

1. **Research or engineering degree** — the gate.
2. **R2 granularity** — fine or coarse; decides whether R2 is distinct from R3.
3. **Client topology** — scattered or clustered; decides whether the edge relay applies.
4. **Two scenarios or one** — cloud-WAN + edge-LAN (needs a PRD amendment) or cloud alone (weaker).
5. **Measured constants** — `b`, `c`, `p`, real cloud prices — each converts estimates into results.
6. **Integrity** — derive decisions cold; no tuned parameters as findings.

## 20. Bottom line after Part II

- The **method** changed from "propose a design and hit self-set KPIs" to "characterise where selection/aggregation placement is optimal, by deployment." A **reference scenario** with external anchors replaced the PRD as the parameter source.
- The **central result** is a deployment-driven crossover: on an **edge LAN**, broadcast wins and the cut is CPU-bound at a few hundred viewers; in **cloud/WAN**, egress is scarce and metered, interest pub/sub is cheapest, and the cut occupies a defensible middle with the only **hard per-client CPU bound**.
- The honest headline does not flatter the design: the cut is not the general winner; it is the *bounded* option, and its advantage lies in the region the PRD excludes.
- The **contribution** remains **exact live aggregation over a semantic hierarchy**, with **selection placement as the variable**.
- **Cheapest path:** two measured arms (R1, R3) off one crate and one oracle; R2 modelled; cloud-WAN measured with `netem`; edge-LAN modelled; N, V, r, link, B swept; every constant cited.
---

# Part III — Domain pivot, libraries, and viva (third session)

Part III records the third session (2026-10-14): the "IoT" pivot, what "the cut is solved" means, client-cut mechanics with a tiles library, attention rendering and instance mapping, and the viva that rejected the crossover framing.

## 21. The IoT pivot: is it defensible?

The proposal was "a semantic filtering and aggregation implementation for IoT devices." **"Semantic" here means a structural hierarchy** — the tree of §4–§8 with per-unit channels — **not** RDF/OWL/SPARQL reasoning.

**Verdict: not defensible as stated.** It is a **more crowded** field than the digital-twin setting; it **removes the coupling** that made aggregation non-trivial (§10), reducing the problem to "roll up a static tree", which is solved; and "implementation for IoT devices" reads as engineering, adding hardware scope without thesis content.

Already solved by: TimescaleDB/InfluxDB rollups; Flink/Kafka Streams/Spark; ThingsBoard/AWS IoT Analytics/Azure Stream Analytics; **MQTT Sparkplug B**; Eclipse Ditto/Azure Digital Twins/FIWARE NGSI-LD; C-SPARQL/CQELS/RSP-QL/Streaming MASSIF; TAG/TinyDB and sketches.

**The gate is the scarce resource.** A pivot is defensible only if a real scarce resource is retained **and** selection stays coupled to aggregation; the natural one is **edge/cloud placement under a metered or constrained link**. Plain aggregation with no named resource is engineering-only. The degree gate (§19.1) remained open.

## 22. What "the cut is solved" means

"The cut" is the per-viewer, per-frame visibility + LOD selection producing a bounded set under `B` (§9.2). "Solved" is a claim about **novelty**, not effort: **as an algorithm it is solved** (prior art at scale); **as a design variable it is open** (the same cut compiles server-side and to WASM, so placement is the question).

| Piece | Prior art |
| --- | --- |
| Frustum culling | Umbra, every engine, three.js/Cesium math |
| Hierarchical LOD under an SSE/projected-size budget | **3D Tiles/CesiumJS** (`skipLevelOfDetail`), **Potree**, OSG/osgEarth `PagedLOD`, UE5 World Partition/HLOD/Nanite |
| Server decides what each client is sent | SpatialOS, Photon, Mirror, FishNet, Unity NGO |

"Solved" does not mean unimportant: exact, deterministic, hysteretic, oracle-comparable selection is hard — but **hard-to-get-right ≠ hard-to-invent**. Not covered: **exact live aggregation under the cut** and the **placement characterisation**.

## 23. Reusable libraries for the cut

The survey is §18. Added here: **lock-in and visible-set exposure** — CesiumJS/`loaders.gl`/Potree/OSG/deck.gl expose a visible set; Babylon partially; three.js not at all. Each cuts **its own** format, forcing a manifest mapping; **none aggregates**.

**Recommendation:** one **shared crate** (native + WASM) for control; Cesium/`loaders.gl` as cross-check and related-work anchor; reuse only the math (`glam`/`nalgebra`, `Frustum`/`Box3`).

## 24. Tiles, visibility, and the data seam

A **tile** is a **spatial streaming node**: bounding volume + `geometricError` + children (`REPLACE`/`ADD`) + optional content + transform + (1.1) metadata. It is author-defined and **not a semantic object** — a tiler merges, splits, and re-levels geometry, so a rack may be merged, split, or present at several LODs. It is "a rack" only if authored 1:1.

Cesium's screen-space error, `sse = geometricError · H / (distance · 2 · tan(fovY/2))`, is proportional to projected size, i.e. to `Δy / z`. That is *why* the cut is commodity.

**Rack visibility:** yes if rack = tile (Cesium `tileVisible`, `loaders.gl` selected tiles, Potree `visibleNodes`, Babylon `getActiveMeshes`, deck.gl events, OSG cull visitor; three.js excepted).

**Subscribe the drawn rack to a value?** **No — not from the tiles library.** 3D Tiles has no pub/sub topics and no live per-object values; metadata is static. The tile layer answers "what is drawn"; the value/attention layer is yours. Colouring is either (a) a per-feature property + `Cesium3DTileStyle` (feasible, off the intended path, scaling with features/tiles), or (b) **separate** — stream values and render your own markers/instances.

## 25. Client-cut mechanised with a tiles library

Flow: resolve the manifest (§4.6) → **author a tileset** from it (one tile per node, `boundingVolume.box`, `geometricError`, `refine: "REPLACE"`, `node_index` metadata; a build/boot step) → library traverses and selects the frontier → query visible tiles → `node_index`.

**Corrections:**

- **The library's cut ≠ the student's cut.** It refines by SSE — no `B`, no budget hysteresis `Δ`, no blended entries, no oracle parity. Fine only if the client arm is defined as "the standard client-side cut."
- **Renderer vs traversal-only** must be chosen deliberately.
- **Do not upload visibility every frame** — that recreates the feedback loop (§4.4). Send subscription deltas on change, rate-limited and hysteretic (§16).
- **Still missing:** values and identity/metric metadata. A tileset gives **lazy geometry** (A′-like), but **not** lazy metric/channel definitions — those remain O(N).

## 26. Pros and cons: library client-cut vs server-side cut

| Dimension | Library client-cut (regime 2/B) | Server cut (R3) |
| --- | --- | --- |
| Cut implementation | Solved; author tileset + traversal — low effort | Index + frontier + cache + oracle — high effort |
| Cut semantics | No `B`, `Δ`, blend entries, oracle parity | Full control |
| Egress | Unchanged by culling; broadcast = full egress | Bounded, hard per-client byte bound |
| Server CPU/viewer | ≈ 0 (fanout) → scales to many V | `T_cut+T_encode` → `V_cpu ≈ 74–570`, CPU-bound |
| Client CPU/mem | Geometry lazy; values + O(N) metadata unless A′ | Thin client; O(B) geometry if lazy manifest |
| Latency | No camera round trip | Camera loop |
| Thin clients | Bad | **Good** |
| Deployment fit | LAN / capable clients | **Cloud/WAN, metered/constrained** |
| Novelty | None on the cut | Only aggregation + placement |
| Fit to PRD/TDD | Inverts the premise | Native |
| Build risk | Low | High |
| Correctness | Not comparable to oracle | Oracle, bit-exact |

**Four inversions:** build cost ↔ control; server CPU ↔ egress; deployment decides; novelty disappears if the cut is the claim.

**Two hidden costs:** it **drags aggregation to the client** (or duplicates aggregate + raw), and `B` is a **product requirement, not an implementation detail** — without it G2/K3 vanish.

## 27. Adding pub/sub to the client-cut

Pub/sub removes the client-cut's **worst** weakness (broadcast egress), but turns on **granularity** (§16):

| | Fine (frontier) | Coarse (region) |
| --- | --- | --- |
| Server sends | frontier values | **raw device state** |
| Egress | **≈ R3** | **> R3** when zoomed out |
| Consequence | R2 ≡ *R3 with the cut moved to the client* | the distinct third architecture |

- **Improves:** egress (to ≈ R3), CPU (routing ~0.003 vs cut ~0.025 core/viewer), cloud fit (R2 cheapest, §15).
- **Costs:** aggregation moves client-side (or duplicates); the **feedback channel returns** (rate-limit + hysteresis, or R7 moves to the data channel); binds on **α** (as α→1, egress → broadcast); client holds subscription state; per-frame consistency is harder.
- **Consequence:** fine pub/sub differs from R3 **only in where the cut runs** — the cleanest experimental control, not an alternative.

## 28. Attention rendering in the server-cut design

1. **Server computes** one `u8` per node in the same bottom-up pass — shared across viewers (FR-AG-06, §8.1, Decision 6).
2. **Streams on change:** flag `bit2 attention-present` + trailing `[u8]` (§10.4, Decision 15); a per-session baseline suppresses unchanged values.
3. **Client maps to colour:** per-node attention map; each frame writes `palette[level]` into the colour VBO (§11.1/§11.4, Decision 17). Default display = attention; metric/channel switching is client-only.
4. **Attention vs availability are orthogonal** (severity vs NaN/offline markers).
5. **Cost:** compare-and-selects folded into the existing pass; O(entries) colour writes; one `u8` per changed entry (inside K5).

With a static tileset the attention value cannot ride the tile — it must be overlaid. The server cut is cleaner because values and selection share a frame.

## 29. Client instance mapping (node index → instance slot)

The wire key is the node index; the `InstancedMesh` slot is **compacted** (≤ `B`), so the client keeps `nodeToSlot` built from the **`appeared`/`removed` diff** (§10.3, Decision 11):

- `appeared` → allocate slot; write the **static** transform from `manifest[node_index]`; write colour.
- `removed` → swap-remove; clear mapping.
- otherwise → look up slot; update colour only.
- `mesh.count = activeCount`; two meshes (device markers, group boxes); picking uses `slotToNode` (§11.3); rebuild on reconnect with the keyframe.

Transforms are static, so only colours change per frame; frontier order is unstable, so the diff — not position — defines the mapping.

## 30. Tileset authoring: limitation and performance

Mechanical but scale-sensitive:

| | Estimate |
| --- | --- |
| `tileset.json` per tile | ~400–600 B |
| Building tier (~4,460 tiles) | ~2 MB — fine |
| Full tier (~101,781 tiles) | ~40–60 MB; explicit (implicit tiling needs a regular octree) |

**Implications:** startup parse O(N); client memory O(N) for the tree (content lazy, tree not); traversal moves to the client; on-demand loading causes **pop-in**; steady-state value egress unchanged; no `B`/K3; not oracle-comparable.

**Alternative:** compile the **shared cut crate to WASM** — no authoring, no second artifact, identical semantics, O(B) memory. Use tiles only as a cross-check.

## 31. The viva: challenges and landings

| Challenge | Landing |
| --- | --- |
| "Why server-side cut? It reinvents wheels and bottlenecks CPU. Use client-cut." | Conceded the cut is solved and CPU is a real ceiling (`V_cpu ≈ 74–570`); defended only that client **culling does not reduce egress**. |
| "Client-cut need not broadcast; it can use pub/sub." | Conceded. Fine pub/sub → egress ≈ R3; coarse → > R3 and aggregation moves client-side; R2 binds on α. Retracted "only the server cut bounds egress." |
| "R1 can use a relay so co-located clients share egress." | Conceded — the **edge relay** (§17); one cloud stream divided by clustering factor. Adds client topology and a fourth architecture; client work unchanged. |
| "Name scenarios where R3 beats R2/R1." | Thin/embedded clients; scattered metered links; coarse-subscription overshoot; disclosure control; exact server-side aggregation; locked-down deployments. Conceded **R3 is a bounded niche**. |
| "Defend the topic if it doesn't beat R2." | Reframed: contribution = aggregation model + placement/coupling characterisation; cut = controlled variable. |
| "Is the problem worth study? Is the answer unclear?" | Partly. Direction guessable; crossover locations, the coupling, guarantee differences, and the counterintuitive result are not. |
| "Will you implement all four regimes + benchmarks?" | No. Two measured arms (R1, R3), relay process, R2 modelled with a spot-check, calibrated model + coarse grid. Full study = 40+ weeks; ~19 remain. |
| **Rejection:** too risky, uncompletable, doubtful value, ranking-only, implementation-dependent. | Conceded. Research value must rest on **invariant** results: the aggregation guarantees and the structural **coupling**. |

## 32. Rejection and proposed reposition

The session ended with the professor **rejecting the crossover proposal** as too risky, likely uncompletable, of doubtful value (a surface yielding only a ranking), and non-generalisable (implementation-dependent).

**Concession:** value cannot rest on measured constants. It must rest on what is **invariant**:

1. **The aggregation model** — exact rollups under bounded selection, with provable properties (mergeable accumulators, canonical reduction order, attention/absence semantics, per-frame snapshot consistency), validated by an oracle.
2. **The coupling result** — client aggregation needs the subtree, forcing broadcast/interest and selection client-side. Structural and implementation-independent.

**Proposed reposition:**

> A **hierarchical telemetry aggregation and bounded-delivery engine**: exact live rollups with severity and absence over a semantic tree, delivered per consumer as a bounded slice — evaluated on observability-style rollups and a constrained-link scenario, with the 3D twin as one demo client.

Main component: the **hierarchical semantic aggregation engine**; index, cut, transport, client, simulator, oracle are supporting. **Reusable** toward observability rollups, constrained-link telemetry, disclosure control, and a "worst-state-in-subtree" primitive. **Reuse broadens usefulness, not novelty**; the differentiator remains exactness/consistency under bounded selection. **Status: unresolved.**

## 33. Open questions carried forward

1. **Degree framing** (§19.1) — the gate.
2. **Approve the reposition?** — observability + constrained-link, or something narrower.
3. **The anchor application** — observability or constrained-link telemetry.
4. **The coupling result** — strong enough to carry a thesis; statable as a bound?
5. **Guarantee ordering** — do exactness/consistency flip the ranking independent of constants?
6. **Integrity** — no tuned parameters; decisions derivable cold.
---

# Part IV — Viva continuation, problem-first discipline, and the Google-Maps framing (fourth session)

Part IV records the fourth session (2026-10-15): the viva continuation, COMP4299's scope, re-skinning versus anchoring, the failing-baseline method, candidate pains, uplink/egress and edge/cloud, the field-technician scenario, and the Google-Maps framing.

## 34. The viva continuation: the defence narrows

The student conceded that the original proposal **"does not stand at the scenario it describes"** and that better solutions exist. The challenges and landings were those of §31, with no new concession: library cut, CPU ceiling, no requirement for server-side selection, fine pub/sub ≈ R3, edge relay, the thin/disclosure/metered niche, and the aggregation + coupling reframe. It ended in the same rejection and the same conclusion: value must rest on **invariant** results, not measured constants.

## 35. Programme scope: COMP4299 permits a system-development project

The module supports **either a system development project or a research-oriented project with a software component**, requiring students to "integrate and extend knowledge" and covering the full lifecycle — problem formulation, related work, methodology/design, development and verification, critical evaluation, risk management, communication, reflection, and societal/ethical implications. **All projects need substantial software.**

- **COMP4299 (BSc Computing):** a problem in a relevant computing area — this topic qualifies.
- **CSAI4299 (BSc AI):** needs a **clear AI focus** — this topic would not qualify without change.

The student is on **COMP4299**: **system-development is admissible and novelty is not required.** But "integrate **and extend**" rules out pure assembly, and the missing apparatus (related work, evaluation, reflection/ethics, a dissertation — not a "v1.0 launch") remains owed.

## 36. Re-skinning versus anchoring: the pattern named

The same engine was re-presented as a digital twin, generic IoT, AR glasses, a robot knowledge service, and spatial knowledge for robots. The supervisor named the **solution-looking-for-a-problem** pattern:

| | Domain-hunting | Re-anchoring |
| --- | --- | --- |
| Direction | design → domain | **problem → design** |
| Held fixed | the design | **the problem** |
| Variable | the label | **the design's scope** |
| Requirement from | invented to fit | **external** |
| Output | a demo/label | **a solution + a yardstick** |
| Test | "does the design fit?" | **"does it meet the need?"** |
| Falsifiable? | no | **yes — "wrong tool when…"** |
| Failure mode | contrivance | scope creep |

**Three tells:** (1) if a new domain changes nothing but the label, it is hunting; (2) hunting has a **demo**, anchoring has a **yardstick**; (3) anchoring lets you **abandon parts of the design**.

"Find a problem that needs real-time, massive EPS, and selection" was identified as **hunting** — those are the design's inputs, not a pain. A problem-first start begins from a pain and **derives** requirements. **Sourced constraints** (external facts) anchor; **provisional constraints** (hypotheses) must be tested. The sin is refusing to test them. A **minimal pain hypothesis** — one sentence about the world, no solution properties — is the correct start.

## 37. What "the cut is solved" means (restated), and where it is not

- **Hierarchical tile/LOD traversal — solved:** 3D Tiles/Cesium, Potree, OSG/osgEarth, interest management. Commodity and a **design variable**, not a contribution.
- **Per-instance selection — not provided by libraries.** Cesium `i3dm` gives **tile-level** visibility, not per-instance ids (batch-table properties can carry a node id for picking); three.js/Babylon `InstancedMesh` have no per-instance visible-list API; Potree exposes node-level visibility; deck.gl offers picking, not a visible list. **Consequence:** compute the cut yourself (frustum tests over your boxes — cheap) and maintain `node↔instance` for colouring and picking.
- **Per-node tiles** (one Cesium tile per node, no instancing) give ids for free but are **impractical at scale** (≈100k tiles, huge JSON, O(N) memory/traversal), give **no control** over `B`/hysteresis/determinism/oracle, and are **client-side only** — ruining server-side visibility for bounded delivery.
- **Conclusion:** build the selection; use Cesium/three.js purely as the **renderer** (`i3dm`/`InstancedMesh`). Regime choice becomes **placement**.

## 38. Transport and metadata mechanisms

- **Cesium picking:** `scene.pick` returns a `Cesium3DTileFeature`; store node identity as a **batch-table property**. A blended rack yields the **group node's** index. The client raycasts, maps `instanceId → slot → node index`, and publishes the inspection target (FR-VS-02).
- **Lazy manifest:** necessary, not novel. Shipping descriptors only for seen nodes makes client memory **O(B)** (≈40 MB at 100k otherwise). In the **server-cut** regime it lazifies the *dictionary*; the **client-cut** regime also needs a **coarse culling structure**.
- **Whole-tileset reuse:** good **across clients** (static geometry is cacheable); inefficient as **multiple placements** in one scene (each copy traverses/caches separately — use `i3dm`). Reuse covers **geometry only**; values, attention, and mapping remain the engine's.

## 39. Exactness is necessary but not scarce

Exactness (correct, and bit-reproducible against a reference) matters for trust, cross-consumer consistency, composability, delta stability, and verifiability. But **SQL** aggregates are exact (not plan-independent); **IVM** (Differential Dataflow, DBSP, Materialize, RisingWave) is exact and incremental; **CRDTs** converge exactly; **stream processors** compute exact operators (modulo FP order).

**Exactness alone is commodity.** It needs mergeable reductions/accumulators, a canonical reduction order, availability semantics, a per-frame snapshot, and an oracle — known disciplines. The genuine difficulties are **bit-exactness under parallelism/SIMD**, **exactness under partial availability**, and **exactness coupled to bounded, consistent, per-consumer selection**. The last is the residue.

## 40. Competitors, and the glue question

Closest threats: **IVM engines** for aggregation; **interest management + broker** for delivery; **integrated twin platforms** (Azure Digital Twins, AWS TwinMaker, Eclipse Ditto, NGSI-LD) for product. The hypothesised gap is the bottom row: **exact hierarchical rollup + consistent snapshot + per-consumer selection + hard bound + constrained delivery, coupled.**

**Will it fall to glue?** Iff the naive composition satisfies the requirement. The test is empirical: **wire the components the obvious way and see whether the coupling fails.** It likely does — event-time windows vs per-frame snapshots, relevance vs exactness, routing vs bounding conflict — so reconciliation is engineering. **Defeat the criticism by building the baseline and showing the failure.**

**Baselines are cheap and not a second system:** modes of the engine (broadcast, static/manual selection, no bound, no snapshot) and one configured off-the-shelf stack. The comparison is the *evaluation*; system + coupling is the *contribution*. Alternatives must be **best-of-breed**, and building them must not crowd out the system.

## 41. No access: use public anchors

The student has no industry/lab/dataset access. **External anchors remain:**

| Kind | Available? | Gives |
| --- | --- | --- |
| User/site/contact | No | lived problem, validation |
| **Standards** (OPC-UA, ISA-95, LoRaWAN, IEC 61850, C37.118) | Yes | citable requirements |
| **Literature** | Yes | anchors, related work |
| **Public datasets/benchmarks** | Yes | external validity |
| **Simulators** (Gazebo, `netem`) | Yes | a testbed |

Without a user, the structure is **a cited requirement → a best-of-breed baseline that fails it → the composition → an evaluation** — a legitimate systems/benchmark shape. The burden shifts onto **citation** and **baseline**; resist fabricating a scenario.

## 42. Candidate pains and domains

Telemetry is **not** exhausted: cardinality explosion/cost, anomaly-preserving reduction, energy/carbon, late/out-of-order correctness, cost-optimal placement, semantic interoperability, and root-cause across hierarchies remain open. **But the unsolved pains mostly do not fit the engine; the pain that fits is largely solved by IVM/brokers.** "Everything feels solved" because the domain was judged through the engine's lens.

| # | Pain | Anchor | Fit |
| --- | --- | --- | --- |
| 1 | Substation link cannot carry full PMU telemetry, but operators need wide-area aggregates | IEEE C37.118 | strong |
| 2 | Remote industrial sites over LPWAN/satellite cannot carry the tag stream | LoRaWAN 0.3–50 kbps; NB-IoT ~100 kbps; satellite cost | **strongest, lowest effort** |
| 3 | Vessels/aircraft report over metered satellite | Inmarsat/Iridium cost | strong |
| 4 | High-cardinality observability: costly, stale, unbounded per-tenant delivery | cardinality/cost literature | strong on scale, weak on physics |
| 5 | Renewable-farm SCADA over constrained comms | IEC 61400-25 | moderate |
| 6 | Water/wastewater remote sites | LPWAN limits | low (same shape as #2) |

**#2** was lowest effort: physical citable constraint, minimal engine change (keep state, hierarchy, aggregation, selection, transport, simulator, oracle; cut real-time and LOD), synthetic data, straight byte-budget evaluation.

## 43. Uplink vs egress; edge vs cloud; the relay

- **"Remote industrial" ≠ LPWAN.** LPWAN, satellite, cellular, private radio, and leased line impose different constraints; the pain exists only if the chosen link is **constrained**.
- **The cloud relocates the constraint.** The site→cloud **uplink** remains the site's link; cloud→consumer **egress** may be constrained or **metered**. The bottleneck decides placement: **uplink-constrained → engine at the edge**; **egress-constrained → engine in the cloud**; **neither → no pain**.
- **Selection does not help the uplink** — it carries the *union* of interests; per-consumer selection reduces only the **downlink/egress**. A gateway aggregate-and-forward suffices upstream and is commodity.
- **The office relay** amortises **cloud egress** for **clustered** consumers. It fails for **scattered** consumers, **thin** consumers (they still receive the full stream), and **heterogeneous** interests (the union). The pain needs consumers that are **scattered**, **thin**, or **heterogeneous**.
- **"Local aggregate + upload"** fails on consumer-side requirements: bounded per-consumer delivery, cross-consumer consistency, exactness under partial availability, mixed freshness, dynamic selection under a bound, severity propagation, fleet-wide consistency — i.e., the coupling.

## 44. The field-technician scenario

A technician uses a tablet to see telemetry **near the point of issue**, to grasp cause and the effect of a fix. The twin gives location, status by severity, adjacent equipment and zone aggregates, readings/history, fault extent, and safety context. Position drives a **bounded, relevant slice** — the engine's shape.

Two corrections: the need is **deliberative, not real-time** (safety-critical reaction comes from onboard sensors; the twin supplies seconds-scale context); usage is **intermittent**, not continuous, so "many viewers at 60 Hz" still fails. Requirements graded: **position-dependent selection holds**; **aggregation holds**, but **exactness is weakly motivated** for human diagnosis; "real-time" is over-claimed — **responsive** (sub-second to ~1 s) is honest. One consumer is a weak bound pain, and alternatives are untested.

## 45. The Google-Maps analogy

Google Maps is a **tiled, view-dependent, lazily streamed, edge-cached, bounded** system for **static substrate + a live layer**. Selection is **client-side** (viewport); object properties ride **vector-tile features**; live traffic is a **separate channel joined by id**.

| | Google Maps | The engine |
| --- | --- | --- |
| Substrate | map tiles (geometry + identity) | manifest/tiles (geometry + node id) |
| Live layer | traffic (joined by id) | value stream (joined by id) |
| Selection locus | **client** | **server** (for bounded/per-consumer delivery) |
| Personalization | **shared** | **per-consumer slice** |
| Aggregation | pre-aggregated, approximate, **shared** | **exact, hierarchical, per-consumer, consistent** |

Maps can select client-side **because content is shared**; **personalised content forces server-side selection**. The skeleton (substrate + live layer + join + view-dependent selection + LOD + lazy + bounded) is proven; the residue — **personalised, exact, per-consumer, consistent hierarchical rollup** — is not.

## 46. Working framing at the end of the session

> **A Google-Maps-shaped service for live telemetry**: static substrate (geometry + node id) plus a live value channel joined by id, **view/position-dependent**, **lazily streamed**, delivered as a **bounded, exact, consistent, per-consumer slice** — the streaming half solved, the **selection↔aggregation coupling** the contribution.

As a COMP4299 project: reuse the engine (state, hierarchy, aggregation, selection, transport, simulator, oracle), cut real-time/LOD/3D where the pain does not demand them, **build the selection** (cheap; control, placement, determinism), use a library renderer for drawing only, and **prove the coupling with baselines** (broadcast, static selection, relay, naive IVM+broker) and measured failures.

**Status:** a working framing, not an approved problem. Gates: a **sourced pain/requirement** (#2 leads) and closing the open decisions in time.

## 47. Open questions carried forward (Part IV)

1. **The sourced requirement** — which pain, from which citable anchor (#2 leads)?
2. **The constraint** — **uplink** (aggregation suffices; no project) or **egress/consumer-side** (the coupling is the project)? Must be the latter.
3. **Consumer topology** — clustered, scattered, thin, or heterogeneous? The pain lives only in the latter three.
4. **Placement** — edge or cloud, driven by the bottleneck link.
5. **The failing baseline** — does IVM + broker fail the coupling? Run it first.
6. **One extension, one evaluation, one consumer class** — scope that fits the time.
7. **Missing apparatus** — related work, evaluation, reflection/ethics, dissertation, resynced PLN.
8. **Integrity** — decisions cold; no tuned parameters.
---

# Part V — Candidate testing, the supervisor's challenge, and the mechanics of the cut (fifth session)

Part V records the fifth session (2026-10-16): the claims tested, the library audit, the pivots, the supervisor's challenges, the recalibrated bar, feasibility, and the cut/delivery mechanics.

## 48. Orientation

Reopened from the handover and the gates of §47. Method: **problem-first**; no re-skinning; test every claim against the world.

## 49. The candidate, stated

**A hierarchical telemetry aggregation-and-delivery engine serving each consumer a bounded, exact, consistent slice, with the selection↔aggregation coupling as the contribution** (§46). The problem statement: live systems produce more data than any viewer can examine; consumers care about different parts, often over constrained/metered links; broadcast exceeds the link, while a shared aggregate discards relevance and exactness; the need is a per-consumer, current, correct, consistent view under a fixed budget, while accounting exactly for what is not viewed.

## 50. The three load-bearing claims, tested

| Claim | Verdict |
| --- | --- |
| **1. The constrained link is the *delivery* link** | **Only in a specific topology.** The canonical LPWAN/satellite constraint is the **uplink**, which aggregate-and-forward solves. It becomes a delivery constraint only when consumers and data share the link (fleet/maritime, isolated site). Candidate #2 fails it by default. |
| **2. Consumers are numerous/heterogeneous/scattered** | **The weakest.** Human consumers of an industrial system are **few** (10 operators, one technician). It holds only for query/tenant/dashboard/subscriber at platform scale. |
| **3. Exactness/consistency is required** | **Fails for viewing.** Situational awareness runs approximate and stale. Genuine exactness lives in **control**, **billing/audit**, and **disclosure** — none is live viewing. |

**Joint finding:** the claims live in **three different worlds** — remote/fleet, platform/observability, control/accounting — and their intersection is thin and deliberately constructed, the signature of a problem assembled around a solution (§36). **Claims 2 and 3 fail naturally; claim 1 holds only in an unchosen topology.**

## 51. Do we need the aggregation? (the deduction)

| Consumers | Exactness required? | Outcome |
| --- | --- | --- |
| Homogeneous | No | One shared summary → broadcast + rollup + cache |
| Heterogeneous | No | Personalised approximate slices → caching/CDN (solved) |
| Heterogeneous | **Yes** | **The coupling — the only cell that is a project** |

**Aggregation alone collapses into a commodity** (SQL, IVM, streams; §39); it is a contribution only when forced to be **exact and per-consumer under a bound at once**. **Claims 2 and 3 are the problem.** Without them the system is dashboards behind a CDN — solved.

## 52. Google Maps tile selection, and where the analogy breaks

The client projects viewport corners through Web Mercator into world pixels at zoom `z`, converts to `tileX = floor(worldX/256)`, `tileY = floor(worldY/256)`, and fetches the uncached rectangle plus a prefetch ring. The server is a **dumb CDN cache keyed by `(z,x,y)`**; it never sees the camera.

**The analogy breaks on two properties:** tiles are **shared** (byte-identical, hence cacheable and computed once) and **attribute-carrying** (vector tiles style locally). The engine's layer is **live and personalised**, so there is no shared key and nothing the client can verify alone. Maps proves the skeleton only for shared, view-independent content.

## 53. Everything is pub/sub; topics altered by selection

The design is **pub/sub whose topic set is altered by the selection** (the frontier becomes the subscription set). Crisp statement: topic namespace = node tree; frontier = subscription set; each topic's payload = the exact rollup of the subtree it stands for; grain of publication = grain of selection.

But **who alters the topics decides the regime**: client-altered = fine pub/sub = R3 with the cut moved; server-altered = R3 plus a broker. **No third option.** Dynamic topics delete the Maps free lunch (sets are no longer shared) and impose churn: a camera-tracking topic set is a per-frame feedback channel, needing coalescing + hysteresis (R7 on the data channel, §16). It renames the mechanism; it does not show the requirement is needed (§36).

## 54. The lazy manifest reconsidered (A′) and the analytic/irregular distinction

- **Without A′, the server-side arm has no client-side advantage** over broadcast (client O(N) regardless). If server placement is argued at all, **A′ is mandatory**.
- **The structural distinction:** client-side selection is cheap only for **regularly addressable** spaces — Maps from a closed-form projection (O(1)), 3D Tiles from a regular quadtree/octree (**implicit tiling**). The semantic tree is **irregular**, so it ships explicitly (40–60 MB at 100k). **This is why the Maps skeleton does not transfer.**
- **Candidate invariant:** for an irregular tree of N nodes selecting B, one cannot have (i) **O(B) client memory**, (ii) **purely client-computed selection**, and (iii) **no extra round trips**. It is a **trade** among memory, locus, and latency — not an impossibility (a coarse top structure plus lazy descent approaches O(B) at round-trip cost).
- **Necessary, not novel** (ordinary on-demand loading; 3D Tiles external tilesets already lazy-load the tree), so a buildable **extension**, not a contribution. It supplies a **better justification axis than exactness**: **bounded client memory/CPU** — the thin-client niche.

## 55. Consumer topology: single roaming consumer

A **single consumer whose position varies** is not "all consumers sharing the cut." Sharing means **concurrent coincidence**; one consumer over time is **temporal variation**, with no fan-out and only revisit-cache. **V = 1 kills the CPU-scaling argument** (§12) while keeping the view-dependence that client-side lazy loading handles naturally — so a single, intermittently-moving consumer **tilts toward the client**. The O(B) bound needs an **eviction policy**; over a long session the working set drifts toward O(N), and re-entry costs a fetch.

## 56. The tight view: the bound never binds

A **5 × 10 m room** within a **100 × 200 m site** (20,000 m²) is **0.25%** of the area → ~**25 devices** at 10k uniform devices, versus `B ≈ 2,000` — **two orders of magnitude under**. The whole mechanism is unexercised, the criticism that sank the original demo.

- **`B` is a cap, not an every-frame mechanism.** It binds **iff the view's device count exceeds `B`** — a **wide/dense** view. At 10k and B = 2,000, that is **more than ~1/5 of the site**.
- **The coupling presupposes the bound binds:** no bound → no selection → no "rollup of the rest" → no coupling. The contribution lives only in a **middle band** — dense, per-consumer, constrained. Tight views (just send it) and shared wide views (same aggregate for all) are commodity.
- `B` is **perceptual**, not physics, so an examiner can argue it away by tolerating clutter.

## 57. Device density calibration

N = 10,000 on 2 ha is **0.5 devices/m²** — **dense**, an upper-end instrument-heavy figure, not a site average. As *tags* it is plausible; as *physical devices* it is high.

| Facility (2 ha) | Points/tags | Rough devices |
| --- | --- | --- |
| Building BMS/BAS | ~1k–10k | hundreds–~2k |
| Mid-size plant / unit | ~10k–50k | a few hundred–few thousand |
| Large refinery / power plant | 100k–1M | tens of thousands |

A plausible average is **~1,000 devices** (0.05/m²). The binding quantity is **budget vs on-screen density**; the mechanism lives at **zoom-out**, always. N must be externally anchored (§13).

## 58. The server-room case

~**40 racks × 20–30 servers ≈ 1,200 servers** (~1,240 nodes). Servers per rack: ~20–30 for 1U, ~10–20 for 2U, ~64–100+ for blades — **tens**, not hundreds.

- **Device or group?** A server is a telemetry source either way (leaf with metrics, or group with sensor leaves).
- **The count is low:** at `B ≈ 2,000` the **bound may not bind** — the same trap; a data centre is *smaller* than the industrial case.
- **The tree is strongly spatial** (rows → racks → U), so it nests cleanly.
- Per-server metrics inflate **channels per entry, not entries per consumer** — they do not make the bound bind, and they push toward **time-series monitoring** (Grafana/DCIM), a crowded field.

## 59. The bound and EPS

A site easily exceeds **100k EPS** (10k servers × 10 metrics at 1 Hz). But **EPS is the wrong axis**: it is the **source rate** and stresses **ingest/aggregation** (commodity, §39), not the **selection/delivery bound** (on *entries*). High EPS tilts toward bounding but does not make the coupling mandatory. **"100k devices" ≠ "100k EPS".**

## 60. Dropping the visualisation → dashboards

A dashboard/query spec replaces the camera — many concurrent, heterogeneous consumers (helping claim 2) — and sharpens claim 3 to "**consistent across consumers and drill-downs**."

| Pain | Real? | Anchor | Incumbent | Coupling survives? |
| --- | --- | --- | --- | --- |
| Cross-consumer disagreement | Yes | ops/SLO practice | none guarantees it | **Yes** |
| Drill-down vs overview mismatch | Yes | TSDB bucketing | continuous aggregates | **Yes if exact** |
| Unbounded per-tenant cost | Yes | SaaS scaling | query limits | partial |
| Absence / partial availability | Yes | Prometheus staleness | partial | partial |
| Cardinality explosion | Yes | cardinality literature | sketching/labels | weak |
| Rollup / retention cost | Yes | downsampling | TSDB | No |
| Dashboard latency | Yes | caching | cache/CDN | No |

The residue is rows 1–3 — **the coupling again** — all contested by strong incumbents (Grafana, Materialize, Timescale, Sparkplug B). Legitimate only if it **starts from one of these pains** with a specific complaint.

## 61. The smart-city pivot

Smart city has real problems (interoperability, privacy-preserving data, cross-agency consistency, entitlement-bounded viewing, twin fusion, edge latency, cost, data quality), but it is **more crowded than industrial** (FIWARE, NGSI-LD, oneM2M, OGC, funded platforms, DP research), the problems are **the wrong size/kind**, and the fit is **the same coupling**. **Entitlement/disclosure-bounded aggregate release** turns the bound into a **hard authorization bound** and supplies heterogeneity — but is partly commodity (ACL/filtering) and partly a research field (DP, contradicting exactness). It also **re-introduces spatial views**.

## 62. The abstraction test: "hierarchical knowledge management + selective publishing"

- "Hierarchical" and "selective publishing" describe the **substrate**.
- **"Knowledge management" over-reaches** — the engine holds numeric state in a tree and rolls it up; no semantics, inference, querying, or retrieval. Better: *hierarchical state aggregation with per-consumer bounded delivery.*
- **It names the two commodity halves and omits the coupling.** As a definition it describes a commodity composition. **Abstraction does not help grounding**; its legitimate use is **related-work positioning**.

## 63. Best-of-breed libraries and their universal gaps

- **Aggregation/IVM:** Materialize, Feldera/DBSP, RisingWave, Flink, Kafka Streams, Timescale, ClickHouse/Druid/Pinot, Redis.
- **Pub/sub:** NATS/JetStream, MQTT (EMQX/HiveMQ), Kafka, Pulsar, RabbitMQ, Centrifugo.
- **Selection/entitlement/twin:** Hasura, PostgREST+RLS, FIWARE Orion/NGSI-LD, Eclipse Ditto, Azure Digital Twins, AWS TwinMaker, ThingsBoard, Sparkplug B.

**Category gaps:** aggregation is **view-independent**; brokers are **compute-free**; query/entitlement layers are **unbounded and refetch-based**; twin platforms **route events, not rollups**; TSDBs are **bucketed/sampled**; renderers **own no values**.

**Cross-cutting gaps:** no per-consumer bound; no aggregation coupled to selection; no cross-consumer snapshot; hierarchy is *either* a namespace *or* a computation, never both; view-dependence exists only in rendering; cost scales with consumers, unbounded.

**Conclusion 1:** each single-category gap is a separation of concerns filled by composition; only the **coupling** has no combination that fills it.

**Conclusion 2 (caution):** a **universal gap is evidence of absent demand more often than unmet need** — in mature categories, "nobody built X" usually means it is assembled when needed, or nobody needs it. A universal gap is a negative observation about **supply**, not **demand**; distinguishing needed from unneeded requires a **published workaround or complaint** — none found yet.

## 64. Cross-consumer consistency: a constraint, not a pain

The shared per-frame snapshot (§28) is **real**, and better than change-based brokers (late-joiner divergence) and refetch layers (per-subscriber staleness). But:

- **It is an artefact of centralisation, not a technique.** Broadcast of a snapshot is also consistent; centralising costs the CPU ceiling.
- It is **fragile** (per-viewer deltas/baselines can break it), holds **only at frame boundaries**, and matters **only if consistency is required** (claim 3).
- **Consistency is a constraint, not a pain:** a pain is observable and citable. "Our dashboards disagree" resolves to **"use one source of truth"** — a data-governance fix, not a delivery-engine problem.

**This was the fourth claim in a row to fail** (link, consumers, exactness, consistency). The requirements are **self-generated**; claims that dissolve on contact with a user suggest they have no natural home.

## 65. Composition and the defensibility of a composition project

**Yes — with the right framing.** Composition is a *method*, not disqualifying; COMP4299 requires "integrate **and extend**", which composition-plus-reconciliation passes.

Defensible **iff**: the problem is real and cited; composition is the method, not the contribution; the parts conflict and the **seam is built**; there is a **baseline** and a **failing demonstration**; verification and critical evaluation exist. It fails if the problem is invented, composition alone suffices, the extension is configuration, or there is no evaluation. **The failing-baseline method is the machine**: compose → show the failure → build the reconciliation → evaluate.

## 66. The professor's offered topics (assessed, held as last resort)

Six topics: RFID attendance, smart mailbox, plant watering, smart parking, server-room monitoring, home-security motion. All are one template — `sensor → Pi/ESP32 → threshold → dashboard → Telegram/email` — with identical architecture and depth. At the COMP4299 bar **none is defensible as pitched** (assembly, no extension, no algorithm, no evaluation; blurbs self-describe as "introductions"). RFID additionally **resolves away its own hard parts** (it relocates proxying; MIFARE Classic is clonable; privacy persists). Best anchor: smart parking (~30% cruise-for-parking). **Last resort, not the objective.**

## 67. The bar recalibrated

The bar had been set too high on *sourcing*: it demanded a **sourced, externally-cited pain with an anchoring consumer** — a **research** standard. COMP4299 system-development needs only a **stated problem in a relevant computing area** plus novelty-free execution.

- **Relaxed:** a **stated engineering need with a justified scenario**, not a market complaint.
- **Not relaxed:** **extension beyond assembly**; **rigorous evaluation**; **integrity** — the scenario must not be tuned for the design (§13).

## 68. The engine reframed as a defensible system-development project

> **Design, build, and rigorously evaluate a hierarchical telemetry delivery engine that provides bounded, exact, consistent per-consumer slices — measured against best-of-breed compositions, with the selection↔aggregation coupling as the extension.**

A *passable, well-evaluated engineering project*, not novel — what §9/§35 permits. Residual risks: the **failing baseline** is an engineering test (if the composition passes, it is glue and the dissertation is thin); and **supervisor fit** (server-side streaming vs embedded/IoT).

## 69. Feasibility: 19 weeks × 16 h/week

**304 h**; dissertation ~70–100 h; **~200 h of code**. The minimal build (state + index; aggregation + oracle; bounded selection + snapshot; socket-free encoding; simulator; one composition baseline; evaluation) is **~225 h** — over budget before any visual demo.

- **Avoid the socket gap:** evaluate bytes/consistency/latency **socket-free** (serialise, count, model the ~52 B overhead). Sockets only for an **optional demo**, then **thread-per-connection**, not async.
- **Rust coverage is fine**; the constraint is **hours, not skills**.
- Fits **only** as a **minimal, socket-free, one-baseline** project with heavy reuse, no richness, one scale, and **continuous writing**.
- A visual demo, second composition, 100k, or async **pushes it over budget**.

## 70. The supervisor's challenges

- **Simulator realism** — the anti-tuning/grounding discipline (§13) in his words. One **reference scenario**, **cited** parameters, **sweeping** the answer-shaping ones, ideally a public dataset.
- **Frontend demo** — cheapest credible is a **2D Leaflet map** whose viewport drives selection (~20–30 h), not 3D/Cesium. Both add **~35–55 h**, taking the build to its edge.
- **"The engine is trivial"** — he independently reached §37.
- **"Frontend libraries can do the cutting"** — right for this spatial tree; the library traversal can serve as the semantic cut, so the cut is not the contribution.

## 71. The cut, concretely

- **The cut = culling *and* LOD** — visibility + resolution — producing a **bounded frontier under `B`** with mixed-depth entries. The engine's LOD is **aggregate/semantic**, distinct from geometric LOD.
- **i3dm** = Instanced 3D Model: one geometry + N transforms, **one draw call** (batching, not parallelism). It trades **per-instance visibility** for scalability: `tileVisible` fires **once per tile**, and all N instances are submitted when drawn (off-screen ones **clipped**, not culled).
- **Tile lifecycle:** `tileLoad` → `tileVisible` (**per tile per frame**) → `tileUnload`; accumulate a frame's firings, then **coalesce + hysteresis**.
- **A tile is a spatial streaming node** (bounding volume + `geometricError` + children + transform + optional content), **not an art asset**; content may be geometry, instances, points, or nothing.
- **External tilesets** let a tile's content be another `tileset.json` loaded lazily — so the **tree itself can lazy-load**. One listener on the root covers external tilesets (filter by subtree); separate tilesets have separate events.
- **Visibility exposure:** `tileVisible` is **per tile**; features can be enumerated, but there is **no per-feature visibility callback or visible list**. Per-feature visibility is user-computed.
- **Picking/colouring:** identity is a **batch-table property** (`node_index`); `scene.pick` → `Cesium3DTileFeature` → `node_index`; colour with `feature.color`.

## 72. Per-node coupling vs regular cells + own check

- **A — per-node coupled tile:** free ids, simple, fits the spatial tree — but O(N) tiles (~10,180 at 10k), no tile optimisation, poor occlusion, re-authoring, no bound or aggregation.
- **B — regular cells + own check:** efficient tiles, scalable, semantic-agnostic, **imposes the bound** — at the cost of writing traversal/policy (a few dozen lines; intersection math is library-provided: Cesium `CullingVolume`, three.js `Frustum`, `glam`/`parry3d`). Occlusion is the only part with no off-the-shelf answer (optional).

At server-room scale the performance difference is modest and concentrated in draw calls — not disqualifying. **The bound, aggregation, and maintenance decide.**

**Strong concession:** the semantic tree is **spatial by design**, so the tileset **can be coupled** and the library cut **delegated** — the "spatiality" objection fails. What remains: the **bound** (SSE ≠ `B`), **aggregation**, and **consistency**.

## 73. Snapshot vs delta: transport and the recommendation

For **10,000 × 3 = 30,000 values = 60 KB** (f16):

- **Full-site snapshot:** 60 KB/tick × V (one message → amortised headers).
- **Interest delta:** ≈ 6,000 values × `p` × 4 B = **24 KB × p** per tick.

**Batching is the answer.** ~**52 B** per message at IP (~70 B with Ethernet); one snapshot → ~**0.09%** header; per-value delta (4 B + 52 B) → **14× inflation**; batched delta → near-zero. Batched delta beats snapshot while `24p < 60` (essentially always); per-value only while `p < 0.18`. **Batching is a requirement, not an optimisation.**

**Aggregation is shared across viewers**, amortised to ~nothing; per-viewer costs are selection and delivery.

A stateful-twin server **also aggregates on the snapshot path**, so server aggregation is not unique to pub/sub. The differentiator is **delivery scope**: the snapshot leaves **nothing unseen** (client can recompute; "rollup of the unseen" is vacuous), while pub/sub **creates an unseen**. Snapshot may use client *or* server aggregation; pub/sub forces server-side aggregation.

**Recommendation: full-site snapshot** — 60 KB, trivial consistency, the client can cut and aggregate exactly (dissolving the coupling), no feedback loop, far less machinery. Delta/pub-sub wins only at **larger V × cadence** or **metered/constrained egress**. **At this scenario, the engine is not needed.**

## 74. Cadence and consumers

- **Cadence:** **1–10 Hz** for viewing (Prometheus 15 s; Grafana 1–5 s; SCADA 1–2 s; process 1–10 Hz; humans read at a few Hz). **60 Hz is an animation holdover**; cadence **is** the latency bound.
- **Consumers:** **10–100** (control room 5–20; plant tens; field fleet 10–50); above ~100, share/cache rather than compute per consumer.

At `B = 2,000`, 3 metrics, ~12 KB/tick, `T_cut ≈ 0.35 ms`: 2 Hz/100 ≈ **2.4 MB/s, ~7% of a core**; 10 Hz/1,000 ≈ 3.5 cores. **At reasonable values the cut and egress are trivial** — necessity must come from a **non-performance** constraint (cost cap, thin client, disclosure, consistency). The 60 Hz cadence is the conceded 3D premise.

## 75. The design-space + implementation framing (current candidate shape)

> **A design-space characterisation and reference implementation for bounded per-consumer delivery of live hierarchical telemetry** — criteria from external anchors, one implementation of the preferred combination, evaluation against best-of-breed baselines.

Defensible **only if**: criteria are **anchored**; the **requirement is real or stated and justified**; baselines are **best-of-breed**; scope is **one combination**. **Not** defensible as only a design-space report. Fits "integrate and extend" if the **coupling** supplies the extension.

## 76. Ingress scope

**Minimal ingress, not a subsystem.** Adapters, brokers, storage, retention are commodity and off-contribution. The needed ingress is the **grounded simulator** (optionally a thin MQTT/OPC-UA adapter); storage is off-the-shelf. Test every component: **does it carry the contribution? If not, use off-the-shelf or skip.** Thin ingress is the "integrate" half; the coupling is the "extend" half.

## 77. Open questions carried forward (Part V)

1. **The requirement** — is there a consumer for whom the **bounded, consistent, per-consumer** slice is **mandatory**? If not, §73 stands.
2. **Supervisor fit** — does he accept a student-proposed server-side topic and relax the "trivial/libraries" objection?
3. **The failing baseline** — wire `IVM + broker` and test whether the coupling fails.
4. **Scenario and scale** — industrial (~10k tags) vs server room (~1,200 servers, bound weak); make the bound bind or supply a non-performance constraint.
5. **Snapshot vs delta** — provisionally snapshot; delta only under metered egress or large V × cadence.
6. **Cadence and V** — provisionally **1–10 Hz** and **10–100**.
7. **Cut delegation** — couple the tileset to the spatial tree; residue = bound + aggregation + consistency.
8. **Scope** — 304 h, ~200 h code; minimal, socket-free, one baseline; Leaflet demo.
9. **Supervisor topics** — last resort; extend with depth, not as pitched.

## 78. Bottom line after Part V

- **All three load-bearing claims fail** naturally, and consistency is a constraint, not a pain — **four self-generated requirements** with **no natural home**.
- **But the bar was too high on sourcing.** For COMP4299 system-development a **stated requirement suffices**, and the engine is defensible **engineering** — coupling as extension, composition as baseline, rigorous evaluation as content.
- **The cut is conceded** (commodity; spatial tree allows delegation). The residue is **exactly the coupling**: bounded, exact, consistent, per-consumer delivery, where bound, aggregation of the unseen, and consistency are inseparable (§10).
- **At reasonable scale the problem is small** (1–10 Hz, V 10–100, ~60 KB): the **snapshot** suffices and the engine is unnecessary. Necessity must come from a **non-performance constraint**.
- **Feasibility is tight:** 304 h, ~200 h code, socket-free, one baseline, no richness; simulator + demo push it to the edge.
- **Provisional candidate:** a **systems study** — design-space characterisation + reference implementation, coupling as extension, snapshot-vs-interest as axis. **Not approved; requirement and supervisor acceptance are the gates.**
---

# Part VI — The engine's disposition and the search for a topic (sixth session)

Part VI records the sixth session (2026-10-16, continuing): the engine's status, six papers and one repository, the recommendation, and the search for a system-development topic.

## 79. The engine's status review

- **All four load-bearing claims fail** naturally (§50, §64): the constrained link is the uplink; consumers are few; exactness is not required for viewing; consistency is a constraint, not a pain.
- **Every component is commodity** (§39, §63): tree, topology, aggregation, broker, cut, transport.
- **The supervisor independently** calls it trivial (§70).
- **The build is heavy:** ~225 h of code + ~90 h dissertation against 304 h (§69).

**Characterisation:** **trivial in novelty, substantial in engineering** — no hard algorithm to invent; the work is volume, correctness, consistency. Admissible for system-development (novelty not required), but the burden shifts to the **evaluation** (§65): if the components are standard, the build is not the contribution — the measurement is.

## 80. What novelty is (and is not) open

| Open seam | Nature | Feasible in ~19 weeks? |
| --- | --- | --- |
| **Coupling** (selection ⊗ aggregation) | characterisation (invariant) | ✅ cheap |
| **Consistency vs bounded delivery** | characterisation | ✅ cheap |
| **Memory ↔ latency ↔ locus** for irregular hierarchies (§54) | characterisation (trilemma) | ✅ cheap |
| **Absence / partial availability** | technique + semantics | ✅ moderate |
| **Frontier stability** (hysteresis) | technique + evaluation | ✅ moderate |
| **Incremental maintenance of only frontier rollups** | algorithm | ⚠️ risky |
| **Overlapping-hierarchy aggregation** | algorithm | ⚠️ risky |
| **Bit-exactness under SIMD** | technique | ⚠️ known-ish |
| **DP + exactness** | research | ❌ too big |
| **Cost-optimal placement** (crossover) | study | ❌ rejected (§31) |

**Recommendation:** **characterisations first** (cheap, invariant — survives the "implementation-dependent" objection, §32), plus **one small technique** (absence semantics) to show extension. **Caution:** novelty is not required; hunting for it is the design-first trap (§36).

## 81. The papers read (catalogue)

| Paper / artifact | Venue · domain | Addresses | Misses |
| --- | --- | --- | --- |
| **Decision-Making in Digital Twins** | OR/ML/control | optimisation, RL, risk-aware autonomy, human-in-the-loop | **delivery** (streaming assumed) |
| **Semantic Edge-Cloud Communication for Urban Traffic DTs** | networking · DT | **uplink** compression: task-relevant semantics, not raw video (99% saving; 0.12 m error) | egress, per-consumer |
| **TwinGuard-Lite** | **cs.CR** · HealthSec'26 | **state-admission gateway** (ingress integrity); GSC and CPN properties | egress delivery |
| **S²GS: Structured Sparse Gaussian Streaming** | **cs.CV** | sparse Gaussian residual selection over an octree for FVV streaming | delivery, exactness (photometric) |
| **Mixed-Reality Feedback Control** | **cs.RO** | fusing control signals across realities; clock sync and sample alignment | delivery, per-consumer |
| **Missing Dimensions: Human and Social Systems in DT Engineering** | **cs.SE** | DT dimensions (Engineered/Natural/Human/Social); four open challenges | delivery |
| **`dt-semantic-drift-management`** (repo) | MDE · semantic web | semantic/model drift across modelling layers (JSMF, Brick, SSM-OM, OAEI) | twin↔reality drift; delivery |

Each is a **neighbour**, not the seam.

## 82. The pattern: neighbours, not the seam

> **Every artifact touches a *piece* — hierarchy, selection, streaming, consistency, integration, fusion — and *none* names the *combination*: bounded, exact, consistent, per-consumer rollup delivery.**

Two readings, per §63: the gap is either genuinely open or genuinely not a felt problem. **No paper states it as a limitation**, which leans toward "not a named problem." The field's attention is **upstream** of the view. **Useful output:** a strong related-work chapter and the positioning sentence — *"the field optimises decisions, uplinks, ingress, rendering, and scope; bounded, exact, consistent per-consumer delivery of state is assumed, not solved."*

## 83. The engine's disposition

- **Is it a good topic?** Evidence is one-directional: **no sourced need** (seven artifacts), **commodity parts**, a **skeptical supervisor**, a **heavy build**.
- **Is building it worth it?** Admissibility does not depend on need (§67), but **effort is the binding constraint** and the payoff is a coin-flip (if the composition passes, the dissertation is a build report).

**Recommendation:** *unless "building the engine" is the goal*, **let it go** — most expensive, weakest motivation. If kept, shrink to **coupling + one baseline + evaluation** (§65, §68). **Sunk-cost caution:** 80% done is not a reason to continue, but the **analysis is not wasted** — coupling, commodity audit, failing-baseline method, and seven anchors transfer.

## 84. Finding a topic: the module's criteria

> *"students are responsible for identifying a problem, analysing stakeholder needs, designing an effective solution using appropriate technologies and tools, and ensuring system quality through verification processes."*

Every candidate needs **problem + stakeholder + buildable solution + verification**. **The engine had solution and verification but no problem and no stakeholder.** The operative question is **"who is the stakeholder, and can you reach them?"**

## 85. Where to look: sources and methods

**Sources (nearest first):** your environment (department, labs, societies, IT); local organisations (charities, clinics, small businesses, schools); public sector/open data; **standards and compliance** (mandated requirement); **open-source issue trackers**; the supervisor's domain; your own workflow.

**Methods:** observation (manual, repetitive, error-prone steps — "where there's a spreadsheet and a copy-paste, there's a project"); interviews ("what's the most annoying part of your week?"); complaint mining; gap analysis (process vs standard); 5 whys.

**Fifth criterion:** the pain must be **reachable and observable** — *a citable standard is good; a citable standard plus a person you can interview is a project.*

## 86. The "Missing Dimensions" paper: the four dimensions and the strongest candidate

The paper's Table 1 (Smart Building) gives **Engineered, Natural/Biological, Human, Social**, each with entities, sensors, models, services, actuation, challenges.

| Candidate (dimension) | Stakeholder | Buildable? | Verified? | Verdict |
| --- | --- | --- | --- | --- |
| **Neurodivergent sensory accessibility** (Human) | **occupants + accessibility office** | ✅ software | ✅ user testing | **strongest** |
| Personalised comfort (Human) | occupants, FM | ✅ | ✅ face validity | strong |
| Space allocation (Social) | facilities | ✅ | ✅ utilisation | medium — planning |
| Crowd flow (Social) | facilities | ⚠️ infra | ✅ | weak |
| Air quality / daylight (Natural) | occupants | ⚠️ hardware | ✅ | weak |
| Predictive maintenance (Engineered) | FM | ⚠️ access | ✅ | weak |

**Why Human dominates:** **named, reachable stakeholders**; a **mandated requirement** (accessibility law); a **software** solution (profiles, recommendations, accessible routing, notifications); **measurable verification**; and **ethics built in** (privacy, sensitive disability data, agency and consent). Candidate:

> **A decision-support system helping a building adapt to (or advise) neurodivergent / sensory-sensitive occupants** — real users, lawful requirement, software-only, measured evaluation; ethics approval is the hurdle.

## 87. The four research challenges, filtered

The paper's §5 names four open challenges. Filtered on problem + stakeholder + buildable + verifiable:

| # | Challenge | Nature | Buildable / verifiable | Verdict |
| --- | --- | --- | --- | --- |
| 1 | Cross-dimensional collaboration & **tooling** (DSLs) | tools/SE | ✅ / ⚠️ experts unreachable | plausible; hard stakeholder |
| 2 | Multi-dimensional **reference architecture** | architecture | ⚠️ / ✅ locality of change | large & vague |
| 3 | **Heterogeneous data & model integration** | data/systems | ✅ / ✅ consistency, provenance | **strong** |
| 4 | **DT drift detection & management** | monitoring/detection | ✅ / ✅ inject, measure | **strongest** |

**Caveats:** research-shaped (need narrowing); stakeholders are abstract; **none is the engine**. #3 is consistency at the **source**; #4 is freshness/staleness at the **twin** — again, upstream of the view.

## 88. The DT-drift topic (#4), detailed

**Working title:** *A drift-aware digital twin for a building: detecting and managing divergence between the twin and reality.*

- **Problem (stakeholder's words):** *"we can't tell when the twin has quietly stopped matching the real building — and then we make decisions on wrong numbers."* Drift is **model**, **data**, or **scope** drift.
- **Stakeholder:** whoever **trusts the twin** — facility manager, lab operator, accessibility office.
- **Solution:** simulated twin (temperature/CO₂/humidity/occupancy; the simulator is central and satisfies §70); a **ground-truth reference**; a **statistical/sequential detector** (residual monitoring; PSI/KS; ADWIN/Page-Hinkley) with a threshold baseline; a **management loop** (alert → recalibrate → re-scope → fall back) with provenance; an evaluation harness.
- **Verification:** inject known drift types; measure **detection latency, precision/recall, false-alarm rate, recovery time** against the baseline (§40).
- **Ethics:** privacy, consent, fairness if human/social data is carried.
- **Feasibility:** ≈ **140–160 h** — half the engine (simulator ~40, detector ~20, management/provenance ~40, evaluation ~40).
- **Cautions:** frame as a **system, not ML**; do not imitate the repository (§90), which is model drift across modelling layers.

## 89. Drift detection is not inherently ML

| Sub-problem | Methods | ML? |
| --- | --- | --- |
| **Data drift** (covariate shift) | KS, PSI, χ², MMD | No |
| **Concept/model drift** | ADWIN, DDM, Page-Hinkley, residual monitoring | No |
| **Scope drift** | rules + anomaly heuristics | No |
| **Diagnosis/attribution** | sometimes ML | *maybe* |
| **Management/remediation** | systems (a control loop) | No |

The classic concept-drift literature is largely **statistical**; the **management half is pure systems** and is the paper's **under-studied** half (*"detection must trigger appropriate DT evolution"*). Frame it as a **monitoring + management system** with a statistical detector.

## 90. The repository (`dt-semantic-drift-management`)

A research artifact (arXiv 2512.15281) for **semantic drift management in a multi-layered, model-driven DT** — identification → evaluation → propagation; air-quality use case; layers = data / model-metamodel (JSMF) / ontology (Brick via SSM-OM); OAEI benchmarks, ablations, interoperability.

**Its "semantic drift" ≠ the paper's "DT drift":** it is **model/metamodel/ontology drift against data** ("the data outgrows the model"), detected by model alignment/ontology matching — MDE + semantic web, much bigger than a 150 h project. It **occupies a variant** of the space, so "semantic drift management" would duplicate it. **Use it as** a related-work anchor, a scope warning, and a source of a concrete open dataset (room air quality).

## 91. Open questions carried forward (Part VI)

1. **Engine disposition** — drop it or shrink it to coupling + baseline + evaluation (§83). Unresolved; recommendation leans to drop.
2. **Supervisor conversation** — what bar, and whether he accepts a student-proposed topic; bring candidates.
3. **Candidates** — drift-aware twin (§88); accessibility (§86); extend a supervisor topic (§66, §84).
4. **The stakeholder** — the missing asset: name one and reach one.
5. **The four-part filter** — problem + stakeholder + solution + verification (+ reachable/observable).
6. **Integrity** — anchor criteria; no tuned parameters; derive, don't fit.

## 92. Bottom line after Part VI

- **The engine is a solution without a problem** — trivial in novelty, heavy in build, unsourced in need — and six papers across six domains confirm the field's attention is elsewhere. **Drop it (or shrink it); the analysis becomes rationale and related work.**
- **The module's filter is better:** problem, stakeholders, buildable solution, verification — with the **stakeholder** as the asset the engine lacked.
- Strongest sources: **accessibility/neurodiversity** (named stakeholders, mandated requirement, software-only, measurable, ethics) and **DT drift** (#4 — smallest, measurable, simulator central); pragmatic route: **extend a supervisor topic** with a reachable stakeholder and real evaluation.
- **Drift detection is not inherently ML** — management is systems and is the actual gap.
- **Next step:** bring the supervisor **2–3 candidates** as *problem → stakeholder (how reached) → minimal solution → verification → ethics*, and talk to a stakeholder.
- **Status: engine provisionally abandoned; no new topic adopted. Gate: a sourced, reachable stakeholder.**
---

# Part VII — The drift topic's sufficiency, the "trivial" question, and device calibration (seventh session)

Part VII records the seventh session (2026-10-17): the drift topic under the same critique, the recurring "sounds trivial" reaction, the track decision, an IoT calibration survey, and explicit state/gates.

## 93. Orientation

The session re-applied the lens that killed the engine (§50, §64, §83). Two questions dominated: *"is drift just inject → detect?"* and *"does it sound trivial?"* A calibration survey followed.

## 94. The drift topic under critique: "inject → detect"?

**Yes, framed that way — and too thin.**

- **Injection is the harness, not the system** (like the simulator, §70).
- **A detector alone is commodity.** ADWIN, Page-Hinkley, PSI, KS, residual monitoring are simple to apply — the paper's starting point, not its gap. "We ran ADWIN" is the engine's failure mode: a commodity component dressed as a project.
- **The system must be the response:** a **closed loop** — detect → decide → act → verify — with the **management policy** (recalibrate, re-scope, fall back) and **provenance/attribution**. That is the paper's open gap: *"detection must trigger appropriate DT evolution."*
- **Evaluation must exceed detection:** recovery time, false-alarm rate, latency, and cost, over several drift types, against a baseline, grounded.
- **Validity risk:** injecting your own drift into a simulator detects planted drift and is coupled to the injector. Mitigate by injecting into a **real dataset** (the repository's air-quality data, §90).
- **Advantage over the engine:** the paper **names the management gap** — a cited open problem.
- **Framing that fixes it:** *"a closed loop that keeps the twin trustworthy: detects divergence, decides the response, executes, and verifies recovery — evaluated on recovery time, false alarms, and cost against a naive threshold."* Subject = the loop and policy; detector = component; injection = harness; evaluation = evidence.

**Test:** is the **management policy** a real problem or a line of code? If a competent engineer would just write "threshold + recalibrate", it is thin; if *when and how* to evolve without thrashing is non-obvious, it is a project.

## 95. "It sounds trivial": novelty vs the module's bar

| Considered | Verdict |
| --- | --- |
| The engine | commodity parts + thin residue |
| The supervisor's six topics | commodity parts (one template) |
| Dashboard / smart-city / robot pivots | commodity or crowded |
| The four DT challenges | research-shaped |
| The drift topic | commodity detector + thin residue |
| Every paper read | a neighbour, not a seam |

**Diagnosis:** a **research bar** was applied to a **system-development module** where **novelty is not required** (§35, §67). By the research bar almost every systems project is trivial; by the module's bar they are what is asked.

**The question changes:** not *"is it novel?"* but *"does it meet the four criteria — problem, stakeholder, solution, verification — and can it be executed and evaluated well?"* Drift is **not disqualified**: named open problem, potential stakeholder, buildable solution, measurable verification.

**Depth comes from:** a **real stakeholder** with a lived pain; a **rigorous, grounded evaluation**; a **genuine constraint** (hard bound, safety/ethics, scale); for *research* only, a real gap.

**Engine comparison:** it died not because trivial but because **trivial *and* unmotivated *and* heavy**. Drift is **trivial, better-motivated (cited gap), and lighter** — strictly better, if the stakeholder and evaluation arrive.

## 96. The track decision

- **System-development:** trivial-in-novelty is fine; depth from **stakeholder + evaluation**; drift, accessibility, and an extended supervisor topic are admissible.
- **Research:** novelty required; seven papers found no named seam; the crossover was rejected (§31); harder and riskier.

**Recorded:** quality is decided by the **stakeholder** and **evaluation**, not component novelty. Until the track is chosen, every candidate will "sound trivial" — the bar is unstated.

## 97. IoT device survey (calibration)

Indicative ranges (not sourced; anchor to a deployment before use):

| Scenario | Devices / points | Update rate | Metrics/device | EPS (order) | Link / protocol | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| Smart home | 10–100 | event + 1/min–1 Hz | 1–5 | 10¹–10² | Wi-Fi, Zigbee, Matter, BLE | privacy; low criticality |
| Commercial building (BMS) | 1k–50k points | 1–15 min (critical 1–60 s) | 1–3 | 10²–10³ | BACnet, Modbus, KNX | energy/HVAC |
| Industrial plant (SCADA/DCS) | 10k–1,000k **tags** | 0.1–10 Hz | 1–10 | 10⁴–10⁵ | OPC-UA, Modbus, PROFINET | safety; exactness |
| Manufacturing line (IIoT) | 10²–10⁴ sensors | 1–1,000 Hz | 1–10 | 10³–10⁵ | OPC-UA, TSN, Sparkplug B | vibration; maintenance |
| Smart grid — meters | 10⁵–10⁷ per utility | 15 min–1 h | 2–5 | 10²–10³ | DLMS/COSEM, PLC, cellular | **billing → exactness** |
| Smart grid — substation/PMU | 10–10² per substation | **30–120 Hz** (C37.118) | 10+ phasors | 10⁴–10⁵ | IEC 61850, DNP3, C37.118 | **hard latency** |
| Smart city — traffic | 10²–10⁴ | 1–30 Hz | 2–10 | 10²–10⁴ | cellular, fibre, LoRaWAN | latency-sensitive |
| Smart city — air quality | 10²–10³ stations | 1 min–1 h | 3–6 | 10⁰–10² | LoRaWAN, NB-IoT | calibration drift |
| Smart city — lighting/parking | 10³–10⁵ | event / daily | 1–3 | 10⁰–10² | LoRaWAN, NB-IoT | constrained; low rate |
| Agriculture | 10²–10³ per farm | 5–60 min | 2–5 | 10⁰–10¹ | **LoRaWAN, NB-IoT** | constrained; battery |
| Livestock / asset tracking | 10²–10⁵ | minutes–hours | 1–3 | 10⁰–10² | BLE, LoRaWAN | battery; geofencing |
| Fleet / logistics | 10²–10⁵ | 1–60 s | 5–20 | 10³–10⁵ | **cellular, satellite** | metered; intermittent |
| Maritime / aviation | 10²–10⁴ | 1 s–1 min | 5–20 | 10²–10⁴ | **satellite (metered)** | cost/latency |
| Healthcare — vitals | 10²–10⁴ per hospital | 1 s–1 min | 3–8 | 10²–10³ | Wi-Fi, BLE, FHIR | privacy; safety |
| Wearables | 1–10 per person | 1–100 Hz | 3–10 | 10⁰–10² per person | BLE, cellular | battery; high rate |
| Retail — ESL / shelves | 10³–10⁵ | event / daily | 1–3 | 10⁰–10² | BLE, Zigbee, Wi-Fi | huge count, low rate |
| Water / utilities | 10⁴–10⁶ meters | 15 min–1 day | 2–5 | 10¹–10³ | LoRaWAN, NB-IoT, PLC | constrained; billing |
| Data centre (DCIM) | 10³–10⁵ servers | 1–10 s | **10–100** | **10⁵–10⁶** | Redfish, IPMI, SNMP | high cardinality |

**Three observations:**

1. **"Big data" ≠ "many consumers."** High EPS is in plants, grids, data centres, fleets — with **few** consumers. Many consumers are in home/city/SaaS — with **low** rates and often **shared** content.
2. **Constrained links are low-rate.** LPWAN/satellite scenarios are minutes-to-hours with few metrics. "Constrained" and "high rate" rarely co-occur.
3. **Exactness and privacy cluster by domain.** Billing and protection need exactness; health/home/meters need privacy; industrial viewing needs neither urgently.

**Structural finding:** the properties are common but **live in different scenarios** — the "three different worlds" of §50/§63.

## 98. State and gates for a new Plan agent

**The engine:** provisionally **abandoned** (§83, §92) — trivial in novelty, heavy, unsourced, and the supervisor calls it trivial. If revived, shrink to **coupling + one composition baseline + evaluation**, framed as stated-requirement system-development. The **analysis is the asset**: coupling, commodity audit, failing-baseline method, seven related-work anchors (§81).

**Candidates:**

| Candidate | Source | Status |
| --- | --- | --- |
| **Drift-aware twin** | §87–§89 | **primary** — cited gap, reachable stakeholder, measurable evaluation, ~140–160 h; risks thinness if "inject → detect" (§94) |
| **Accessibility / neurodiversity** | §86 | **strong** — named stakeholders, mandated requirement, software-only, measurable, ethics |
| **Extend a supervisor topic** | §66, §84 | **pragmatic** — his domain, his approval, ~120 h |

**Critiques to answer:**

- **Not "is it novel?"** — the four criteria: problem, stakeholder, solution, verification (§95).
- **Is the residue real or a line of code?** For drift: management policy. For accessibility: decision support beyond a form.
- **Is it grounded?** Synthetic work must rest on real data or a real stakeholder (§94).
- **Is it right-sized?** ~304 h total, ~90 h dissertation; prefer **120–160 h** builds.

**Gates, in order:**

1. **Choose the track** — system-development or research. Everything depends on this (§96).
2. **Choose the candidate** — drift (primary), accessibility, or extended supervisor topic.
3. **Name a reachable stakeholder** — or the candidate is hypothetical (§84–§85).
4. **Talk to the supervisor** — 2–3 candidates as *problem → stakeholder (how reached) → solution → verification → ethics*; ask his bar and whether he accepts student-proposed topics (§91).
5. **Run the four-part filter** (+ reachable/observable).
6. **Integrity** — anchored criteria; no tuned parameters; derive, don't fit.

**What a new Plan agent should do:** ask **which track** and **which candidate**; help **name and reach a stakeholder** and **draft the paragraph**. Do not re-litigate the engine; do not drift back to "find a domain for the engine" (§36). Fix track and stakeholder first, then judge on stakeholder + evaluation, not novelty.

**Bottom line:** the discussion moved from **"can the engine be defended?"** (not as a problem; only as a costly stated-requirement project) to **"which system-development topic has a real stakeholder and a measurable evaluation?"** — drift primary, accessibility the strongest human-centred alternative, extending a supervisor topic the pragmatic fallback.
---

# Part VIII — Track closure, the competitor hunt, the design space, and the cut (eighth session)

Part VIII records the eighth session (2026-10-17, continuing): the track gate closed, candidate exploration under no-access, the campus-twin and AI proposals, the competitor feature-request hunt, and the design space and cut in depth.

## 99. Orientation

The student **closed the track gate**. The engine was re-examined with a new instrument — **competitor repositories' feature requests** — and a sharper account of the **cut's design space**. Method unchanged: apply the engine's critique to every candidate; do not re-skin.

## 100. The track decision (the gate of §96 closes)

Committed to **system-development** (COMP4299): **novelty not required**; depth from a **stakeholder** and **rigorous evaluation**; the engine remains admissible as a stated-requirement engineering project, not a research contribution. The track fixes the **bar**, not the **topic**; it does not make any candidate defensible.

## 101. Candidate exploration and the access constraint

The student declared **no reachable stakeholder** and **laptop-only** (no hardware, no institutional data). Lanes triaged:

| Lane | Verdict |
| --- | --- |
| Accessibility / assistive | **blocked** — verification *is* user testing |
| Learning / education | **blocked / weak** — same |
| Campus / dept tooling | **weakened** — needs champion and data |
| Embedded / sensing | **blocked** — no site, no devices |
| Security / privacy | partly open — public corpora, no participants |
| Public / open data | open — the dataset substitutes for access |
| Developer / ops tooling | open — OSS trackers reachable |
| Simulation / benchmark systems | open — but this is the engine's shape |

**The stakeholder is the scarce resource.** With none, the **external anchor** — public dataset, standard, published complaint — must carry justification (§41). The student **declined to test the no-access assumption** (a short, cheap effort asking around), so the anchor-only route was adopted.

## 102. The campus digital-twin proposal

**As stated, not defensible.** It is a *means* statement — an artifact (twin) plus a presentation (dashboard) — with no problem, stakeholder, or success definition; twins and dashboards are commodity; "digital twin" must be **earned** (bidirectional sync/what-if) or it is a map with charts. The **campus is the nearest stakeholder** (estates, sustainability, IT, accessibility), so it is promising **iff** anchored to a specific measured pain (energy/carbon, ventilation/CO₂, space, maintenance, accessibility) and a reachable person; the **extension must be a mechanism**, not a screen. With no access, it is a **synthetic twin of the campus** — the invented-scenario trap in a smaller box.

## 103. AI-model integration

**Not as "engine + a model."** It touches none of the engine's failures (problem, stakeholder, need); "integrate a model" is a *means*; it adds a second contribution and evaluation burden to an over-budget build; and an AI-centric project risks the **CSAI4299** bar. Test: *is there a requirement only inference can satisfy; can the model be isolated and measured; does the problem survive removing it; which module owns the result?* Roles graded — learned selection, imputation (contradicts exactness), anomaly detection (crowded, CSAI-shaped), threshold tuning (thin), forecasting (shifts topic) — **none rescues the engine**. **AI as decoration** vs **AI as the mechanism for a stated problem** (then the AI problem is the topic).

## 104. The competitor feature-request hunt

GitHub issue search plus direct reads (Cesium, deck.gl, loaders.gl, Potree, three.js, Ditto, Orion, ThingsBoard, Materialize/RisingWave/Feldera, Mirror/FishNet, tippecanoe, Grafana). Rate-limited: **sample, not exhaustive**; **absence of a request is weak evidence**. Target: the engine's **conjunction** — bounded + exact + consistent + per-consumer rollups under a live, view-dependent selection.

| Competitor | Issue | Age / state | What it asks | What it proves |
| --- | --- | --- | --- | --- |
| **Eclipse Ditto** | [#1525](https://github.com/eclipse-ditto/ditto/issues/1525) | 2022-10, **open** (`outlook`) | Aggregation over "hundreds/thousands of selected things" | **Real, dated, unmet demand** — *"clear need to have aggregations … to manage a large fleet"*; but **view-independent**; 4 years unbuilt |
| **Eclipse Ditto** | [#2291](https://github.com/eclipse-ditto/ditto/issues/2291) | 2025-12, open | Timeseries facade | Workaround: **manual TSDB integration** |
| **FIWARE Orion** | [#3816](https://github.com/telefonicaid/fiware-orion/issues/3816) | 2021-03, **open** | `?aggregate=category&operation=avg:score` | Same shape; workaround: **fetch all, aggregate client-side** |
| **ThingsBoard** | `RELATED_ENTITIES_AGGREGATION` | productized | Hierarchical aggregation over the entity tree | Aggregation half **already productized** |
| **Mapbox tippecanoe** | [#324](https://github.com/mapbox/tippecanoe/issues/324), [#354](https://github.com/mapbox/tippecanoe/issues/354), [#830](https://github.com/mapbox/tippecanoe/issues/830) | 2016–2020, closed | Combine features, aggregate properties to fit tiles | **Coupling exists** (`--accumulate-attribute`); **precomputed, shared, offline** |
| **deck.gl** | [#10795](https://github.com/visgl/deck.gl/issues/10795) | 2026-10, open | Aggregation layers that refine to raw points on zoom | View-dependent aggregation **solved — a documentation task** |
| **Cesium** | [#6182](https://github.com/CesiumGS/cesium/issues/6182) | 2018, closed | Aggregate geometries for zoom levels | User built the **workaround**; redirected to tile pipelines |
| **Mirror** | [#3197](https://github.com/MirrorNetworking/Mirror/issues/3197) | 2022, closed | Send state to a **subset of observers** | Per-consumer selection **solved** |
| **Grafana** | [#347](https://github.com/grafana/grafana/issues/347) | 2014, closed | Tree explorer of metric keys | Topic-namespace = tree exists as navigation |
| **misc** | `autheris` #112 | 2026, open | GraphQL sums/counts per group "without reading every row" | Another live aggregation-over-hierarchy request |

## 105. What the hunt establishes

1. **Aggregation has real, dated, open demand** — the **first published complaints and workarounds** in the review, in users' words (§63's missing evidence).
2. **View-dependent coupling is solved where content is shared** — tippecanoe, Cesium clustering, deck.gl; **precomputed/offline** (the Google-Maps case).
3. **Per-consumer selection is solved** — interest management.
4. **No published demand for the conjunction** — bounded + exact + consistent + per-consumer under a live cut; nor for cross-consumer snapshot consistency; nor for a hard per-consumer bound.

**Net:** demand is for **aggregation, not the coupling** — the residue is **unrequested**. This **corroborates and sharpens** §83/§92. Assets: citable **related work**; a narrower **live-aggregation-service** framing; material for the **"why not"** argument. Caution: **age is not demand** — Ditto is four years old and labelled `outlook`; platforms declined to pay for it.

## 106. The design-space comparison (for the professor)

Five architectures, plus three layered choices:

- **D1 — Broadcast + client cut:** fanout only; egress **O(N)/client**; no bound.
- **D2 — Interest/subscription:** fine (frontier) → **≈ D3**; coarse → **> D3**; CPU ≈ routing; binds on α.
- **D3 — Server cut:** only **hard per-client byte bound**; exact server-side aggregation of the unseen; deterministic, oracle-comparable, consistent; **CPU-bound** (`V_cpu ≈ 74–260`).
- **D4 — Server cut + lazy manifest (A′):** client **O(B)**; ordinary lazy loading, not novel.
- **D5 — Broadcast + edge relay:** cuts wide-area egress by the clustering factor; clustered clients only.

Layered: **transport** (snapshot vs batched delta; ~52 B IP overhead makes batching a requirement); **aggregation placement** (coupled); **deployment** (edge LAN vs cloud/WAN flips the ranking; cloud bytes cost ~40× the CPU to avoid them). Conclusions: **three designs bind on three resources**; **deployment decides**; **D3 is a bounded niche**; **the cut is commodity**; **at reasonable scale a snapshot with client cut suffices**, so necessity must be non-performance. This is a design-space characterisation — legitimate, **not novel**. (Detail: §15, §75, §107–§112.)

## 107. The cut, sharpened: placement × implementation, and the substrate

The proposed three choices mix **two axes** — **placement** (client/server/split) and **implementation** (library / library + own policy / own) — closer to a **2×2**, with the missing cell **server + library** (`loaders.gl` headless; possible, not sensible). **"Own check"** can mean *own traversal* or *own policy on library output*; the latter is fragile because the library selects by SSE and has already pruned — the honest client option is **own traversal using library math**.

**Pure client-library is weaker than it sounds:** three.js exposes no visible set; Cesium `i3dm` is tile-level; per-node tiles are impractical at 100k. The realistic client route is **regular cells + own check** (§72B). The **control** is a **single cut crate native + WASM**, so **placement is the only variable** and a third-party cut does not confound the comparison.

## 108. Direct select (D6): explicit subtree subscription

Clicking a room and receiving its readings is an **explicit subtree subscription** — the simplest regime 2, user-chosen, **not a cut**: no view-dependence, no `B` unless imposed. A subtree is a **contiguous preorder range** (membership O(1)); routing is a **per-node subscriber bitset**; `appeared`/`removed` + keyframe on change; no camera loop, frontier, or hysteresis. Variants: **flat** (all devices) and **progressive drill-down** (aggregates + child summaries; manual LOD). Egress: a benchmark room (~625 × 3) ≈ **3.75 KB/snapshot**; the **root = broadcast** (~60 KB/snapshot).

Pros: trivial, deterministic, works **off-screen**, maps to picking, generalises FR-VS-02. Cons: **commodity** (MQTT/Sparkplug/OPC-UA/Ditto/ThingsBoard); **unbounded at the root**; no free-camera relevance; churn on rapid clicking. Consequence: it covers **drill-down/inspection**, so **the cut is unnecessary for inspection**; it is the honest baseline that **narrows D3's niche**; cut + direct select compose.

## 109. View-dependence: only the selection is view-dependent

The **properties are view-independent** — a group's aggregate, attention, and device values are the exact rollup regardless of camera (hence aggregation is shared); only the **selection** is view-dependent, a **delivery/presentation** decision.

View-dependence buys relevance without asking, a **server-side** egress bound (client culling cannot reduce egress), LOD/blending, and "just works"; it costs the camera loop, per-viewer CPU, hysteresis/oracle complexity, latency sensitivity, and commodity. The three selection models — geometric / explicit / none — and the conditions where geometric is necessary (exploration; too large to enumerate; continuous pan/zoom; thin client; metered link; many viewers) versus not (dashboards, drill-down, inspection, alerting, field technicians). Dropping it leaves **state + hierarchy + aggregation + attention + subscription/delivery** — smaller, more commodity, and the shape with published demand.

## 110. Delivery versus rendering selection; colouring and compaction

**Colouring does not require the cut.** It needs the value, a **node → instance mapping**, and the instance in the draw buffer — none is a cut. The engine **conflated delivery selection** with **rendering selection** because the client only held what the server sent. With direct select, delivery is the subscription and rendering is client-side; **rendering all instances** makes `slot = node index` **static** — no compaction, slot table, or per-frame diff; picking is `instanceId → node index` (the engine needed `slotToNode` *because* the cut compacted). The "40 MB client" is the full manifest; a compact render buffer is ≈ **1.6 MB at 100k**. **The cut is a delivery mechanism; rendering does not require it.**

When compacted (`mesh.count = count ≤ B`), mapping is maintained by the diff (`nodeToSlot`/`slotToNode`, swap-remove). Example: `{A,B,C}`; after A leaves and D enters, `C↦0,B↦1,D↦2` — C moved without a decision, so its matrix is copied and picking resolves slot 0 to C. Transforms are static, so matrices change only on `appeared` and the swap; colours are the only per-frame write. The mapping is stable only under an unchanged cut (Decision 26). The alternative — **identity mapping** (render all, hide per instance) — trades churn for an **O(N) buffer**.

## 111. Tilesets, traversal, draw calls, and the control configuration

Use **per-node *tile* inside *one* tileset**, not per-node *tilesets* (which multiply traversals, add N roots, and destroy pruning). The library's traversal is **pruned** (frustum + SSE + `skipLevelOfDetail`), but a per-node tileset is **40–60 MB at 100k**, O(N) memory, and ~one draw call per visible tile. **Draw calls are issued by the library**; the developer controls scene membership, selection parameters, and authoring.

The **scaleable** structure is **coarse group tiles + `i3dm` batches + own per-instance check** (§72B). **Control configuration:** own the cut, write an instanced buffer, let the renderer issue one instanced draw — the library is a **renderer, not a culler** (disable `frustumCulled`, preallocate `max = B`, set `mesh.count`). **Manifest and tileset are two encodings of one hierarchy**; the tileset is derived and must coexist for value semantics — two sources of truth, double metadata, lossy two-box→one-box mapping. The own-cut route needs the **manifest alone**.

## 112. Client memory as the surviving justification

For minimal client memory, the **lazy manifest** turns the dominant **~40 MB** metadata into **~120–200 KB** at `B = 5,000` (O(B)), with total footprint ~200 KB–1 MB. But it **forces server-side selection** — the client cannot cut what it does not hold (§54: bounded client memory is a physical constraint and a better justification axis than exactness). **Trilemma:** O(B) memory + purely client-side selection + no extra round trips — **pick two**. Other levers: own cut + renderer-only; compaction; **server-computed attention**; compact binary encoding; progressive drill-down; remote/pixel streaming.

**Verdict:** the memory bound is the **strongest justification** — physical, citable, non-performance — but it justifies **server-side selection**, not the cut uniquely (direct select and drill-down meet it too); it needs an **anchored device class**, an **O(N) baseline that exceeds** the bound, and **A′**; and in the **PRD's GPU-laptop scenario it does not bind**. It makes the engine **defensible in a thin-client, free-camera, hard-bound niche**, not necessary.

## 113. The shared value: "if it does not bind, is it worth it?"

The value is not *binding* but **causality/falsifiability at the studied operating point** — the **ablation test**: the benefit must be **caused** by the mechanism, **observable**, and the study must **include** that point. Where the crude form is wrong: **tail value** (caps, rate limits, safety bounds are valuable when they do not fire), **structural value** (correctness, consistency, privacy constrain rather than bind), and **latent value**. Its **dark twin**: requiring the bound to bind invites the **tuning trap** (§13). **Rule:** *name the operating point where the mechanism is load-bearing, anchor it externally, and study it there; if no such point can be named honestly, the mechanism is decoration.*

## 114. Performance, must-have, and the product arena

**Performance improvement is unnecessary unless needed** — optimise against an **anchored requirement**, measure first; unneeded optimisation adds complexity and masks the real problem (the engine's problem was an ungrounded premise, not speed). "Needed" includes the **tail** and **metered costs**; **improvement ≠ study** (a design-space characterisation is a contribution with no improvement, and COMP4299 requires evaluation, not improvement). Need is **regime-relative** ("unnecessary *at P*"); sometimes the right response is to **avoid** the requirement.

**Nice-to-haves** are preferences with no owner/measure and cannot justify a project — but the must/nice boundary is set by the **anchor** (60 FPS is a must in VR, a nicety in a dashboard), and **even a self-set "must"** (K3/K6) does not save an unsourced, trivially met project. A nice-to-have **can justify a product by demand** (owner + measure, revealed preference). Distinctions: **feature vs quality**; **product** justified by demand, **FYP** by problem + method + contribution + evaluation, **research** by a gap. The **GitHub hunt was the product-style demand test inside a project** — demand for aggregation, none for the coupling. The rule did not change; the currency did.

## 115. State and gates

- **Track:** **decided — system-development** (COMP4299).
- **Candidate:** **unresolved**; drift is no longer the leading default.
- **Engine:** **not revived.** The hunt supplies the demand test; verdict **negative for the coupling, positive for plain aggregation.** The analysis remains the asset.
- **Stakeholder:** declared **none**; the **external anchor** must carry justification.
- **Open gates:** choose a candidate/anchor (aggregation service; drift; standards/dataset; extended supervisor topic); **talk to the supervisor**; run the four-part filter; **integrity**; **module-fit check** for AI/ML (COMP4299 vs CSAI4299).
- **What a new agent should do:** ask which **candidate** and **anchor**; help **name the anchor** and **draft the paragraph** (*problem → anchor (how it substitutes for a user) → minimal solution → verification → ethics*).

## 116. Bottom line after Part VIII

- The **track gate closed**. The **feature-request hunt** produced the review's **first real demand evidence** — for **aggregation**, not the **coupling**: the residue is **unrequested**.
- The **design-space comparison** and **cut taxonomy** make the trade-offs reportable: placement × implementation, shared-crate control, **direct select**, **view-independence**, **delivery/rendering separation**, **compaction**.
- **Client memory** is the engine's **only surviving justification**: physical and citable, but **narrow** — it justifies server-side selection generally, binds only for a thin-client free-camera consumer, needs A′, and **does not bind in the PRD's scenario**.
- One shared value: a mechanism must be **causal at an anchored operating point**. Applied precisely it separates a project from a demo; applied crudely it invites the **tuning trap**. A nice-to-have can justify a **product by demand**, not a project without owner and measure.
- **Status:** track fixed; **candidate and anchor open.**
---

# Part IX — Defensibility, disclosure, supervisor fit, and trust (ninth session)

Part IX records the ninth session (2026-10-17 → 2026-10-19): what makes a topic admissible; the disclosure direction worked to a concrete problem and extension; two papers positioned; the supervisor's profile; and IoT telemetry upload and trust.

## 117. Orientation

The session moved from "which topic?" to "what makes *any* topic defensible?", and stress-tested the disclosure direction against the literature and the supervisor's work.

## 118. The data-centre digital-twin proposal

**As pitched, not defensible**, and the domain weakens the premise: (i) a server room is ~**1,200 servers** (§58), so the bound never binds; (ii) the real pains — hot spots, PUE, capacity, blast radius — are **severity/correlation-driven, not view-driven**; (iii) DCIM runs on a **LAN with capable clients**, the edge regime where broadcast wins; (iv) crowded (Vertiv, Sunbird, Nlyte, Cadence, Grafana, Omniverse). "100k EPS ≠ 100k devices" (§59).

## 119. Do data-centre twins use visualization?

Yes — 3D is close to the signature (Sunbird, Raritan, Cadence Reality, Omniverse DSX). **But visualization is the interface, not the twin**: the industry says a true twin "goes far beyond 3D" (Modius) — the substance is the synchronized model plus simulation (CFD/thermal, capacity, what-if). Visualization is **table stakes**; 3D earns its place for **airflow/CFD**, not telemetry.

## 120. Does an FYP rule out simple CRUD?

No — it rules out **simple**, not **CRUD**. Test: **delete the framework, ORM, and UI kit — is there a paper left?** A project passes on **problem + mechanism + evaluation** (plus substantial software and "integrate *and extend*"), not the technology label. The failing pattern is assembly — the supervisor's six topics are one template: `sensor → threshold → dashboard → notify`.

## 121. Mature solutions and defensibility

Prior art is the **baseline**, not a disqualifier (novelty not required). Four shapes: (a) build-where-exists → **assembly**; (b) build-for-a-context-the-incumbents-fail → defensible; (c) **study/evaluate** → defensible; (d) integrate-X+Y-where-the-composition-fails → defensible. Only (a) is fatal. In a mature area the routes are a **wanted residual** (needs access), a **study**, or an **engineering fit** judged on execution.

## 122. The salon booking example

A local salon clears the gate the engine never did — a **real, reachable stakeholder** (reopening access). But as pitched it is **assembly**: booking is commodity (Fresha, Booksy, Square, Vagaro, Google Reserve). Booking does **not** eliminate waiting; the real pains are **no-shows, service-time variability, walk-in mix, skill assignment**. Reframes: no-show-aware overbooking, variable-duration scheduling, hybrid queues — each a mechanism plus evaluation against the incumbent/manual baseline.

## 123. FYP versus product

The FYP is an **assessment** (judge = examiner; success = defensible contribution); the product is **value capture** (judge = market; users/revenue). The FYP excludes distribution, support, maintenance, monetisation. They share **problem legitimacy**; the product needs **existence → salience → willingness to pay → viability**. The FYP's stakeholder is an **epistemic device**, not a customer. A nice-to-have can justify a **product by demand**, not a project without owner and measure.

## 124. One citation and specificity

A generic citation ("someone wants a view-dependent visualization") validates the **commodity half** and would be satisfied by Maps/Cesium. The citation must match the **contribution's specificity**. Evidence ladder: paper's "future work" (weak) < one user (weak–moderate) < **published complaint/workaround** (strong) < **standard/regulation** (strong). One strong citation is **necessary, not sufficient**: extension, evaluation, feasibility, and integrity are still owed.

## 125. QR-code attendance

Public repos already do IP restriction, geofencing, and short-lived codes. A QR code is a **data carrier, not a presence proof**, so the design **resolves away its own hard part** (proxy attendance), as §66 found for RFID. Residual seams: **relay/"mafia-fraud" resistance** (distance bounding, UWB, acoustic — mostly hardware), **privacy-preserving presence** (Semaphore, BBS+, eIDAS 2.0), comparative evaluation, **collusion detection**, **offline integrity**. The failing-baseline method applies: build naive, red-team, reconcile, evaluate (proxy rate, false accept/reject, usability, privacy).

## 126. Justifying privacy without demand

"Students want privacy" is weak demand and probably false for the median student (privacy paradox; attendance mandatory). Privacy survives on **regulation** (GDPR minimisation/proportionality), **harm** (visa students, disabled or mentally-ill students, the stalked, marginalised groups), and **design obligation**. The "wanter" is the regulator/DPO, not the student. The design tension: **verifiable accountability without central surveillance**.

## 127. The disclosure problem

A campus operator shares a **live, aggregate view** of a building twin with an **external FM contractor** (one building; no other tenants; no identifiable individuals). Fourteen requirements across **access/scope, disclosure, delivery/correctness, accountability**, evaluated against the **engine** and the **best-of-breed composition**:

- The engine **supplies** aggregation, exactness plus consistent snapshot, freshness, and bounded delivery — the **delivery half**.
- It **lacks** policy, suppression, temporal control, audit, revocation, reconstruction evaluation.
- It **conflicts** where exactness/consistency **enables subtraction**, where **delta encoding hands over the differencing signal**, and where camera-driven selection conflicts with a fixed policy scope.
- The composition supplies policy/audit/static SDC but is **weak on live consistency, freshness, bounds**.
- **Both fail** on live suppression, temporal privacy, and a reconstruction adversary — the seam.

## 128. Visualization not needed

Disclosure is a **delivery** problem; rendering is orthogonal. Dropping 3D removes the camera loop, per-viewer cut, roaming problem, and build cost; scope becomes a **policy-defined subtree**. Keep at most a minimal demo (dashboard/table) and an **operator audit view**. State the project as **disclosure-bounded delivery**, not visualization.

## 129. The key extension

**Online statistical disclosure control over a live hierarchical model, coupled to per-consumer bounded delivery, verified by a reconstruction adversary.** Components: **E1** live hierarchical disclosure control (minimum group size + **complementary suppression**; `min`/`max` leak a unit); **E2** temporal control (differencing, timing channel, release budget); **E3** suppression stability (transitions leak; hysteresis); **E4** absence confidentiality; **E5** reconstruction-adversary harness; **E6** the coupling (shared aggregation, per-consumer disclosure, bounded delivery). It is an extension, not assembly: the composition **fails** (exact aggregates leak) and the reconciliation is the engineering.

## 130. Existing solutions

In parts: **data clean rooms** (AWS Clean Rooms, BigQuery thresholds) do scope + aggregate-only + minimum group size, **query-based**; **SDC** (sdcTable, tau-ARGUS, complementary suppression) is **static** and known-vulnerable (Cox 2010); **DP under continual observation** (Dwork 2010; Jain 2023; SPAS; ToPS) is **noisy**; **hierarchical DP** (Fioretto/Census) is noisy. The **combination** — live, hierarchical, suppression-based, bounded per consumer — is **not productized**, but well studied: research-shaped, modest novelty. If periodic suffices, a clean room plus a scheduler is close.

## 131. Live versus periodic

In **rate**, live ≈ periodic with Δ → 0; **event-driven** adds a **timing channel**. The real problem is **repetition over a horizon** (≈ T/Δ releases), because differencing accumulates — static per-release SDC is unsafe under repetition. Drop "live": it is **online/repeated release**; the controlling variables are **freshness Δ, horizon T, and whether timing is published**.

## 132. Why it is important

Three legs: the setting is real and growing (data sharing, clean rooms, data spaces); the obligation is citable (GDPR plus sectoral confidentiality); **current practice is unsafe and the safe alternative missing** — the **silent failure** of repeated thresholded release. The strongest leg is the third. Counterpoints: demand may be weak or unspoken; mature fields may be "good enough"; the scenario may be contrived; the problem can be avoided. Importance is **conditional** on a documented deployment or standard.

## 133. How to address, why nobody, how to prove

Two routes: **DP under continual release** (formal, noisy) versus **suppression-based online SDC** (exactness, no general proof). "Why nobody": most likely **exactness is the idiosyncratic requirement** (drop it and DP solves it), or demand is weak, or it is designed away. **Proving no leak:** DP is the only general proof; complementary-suppression correctness proves **single-release** bounds, not sequences; otherwise **measured** resistance. Honest claim: **exactness where released + no single-release reconstruction (proved) + measured sequence resistance (evaluated)** — not a universal no-leak proof.

## 134. SHIP (arXiv 2610.21048)

**SHIP: Structured and Hierarchical Insight with Privacy** (UVA + Google Research; cs.CL; 15 Oct 2026): DP insight extraction from **textual corpora**; hierarchical hybrid taxonomy (open-set DP partition selection + closed-set supplementation); a DP synthetic table (AIM) preserving cross-attribute marginals; **zCDP + composition**. Relevance: a **related-work anchor**, not a competitor — hierarchical DP release is established and DP-with-composition is the **proof** answer. It is **textual, one-shot, noisy**: no repeated release, per-consumer delivery, or exactness. Applying it joins the DP route (conceding exactness).

## 135. The supervisor's profile and authorship

**Su-Kit Tang**, Associate Professor, Faculty of Applied Sciences, **Macao Polytechnic University**. Five lines: **embedded/edge AI** (review; zero-copy instrumentation); **lightweight vision for conservation** (eBirdNet-Nano); **LLM + built environment** (floor-plan compliance); **air quality** (feature screening); **auditability/privacy** (blockchain confidentiality). Authorship (Crossref): **never first author**; **last/senior** on edge-AI/floor-plan/embedded-AI papers (students lead); **middle** on air-quality, blockchain, employment, sentiment. The last-author lines are where he **supervises students** with data/hardware — the best bets.

## 136. Fitting the engine to those applications

The engine's parts can appear (hierarchy, aggregation, filtering), but the **engine as contribution** does not fit: bird monitoring's hard part is detection; floor plans' is extraction/rules; the embedded benchmark has no hierarchy; air quality's is calibration. The distinguishing mechanism — live, bounded, per-consumer, view-dependent — **does not bind**. "Hierarchy + aggregation + filtering" is **infrastructure** — ubiquitous as a component, never a product alone. Invert the search: not "where does my solution fit?" but **"whose requirement fails the existing tools?"**

## 137. Résumé value and engineering depth

**Portfolio value** is high for systems/backend/infrastructure/real-time roles, **conditional on building and measuring** (docs-only is weak). Present **engineering metrics**, not users. **Depth ≠ from scratch**: refusing libraries is a yellow flag (NIH); depth = justified dependencies + understanding + measured trade-offs + correctness discipline. Reuse transport/math/rendering; build the **state table, aggregation, cut, delta protocol, oracle, harness**.

## 138. The edge-IoT security-monitoring paper (arXiv 2610.09817)

**Hierarchical Security Monitoring for Edge-IoT: A Formal Methods Approach** (IEEE CSR 2026): two-level runtime verification — **L1 TeSSLa** per-device predicates (size, fuzz, monotonic time, rate) aggregated to four-valued verdicts (⊤/⊤p/⊥p/⊥); **L2 MonPoly MFOTL** at the gateway correlating verdicts (three nodes with ⊥p within 60 s) to catch coordinated APT; lattice-meet composition with witness sets; a shield (block/confirm/log/allow); uplink **one Boolean per window per node**. Docker testbed: APT 12/16 → 14/16 with L2; FPR 24% (timestamp artifact) → 4% on cleaner replay; ~0.52 µs/event; ~200 bit/s at N = 1000. Limits: synthetic, offline, single gateway, **non-adaptive adversary**. **Lesson: start from the cost asymmetry (edge vs cloud vs attention), then derive the hierarchy** — the inversion the engine never made.

## 139. IoT telemetry: upload and trust

**Upload:** link (Wi-Fi/Ethernet/cellular/LPWAN/BLE/Zigbee/Modbus/CAN/satellite); protocol (MQTT, HTTP(S), CoAP, AMQP, OPC UA, Modbus/DNP3/IEC 61850, Sparkplug B, LoRaWAN, WebSocket); patterns (gateway-mediated, store-and-forward, periodic/delta/event, JSON/CBOR/Protobuf); **time** (device vs gateway vs server). **Trust** is six questions: authenticity (mTLS/certs/PSK), integrity (TLS/MAC/signature), freshness (nonce/sequence/timestamp), device integrity (secure boot, **remote attestation**), **data truth** (range/rate/plausibility/redundancy/calibration), and **behaviour** (rate/size/time, cross-device correlation). **Cryptography gives authenticity and integrity, not truth**; a compromised sensor signs a wrong value perfectly. Governance adds provisioning, revocation, access control, **audit/provenance**. Mapping: the edge-IoT paper is the **behavioural** layer (assuming identity/attestation); drift is the **data-truth** layer; disclosure is the **governance/audit** layer.

## 140. State and gates

- **Track:** system-development (COMP4299) — **decided**.
- **Candidate:** **open**, and the landscape has contracted. The engine is **not revived**; parts are reusable, analysis is the transferable asset.
- **Access:** no stakeholder declared; **the supervisor's lab is the realistic source** (data/hardware), and **last-author lines** are the strongest bets.
- **Disclosure direction:** most intellectually coherent, but **research-shaped** and dependent on the exactness/DP decision and the "repetition over a horizon" framing.
- **Gates:** (1) choose candidate/anchor; (2) confirm **COMP4299 vs CSAI4299**; (3) establish **data/hardware availability** with the supervisor; (4) decide **exactness vs DP** if disclosure; (5) integrity.
- **Supervisor questions:** which line has data/hardware this term; does he accept a student-proposed topic; which module; what bar.

## 141. Bottom line after Part IX

- The session gave the review's **most complete account of defensibility**: prior art is the **baseline**, not a disqualifier; the fatal shape is only "build the solved thing again, for nobody, measured by nothing"; a **study** or **failing-baseline extension** is the honest escape.
- **Disclosure** reached a concrete problem, requirements, and a **key extension** (online hierarchical SDC + reconstruction adversary) — but it is **well studied**, "live" collapses to **repetition over a horizon**, and **exactness** is the idiosyncratic bet.
- The **supervisor's profile** is the strongest practical asset: **last-author** lines (edge AI, bird monitoring, floor plans) provide data, hardware, supervision; middle-author collaborations are weaker.
- The **engine** retains value as a **portfolio** piece (conditional on building/measuring) and as **analysis** (coupling, design space, commodity audit) — not as the FYP contribution.
- **Status:** track fixed; candidate, anchor, and module fit open; next move is a **supervisor conversation** with 2–3 one-paragraph candidates and two questions (data/hardware; COMP4299 vs CSAI4299).
