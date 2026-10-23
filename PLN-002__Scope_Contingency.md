# PLN-002 — Scope Contingency Plan for the PoD E2E Slice

| Field | Value |
| --- | --- |
| Document ID | PLN-002 |
| Title | Scope contingency plan for the proof-of-delivery E2E slice |
| Version | 0.1.0 |
| Status | Draft — companion to PRP-001; for supervisor discussion |
| Last Updated | 2026-10-22 |
| Related documents | PRP-001 §9.2 (WBS), §9.4 (cut-down ladder), §10 (gates); ANL-002; TUT-001 |
| Purpose | Keep the 290 h plan as the target and pre-authorize scope reductions if the schedule slips |
| Supersedes | PRP-001 §9.4, for schedule-driven cuts only (see §7). PRP-001 otherwise remains the baseline. |

---

## 1. Purpose and rule

The baseline remains the 290 h plan at 18 h/week over 16 weeks (PRP-001 §9.2). That is **288 h of calendar capacity against 290 h of estimated work**, so the plan has no slack. A slip cannot be absorbed; it must become a scope decision.

This plan makes that decision in advance:

1. The 290 h plan stays the target.
2. The work is split into a **protected core** and a **contingency pool** (§2).
3. Reductions are triggered by **milestone status and effort variance**, not by how the project feels (§4).
4. Tiers 1–2 are **pre-authorized**; the supervisor is informed at the checkpoint.
5. Tiers 3–4 **change the claim** and require written supervisor sign-off (§3).
6. Every cut is recorded in the decision log (§6).

The lower contribution bar changes what may be cut, not what must be built. Evaluation depth is optional; the demonstration is not.

All hours in this document are **planning estimates**, not measurements. They allocate the PRP-001 §9.2 estimates; they do not add a second budget.

---

## 2. Protected core and contingency pool

### 2.1 Protected core — ~217 h

| Work package | Hours | Why it is protected |
| --- | --- | --- |
| Requirements, threat model, standards review | 20 | The report's spine; needed even for a demo. |
| PoD profile, statement schema, signing client | 35 | The application itself. |
| Log integration and RFC 9942 receipt wrapper | 25 | Core level-3 work; the main technical risk. |
| Witness integration | 15 | The equivocation demonstration. |
| Verifier and dispute bundle | 15 | Proof that the slice works offline. |
| Offline queue and TSA (of the 45 h line) | 20 | The handover case is not credible without it. |
| Minimal privacy (hashes only) | 2 | A design decision, not a study. |
| Test harness (of the 30 h line) | 15 | Makes the demo repeatable. |
| Core artifact documentation | 10 | Reproducibility. |
| Demo evaluation (timings, four attacks) | 5 | The "it works" evidence. |
| Dissertation | 55 | The deliverable. |
| **Core total** | **~217** | |

### 2.2 Contingency pool — ~73 h

| Work package | Hours | What is lost if cut |
| --- | --- | --- |
| Evaluation beyond the demo: 30-day workload, L1/L2/L5 comparison | 25 | H5 becomes discussion, not a test; no comparative numbers. |
| Revocation study (of the 45 h line) | 25 | C3 reduces to a minimal signed status list; the compromise-window analysis becomes a paragraph. |
| Red-team beyond the core five | 15 | Detection claims are limited to the demonstrated cases. |
| Privacy/linkage study | 3 | C5 becomes a short design section. |
| Artifact documentation polish | 5 | Reproducibility is thinner; the demo still runs. |
| **Pool total** | **~73** | |

**217 + 73 = 290.** The pool is the part that turns a working system into a research evaluation. Under the lower contribution bar, it is the right part to spend first.

---

## 3. The ladder

Cuts are cumulative and applied in order. Do not skip a tier to protect a later one (the spike exception is in §3.2).

