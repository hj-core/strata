# PLN-001 — Master Execution Sequence

| Field | Value |
| --- | --- |
| Document ID | PLN-001 |
| Title | Master Execution Sequence |
| Version | 0.1.1 |
| Status | Draft |
| Date | 2026-09-20 |
| Owner | Project maintainer |
| Related documents | Upstream: PRD-001 (Product Requirements), TDD-001 (Technical Design). |

## 1. Purpose

This document is the single execution sequence for the project. It fixes the order of work, the dependencies between activities, the milestone gates, and the cut-down ladder (engineering rungs pre-approved; scope-reducing rungs require a PRD-001 amendment). It derives its scope from PRD-001 and its technical sequence from TDD-001. It is the reference the maintainer follows each week and the instrument by which slippage is detected early. The project is built in public: every activity's output is committed to, and written up in, the public GitHub repository as it is completed. This is a personal project with no external deadline or contributions; the 25-week cadence is self-imposed.

## 2. Timeline at a Glance

The roadmap spans 25 weeks (2026-09-14 → 2027-03-01), run with a weekly public build-log and milestone-review cadence.

| Phase | Weeks | Dates | Focus |
| --- | --- | --- | --- |
| P1 Research | 1 | 2026-09-14 | Prior-art & reference review |
| P2 Architecture | 2–4 | 2026-09-21 → 2026-10-05 | Config, protocol, architecture spec |
| P3 Engine core | 5–10 | 2026-10-12 → 2026-11-16 | State, index, aggregation, culling, alpha |
| P4 Transport | 12–13 | 2026-11-30 → 2026-12-07 | WebSocket, quantised delta pipeline |
| P5 Benchmarking | 11, 14–16 | 2026-11-23; 2026-12-14 → 2026-12-28 | Simulator, KPI + scaling + ablation |
| P6 Client integration | 17–20 | 2027-01-04 → 2027-01-25 | Three.js SDK, picking, demo, testing |
| P7 Docs & Launch | 21–25 | 2027-02-01 → 2027-03-01 | Documentation, demo video, v1.0 release |

Activities A11 (simulator) and A12–A13 (transport) interleave: the simulator is produced in week 11, before the transport work in weeks 12–13.

## 3. Workstreams / Components

| Component | Activities | Deliverable | Design reference |
| --- | --- | --- | --- |
| Research | A1 | Prior-art & reference review | — |
| Architecture | A2–A4 | Config schema, protocol spec, architecture spec | TDD §4, §9 |
| Engine | A5–A10 | Working engine, semantic hierarchy index, backend alpha | TDD §5–§8 |
| Transport | A12–A13 | WS transport, quantised delta pipeline | TDD §9 |
| Benchmark | A11, A14–A16 | Simulator, baseline KPIs, scaling envelope, report draft | TDD §13–§14 |
| Client SDK | A17–A20 | Client SDK, end-to-end demo, working prototype | TDD §10 |
| Docs & Launch | A21–A25 | Documentation set, demo video, v1.0 release | — |

## 4. Milestone Gates

| Gate | Name | Date | Exit condition |
| --- | --- | --- | --- |
| M1 | Architecture frozen | 2026-10-05 (A4) | A2–A4 documents reviewed and merged; benchmark host spec and the fixed benchmark topology recorded (FR-BR-05). |
| M2 | Ingestion + index working | 2026-10-19 (A6) | Readings ingested; index built and passing invariant scan. |
| M3 | Aggregation correct | 2026-10-26 (A7) | Per-device state-row integrity invariant enforced; aggregates match scalar reference on fixtures. |
| M4 | Culling prototype | 2026-11-02 (A8) | Scalar frustum traversal + expansion budget demonstrated. |
| M5 | Backend alpha | 2026-11-16 (A10) | Engine passes integration tests; unit/property suite green. |
| M6 | Transport complete | 2026-12-07 (A13) | f16 + delta + keyframe + resync pipeline working. |
| M7 | Benchmark evidence | 2026-12-28 (A16) | K1–K5, K8–K10 measured at 4,280/50k/100k; ablation if schedule permits (FR-BR-03, Should); report draft. |
| M8 | End-to-end prototype | 2027-01-25 (A20) | Full demo runs simulator → engine → browser at building scale (~4,280 devices); client KPIs (K6, K7, K11) met. |
| M9 | Documentation complete | 2027-02-15 (A23) | Full documentation set and public write-up series complete. |
| M10 | v1.0 release | 2027-03-01 (A25) | v1.0 tagged and published; repository license selected; demo video released. |

## 5. Weekly Execution Sequence

