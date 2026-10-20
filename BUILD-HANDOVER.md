# BUILD Handover — DEC-001 and ANL-001

| Field | Value |
| --- | --- |
| Prepared | 2026-10-20 |
| For | a new BUILD agent continuing the ANL-001 / DEC-001 discussion |
| Repo | `/Users/toddhuang/Projects/strata` |
| Branch | `tdd` |
| HEAD | `74cdfa4` (“Save”) |
| Working tree | modified: `BUILD-HANDOVER.md`, `DISCUSSION-SUMMARY.md`, `docs/ANL-001`, `docs/DEC-001`; some files staged by the user |
| Read first | `docs/DEC-001__Decision_Options_Brief.md` (v0.4.0), `docs/ANL-001__Existing_Solutions_Contrast.md` (v0.2.0), `DISCUSSION-SUMMARY.md` Part XI (§151–§162), this report |

---

## 1. Status at a glance

| Item | State |
| --- | --- |
| Track | **COMP4299 system-development — decided.** Novelty is not required; depth comes from the stakeholder and evaluation. |
| Supervisor conversation | **Scheduled, not yet held — this is the gate.** |
| Candidate / anchor | **Open.** No candidate or scenario anchor is fixed. |
| Engine | **Not revived, not being built.** The analysis is the transferable asset. |
| DEC-001 | **v0.4.0**, “Draft — for supervisor discussion,” last updated 2026-10-20. |
| ANL-001 | **v0.2.0**, analysis annex, last updated 2026-10-20. |
| Current focus | **Continue the ANL/DEC discussion at the analysis level.** Materialize, model bases, K3, S5, the SQL benchmark, the bitmask, and the supervisor questions. |
| Main build | **Not started.** Do not start before the candidate and anchor are fixed. |
| Git | HEAD `74cdfa4`; do not stage or commit unless the user asks. |

---

## 2. Git and document state

- Branch: `tdd`.
- HEAD: `74cdfa4` (“Save”).
- Working tree: `BUILD-HANDOVER.md`, `DISCUSSION-SUMMARY.md`, `docs/ANL-001__Existing_Solutions_Contrast.md`, and `docs/DEC-001__Decision_Options_Brief.md` are modified. Some are staged; check `git status` before acting. Do not stage or commit.
- Recent history:
  - `16469d0` — ANL-001 and the DEC-001 rework.
  - `74cdfa4` — DISCUSSION-SUMMARY Part X and the ANL/DEC fixes.
  - Earlier: `5b1244f`, `75fd948`, `ec9304a`, `77ac88e`, `486b40a`.
- Commit messages are terse. Read the documents; do not infer scope from the messages.

Header state:

- DEC-001: version `0.4.0`, status “Draft — for supervisor discussion,” last updated `2026-10-20`.
- ANL-001: version `0.2.0`, status “Draft — analysis annex to DEC-001,” last updated `2026-10-20`.
- DISCUSSION-SUMMARY: date `2026-10-20`, eleven sessions, §1–§162.
- PRD-001, TDD-001, and PLN-001 are **legacy references**, not the source of truth. The project is discussing changes and possible abandonment of the legacy design.

Do not bump versions unless asked.

---

## 3. What the project is

The documents compare designs for delivering live telemetry to 3D digital-twin consumers. The proposed engine is a server-side geometric cut with exact hierarchical aggregation.

DEC-001 asks whether the cut has an advantage; its answer is conditional and largely negative:

- No unconditional advantage.
- Without K3, D2f (fine interest) and D3/D4 (server cut) tie on egress; broadcast is `N/B` worse; D1 wins on LAN; D6 wins for a guided consumer.
- Under K3, the free-camera feasible set is D3/D4; the comparison is a premium, not an advantage.
- K3 is **self-set and unanchored**; on the current evidence it is **likely not a realistic requirement**. The unconstrained comparison is the realistic baseline; the K3-feasible comparison is conditional.
- The demand for the conjunction is unproven; demand exists for plain aggregation.

