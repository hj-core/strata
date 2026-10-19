# BUILD Handover — DEC-001 and the COMP4299 decision

| Field | Value |
| --- | --- |
| Prepared | 2026-10-19 |
| For | a new BUILD agent continuing this work in another workspace |
| Repo | `/home/hj-core/project` |
| Branch | `tdd` |
| HEAD | `75fd948` (“Save”); clean except this report (untracked) |
| Read first | `docs/DEC-001__Decision_Options_Brief.md` (v0.4.0), `DISCUSSION-SUMMARY.md`, this report |
| Predecessor handover | `~/.opencode/build/fyp-build-handover.md` (machine-local; this report supersedes its current-state sections) |

---

## 1. Status at a glance

| Item | State |
| --- | --- |
| Track | **COMP4299 system-development — decided.** Novelty not required; depth from stakeholder and evaluation. |
| Supervisor conversation | **Scheduled, not yet held — this is the gate.** |
| Candidate / anchor | **Open.** No candidate or scenario anchor is fixed. |
| Engine | **Not revived.** The analysis is the transferable asset. |
| DEC-001 | **v0.4.0**, status “Draft — for supervisor discussion”; analysis corrected and simplified. |
| Current user focus | **A correct DEC-001 to discuss with the supervisor. Stay at the analysis of the engine.** |
| Main build | **Not started.** Do not start before the candidate and anchor are fixed. |
| Git | HEAD `75fd948`; clean. Do **not** stage or commit unless the user asks. |

The user’s current instruction is explicit: they need DEC-001 to be correct for the supervisor meeting, and they want the discussion to stay at the **analysis of the engine**, not a pivot to project management or a build.

---

## 2. Current git and document state

- Branch: `tdd`.
- HEAD: `75fd948` (“Save”). The commit includes the latest DEC-001 refinements.
- Working tree: clean except for this report (`BUILD-HANDOVER.md`, untracked). The report is intended to be committed and pulled into the other workspace.
- Recent history:
  - `486b40a` — bump DEC-001 to `0.4.0`; record K3 as self-set.
  - `77ac88e` — correctness pass: bound definitions, D2f placement, S4 preference, manifest egress.
  - `ec9304a` — transpose §4 dimension tables to options-as-rows.
  - `75fd948` — latest refinements (columns, client memory, latency, dedup).
- The commit messages are terse; do not infer scope from them. Read the document.

DEC-001 header currently reads:

- Version `0.4.0`
- Status `Draft — for supervisor discussion`
- Last Updated `2026-10-19`

Do not bump the version again unless asked. `0.4.0` is the current baseline.

---

## 3. What the project is

The document compares designs for delivering live telemetry to 3D digital-twin consumers. The proposed design is a **server-side geometric cut**: the server computes a per-viewer visible set (frontier) and sends only that set, with exact hierarchical aggregation.

The question in §1.2 is:

> Under an anchored reference scenario, which placement of selection and aggregation minimises delivery cost subject to a hard per-client bound (K3, self-set), and under what conditions, if any, does the server-side cut outperform the alternatives?

The answer is deliberately negative:

- **No unconditional advantage.**
- Fine-grained interest delivery (D2f) and the server cut (D3/D4) tie on egress at S0.
- Broadcast is worse by the ratio `N/B` (~5× at S0).
- The server cut’s only unique claim is the **hard per-client bound with server-authoritative relevance**.
- That bound is **K3, a self-set requirement**, not a sourced one.
- Demand for the conjunction is unproven; demand for plain aggregation is documented.

This conclusion is the asset. Do not turn it back into a sales pitch.

---

## 4. User decisions already made

These are settled for the current discussion:

1. **Supervisor conversation is scheduled but has not happened.** It is the gate.
2. **K3 is marked as self-set.** It is recorded in §1.3, §5.5, §6.4, §7 (Path A), and §8 Q4.
3. **DEC-001 is at v0.4.0.** The version bump was requested and applied.
4. **The current task is a correct DEC-001 for the supervisor.** The user is not asking for a build, a path recommendation, or a new topic.
5. **Do not stage or commit for the user.** The user has been committing the document themselves.

Do not re-open these unless the user does.

---

## 5. Current DEC-001 content and key numbers

### 5.1 Base scenario

- 10,000 devices (≈30,000 tags), three metrics per device, 1 Hz reading rate.
- 10 viewers, cloud/WAN, f32 values.
- Entry budget `B = 2,000`.
- Group hierarchy ~180 groups, 3 channels per group.
- Full state ~122 KB/tick; full-state delta ~65 KB.
- Frontier ~13 KB/s; D2c region ~33 KB/s; D6 room ~4 KB/s.
- Egress rates at $0.09/GB: 65 KB/s → $0.021/h; 13 KB/s → $0.0042/h; 33 KB/s → $0.011/h; 4 KB/s → $0.0013/h. A continuous month is 730 h.

