# BUILD-HANDOVER-003 — Roast of the Adjudicated Delivery Concept

| Field | Value |
| --- | --- |
| Prepared | 2026-10-23 |
| For | The next BUILD or REVIEW agent asked to **attack** the current proposal |
| Repo | `/home/hj-core/project` |
| Branch | `tdd` |
| HEAD | `cdb5a93` (per BUILD-HANDOVER-002; not re-verified — no shell in the preparing session) |
| Working tree | Uncommitted. `BUILD-HANDOVER.md` modified; the documents in §11 are untracked. **Do not stage or commit.** |
| Read first | `CNP-001__Adjudicated_Delivery_Concept_Note.md` |
| Status | Concept note drafted; **not yet reviewed**; **no build started** |
| Supersedes | `BUILD-HANDOVER-002` as the current handover for this topic. -002 remains the historical record of the v0.2.0 state. |

---

## 1. Status at a glance

| Item | State |
| --- | --- |
| Module | **COMP4299** — confirmed |
| Deliverable | Dissertation + demo + evaluation — confirmed |
| Stakeholder | **Not required** — a plausible concrete scenario is required instead |
| Mechanism | Witnessed Merkle transparency log (RFC 9943 architecture, RFC 9162 log via Tessera) |
| Scenario | **Payment-linked B2B delivery, adjudicated after the fact** (insurer / arbitrator / court) |
| Anchor | Four adjudicated delivery-forgery cases (ANL-006) |
| Claim | The log supplies complete, non-equivocable, offline-verifiable evidence to a **non-party adjudicator** — a demand no incumbent meets |
| Build | Not started |
| Review | Pending. This handover exists to commission an adversarial review |

---

## 2. Your task

**Roast `CNP-001`.** Do not improve it. Do not rewrite it. Find what a viva examiner, a hostile reviewer, or a sceptical supervisor would find, and report it.

The question to answer:

> Is this a defensible COMP4299 system-development project under the stated bar — and if not, what specifically kills it?

Attack the claim, the scenario, the baselines, the evaluation design and the contribution. Assume the author is motivated to believe it works.

---

## 3. The bar (supervisor's answers, verbatim meaning)

These were given on 2026-10-23 and are the criteria against which the proposal must be judged. Do not apply a stricter bar than this — in particular, **do not require the proposal to beat incumbents on every axis.**

1. Assessed under **COMP4299**.
2. **No real stakeholder is required**, but a **plausible concrete scenario** must demonstrate that the proposed solution meets **some unmet demand**.
3. The implementation demonstrating the solution **can itself be a contribution**.
4. Deliverable: **dissertation + demo + evaluation**.
5. The result should **lean positive**. It need not beat existing solutions in every aspect, but the proposal **must possess its values**.
6. Both a witnessed Merkle log and a consensus ledger are acceptable.

The bar is therefore: plausible scenario + unmet demand + demonstrated value + positive-leaning result. Not market dominance.

---

## 4. Read order

1. `CNP-001__Adjudicated_Delivery_Concept_Note.md` — **the object of the roast**
2. `ANL-006__Adjudication_Anchoring_Cases.md` — the four cases and the derived scenario
3. `ANL-007__PoD_Strongest_Alternative_Test.md` — the incumbent matrix and falsification conditions
4. `PRP-001a__SHP-42_Four_Statement_Walkthrough.md` — the four-statement chain and handover primitive
5. `DISCUSSION-SUMMARY-003__PoD_Scenario_Search.md` — how the current position was reached, including the eliminations
6. `PRP-001__Proof_of_Delivery_Proposal.md` v0.2.0 — the baseline being replaced; read for the sections CNP-001 would rewrite

Supporting: ANL-002, ANL-003, ANL-004, ANL-005, PLN-002, TUT-001, REF-001.

---

## 5. What the proposal now claims

In one paragraph, so you can attack it directly:

> Delivery disputes are adjudicated by parties that were not present and have no right of access to the records they must judge. Existing controls — ERP three-way matching, eIDAS qualified signatures and registered delivery, EDI receiving advice, e-CMR — each solve part of the problem, but all are verifiable only by the parties to them, or by an auditor with access. The project builds a thin profile and verification path on a witnessed Merkle log that exports those controls' results as a **complete, non-equivocable, offline-verifiable record**, and demonstrates against four documented fraud patterns that it detects what L1 and L2 do not. The contribution is the handover profile, the RFC 9162 receipt path — for which no standard offline verifier exists — and a measured demonstration at the adjudication boundary.

---

## 6. Known attack surface