ANL-001 sets the cut aside and asks whether the engine’s hot-state, rollup, and snapshot pipeline is commodity at 30k EPS. Its answer:

- The scale is trivial; existing frameworks handle it.
- The semantic contract is the hard part: exact rollup over a tree with channel wiring, shared frame snapshot, and candidate mask-aware aggregation.
- A generic SQL/streaming stack can express each part; it does not provide the conjunction out of the box.
- Materialize, configured to the engine’s contract, is the closest packaged aggregation baseline. It matches the simple rollup, wiring, and shared snapshot; it cannot match IEEE f32 parity; it expresses availability and masks at a cost.
- Delivery is excluded from the Materialize comparison; it is a separate comparison with its own baselines.

---

## 4. User decisions already made

1. **K3 is self-set and unanchored.** It is ≤ 5,000 active entries per viewer per frame in the worst case, where an entry is a device instance or a blended group entry (legacy PRD K3, FR-VS-07); S0 uses the tighter B = 2,000 perceptual budget.
2. **K3 is likely not a realistic requirement.** The 2026-10-19 decision to adopt it is preserved, but DEC-001 now leans: no external anchor, no consumer or standard found; the unconstrained comparison is the realistic baseline, the K3-feasible comparison is conditional.
3. **The topology is a tree.** Exactly one root; every non-root node has exactly one parent; no cycles; devices are leaves and attach to exactly one group. Channel wiring fans out/fans in only within the unique parent edge; there are no multi-parent nodes.
4. **Bitmask access control is a candidate requirement, not sourced.** PRD §10 puts auth out of scope; DISCUSSION §127 is a candidate direction. Per-device mask (10k bits ≈ 1.25 KB); metric-slot expansion (30k ≈ 3.75 KB if materialized).
5. **Delivery is not compared with Materialize.** It is kept as a separate comparison (ANL §9) with its own baselines: 3D Tiles/Cesium (`skipLevelOfDetail`), Potree, OSG/osgEarth `PagedLOD`, deck.gl, interest management (SpatialOS, Photon, Mirror, FishNet, Unity NGO).
6. **Client CPU and p99 are model columns, not measurements.** The bases are in DEC-001 §5.3. The client CPU column is the frame in which a data tick arrives; decode/update are charged in full. p99 is `1,000 ms + per-package overhead`.
7. **S5 is an assumed scenario, not anchored.** p99 < 250 ms from server data availability (ingress) to client receipt, including the tick wait and excluding the device path; `r = 4 Hz`; egress 10 Hz, decoupled from `r`.
8. **PRD-001 and TDD-001 are legacy references.** They are not the source of truth while the project is reconsidered.
9. **Do not stage or commit** for the user. The user commits the documents themselves.
10. **The supervisor conversation is the gate.** It has not happened.

Do not re-open these unless the user does.

---

## 5. Current numbers and scope

### 5.1 S0

- 10,000 devices, ≈30,000 tags, 3 metrics each, 1 Hz reading rate.
- 10 viewers, cloud/WAN, f32 values.
- 180 groups, 3 channels each; 540 group-channel aggregates.
- Entry budget B = 2,000; K3 ceiling = 5,000.
- Full state ≈122 KB/tick; full-state delta ≈65 KB/tick.
- Frontier ≈13 KB/s base case; Appendix B gives ≈14 KB/s under independent per-slot `p = 0.5`.
- D2c region ≈33 KB/s; D6 room ≈4 KB/s.
- Egress at $0.09/GB: 65 KB/s → $0.021/h; 13 KB/s → $0.0042/h; 33 KB/s → $0.011/h; 4 KB/s → $0.0013/h. A continuous month is 730 h.

### 5.2 Client CPU model (DEC-001 §5.3)

| Package | Printed (ms/frame) | Model |
| --- | --- | --- |
| D1 | 0.5–2 | 0.57–1.77 (full state + cut + client aggregation) |
| D2f | 0.3–1.2 | 0.30–1.16 (×1.5–2 WASM; 0.1–0.4 frontier-only) |
| D2c | 0.2–0.8 | 0.18–0.78 (region + aggregation 25k–41k) |
| D3/D4 | 0.1–0.3 | 0.07–0.21 (frontier decode + update) |
| D5 | 0.5–2 | 0.57–1.77 (same as D1) |
| D6 | 0.05–0.15 | 0.04–0.12 (room + cull/push-pop) |