### 5.2 Manifest egress

The manifest is a **connection/session cost**, not a per-tick rate. §5.3 reports it separately:

- ~4 MB at 10k devices, ~40 MB at 100k (full JSON; format-dependent).
- At 13 KB/s frontier, ~5 min of frontier-equivalent egress at 10k, ~51 min at 100k.
- Over an 8 h session: ~0.14 KB/s at 10k, ~1.4 KB/s at 100k.
- Reduced by compact binary, client cache/versioning, and a lazy manifest.
- A full JSON manifest is a bad default at 100k.

### 5.3 CPU

CPU is **unmeasured**. The only CPU figures are model hypotheses:

- `T_cut ≈ 0.05 ms + bB`, `T_encode ≈ cB`.
- `b ∈ [0.06, 0.19] µs`, `c ∈ [0.03, 0.07] µs`.
- At `B = 2,000`: ~0.23–0.57 ms per cut+encode.
- ~0.014–0.034 core/viewer at 60 Hz.
- At 10 Hz the model overlaps routing; it is a model tie.

Do not cite `V_cpu ≈ 320`, `8×`, or `1.3×` as measured. Do not present the range as a result.

### 5.4 Client memory and `O(B)`

The document’s claim is **`O(B)` only with a lazy manifest**, not with server-side selection alone. The discussion clarified what `O(B)` actually requires:

- A compacted B-slot buffer, not an N-slot identity buffer.
- A sparse `node ↔ slot` map over B entries (slot table, hash map, or sorted array), not a dense array over N.
- A dynamic mapping updated on membership change; swap-remove can move the last entry.
- Client CPU `O(changes)` per tick (worst case `O(B)`), plus `appeared`/`removed` events and keyframes on desync.

The document’s total footprint is ~0.1–0.4 MB at `B = 2,000`. The discussion’s constant-factor estimate including the instance buffer is ~0.2–0.3 MB at `B = 2,000`, and ~0.5–0.75 MB at `B = 5,000`.

`O(B)` is **not unique to the server cut**:

- D6 can achieve `O(subtree)` with a scope-limited lazy manifest.
- D2f approximates `O(B)` via a coarse-top lazy descent, at a round-trip cost.

The hard bound (egress) and `O(B)` (client memory) are separate claims. Do not conflate them.

### 5.5 The `O(B)` judgment from the discussion

The discussion concluded that `O(B)` is worth the engineering hurdle only in a narrow band:

- Above ~0.3 MB and below ~4 MB at 10k devices (or below ~40 MB at 100k).
- Above the `O(N)` manifest size, the manifest fits and `O(B)` buys nothing.
- Below the `O(B)` footprint, `O(B)` also fails; a smaller `B` or a different delivery model is needed.
- S2 has no anchored client class, so the band is currently speculative.
- At S0, `O(B)` buys nothing observable: 4 MB is trivial for a laptop.

This is a discussion judgment, not a measured result; label it as such if it enters the document.

---

## 6. What changed in this session

The following corrections and refinements are in the current document. They are the important context for a new agent.

### 6.1 Bound definition and the unique claim