| # | Activity | Week | Component | Deliverable | Date | Depends on |
| --- | --- | --- | --- | --- | --- | --- |
| A1 | Prior-art & reference review | 1 | Research | — | 2026-09-14 | — |
| A2 | Topology config design, data model | 2 | Architecture | Draft config schema | 2026-09-21 | A1 |
| A3 | Binary protocol design, memory layout | 3 | Architecture | Protocol spec | 2026-09-28 | A2 |
| A4 | Architecture doc finalisation | 4 | Architecture | Architecture spec doc; fixed benchmark topology (FR-BR-05) | 2026-10-05 | A2, A3 |
| A5 | Atomic state table, basic ingestion | 5 | Engine | Working ingestion prototype | 2026-10-12 | A4 |
| A6 | Semantic index build (DFS layout, ids, AABBs, subtree sizes, device-row state layout) | 6 | Engine | Working semantic index | 2026-10-19 | A5 |
| A7 | Hierarchical aggregation, contiguous-block invariant, and the reserved attention channel; SIMD stretch (OI-4, OI-8) | 7 | Engine | Hierarchical aggregation | 2026-10-26 | A6 |
| A8 | Frustum-culling traversal, expansion budget (scalar reference first) | 8 | Engine | Frustum-culling prototype (scalar path) | 2026-11-02 | A7 |
| A9 | Boot-time validation, unit tests; parallel client spike (InstancedMesh + VBO hello-world) | 9 | Engine | Backend unit test suite; boot-time measurement; client spike findings | 2026-11-09 | A8 |
| A10 | Backend integration testing | 10 | Engine | Backend alpha | 2026-11-16 | A9 |
| A11 | Synthetic telemetry generator | 11 | Benchmark | Telemetry simulator | 2026-11-23 | A4 |
| A12 | WebSocket server, binary frame format | 12 | Transport | Working WS transport | 2026-11-30 | A10, A11 |
| A13 | f16 quantizer, delta encoder, per-value ε-filter (metric/channel), keyframe snapshots, session re-sync | 13 | Transport | Quantised delta pipeline | 2026-12-07 | A12 |
| A14 | Single-facility benchmarks (4,280 nodes) | 14 | Benchmark | Baseline KPIs | 2026-12-14 | A4, A13 |
| A15 | Intermediate (50,000) and full (100,000) benchmarks; concurrent-session run | 15 | Benchmark | Scaling envelope | 2026-12-21 | A14 |
| A16 | Ablation studies (sibling ordering on/off; FR-BR-03, Should — if schedule permits); benchmark report draft | 16 | Benchmark | Benchmark report draft (client KPIs appended after A20) | 2026-12-28 | A15 |
| A17 | Three.js InstancedMesh, VBO writer | 17 | Client SDK | Client SDK alpha | 2027-01-04 | A16 |
| A18 | Object picking; camera-view publisher | 18 | Client SDK | Full client SDK | 2027-01-11 | A17 |
| A19 | Demo application integration | 19 | Client SDK | End-to-end demo (building scale, ~4,280 devices) | 2027-01-18 | A18 |
| A20 | End-to-end testing, bug fixing | 20 | Client SDK | Working prototype | 2027-01-25 | A19 |
| A21 | Documentation & write-up (first half, incl. refreshed prior-art review) | 21 | Docs & Launch | Write-up series (first half) | 2027-02-01 | A20 |
| A22 | Documentation & write-up (second half) | 22 | Docs & Launch | Write-up series (second half) | 2027-02-08 | A21 |
| A23 | Documentation finalisation (overview, conclusions, revisions) | 23 | Docs & Launch | Complete documentation set | 2027-02-15 | A22 |
| A24 | Demo video recording | 24 | Docs & Launch | Demo video | 2027-02-22 | A20, A23 |
| A25 | v1.0 release preparation and public launch (incl. repository license) | 25 | Docs & Launch | v1.0 release | 2027-03-01 | A23, A24 |

## 6. Critical Path

```
A1 → A2 → A3 → A4 → A5 → A6 → A7 → A8 → A9 → A10
                                                   │
             A11 ─────────────────────────────────► A12 → A13 → A14 → A15 → A16
                                                                          │
                                                              A17 → A18 → A19 → A20
                                                                                │
                                                              A21 → A22 → A23 → A24 → A25
```

The longest path runs through the engine core (A5–A10) into transport (A12–A13), benchmarking (A14–A16), and finally the client and documentation/launch. Any slip in A5–A10 propagates to every downstream gate; these weeks carry the highest schedule risk. A11 is off the critical path and can float within week 11.