Constants: 10–30 ns per value decode; 2–5 ns per value update; 10–30 ns per node frustum; 20–50 ns per frontier entry push/pop; 2–10 ns per contribution aggregate. All assumed, not measured.

### 5.3 p99 model (DEC-001 §5.3)

`p99 ≈ 1,000 ms + overhead`, where 1,000 ms is the 1 Hz egress cadence.

| Package | Overhead (ms) | Path |
| --- | --- | --- |
| D1 | 20–80 | one WAN round trip |
| D2f | 50–150 | subscription RTT + frontier churn |
| D2c | 50–150 | subscription RTT + region switch |
| D3 | 20–100 | camera uplink + downlink |
| D4 | 50–200 | camera path + lazy manifest (partially warm) |
| D5 | 30–100 | WAN + relay hop |
| D6 | 20–80 | subscription RTT, steady after user action |

D4’s 1,050 lower bound assumes a partially warm manifest; a warm cache reduces it to D3’s range. The b/c entry sum (0.18–0.52 ms) plus framing reconciles the 0.23–0.57 ms cut+encode figure.

### 5.4 S5

- Requirement: p99 < 250 ms from server data availability to client receipt; tick wait included, device path excluded.
- Parameters: `r = 4 Hz`; egress 10 Hz, decoupled.
- At uniform 10 Hz: D1/D3/D5/D6 pass; D2f/D2c borderline at 250 ms; D4 fails at 300 ms.
- Per-package: D4 needs ~20 Hz (25 Hz with margin), 2–2.5× the uniform 10 Hz and ~3× the ~6 Hz D1/D6 minimum.
- `N/B` is unchanged; `r = 4 Hz` raises the changed-value payload ~4×; the 10 Hz cadence adds per-tick headers/bitmaps; the cut is unchanged, encode/send rises.
- Under K3 at 10 Hz, the feasible set collapses from {D3, D4} to {D3}; at 20–25 Hz D4 returns, with its memory advantage offset by cadence cost.

### 5.5 Manifest egress

- The manifest is a connection/session cost, not a per-tick rate.
- ~4 MB at 10k devices, ~40 MB at 100k (full JSON; format-dependent).
- At 13 KB/s frontier, ~5 min of frontier-equivalent egress at 10k, ~51 min at 100k.
- Over 8 h: ~0.14 KB/s at 10k, ~1.4 KB/s at 100k.
- Reduced by compact binary, client cache/versioning, and a lazy manifest. Report separately; it can dominate a short session.

### 5.6 CPU

- Server CPU unmeasured. Model: `T_cut ≈ 0.05 ms + bB`, `T_encode ≈ cB`; `b ∈ [0.06, 0.19] µs`, `c ∈ [0.03, 0.07] µs`.
- At B = 2,000: ~0.23–0.57 ms per cut+encode; ~0.014–0.034 core/viewer at 60 Hz.
- Client CPU and p99 are models, not measurements.
- Do not present the range as a result.

### 5.7 Client memory

- `O(B)` requires a lazy manifest plus compaction, a sparse `node ↔ slot` map, and dynamic remapping.
- At B = 2,000, D4’s descriptors are ~0.05–0.1 MB; total ~0.1–0.4 MB.
- D6 can reach `O(subtree)` with a scope-limited lazy manifest.
- D2f approximates `O(B)` via a coarse-top lazy descent, at a round-trip cost.
- The hard bound (egress/K3) and `O(B)` (client memory) are separate claims.

---

## 6. ANL-001 highlights

ANL-001 sets the cut aside and states three defensible highlights:

1. **Exact rollup over a tree with channel wiring.**
   - Fan-out/fan-in within the unique parent edge.
   - Same-unit/same-reduction; violations abort boot.
   - Exact mean `(sum, count)`; count of available readings; no-value channels.
   - Per-metric freshness; attention/absence.
   - Canonical blocked-lane f32 order (`W = 8`), bit-exact to the oracle.
   - ≤5 µs per channel per group with up to 1,000 contributions (K9, legacy target).
2. **Shared frame snapshot independent of viewers.**
   - One rollup per frame; double-buffered; all viewers on the same snapshot.
   - Cost independent of viewer count.
   - It is a contract property, not a differentiator: a shared MV + `SUBSCRIBE` provides the same consistency.
3. **Mask-aware aggregation (candidate).**
   - Raw mask vs aggregate mask are different:
     - Raw: filter delivered entries; global rollup remains.
     - Aggregate: exclude unauthorized contributions before the reduction; the aggregate is per mask class.
   - Per-device mask; metric-slot expansion.
   - SQL can express per-mask aggregation; cost scales with mask cardinality.
   - Engine candidate advantage: per-mask classes; global-minus-unauthorized for sum/count/mean; mask-aware tree for min/max/attention.
   - Exactness caveat: global-minus-unauthorized is **not bit-exact** to a direct masked reduction in f32. Use a direct per-class canonical pass or exact arithmetic if bit-exactness is required.
   - The bitmask is authorization, not privacy; inference/differencing remain.

ANL-001 §8 is the engine-vs-Materialize comparison (8.1–8.10); §9 is delivery as a separate comparison; §10 is the failing-baseline tests; §11 is positioning; §12 is the summary.

---

## 7. What changed since the previous handover

### 7.1 DEC-001

- **Client CPU column** is now “(ms/frame; model)” with a basis note and per-package table. Values: D1/D5 0.5–2; D2f 0.3–1.2; D2c 0.2–0.8; D3/D4 0.1–0.3; D6 0.05–0.15.
- **p99 column** is now “(ms; model)” with `p99 ≈ 1,000 ms + per-package overhead` and a per-package allocation table.
- **§5.3 notes reordered and trimmed:** Dominance set → table → Reading the table → Egress ratio → Egress scope → Manifest → At S0 → CPU cost → Client CPU basis → p99 basis → Package notes → K3-feasible table.
- **K3 lean:** self-set, unanchored, likely not realistic; unconstrained comparison is the realistic baseline; K3-feasible comparison conditional. The 2026-10-19 adoption is preserved.
- **S5 added:** p99 < 250 ms from server availability to client receipt; r = 4 Hz; egress 10 Hz; per-package results; K3 intersection.
- **Appendix A** links verified and corrected; deck.gl and Mirror recharacterised.
- **Last Updated** 2026-10-20; version 0.4.0 unchanged.

### 7.2 ANL-001

- **§8 engine vs Materialize** (8.1–8.10): configuration, contract table, one-parent constraint, update/arithmetic, availability, masks, memory, where each wins, other candidates, what to measure.
- **§9 delivery: separate baselines**; delivery is not scored against Materialize.
- **§10 failing-baseline tests** (was §8 before the new §8).
- **§11 positioning**, **§12 summary** (renumbered).
- Arithmetic: Materialize fixed-point 2⁻²⁴, deterministic and order-independent but not f32; the `VALUES`-vs-MV benchmark trap; the shared snapshot is a tie; memory is O(N·m) under current-state Materialize; the structural memory gap is only in the arbitrary-mask regime.

### 7.3 DISCUSSION-SUMMARY

- Added **Part XI** (§151–§162): Materialize, head-to-head, persistence/history, one-parent constraint, client CPU/p99 models, K3 lean, S5, Appendix A verification, external sources, review audits, state.
- Header updated to eleven sessions; Part XI row added.

---

## 8. Analytical points to preserve

Do not re-litigate these:

1. **The cut is commodity** (3D Tiles/Cesium, Potree, OSG, interest management).
2. **Per-instance selection is not provided by libraries**; you compute it and keep a node↔instance map.
3. **Exactness is commodity** (SQL, IVM, CRDTs, stream processors).
4. **The residue is the coupling** (selection ⊗ aggregation); the demand hunt found it unrequested.
5. **Prior art is the baseline, not a disqualifier.**
6. **No unconditional advantage.** D2f ties the server cut on egress; D1 wins on LAN; D6 wins for guided.
7. **View-dependence is a delivery design choice**, not a requirement.
8. **K3 is self-set and unanchored; access control is a candidate requirement; exact aggregation and consistency are not sourced.**
9. **Deterministic constants cancel:** duty cycle, clock speed, value width, `p` scale.
10. **The egress ratio is approximately `N/B`**, not a bandwidth saving.
11. **Split is not a separate placement.**
12. **D2f is client-side selection + frontier subscription.**
13. **TDD/PRD budgets are design targets, not measurements.**
14. **CPU is unmeasured.**
15. **Manifest egress is separate** from per-tick egress.
16. **`O(B)` requires lazy manifest + compaction + sparse map + dynamic remapping**; it is not unique to the cut.
17. **The hard bound (egress/K3) and `O(B)` (client memory) are separate.**
18. **The free camera is the engine’s niche, not the modal consumer.**
19. **The node topology is a tree**, not a multi-parent DAG. Channel wiring is many-to-many only within the unique parent edge.
20. **30k EPS is not the gap.** Existing frameworks handle the scale; the semantic contract is the differentiator.
21. **SQL is a language, not one engine.** Do not say “SQL cannot”; say which engine/query and whether the shared cost/guarantees survive.
22. **The bitmask is authorization, not privacy.** Inference/differencing remain.
23. **Global-minus-unauthorized is not bit-exact** to a direct masked reduction in f32.
24. **Materialize sums are fixed-point at 2⁻²⁴**, not IEEE f32; test through a source/MV, not a `VALUES` list.
25. **The shared snapshot is a tie**, not a differentiator. The fair baseline is a shared MV + `SUBSCRIBE`, not a per-viewer pull query.
26. **Delivery is not compared with Materialize.** The cut has its own baselines.
27. **Client CPU and p99 are models.** State the frame semantics and the per-package overhead; do not present them as measured.
28. **K3 is likely not realistic.** The unconstrained comparison is the realistic baseline.
29. **S5 is a cadence requirement.** It changes the feasible set, not the `N/B` ratio.
30. **Appendix A evidence is verified.** The old `FIWARE/orion` link is broken; the canonical repo is `telefonicaid/fiware-orion`.
31. **The two IEEE papers support the latency/consistency framing**, but not as quantitative anchors; the Anvil post is vendor context.

---

## 9. Review session

The review line is tracked **outside the project**:

| Field | Value |
| --- | --- |
| External record | `/private/var/folders/py/tgxh0jws1xs34r2rbbgrnhxw0000gn/T/opencode/strata-review-session.md` |
| Original session | `ses_ea8c8529fffePn16RPUCljzDgc` (not resumable from a later BUILD session) |
| Current child | `ses_ea8261124ffewIT7wHmP4QtF15` |
| Last audits | ANL §8: 3 high, 6 medium, 7 low; DEC: 2 high, 3 medium, 6 low; fixes applied |

**Resumability caveat.** A subagent session can only be resumed from the BUILD session that spawned it. A new BUILD agent should start a fresh `review` session and port the external record; do not assume either ID is directly resumable. Do not recreate a review record under the project root.

The ANL fixes were applied after the audit; the DEC fixes were applied after the audit. **Neither document has had a fresh verification pass.**

---

## 10. Open questions