- §2.3 now defines **hard bound** (sender-enforced cap, sender determines the delivered set) and **soft bound** (depends on the client's selection/subscription).
- The server cut’s unique claim is now stated as a **hard bound with server-authoritative relevance**, not just “a hard bound.” A capped subscription can enforce a byte cap; it cannot guarantee server-authoritative relevance.
- §4.2 and §5.1 were corrected: a client cut over a **broadcast source** cannot bound egress; a client cut **with a subscription** (D2f) can, softly. The old blanket “a client-side cut cannot bound egress” contradicted D2f.
- §5.1 rule 3 now says *strict* `O(B)` requires server-side selection; D2f approximates it with a coarse-top lazy descent.

### 6.2 S4 and S2

- S4 now says D3/D4 are preferred **only if the hard bound is required**; otherwise D2f on cost. D2f ties on egress, so the hard bound is the only separator.
- §6.4 no longer says “D4 wins S2 outright”; it says D4’s memory claim becomes anchored, while D2f remains a trade at round-trip and extra-egress cost.
- Path A now says “an anchored thin-client class (S2) or scatter (S4); K3 is adopted as self-set,” not “a sourced hard bound.”

### 6.3 CPU and manifest

- The last unlabelled CPU point (`~0.25 core at V = 10`) was replaced with “model 0.14–0.34 core at V = 10 (unmeasured).”
- All CPU figures are labelled model/unmeasured.
- Manifest egress was added as a separate connection cost in §5.3; Table 5.3 still excludes it.
- Appendix B discloses that the 1.5-slot average is an assumption; under independent per-slot `p = 0.5` the slot-table estimate is ~14 KB/s, not ~13 KB/s. The ranking is unchanged.

### 6.4 Table 4.1

Table 4.1 now has options as rows and these columns:

`Model + placement | Delivered set | Membership | Supported transports | Dominant per-viewer server work (unmeasured) | Client memory | Added selection-path latency | Bound | Verdict / role`

Key changes:

- The server-work column was renamed and given a **column-basis note**: it is per-viewer marginal work, `C_server(V) = C_shared + V × C_per_viewer`, excluding shared state update, aggregation, and serialization.
- **Client memory includes the manifest.** Without a lazy or scope-limited manifest, every row is `O(N)`. The explicit rows now read `O(N) manifest + O(subtree) data; O(subtree) with scope-limited lazy manifest`.
- D6 in §5.3 now reads `~4 MB + subtree (~0.1–0.2 MB); ~subtree with lazy manifest`.
- The latency column was renamed **“Added selection-path latency”** and the cells now name the path: no added path for broadcast; subscription RTT for D2f; camera uplink + cut + downlink for the server cut; subscription RTT for explicit; none for None.
- “Geometric + client (broadcast source)” was renamed **“Geometric + client (broadcast)”**.
- The notes under Table 4.1 were reordered and deduplicated: reading the table → model notes → broadcast/None equivalence → server-work basis → client memory → added selection-path latency → membership/transport.

### 6.5 §4.2

§4.2 was simplified and deduplicated:

- Removed the opening meta-sentence and the bullets that repeated it.
- The three placement cases are one paragraph: client broadcast cannot reduce egress; client subscription (D2f) can but is soft; geometric + server hard-bounds at the cost of per-viewer CPU and the camera loop.
- The camera-loop and client-cull consequences are stated directly.
- The long “rendering selection is always present” paragraph was compressed to a cross-reference to §4.1 plus the trade: server CPU vs client memory/client CPU.
- “Server placement produces a hard bound” was corrected to **“Geometric + server produces a hard bound,”** because explicit + server (D6) is soft unless capped.

### 6.6 §4.3–§4.8 and Appendix B

- The dimension tables in §4.3, §4.4, §4.5, §4.6, §4.8 and Appendix B were transposed to **options as rows**. §4.6 is a matrix: encodings are rows, transports are columns; §4.7’s reference was updated to “rows.”
- §4.8 now has a **“What O(B) requires”** note: compacted B-slot buffer, sparse `node ↔ slot` map, dynamic mapping, client CPU `O(changes)`, protocol events. Without compaction, the client is `O(N)` and the claim fails.

---

## 7. Analytical points to preserve — do not re-litigate

These are settled in the current analysis:

1. **The cut is commodity** (3D Tiles/Cesium, Potree, OSG, interest management).
2. **Per-instance selection is not provided by libraries**; you compute it and keep a node↔instance map.
3. **Exactness is commodity** (SQL, IVM, CRDTs, stream processors).
4. **The residue is the coupling** (selection ⊗ aggregation); the demand hunt found it unrequested.
5. **Prior art is the baseline, not a disqualifier.**
6. **No unconditional advantage.** D2f ties the server cut on egress in S0/S1/S3/S4; in S2 it approaches `O(B)` memory only via a coarse-top lazy descent, at a round-trip and extra-egress cost.
7. **View-dependence is a delivery design choice**, not a requirement; rendering selection is always the client's.
8. **The self-set requirements are not sourced:** the hard bound (K3), cross-viewer consistency, exact aggregation.
9. **Deterministic constants cancel:** duty cycle, clock speed, value width, `p` scale.
10. **The egress ratio is `N/B`**, not a bandwidth saving.
11. **Split is not a separate placement**; a server-coarse/client-fine split is a hybrid variant, not evaluated.
12. **D2f is client-side selection + frontier subscription**, not “client placement always broadcasts.”
13. **TDD/PRD budgets are design targets, not measurements.**
14. **CPU is unmeasured.** The model range is a hypothesis.
15. **Manifest egress is separate** from per-tick egress; report it as a connection cost.
16. **`O(B)` is a property of the lazy manifest + compaction + sparse map + dynamic mapping**, not of the server cut alone.
17. **The hard bound and `O(B)` are separate claims.** K3 is egress; `O(B)` is client memory.
18. **The free camera is the engine’s niche, not the modal consumer.** D6 is the guided-view baseline.

---

## 8. Open questions for the user and supervisor

The supervisor conversation is the gate. The user may return with answers. Ask, in this order:

1. **Has the supervisor conversation happened?** What line has data or hardware this term?
2. **Is K3 acceptable as a self-set requirement**, or must the bound be sourced from a standard or deployment?
3. **Is the free-camera profile the intended S0 requirement**, or is the modal consumer a guided view/dashboard? If guided, D6 is the baseline and the cut is unnecessary.
4. **What counts as a thin client (S2)?** What is the memory budget, and does it count the JS heap, GPU buffers, or both?
5. **Is there an S4 anchor** (fleet/maritime, no shared point of presence)?
6. **COMP4299 or CSAI4299** if the work touches a model?
7. **Deliverables:** dissertation only, or demo and benchmarks?
8. **Next artefact:** full proposal, design-space report, or prototype spike?
9. **Path A/B/C:** A is keep-and-narrow (S2/S4, ~200 h); B is comparison as contribution (~150–180 h); C is change topic (~120–160 h). No decision has been made. The analysis supports B as the default if no anchor appears; A only if the supervisor supplies a scenario anchor.

Do not choose a path for the user. Present the trade.

---

## 9. What the new agent should do

1. Read `docs/DEC-001__Decision_Options_Brief.md` (v0.4.0), `DISCUSSION-SUMMARY.md`, and this report.
2. Check `git log` and `git status`; do not stage or commit unless asked.
3. Continue the discussion at the analysis level. The user is preparing for the supervisor; do not pivot to a build, a new topic, or a path recommendation unless asked.
4. If editing, preserve the document’s conventions:
   - every number is measured, derived, or assumed, and labelled;
   - no weighted scores;
   - no strawman baselines;
   - K3 is self-set and declared;
   - CPU is unmeasured;
   - manifest egress is separate.
5. If asked for next steps, the safe builds are:
   - CPU microbenchmark (cut and encode; fanout and routing; fixed host; mean/p99);
   - manifest format/egress spike (JSON vs compressed vs binary; 10k/100k; session lengths);
   - failing-baseline spike (IVM + broker + entitlement) only if Path A/B is live.
   The main build is gated on the candidate/anchor.
6. Do not re-litigate the engine or the demand question; both are settled in the analysis.
7. Do not invent a stakeholder or a scenario. The external anchor must carry justification.

---

## 10. Assets and references

| Asset | Path / reference |
| --- | --- |
| Current comparison | `docs/DEC-001__Decision_Options_Brief.md` (v0.4.0) |
| Review record | `DISCUSSION-SUMMARY.md` (~20k words; §1–§141) |
| Legacy design | `docs/PRD-001`, `docs/TDD-001`, `docs/PLN-001` |
| Predecessor handover | `~/.opencode/build/fyp-build-handover.md` |
| Papers | SHIP (arXiv **2610.21048**); edge-IoT hierarchical monitoring (arXiv **2610.09817**) |
| Competitor issues | Ditto **#1525**; Orion **#3816**; tippecanoe **#324/#830**; deck.gl **#10795**; Cesium **#6182**; Mirror **#3197** |
| Demand | Verdantix industrial-twin market; Siemens Digital Twin Composer; NVIDIA Omniverse; Azure 3D Scenes Studio; Maiterth et al., VIS 2024 |

---

## 11. Cautions

- **The crudest trap:** K3 is self-set. If the supervisor rejects a self-set bound, the unique claim dissolves and the comparison’s conclusion does not move.
- **Do not present `O(B)` as free or unique.** It requires a lazy manifest, a compacted buffer, a sparse map, and dynamic remapping; D6 can do `O(subtree)` with a scope-limited lazy manifest; D2f approximates `O(B)`.
- **Do not present the CPU range as measured.** `b`, `c`, fanout, and routing are unmeasured.
- **Do not send a full JSON manifest by default at 100k.** Use compact binary, version/cache, and/or lazy manifest.
- **Manifest egress can dominate short sessions.** Report it separately.
- **Do not conflate the hard bound (egress/K3) with `O(B)` (client memory).**
- **Do not re-litigate** the engine or the demand question; both are settled.
- **Do not start the main build** before the candidate and anchor are fixed.
- **Do not stage or commit** for the user.

---

## 12. One-paragraph summary

DEC-001 v0.4.0 is a corrected, internally consistent design comparison for bounded per-consumer delivery of live telemetry. Its conclusion is negative and conditional: no design has an unconditional advantage; D2f ties the server cut on egress; the server cut’s only unique claim is a hard per-client bound with server-authoritative relevance; that bound is K3, a self-set requirement; demand for the conjunction is unproven. The document now separates the hard bound from `O(B)` client memory, makes the manifest a separate connection cost, labels all CPU figures as an unmeasured model, and states that `O(B)` requires a lazy manifest plus compaction, a sparse map, and dynamic remapping. The user is preparing for a scheduled supervisor meeting and wants the discussion to stay at the analysis of the engine. The next move is the supervisor conversation, not a build.
