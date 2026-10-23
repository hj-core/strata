# BUILD-HANDOVER-004 — Automotive Test-Evidence Integrity

| Field | Value |
| --- | --- |
| Prepared | 2026-10-23 |
| For | The next BUILD agent continuing after the parcel pivot |
| Repo | `/home/hj-core/project` |
| Branch | `tdd` |
| HEAD | `cdb5a93` (verified with `git log`; working tree uncommitted) |
| Working tree | Uncommitted. `DISCUSSION-SUMMARY-004` and this file are untracked; the `-003` files and the parcel assets remain untracked/modified. **Do not stage or commit.** |
| Read first | `DISCUSSION-SUMMARY-004`, then `CNP-001` (the roast object), `ANL-006`, `ANL-007`, `PRP-001a` |
| Status | **Pivot proposed; supervisor approval not yet sought; no concept note for the new direction; no build started** |
| Supersedes | `BUILD-HANDOVER-003` **for the current state**. -003 remains the historical record of the roast commission. |

---

## 1. Status at a glance

| Item | State |
| --- | --- |
| Module | **COMP4299** — confirmed |
| Deliverable | Dissertation + demo + evaluation — confirmed |
| Old topic | Parcel / adjudicated delivery — **out as the primary case** |
| New topic | **Automotive software/firmware test-evidence integrity** — proposed |
| Mechanism | Thin attestation layer; independent log operator; RFC 9162 Merkle log via Tessera; RFC 9942 receipts; RFC 3161 time; optional witnesses; type-approval authority as auditor |
| Scope | Chain **ends at the test result**; production conformity out of scope; accredited test lab as issuer; build hash + calibration ID in the attestation |
| Claim | The regulator can verify that the test result it relies on was produced for the exact artefact tested, by an accredited lab, and that the test-evidence set is complete and non-equivocal, without trusting the manufacturer’s compilation |
| Anchors | Toyota Industries (2024, regulatory admission); Hino (2022–2025, criminal plea/settlement); Daihatsu (2023–2024, contrast) |
| Build | **Not started** |
| Approval | **Supervisor not yet asked.** The pivot is a proposal |

---

## 2. The job

1. **Do not build yet.** The pivot is not approved. The next action is a conversation with the supervisor, not a commit.
2. **Write a one-page options brief for the supervisor**, with three options:
   - A: the automotive test-evidence thin layer (the recommended pivot);
   - B: a pivot to the employment-verification case (the strongest demand evidence in `ANL-004` §5.3);
   - C: the artifact/boundary study (`DISCUSSION-SUMMARY-003` T1).
   State for each: the claim, the evidence, the strongest alternative, the risk, and what is needed from the supervisor.
3. **If approved, write `CNP-002`** for the automotive direction, using `DISCUSSION-SUMMARY-004` §3–§6 as the draft.
4. **Update `ANL-006`** with Toyota Industries, Hino and Daihatsu, in the existing case format, with accurate status labels.
5. **Re-cut `ANL-007`** by question, adding the registry/QERDS alternative.
6. **Re-cut the WBS** (`PLN-002` is parcel-shaped and does not carry over line-for-line).

---

## 3. Why the pivot

The parcel case failed two tests that no amount of technical work fixes:

- **Incentive test:** the insurer had weak power and weak per-claim interest; the courier had a veto; the burden fell on the courier’s customers. The beneficiary, the cost bearer and the burden bearers were three different parties.
- **Strongest-alternative test:** a neutral registry or a QERDS is simpler, cheaper, easier to adopt, and easier to explain to a court. The log’s only distinctive answer is **trust without custody**, and the parcel scenario could not show that an insurer cared enough to mandate it.

The automotive test-evidence case has the conditions the parcel case lacked: a real mandate (type approval), a powerful verifier (the authority/OEM), a machine-generated statement (build/test output), a record holder with an incentive to falsify (the manufacturer/tier), a high cost of failure (recall), a distributed burden (each tier attests), and manageable privacy (technical data).

---

## 4. The problem statement