1. **Supervisor meeting:** has it happened? What line has data or hardware this term?
2. **K3:** is a self-set bound acceptable, or must it be sourced? Current lean: likely not realistic; should it be dropped and the comparison run unconstrained?
3. **Free camera vs guided:** is the free-camera profile the intended S0 requirement? If guided, D6 is the baseline.
4. **S2:** what thin client can be anchored, and is it free-camera or guided?
5. **S4:** is there a scatter anchor (fleet/maritime)?
6. **Bitmask:** does the candidate access-control requirement have a real consumer? Is exact masked aggregation required, or is ±ε acceptable?
7. **SQL benchmark:** at 30k EPS, 1 Hz and 60 Hz, V = 10/100, K = 1/10/100, does PostgreSQL/Materialize/Flink meet the contract? Include the IVM mask-class variant, not just per-consumer pull.
8. **S5:** is p99 < 250 ms a real requirement? What read rate is paired? Is it anchored?
9. **Papers:** should the two IEEE papers be added to DEC-001’s related work, and where?
10. **Delivery:** does the delivery comparison stay in ANL §9, or move to DEC-001?
11. **Module fit:** COMP4299 or CSAI4299 if the work touches a model?
12. **Deliverables:** dissertation only, or demo and benchmarks?
13. **Next artefact:** full proposal, design-space report, or prototype spike?

Do not choose a path for the user. Present the trade.

---

## 11. What the new agent should do

1. Read DEC-001 v0.4.0, ANL-001 v0.2.0, DISCUSSION-SUMMARY Part XI, and this report.
2. Check `git log` and `git status`; do not stage or commit unless asked.
3. Continue the analysis at the analysis level:
   - Materialize vs the engine: arithmetic, snapshot, masks, memory, persistence, topology.
   - The client CPU and p99 model bases; the K3 lean; S5.
   - The failing-baseline SQL benchmark and the bitmask cost model.
   - The supervisor question set.
4. Do not start the main build; the candidate and anchor are open.
5. Preserve the document conventions:
   - every number is measured, derived, or assumed, and labelled;
   - no weighted scores;
   - no strawman baselines;
   - K3 is self-set and unanchored; access control is a candidate;
   - CPU is unmeasured; client CPU and p99 are models;
   - manifest egress is separate;
   - the topology is a tree;
   - the bitmask is authorization, not privacy;
   - delivery is not compared with Materialize.
6. For reviews, start a fresh session and port the external record; the original is not resumable from a new BUILD session.
7. If asked for next steps, the safe analytical/benchmark work is:
   - the SQL failing baseline: PostgreSQL/Materialize/Flink vs the Rust rollup at 30k EPS, 1/60 Hz, masks; include the IVM mask-class view;
   - the bitmask cost model: per-mask classes, global-minus-unauthorized vs direct canonical reduction, min/max/attention;
   - the S5 anchor and read-rate pairing;
   - the supervisor question set.
   The main build is gated on the candidate/anchor.

---

## 12. Assets and references

| Asset | Path / reference |
| --- | --- |
| Current comparison | `docs/DEC-001__Decision_Options_Brief.md` (v0.4.0) |
| Engine highlights and existing-solutions contrast | `docs/ANL-001__Existing_Solutions_Contrast.md` (v0.2.0) |
| Discussion record | `DISCUSSION-SUMMARY.md` Part XI (§151–§162) |
| Legacy design | `docs/PRD-001`, `docs/TDD-001`, `docs/PLN-001` (legacy references, not source of truth) |
| Review record | `ses_ea8261124ffewIT7wHmP4QtF15`; external path above |
| Previous handover | Git history at `74cdfa4`; this report supersedes it |

**Papers**

- T. Uehara and D. Kasamatsu, “A Method of Consistency Control Based on Latency for Digital Twin Systems,” *2025 International Technical Conference on Circuits/Systems, Computers, and Communications (ITC-CSCC)*, 2025. DOI: `10.1109/ITC-CSCC66376.2025.11137613`.
- S. Rinaldi, S. Dello Iacono, P. Bellagente, and M. Pasetti, “Analysis of Time Synchronization Challenges in Digital Twins for Edge-Enabled Data Centers in Smart Cities Scenario,” *2025 33rd Euromicro International Conference on Parallel, Distributed, and Network-Based Processing (PDP)*, 2025, pp. 506–511. DOI: `10.1109/PDP66500.2025.00078`.
- Anvil Labs, “Reducing Latency in Drone Digital Twin Models,” blog, 2026-03-02. Vendor content; industry context only.

**Competitor issues (verified 2026-10-20)**