These are the weaknesses the author already knows about. Your job is to determine whether any of them is fatal, and to find the ones not listed here. Each is given with the defence already offered, so you can test the defence rather than rediscover the objection.

### 6.1 The "no audit rights" premise may be false

Insurers have subrogation rights; courts can subpoena; contracts can grant inspection. If the adjudicator *can* get access, the unmet demand evaporates.

*Defence offered:* access is slow, expensive, contested, and after the fact — the cases show detection taking **two years** (Leicht) and requiring **handwriting experts** (Williams). Test whether that is a real gap or a delay.

### 6.2 Letters of credit may already solve "payment against verified documents"

A documentary credit with bank examination is an established mechanism for releasing payment against documents. If an L/C plus a surveyor covers the same ground, the unmet demand is narrower than claimed.

*Defence offered:* L/C covers payment security, not evidence that goods were delivered; banks examine documents, not physical events. Test it — this alternative is **not** fully treated in ANL-007.

### 6.3 The baselines may be strawmen

L2 is defined as a hash chain plus a TSA. But a competent L2 implementation would also include a payer-held sequence, signed receipts, and a policy. If L2 with those additions detects the same attacks, H1 fails.

*This is the most dangerous attack.* Check whether the L1/L2 arms are given the same policies and the same recipient-held keys as L3. If not, the comparison is rigged.

### 6.4 The completeness claim may be circular

Completeness is measured against the payer's own count. The payer is also the party that benefits from denying acceptances. Establish precisely:

- who issues the count,
- what prevents that party from omitting,
- and why the adjudicator should believe it.

*Defence offered:* the payer's incentive runs against over-claiming, which is the fraud in Leicht. Test whether the incentive argument holds in both directions.

### 6.5 The positive evaluation may be tautological

The author designs the attacks *and* the defences. Every attack is one the design was built to catch. A viva will ask: **what result would have falsified the claim?** Check that CNP-001 §8 contains a falsifier that could actually fire.

### 6.6 The contribution may be too small

The RFC 9162 receipt wrapper is budgeted at 25 h. The profile is a schema plus a state machine. The evaluation is synthetic. Is that a dissertation, or a component plus a write-up?

*Defence offered:* the three together, plus the measured comparison. Test the arithmetic.

### 6.7 The RFC 9162 gap may be smaller than claimed

`scitt-verifier` is CCF-only, but Tessera, Rekor and Trillian all produce inclusion proofs. If serialising an RFC 9942 receipt over an existing proof is a day's work, the "real implementation gap" shrinks.

*Action for the reviewer:* check what Tessera and Rekor already expose. Do not accept the 25 h estimate without evidence.

### 6.8 The scenario is simulated and the adjudicator is a role

The supervisor waived the stakeholder requirement, but "plausible concrete scenario" still has to be plausible. Test whether "an insurer adjudicating a cargo claim" is concrete enough, or whether it is a role invented to need the property.

### 6.9 Witness independence is simulated

Two processes on one host. The non-equivocation claim is demonstrated, not deployed. Test whether the evaluation distinguishes "the mechanism works" from "the mechanism would work with independent operators."

### 6.10 Commercial privacy

A log of supplier–payer relationships, timings and amounts is competitive intelligence, even hashed. Who operates it, who can see the metadata, and what the linkage analysis says.

### 6.11 The anchors are old and foreign

Four cases, 1996–2019, US and England. They establish the pattern, not its prevalence or current relevance. Check that CNP-001 does not drift into a prevalence claim.

### 6.12 The mechanism does not prevent the fraud

It detects after the fact. If the fraud is detected anyway — by audit, volume anomaly, or forensics — what is the marginal value? The claim must be about **speed, cost, and burden of proof**, not prevention. Test whether the note says so consistently.

---

## 7. Settled points — do not re-litigate

These were established with evidence during the preceding sessions. Reopening them wastes the review.