> Vehicle type approval and software-update regulation depend on evidence that the manufacturer produces and controls. The approval authority receives a test report, a software declaration, an SBOM and supplier attestations; it sees the submitted record, not the production state. It cannot independently establish (1) that the software tested is the software the record describes, (2) that the cross-tier attestation set is complete, or (3) that the manufacturer showed the same record to the OEM and the regulator. The record holder has both the incentive and the opportunity to make the record self-consistent while the tested artefact differs from the production artefact. The result is fraud detected after the fact, at recall scale.

This is not a signature problem. The test report, the SBOM and the declaration are signed. The problem is that the signed record is **self-selected and self-consistent**, and no party outside the manufacturer can enumerate the set or compare the tested artefact with the record.

---

## 5. The scenario

**Parties.** Aurelia Motors (OEM), NordECU (Tier 1, VCU-3), Voltparts (Tier 2, DC-DC converter), Kestrel Type Approval Authority (regulator).

**Flow.**

1. Voltparts builds DC-DC firmware `v2.3.1`; CI emits a signed `component` attestation (firmware hash, SBOM hash, test result, time).
2. NordECU integrates it into VCU-3, builds `v7.4.0` with **test calibration TC-17** and **production calibration PC-04**; build system emits a `build` attestation (source hash, `prev` → Voltparts, TC-17 hash, PC-04 hash, test report hash).
3. Aurelia integrates VCU-3 into the A5; MES emits a `vehicle` attestation (ECU hash, VIN/build record, `prev` → NordECU).
4. Kestrel reads the log during type approval: enumerates the set, verifies receipts, checks the chain, compares the test build hash with the test report.
5. A year later, field failures trigger a recall. Kestrel asks: **which software was tested?** The audit report shows the test build (TC-17) and the production build (PC-04) attested by NordECU; the mismatch is visible.

**The manipulation.** The version string is the same; the calibration differs. A version-string attestation would not catch it. A build attestation that commits to both hashes would.

**The honest limit.** If NordECU lies about the calibration hash at build time, the log records the lie. The log is not an oracle. Its value is cross-tier comparison and completeness; the regulator still inspects.

---

## 6. The thin layer

### 6.1 Four functions

1. **Attest** — normalise the source assertion into the profile and sign it.
2. **Register** — submit to the independent log; obtain an RFC 9942 receipt.
3. **Audit** — enumerate the set for a vehicle type; compare checkpoints; report completeness and forks.
4. **Policy** — who may assert what; what set is required; what a missing attestation means.

Q1–Q4 come from the build/test systems. The layer supplies **Q5–Q7**: completeness, non-equivocation, independent verification.

### 6.2 The chain

```
Tier 2 build ──▶ component attestation ──┐
                                         ▼
Tier 1 build ──▶ build attestation ──▶ Test lab ──▶ test attestation ──▶ Log
                  (artefact hash,       (accredited)   (artefact hash,       │
                   calibration)                         calibration,          ▼
                                                        raw data hash,   Regulator audit
                                                        result, time)
```

- The **test lab is an issuer**, not the manufacturer’s own test department. Its attestation carries the artefact hash, calibration ID, raw-data hash, result and time.
- The tested artefact’s identity (build hash, calibration ID) stays in the attestation. The regulator can compare it with the production RXSWIN/COP record separately.
- The named outcomes are `artifact_mismatch`, `test_not_completed`, `result_missing`, `lab_not_accredited`.

### 6.3 The scope cut

The chain **ends at the test result**. Production conformity is out of scope.

| | Chain through production | Chain ending at the test result |
| --- | --- | --- |
| Claim | “Tested = shipped” | “The test result is attributable to the exact artefact tested, by an accredited lab, and the set is complete/non-equivocal” |
| Hardest dependency | Flashing station / ECU secure boot / fleet measurement | Test-lab attestation and build hash |
| Anchor | Toyota Industries (test vs production) | Hino (falsified test data) + Toyota Industries (test-object identity) |
| Production conformity | In scope | **Out of scope**; RXSWIN/COP comparison is the regulator’s separate step |

Do not have it both ways: if you withdraw production, do not claim the layer catches the tested-vs-shipped mismatch by itself.

---

## 7. The claim and the boundary

**Claim.**

> A thin attestation layer over build and test systems gives the type-approval authority an enumerable, non-equivocal record of which artefact was tested, by which accredited lab, with what result — without the authority holding the manufacturer’s data or trusting its compilation of the evidence.