- Eclipse Ditto [#1525](https://github.com/eclipse-ditto/ditto/issues/1525) — aggregation in search; 2022, open.
- FIWARE Orion [#3816](https://github.com/telefonicaid/fiware-orion/issues/3816) — aggregation operations API; 2021, open. (The old `FIWARE/orion` URL 404s.)
- tippecanoe [#324](https://github.com/mapbox/tippecanoe/issues/324), [#830](https://github.com/mapbox/tippecanoe/issues/830) — aggregation in the tile pipeline; closed 2017/2020.
- deck.gl [#10795](https://github.com/visgl/deck.gl/issues/10795) — “Tracker: Zoom-dependent layers”; 2026, open; documentation tracker, not user demand.
- Cesium [#6182](https://github.com/CesiumGS/cesium/issues/6182) — vector-tile/LOD aggregation; 2018–2019, closed completed.
- Mirror [#3197](https://github.com/MirrorNetworking/Mirror/issues/3197) — SyncVars to a subset of observers; 2022, closed completed; per-observer visibility.

**Earlier related work:** SHIP (arXiv 2610.21048); edge-IoT hierarchical monitoring (arXiv 2610.09817); Verdantix industrial-twin market; Siemens Digital Twin Composer; NVIDIA Omniverse; Azure 3D Scenes Studio; Maiterth et al., VIS 2024.

---

## 13. Cautions

- **Do not overclaim SQL.** SQL is a language, not one engine.
- **Do not call access control sourced.** It is a candidate; PRD §10 puts auth out of scope.
- **Do not call the topology a DAG.** The node graph is a tree.
- **Do not conflate raw and aggregate masks.**
- **Do not present global-minus-unauthorized as bit-exact.**
- **Do not present `O(B)` as free or unique.**
- **Do not present server CPU, client CPU, or p99 as measured.**
- **Do not treat K3 as sourced.** It is self-set and unanchored; the current lean is that it is likely not realistic.
- **Do not compare delivery to Materialize.** Use the delivery baselines.
- **Do not call the shared snapshot a differentiator.**
- **Do not use the Anvil blog as an anchor.** It is vendor content.
- **Do not call deck.gl #10795 user demand.** It is a documentation tracker.
- **Do not use the old `FIWARE/orion` link.** Use `telefonicaid/fiware-orion`.
- **PRD-001 and TDD-001 are legacy references**, not the source of truth.
- **Do not send a full JSON manifest by default at 100k.**
- **Manifest egress can dominate short sessions.** Report it separately.
- **Do not re-litigate the engine or the demand question.**
- **Do not start the main build** before the candidate and anchor are fixed.
- **Do not stage or commit** for the user.
- **Do not recreate a review session record under the project root.**

---

## 14. One-paragraph summary

DEC-001 v0.4.0 is a corrected, internally consistent design comparison for bounded per-consumer delivery of live telemetry. Its conclusion is negative and conditional: no design has an unconditional advantage; D2f ties the server cut on egress; under K3 the free-camera feasible set is D3/D4 and the comparison is a premium; K3 is self-set, unanchored, and on the current evidence likely not a realistic requirement, so the unconstrained comparison is the realistic baseline. ANL-001 v0.2.0 sets the cut aside and states the engine’s non-cut highlights: exact rollup over a tree with channel wiring, a shared frame snapshot (a contract property, not a differentiator), and candidate mask-aware aggregation. At 30k EPS the scale is commodity; the semantic contract is the differentiator. Materialize, configured to the contract, is the closest packaged aggregation baseline: it matches the simple rollup, wiring, and snapshot; it cannot match IEEE f32 parity; it expresses availability and masks at a cost; delivery is excluded and compared separately. The client CPU and p99 columns are now model outputs with explicit bases. S5 adds a p99 < 250 ms server-to-client variant; it is a cadence requirement, changes the feasible set, and leaves the `N/B` egress ratio unchanged. Two review audits found and fixed consistency issues; the paper reads support the clock-origin and latency–consistency framing but are not anchors. The supervisor conversation remains the gate. No build, no commit.