1. The system **does not prevent** forged delivery records. Signatures prevent per-document forgery; the log addresses denial, backdating, selective presentation and count manipulation.
2. It is **not** a qualified trust service and has **no legal recognition**. QES, QERDS and e-CMR are legally stronger.
3. It **does not** beat ERP three-way matching on prevention, EDI on bilateral acknowledgment, or QES on signature. The claim is confined to the adjudication boundary.
4. It **does not** prove physical truth, contents, or inspection.
5. The four cases are **a pattern, not a prevalence study**.
6. The scenario is **anchored, not recruited**. No stakeholder is required per the supervisor.
7. Do not call it a blockchain. Call it a witnessed Merkle transparency log.
8. "No public deployment found" is **false for credentials** (OpenCerts, My eQuals) and was corrected. Do not repeat it.
9. Experimental-run provenance was **rejected**: 21 CFR Part 11 and commercial CDS already implement instrument-level audit trails, and Registered Reports is a free procedural alternative at 1.2% adoption.
10. The Macau payment-log directions were **eliminated** (AMCM holds the data; UM's financial regime is special and reports annually).

If you believe one of these is wrong, say so explicitly and give evidence — but do not silently reverse it.

---

## 8. Review method

- **Adversarial, not editorial.** Findings, not prose fixes.
- **Every finding gets a severity**: fatal / high / medium / low.
- **Every fatal or high finding gets evidence**: a document reference, a citation, or a concrete counter-example.
- **Test the defences, not just the claims.** Where §6 gives a defence, say whether it holds.
- **Check the bar, not an imagined bar.** The supervisor does not require dominance; do not fail the project for not beating ERP.
- **One question above all**: would the evaluation's result have been different if the hypothesis were false? If not, the evaluation is decorative.

---

## 9. Output expected

A findings report, in this shape:

| # | Finding | Severity | Evidence | Does the offered defence hold? | Fix or kill |
| --- | --- | --- | --- | --- | --- |

Then:

1. **Verdict** — defensible / defensible with named fixes / not defensible.
2. **The three findings that matter most**, in one line each.
3. **The question the viva will ask that the note cannot yet answer.**
4. **A recommended scope cut**, if the project is too large.

Do not edit files. Do not commit. Write the report and stop.

---

## 10. Conventions

- Every number is **measured, derived, or assumed**, and labelled.
- No weighted scores.
- No strawman baselines.
- Two issuers for the document variant, three for the parcel variant; 2-of-2 witnesses; five anchors.
- No "RFC compliant" without a conformance suite.
- No "trustless". No "blockchain" for level 3.
- No physical or documentary truth. No legal non-repudiation.
- Planning estimates are not measurements (290 h; 18 h/week; <$0.01/handover is a target).

---

## 11. Assets

| Asset | Path | Role in the roast |
| --- | --- | --- |
| **Concept note** | `CNP-001__Adjudicated_Delivery_Concept_Note.md` | **The object of review** |
| Anchoring cases | `ANL-006__Adjudication_Anchoring_Cases.md` | Sources for §6.1, §6.11 |
| Strongest-alternative test | `ANL-007__PoD_Strongest_Alternative_Test.md` | Sources for §6.2, §6.3; check for gaps |
| Four-statement chain | `PRP-001a__SHP-42_Four_Statement_Walkthrough.md` | The design under test |
| Scenario search record | `DISCUSSION-SUMMARY-003__PoD_Scenario_Search.md` | Why earlier candidates died; do not repeat them |
| Baseline proposal | `PRP-001__Proof_of_Delivery_Proposal.md` v0.2.0 | The text CNP-001 would replace |
| Demand evidence | `ANL-004__PoD_Domain_Demand_Evidence.md` | Incumbent map; OpenCerts correction |
| Macau feasibility | `ANL-005__Macau_Payment_Log_Feasibility_Check.md` | Eliminated direction; evidence for §7.10 |
| Contingency | `PLN-002__Scope_Contingency.md` | Cut ladder if scope is too large |
| Learning guide | `TUT-001__E2E_Slice_Learning_Guide.md` | Stale relative to PRP-001a; do not treat as authoritative |
| Glossary | `REF-001__PoD_Glossary.md` | Terminology |
| Review tracker | `REVIEW-SESSION-POD.md` | Historical; refers to PRP-001 v0.2.0, not CNP-001 |
| Historical handover | `BUILD-HANDOVER-002__PoD_Handover.md` | Prior state; do not edit |

---

## 12. One-paragraph summary

The proposal has been through nine candidate domains and survived only after the supervisor relaxed the bar: COMP4299, no stakeholder required, a plausible scenario and an unmet demand, implementation as a contribution, and a result that leans positive without needing to dominate incumbents. The current position is `CNP-001`: payment-linked B2B delivery adjudicated by a non-party, anchored in four adjudicated forgery cases, built as a four-statement handover chain on a witnessed Merkle log, with the RFC 9162 receipt path as the implementation gap. Your task is to break it. The most dangerous lines of attack are the no-audit-rights premise, the fairness of the L1/L2 baselines, the circularity of the completeness count, and whether any result in the evaluation could have come out the other way. The settled points in §7 are not open for re-litigation without new evidence. Report findings; do not edit; do not commit.