**Does not prove.**

- that production runs the tested build;
- that the test was adequate or the software safe;
- that the lab told the truth — only that it signed the result and the raw-data hash;
- that the Toyota Industries production mismatch is caught by the layer alone;
- legal certification or non-repudiation.

---

## 8. The anchors

| Case | Facts | Status | What it anchors |
| --- | --- | --- | --- |
| **Toyota Industries** (Jan 2024) | Horsepower certification used ECUs with software different from mass production; results appeared smoother; three diesel engine models, ten vehicle models; MLIT correction order, Feb 2024 | **Regulatory finding and company admission** — not a court judgment | Tested-vs-production software gap; test-object identity |
| **Hino Motors** (2022–2025) | Altered/fabricated emissions and fuel-economy test data; failed to disclose software functions; 105,000+ non-conforming engines; over **$1.6B** US resolution, Jan 2025 | **Criminal plea and settlement** — not a Japanese court adjudication | Falsified test data; the test evidence is the record in question |
| **Daihatsu** (2023–2024) | Falsified safety/crash-test data across many models | **Regulatory/company admission** | Contrast: a car manufacturer, but safety data, not software |

The cases establish the failure mode, not its prevalence. Label their status accurately. Do not present a regulatory admission as a court finding.

---

## 9. Evaluation plan

| Arm | What it is | What it tests |
| --- | --- | --- |
| **A — status quo** | Signed lab report + manufacturer submission + audit/COP | Can the regulator already answer the test-evidence questions? |
| **B — registry / QERDS** | One trusted custodian, signed submissions, audit API | Is the log over-engineered? |
| **C — thin layer + independent log** | Profile + test-lab attestations + log + auditor (+ witness sub-arm) | Does the layer add completeness/non-equivocation at acceptable cost? |

Measures: time and cost to verify the test-evidence set; completeness under withholding; non-equivocation; trust anchors; burden by party; update evidence; bundle size.

Falsifiers:

- Arm A or B already answers the questions at lower cost.
- The test lab is not independent, so the attestation adds nothing.
- The regulator never disputes the test evidence, so completeness/non-equivocation is not contested.
- The layer costs more than the audit/recall it replaces.
- The manufacturer can lie at the source and the cross-tier attestations do not constrain it.

---

## 10. Decisions already made — do not re-litigate

1. **Parcel out as the primary case.** Keep the assets as historical/method.
2. **Independent log operator is primary.** The manufacturer-operated log is an evaluation arm.
3. **Thin layer, not a full system.** Consume Q1–Q4; answer Q5–Q7.
4. **Chain ends at the test result.** Production conformity out.
5. **Accredited test lab as issuer.** If the manufacturer’s own lab is the issuer, the value is weaker.
6. **Keep build hash and calibration ID.** A version string is not enough.
7. **RFC-aligned minimal deployment is the baseline**: one independent TS, the OEM/regulator as Relying Party/Auditor, no witnesses. The CT-hardened configuration adds two witnesses as a treatment arm.
8. **L1–L5 is an analytical device, not a standard taxonomy.** Use the supervisor’s terms.
9. **Do not move the claim to L2; move the baseline to L2.**
10. **Do not claim “complete, non-equivocable, offline” as one bundle property.** Use the RFC 9943 tiers.
11. **The VDS is not necessarily Merkle.** RFC 9162 is the chosen VDS; RFC 9943 requires the properties and inclusion proofs.
12. **The witness is consistency/non-equivocation**, not completeness, time, content validation or availability.
13. **Single sequencer or DB transaction.** No consensus; 4,000 appends/day is trivial.
14. **The evaluation has a real baseline and a falsifier.** Include the registry/QERDS arm.

---

## 11. Open questions and gates