## 7. Weekly Checkpoint Protocol

For every week:

1. **Plan** — confirm the week's activity, deliverable, and exit criteria (from §5).
2. **Execute** — work the activity to its stated deliverable.
3. **Verify** — run the relevant tests/measurements; record raw numbers for benchmark weeks.
4. **Publish** — commit the work and post a short public build-log entry (progress, screenshots, numbers).
5. **Decide** — state progress, blockers, and any use of the cut-down ladder; if behind, invoke the next pre-approved rung (§9) immediately rather than at the next gate. A scope-reducing rung (2 or 4) requires a PRD-001 amendment before it is invoked.

## 8. Definition of Done

An activity is done only when:

- Its deliverable exists and is committed and pushed to the public repository.
- Relevant automated tests pass (`cargo nextest`, lint/format clean).
- Any KPI it informs has a recorded measurement, not an estimate.
- Findings are written down (spike notes, benchmark log, or public write-up).
- The next activity's dependencies are confirmed satisfied.

## 9. Cut-Down Ladder

If the schedule slips, apply the ladder in order among the pre-approved rungs. Do not improvise alternatives before exhausting the list; a scope-reducing rung (2 or 4) pauses for a PRD-001 amendment. Rungs 1 and 3 are engineering cuts and are pre-approved. Rungs 2 and 4 reduce the benchmark tier coverage defined by PRD-001 §9 and §12, so they are **not pre-approved**: invoking one requires an explicit, versioned amendment to PRD-001, disclosed in the benchmark report (FR-BR-04). Until such an amendment exists, all three tiers remain in scope.

| Rung | Action | Preserves |
| --- | --- | --- |
| 1 | Ship the scalar path first; defer SIMD. | Correctness, K2/K3/K4 |
| 2 | Restrict benchmarks to building scale (~4,280) — requires a PRD-001 amendment. | Core claims, K5 building tier |
| 3 | Revert picking to CPU `Raycaster`; drop LOD transitions. | Demo usability |
| 4 | Defer the 100k-device tier to "future work"; ship 50k results — requires a PRD-001 amendment. | Honest, complete deliverables |

Client weeks (A17–A20) and documentation/launch weeks (A21–A25) are touched last. The A9 client spike deliberately removes client unknowns early, and A14–A16 use the festive weeks for machine-bound, interruptible work.

## 10. Risk-Aware Scheduling

| Risk | Scheduled response |
| --- | --- |
| R1 index bugs (A6) | Unit + property tests from A5; invariant scan from A6; independent flat-list brute-force reference for traversal/selection. |
| R2 SIMD bugs (A7) | Scalar reference kept alongside; SIMD is explicit stretch (aggregation/encoding). |
| R3 group invariants (A7) | Build-time assertions and contiguity tests at A7. |
| R4 Three.js failures (A17) | Early InstancedMesh spike at A9; WebGL inspector at A17. |
| R5 latency (A8, A14–A16) | Expansion budget and pruning designed in at A8; measured at A14–A16. |
| R6 simulator starvation (A15) | `taskset` pinning; pre-generated traces. |
| R7 flicker (A8/A17) | Hysteresis + keyframe-on-expand; cross-fade fallback. |
| R8 slippage | Ladder in §9 (scope-reducing rungs require a PRD-001 amendment); phases P6/P7 last. |
| R9 configuration validity (A6) | Boot-time validation rejects structural and channel/unit inconsistencies (FR-TD-07, FR-TD-08). |

## 11. Dependencies and Parallel Work

- A11 (simulator) can proceed in parallel with A10 once A4 is frozen.
- A9's client spike runs in parallel with backend testing to de-risk A17.
- Documentation (A21+) should absorb the A1 prior-art review; A21 explicitly refreshes it.
- The benchmark report draft (A16) is completed only after client KPIs (K6, K7, K11) are appended from A20.

## 12. Deliverable Acceptance Gates

| Deliverable | Gate | Date |
| --- | --- | --- |
| Architecture spec | M1 | 2026-10-05 |
| Backend alpha | M5 | 2026-11-16 |
| Quantised delta pipeline | M6 | 2026-12-07 |
| Benchmark report draft | M7 | 2026-12-28 |
| Working prototype | M8 | 2027-01-25 |
| Complete documentation set | M9 | 2027-02-15 |
| v1.0 release + demo video | M10 | 2027-03-01 |

## 13. Out of Scope for This Plan

- Production deployment, operational runbooks, and long-term maintenance.
- Real device-protocol integration.
- Anything listed as out of scope in PRD-001 §10.