| Tier | Trigger | Cut | Hours freed | Nominal remaining | Claim after cut | Authority |
| --- | --- | --- | --- | --- | --- | --- |
| **0** | On schedule | None | — | 290 | Full proposal | — |
| **1** | End of week 5: M2 late ≤1 week, or total effort 10–20% over plan | Evaluation beyond the demo, privacy study, docs polish | ~33 | ~257 | Demo intact; H5/H6 become discussion or **untested** | Pre-authorized; inform supervisor |
| **2** | End of week 8: M3 late, or 20–30% over plan | Revocation study to a minimal status list (keeps ~5 h); red-team to the core five | ~35 | ~222 | C3 reduced; detection claims limited to demonstrated cases | Pre-authorized; inform supervisor |
| **3** | End of week 10: M4 late, or >30% over plan; **or** spike day 7/10 fails | **First:** drop the RFC 9942 COSE receipt encoding; verify Tessera's inclusion proof directly against a signed checkpoint. **Only if witnesses are the blocker:** drop to one witness | ~20 (or ~15) | ~202 (or ~207) | "RFC 9943 roles, simplified receipt encoding" **or** "L2; equivocation not detected" | **Written supervisor sign-off** |
| **4** | End of week 12: M5 late and the deadline is fixed | Floor: L2 hash chain + TSA + verifier + demo; no witnesses, no receipts | ~40 | ~162–167 | Working PoD; rewrite and truncation only | **Written sign-off and a scope-change note** |

### 3.1 Why the wrapper fallback comes before the witness drop

The RFC 9942 wrapper is the custom 25 h item and the most likely blocker. Replacing it with a Tessera proof checked against a signed checkpoint preserves:

- inclusion;
- the signed checkpoint;
- the two-witness quorum;
- the equivocation demonstration.

It loses the standardized COSE receipt encoding, which is a format choice, not the security property. Dropping a witness instead would remove the property that distinguishes level 3 from level 2. Cut the encoding before the quorum.

### 3.2 Tier 3 is a claim change, not a schedule tweak

Tier 3 and Tier 4 alter what the dissertation may assert. They require:

1. a written note to the supervisor naming the lost property;
2. a decision-log entry (§6);
3. a short review pass on the amended scope before the write-up is finalized.

Do not apply Tier 3 silently because week 10 is busy.

**Exception — the spike trigger.** If the day-7 or day-10 gate fails, apply the Tier 3 **wrapper branch only** immediately, without first executing Tiers 1–2. That is an encoding decision, not a full ladder descent; Tiers 1–2 may still be applied later if the schedule also slips. The witness branch is not available at the spike: a failed witness integration is a reason to revise the default or extend, not to drop the property in week 2.

### 3.3 Deadline flexibility

If the deadline can move, **extend to 18–20 weeks before applying Tier 3 or Tier 4**. PRP-001 §9.1a already prefers an 18–20 week plan over compression. Extending preserves the claim; cutting does not.

---

## 4. Checkpoints and trigger rule

### 4.1 Checkpoints

Cumulative hours include the dissertation, which runs in parallel from week 2. They are an effort check, not a build-only measure.

| Week | Cumulative planned | Gate |
| --- | --- | --- |
| 2 | 36 h | M1: Merkle tree + inclusion/consistency proofs; spike day 7 minimal M2 |
| 5 | 90 h | M2: RFC 9162 receipt in COSE, verifies |
| 8 | 144 h | M3: Tessera + checkpoint + witnesses |
| 10 | 180 h | M4: PWA capture + co-sign + receipt |
| 12 | 216 h | M5: offline queue + TSA + dispute bundle |
| 14 | 252 h | M6: red-team + evaluation |
| 16 | 288 h | M7: results, discussion, revision |

### 4.2 Traffic light

At each checkpoint, classify the project:

- **Green** — milestone met and effort within 10% of plan. Continue.
- **Amber** — milestone late by ≤1 week, or effort 10–20% over. Apply the first item of the next tier immediately; do not wait for the next checkpoint.
- **Red** — milestone late by >1 week, or effort >20% over. Apply the whole tier.

### 4.3 Early triggers

The spike gates are earlier and more informative than the week-5 checkpoint:

| Spike day | Gate | If it fails |
| --- | --- | --- |
| Day 7 | One statement, one receipt, one verification, end to end (minimal M2) | Do not proceed to the UI. Apply the Tier 3 wrapper fallback, or change the VDS/verifier. |
| Day 10 | Checkpoint and one witness; the verifier rejects a bad checkpoint | Revise the witness/default combination before M3; if it still fails, apply Tier 3 (witness branch). |

### 4.4 Variance rule

Any single work package that exceeds its estimate by **25%** is a stop-and-replan event. Do not absorb it silently; the usual failure is a multi-day crypto debugging session that quietly consumes the evaluation budget.

---

## 5. What is never cut

Even at Tier 4:

1. The **tamper, truncate, and equivocate** demonstration (equivocate only while witnesses remain).
2. The **"what this does not prove"** section: assertions and inclusion, not physical truth; evidence, not a qualified trust service.
3. The **55 h dissertation floor**. Writing is the deliverable, not overhead.
4. The **decision log** (§6). The cuts become the limitations chapter.
5. The **honest labelling** of untested hypotheses. If H5 or H6 is cut, report it as untested, not confirmed.

Do not call the result "RFC compliant" without the conformance suite. Do not call level 3 a blockchain. Do not claim legal non-repudiation.

---

## 6. Decision log

Every cut is recorded within 24 hours of the checkpoint.

| Field | Meaning |
| --- | --- |
| Date | When the cut was decided |
| Trigger | The milestone slip or effort variance that fired it |
| Tier | 1–4 |
| Packages cut | The exact PRP-001 §9.2 lines reduced or removed |
| Hours freed | Planning estimate, not measured |
| Claim impact | Which hypothesis, contribution, or property is lost or weakened |
| Supervisor notified | Yes/No; for Tiers 3–4, attach the written sign-off |
| Evidence | Timesheet, milestone checklist, spike result |

The log becomes the limitations section. A cut that is not logged is a silent scope change, which is the failure this plan exists to prevent.

---

## 7. Relationship to PRP-001 and the review

- **PRP-001 remains the baseline and is not edited by this plan.** Its version stays stable.
- **PRP-001 §9.4 is superseded only for schedule-driven cuts.** Its instruction "do not cut the witness protocol or the red-team suite" continues to apply at Tiers 0–2. Tiers 3–4 are claim-changing and require sign-off, as above.
- If Tier 3 or Tier 4 is invoked, add a **one-page scope amendment** to the repository and notify the review thread. Do not rewrite the full proposal mid-term.
- The review tracker remains `REVIEW-SESSION-POD.md`; a Tier 3/4 amendment gets a short review pass, not a new full audit.

---

## 8. Dependencies and open confirmations

The ladder assumes these are settled. If any is not, the triggers are unreliable.

1. **G1** — written confirmation that a witnessed Merkle log (level 3) is acceptable under "not limited to a hash chain".
2. **G4** — written confirmation that the deliverable is a **demo plus report**, not a benchmark study.
3. **G2** — whether a synthetic SHP-42 workflow is acceptable, or a named external unit is still required.
4. **G3** — which module, and whether its rubric rewards a working artifact or an evaluated contribution.
5. **Pace** — 18 h/week is sustainable. If it is not, extend to 18–20 weeks rather than compress the core.

---

## 9. One-paragraph summary

The 290 h plan remains the target, but it has no slack: 18 h/week over 16 weeks is 288 h. This plan therefore splits the work into a protected core of about 217 h and a contingency pool of about 73 h, and pre-authorizes two schedule-driven cuts: first the evaluation apparatus and privacy study, then the revocation study and the red-team breadth. A third tier, triggered by a late M4 or a failed spike, drops the RFC 9942 receipt encoding in favour of a Tessera proof checked against a signed checkpoint, preserving inclusion and the two-witness quorum; dropping a witness is the last resort before the level-2 floor. Tiers 1–2 are pre-authorized; Tiers 3–4 require written supervisor sign-off because they change the claim. Every cut is logged and becomes the limitations chapter. The demonstration, the honest non-claims, and the writing floor are never cut.