| # | Question | Why it matters |
| --- | --- | --- |
| G1 | **Will the supervisor approve the pivot?** | Do not build before this answer |
| G2 | **Does the module accept an artifact-plus-evaluation project?** | COMP4299 is confirmed, but the pivot is a new domain |
| G3 | **Which mandate/jurisdiction?** | UNECE R155/R156, ISO/SAE 21434, ISO 24089; the regulator’s actual requirement |
| G4 | **Is the test lab independent?** | The value of the attestation depends on it |
| G5 | **What exactly does the profile require?** | Calibration/variant identity; update attestation; required set |
| G6 | **How credible are Arms A and B?** | The evaluation is only as good as the baseline |
| G7 | **What is the “why not a registry/QERDS” answer?** | Trust without custody; multi-verifier; non-equivocation — test it |
| G8 | **What is the 290-hour WBS?** | The parcel WBS does not carry over line-for-line |
| G9 | **What replaces the minimal PWA?** | The parcel edge PWA is superseded; a small auditor UI or build adapter is the likely replacement |
| G10 | **Legal and privacy** | Evidence only; technical data; supplier confidentiality |
| G11 | **Anchor status labels** | Regulatory findings/admissions/pleas, not court judgments |
| G12 | **Prevalence** | The anchors show the failure mode, not its frequency |

---

## 12. Assets and read order

**Current**

1. `DISCUSSION-SUMMARY-004` — the pivot record.
2. This handover.
3. `CNP-002` — to write for the automotive direction.
4. `ANL-006` — to update with the automotive anchors.
5. `ANL-007` — to re-cut by question; add registry/QERDS.
6. `PRP-001a` — method: the handover primitive maps to artefact → lab → result.
7. `PLN-002` — to re-cut for the new WBS.

**Historical (do not edit)**

- `CNP-001` — the roast object.
- `DISCUSSION-SUMMARY-003` — the parcel scenario search.
- `BUILD-HANDOVER-003` — the roast commission.
- `BUILD-HANDOVER-002` — the prior PoD state.
- `PRP-001` v0.2.0 — the baseline; L1–L5 and the evaluation matrix are reusable as a method, not as the claim.
- `ANL-004`, `ANL-005` — demand and Macau feasibility; historical.
- `BUILD-HANDOVER.md` — the older general PoD handover; not current.

**Standards and libraries**

- RFC 9943 (SCITT architecture), RFC 9942 (COSE receipts), RFC 9162 (CT v2 Merkle log), RFC 3161 (TSA).
- UNECE R155 (CSMS), R156 (SUMS); ISO/SAE 21434; ISO 24089.
- Tessera; `scitt-cose` (RFC 9162 receipts); `microsoft/scitt-verifier` (CCF-only); `transparency-dev/witness`; Sigsum.

---

## 13. Git and document state

- Branch `tdd`; HEAD `cdb5a93`; working tree uncommitted.
- `DISCUSSION-SUMMARY-004` and this handover are **untracked**.
- Do **not** stage or commit. The user commits the documents.
- Do not edit the `-003` records; they are historical.

---

## 14. Cautions

- Do not call it a blockchain; call it a witnessed transparency log.
- Do not claim prevention; the layer makes records detectable/comparable.
- Do not claim legal non-repudiation; it is evidence only.
- Do not claim production conformity; it is out of scope.
- Do not claim the test was adequate; the lab’s accreditation is the trust.
- Do not present L1–L5 as a standard taxonomy.
- Do not claim “no verifier exists” (`scitt-cose` exists).
- Do not claim “no audit rights” (the regulator has access; the gap is trust in the record holder).
- Do not claim “complete, non-equivocable, offline” as one property.
- Do not present planning estimates as measurements.
- Do not let the evidence register treat a regulatory admission as a court finding.
- Do not build before the supervisor answers G1.

---

## 15. One-paragraph summary

The parcel/adjudicated-delivery case is out. The roast showed that its claim was not a property of the bundle, its no-audit-rights premise was contradicted by its own anchors, and its baselines were rigged; a neutral registry or QERDS is simpler and may be better. The new direction is automotive software/firmware **test-evidence integrity**: a thin attestation layer over build and test systems, with an accredited test lab as an issuer, an independent log operator, and the type-approval authority as auditor. The chain ends at the test result; production conformity is out of scope. The claim is narrow: the regulator can verify that the test result it relies on was produced for the exact artefact tested, by an accredited lab, and that the test-evidence set is complete and non-equivocal, without trusting the manufacturer’s compilation. The pivot is a proposal, not a settled decision. The next action is a one-page options brief to the supervisor; do not build before that answer.
