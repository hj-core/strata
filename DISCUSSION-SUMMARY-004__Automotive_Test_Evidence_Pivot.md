# DISCUSSION-SUMMARY-004 — From Parcel Adjudication to Automotive Test-Evidence Integrity

| Field | Value |
| --- | --- |
| Document ID | DISCUSSION-SUMMARY-004 |
| Title | Discussion summary — the pivot from parcel adjudication to automotive software/firmware test-evidence integrity |
| Date | 2026-10-23 |
| Status | Working record — **current direction; the pivot is a proposal, not yet approved by the supervisor** |
| Related documents | CNP-001 (roast object); ANL-006; ANL-007; PRP-001a; PRP-001 v0.2.0; BUILD-HANDOVER-004 |
| Supersedes | DISCUSSION-SUMMARY-003 **for the current direction**. -003 remains the historical record of the parcel scenario search; it is not edited. |
| Purpose | Record the roast findings, the decision to leave the parcel scenario, the new automotive test-evidence direction, the scope cut, and the open questions |

**How to read this.** -003 records nine domain eliminations and the parcel anchor, and concludes without a defensible scenario. This file records what happened after: the roast of `CNP-001`, the finding that the parcel case fails the incentive and strongest-alternative tests, and the pivot to an automotive software/firmware test-evidence chain. The pivot is **not settled**; the supervisor has not been asked. Do not build on it until that conversation happens.

---

## 1. What changed

1. **The parcel / adjudicated-delivery scenario is out as the primary case.** It failed the incentive test (the insurer had weak power and weak per-claim interest; the courier had a veto; the burden fell on the courier's customers) and the strongest-alternative test (a registry or QERDS is simpler and may be better).
2. **L3 is on trial, not assumed.** The new scenario is chosen because it has the conditions L3 needs: a real mandate, a powerful verifier, a machine-generated statement, a record holder with an incentive to falsify, and a burden spread across tiers.
3. **The chain ends at the test result.** Production conformity is out of scope. This is the scope cut that makes the project fit a 290-hour plan.
4. **The new direction is automotive software/firmware test-evidence integrity**: a thin attestation layer over build and test systems, with an independent log operator and the type-approval authority as auditor.

---

## 2. The roast findings that forced the change

From `BUILD-HANDOVER-003` and the roast of `CNP-001`:

**Fatal to the claim as written**

- The bundle cannot prove **completeness** (no non-inclusion proof; the checkpoint is global; a selective bundle is indistinguishable from a complete one) or **non-equivocation** (one cosigned checkpoint is one head; a fork needs a second head or gossip). `PRP-001a` Phase 10 claims only who asserted what, when, order and non-alteration; `CNP-001` added “complete” and “non-equivocable.”
- The **no-audit-rights premise** is contradicted by all four anchors: *Leicht* was a coverage dispute with the insurer; *Howe* and *Williams* were criminal prosecutions; *Maersk* was High Court litigation. All had compulsory process. The real gap is cost and delay, not absence of access.
- The **L1/L2 baselines are rigged**: replay and count-gap detection are profile fields credited to L3; L2 is denied the operator policy that `PRP-001` §5.4/§8.6(4) names as the falsifier; the evaluation omits the two log-unique rows (fork, offline).

**High**

- The scenario was a damage claim; the mechanism is delivery forgery. `PRP-001a` Phase 10 says it does not establish who damaged the pallet.
- The “RFC 9162 verifier gap” is already filled: `scitt-cose` (Apache-2.0) verifies and builds RFC 9162_SHA256 COSE receipts; `microsoft/scitt-verifier` is CCF-only, but that is one tool.
- The verifier’s trust anchors are configured by a party (Acme in `PRP-001a`), not by the relying party.
- The 30-minute handover window is unverifiable when the courier is offline; `handover_incomplete` rests on device time.
- The log’s unique property (non-equivocation) addresses log-operator misbehaviour; none of the four anchors involves a dishonest log operator.
- No demand evidence at the adjudication boundary; `ANL-004`’s demand data is employment/credentials.
- Privacy and commercial confidentiality are absent; the operator sees statement bodies.
- Witness independence is simulated; “non-equivocable” was unqualified.

**Constructive conclusion**

The evidence supports a **cost-and-delay** claim, not an access claim. The right architecture is a **thin attestation layer** over existing controls, not a complete PoD system. The strongest alternative is a neutral evidence registry or a QERDS; the log’s only distinctive answer is **trust without custody**. Any new scenario must show why that matters.

---

## 3. The new problem statement

> Vehicle type approval and software-update regulation depend on evidence that the manufacturer produces and controls. The approval authority receives a test report, a software declaration, an SBOM and supplier attestations; it sees the submitted record, not the production state. It cannot independently establish (1) that the software tested is the software the record describes, (2) that the cross-tier attestation set is complete, or (3) that the manufacturer showed the same record to the OEM and the regulator. The record holder has both the incentive and the opportunity to make the record self-consistent while the tested artefact differs from the production artefact. The result is fraud detected after the fact, at recall scale.

This is not a signature problem. The test report, the SBOM and the declaration are signed. The problem is that the signed record is **self-selected and self-consistent**, and no party outside the manufacturer can enumerate the set or compare the tested artefact with the record.

### 3.1 Why the existing controls do not close it

| Control | Provides | Does not provide |
| --- | --- | --- |
| SBOM (CycloneDX, SPDX) | Component list for an artefact | Tested-vs-shipped comparison; cross-tier set; neutrality |
| SLSA / in-toto provenance | Build provenance | Type-approval evidence set; supplier identity; regulator audit |
| Sigstore / Rekor | Public artefact transparency log | Permissioned, supplier-aware, regulator-auditable record; SBOM/ECU semantics |
| ISO/SAE 21434, ISO 24089 | Process requirements and work products | Transaction-level, third-party-verifiable evidence |
| OEM supplier portal | A place for supplier evidence | Neutrality: the OEM is a party |
| Type-approval submission | Manufacturer declaration and test report | Independent comparison; enumeration of the cross-tier set |
| Audit / inspection | After-the-fact, sample-based assurance | Transaction-level completeness; speed; every build |

---

## 4. The scenario

**Parties.** Aurelia Motors (OEM), NordECU (Tier 1, vehicle control unit), Voltparts (Tier 2, DC-DC converter), Kestrel Type Approval Authority (regulator).

**Artefact.** Aurelia A5 compact electric vehicle; NordECU VCU-3 vehicle control unit; Voltparts DC-DC converter firmware.

**Flow**

1. Voltparts builds DC-DC firmware `v2.3.1`; its CI emits a signed `component` attestation (firmware hash, SBOM hash, test result, time).
2. NordECU integrates it into VCU-3, builds `v7.4.0` with **test calibration TC-17** and **production calibration PC-04**; its build system emits a `build` attestation (source hash, `prev` → Voltparts, TC-17 hash, PC-04 hash, test report hash).
3. Aurelia integrates VCU-3 into the A5; its MES emits a `vehicle` attestation (ECU hash, VIN/build record, `prev` → NordECU).
4. Kestrel reads the log during type approval: enumerates the set, verifies receipts, checks the chain, compares the test build hash with the test report.
5. A year later, field failures trigger a recall. Kestrel asks: **which software was tested?** The audit report shows the test build (TC-17) and the production build (PC-04) attested by NordECU; the mismatch is visible.
6. The regulator still inspects and tests the production software; the log is evidence, not a verdict.

**The manipulation.** The version string is the same; the calibration differs. A version-string attestation would not catch it. A build attestation that commits to both hashes would.

**The honest limit.** If NordECU lies about the calibration hash at build time, the log records the lie. The log is not an oracle. Its value is cross-tier comparison and completeness; the regulator still inspects.

---

## 5. The anchors

| Case | Facts | Status | What it anchors |
| --- | --- | --- | --- |
| **Toyota Industries** (Jan 2024) | During horsepower certification, engines were tested with ECUs running software different from mass production, so results appeared smoother; three diesel engine models, ten vehicle models; MLIT correction order, Feb 2024 | **Regulatory finding and company admission** — not a court judgment | The tested-vs-production software gap at the calibration/software level; test-object identity |
| **Hino Motors** (2022–2025) | Altered/fabricated emissions and fuel-economy test data; failed to disclose software functions affecting emission controls; 105,000+ non-conforming engines; over **$1.6B** US criminal/civil resolution, Jan 2025 | **Criminal plea and settlement** — not a Japanese court adjudication | Falsified test data; the test evidence is the record in question |
| **Daihatsu** (2023–2024) | Falsified safety/crash-test certification data across many models | **Regulatory/company admission** | Contrast: a car manufacturer, but a safety-data case, not software |

The cases establish the failure mode, not its prevalence. They are recent, Japanese, software-related and regulator-facing — closer to the scenario than the 1996–2019 parcel cases. Label their status accurately in the evidence register.

---

## 6. The thin layer and the scope cut

### 6.1 What the layer is

Four functions over existing systems:

1. **Attest** — normalise a source assertion into the profile and sign it.
2. **Register** — submit to the independent log; obtain an RFC 9942 receipt.
3. **Audit** — enumerate the set for a vehicle type; compare checkpoints; report completeness and forks.
4. **Policy** — who may assert what; what set is required; what a missing attestation means.

The sources supply Q1–Q4 (what was built, by whom, what was tested). The layer supplies Q5–Q7 (completeness, non-equivocation, independent verification).

### 6.2 The chain ends at the test result

The revised chain is:

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
- The tested artefact’s identity (build hash and calibration ID) stays in the attestation, even though production is out of scope. The regulator can compare it with the production RXSWIN/COP record separately.
- The chain proves **test-evidence integrity**: what was tested, by which accredited lab, with what result, and that the required set is complete and non-equivocal. It does **not** prove that production runs the tested build.

### 6.3 What the scope cut buys and costs

| | Chain through production | Chain ending at the test result |
| --- | --- | --- |
| Claim | “Tested = shipped” | “The test result is attributable to the exact artefact tested, by an accredited lab, and the set is complete/non-equivocal” |
| Hardest dependency | Flashing station / ECU secure boot / fleet measurement | Test-lab attestation and build hash |
| Anchor | Toyota Industries (test vs production) | Hino (falsified test data) + Toyota Industries (test-object identity) |
| Production conformity | In scope | **Out of scope**; RXSWIN/COP comparison is the regulator’s separate step |

---

## 7. Decisions taken

1. **Parcel out.** It is not the primary case; keep the documents as historical assets.
2. **Independent log operator is primary.** The manufacturer-operated log is an evaluation arm, not the deployment.
3. **Thin layer, not a full system.** Consume Q1–Q4; answer Q5–Q7.
4. **Chain ends at the test result.** Production conformity out of scope.
5. **Accredited test lab as issuer.** The lab’s attestation is the result. If the manufacturer’s own lab is the issuer, the value is weaker.
6. **Keep build hash and calibration ID** in the attestation; a version string is not enough.
7. **RFC-aligned minimal deployment is the baseline**: one independent TS, the OEM/regulator as Relying Party/Auditor, no witnesses. The **CT-hardened** configuration adds two witnesses as a treatment arm.
8. **L1–L5 is an analytical device, not a standard taxonomy.** Use the supervisor’s terms: signed records, hash chain + timestamp, witnessed Merkle log, consensus ledger.
9. **Do not move the claim to L2; move the baseline to L2.** L2 is the comparison arm; L3 is the treatment.
10. **Do not claim “complete, non-equivocable, offline” as one bundle property.** Use the RFC 9943 tiers: per-statement offline inclusion; per-set audit for completeness/non-equivocation.
11. **The VDS is not necessarily Merkle.** RFC 9943 requires a VDS with append-only, non-equivocation, replayability, and receipts with inclusion proofs; RFC 9162 is the chosen VDS.
12. **The witness is consistency/non-equivocation**, not completeness, time, content validation or availability.
13. **Single sequencer or DB transaction.** No consensus; 4,000 appends/day is trivial.
14. **Evaluation has a real baseline and a falsifier.** Include the registry/QERDS arm.

---

## 8. Alternatives considered

| Alternative | Status | Why |
| --- | --- | --- |
| **Neutral evidence registry** (OEM or regulator operated, or a regulated trustee) | **Strongest alternative; evaluation arm** | Simpler, cheaper, easier to adopt; answers Q5–Q7 if the registry is trusted |
| **QERDS / QTSP channel** | Evaluation arm / related work | Legal recognition; per-message, not a set; no non-equivocation |
| **e-CMR / eBL platform** | Related work | Legal document/title, not the test-evidence set |
| **Manufacturer-operated log** | Evaluation arm | Adoption is easier, but the log is not an independent copy; the witnesses carry all the independence |
| **L2 cross-party hash chains** | Baseline | Fewer parties, lower cost; no non-equivocation; no single auditable set |
| **Full production chain** | Out of scope | Requires trusted flashing/secure boot/fleet measurement; the hardest oracle |
| **Minimal PWA (parcel)** | **Superseded** | The parcel edge PWA has no direct equivalent; the automotive source is a build/test system. A small auditor UI or build adapter is the likely replacement — confirm. |

---

## 9. Evaluation plan

Three arms:

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

## 10. Open questions

1. **Supervisor approval of the pivot** — the decisive question. Do not build before it.
2. **Module and format** — COMP4299, dissertation + demo + evaluation; confirm the pivot fits.
3. **The mandate and jurisdiction** — UNECE R155/R156, ISO/SAE 21434, ISO 24089; which market; what the regulator actually requires.
4. **The test lab’s independence** — accredited third party vs the manufacturer’s own lab; this determines the value.
5. **The profile** — exact fields; calibration/variant identity; update attestation; the required set.
6. **The evaluation baseline** — how to model Arm A and B credibly; what data is available.
7. **The “why not a registry/QERDS” answer** — trust without custody; multi-verifier; non-equivocation. Test it, do not assume it.
8. **The 290-hour budget** — the pivot needs a new WBS; the parcel WBS is not reusable line-for-line.
9. **The minimal PWA** — superseded; decide whether a small auditor UI or a build adapter replaces it.
10. **Legal and privacy** — evidence only, no legal recognition; technical data, supplier confidentiality.
11. **Anchor status labels** — regulatory findings/admissions/pleas, not court judgments.
12. **Prevalence** — the anchors show the failure mode, not its frequency.

---

## 11. What not to re-litigate

These are settled from the roast and the preceding analysis:

- The system **does not prevent** fraud; it makes records detectable/comparable. Do not use “prevent” for non-equivocation.
- It is **not a blockchain**; call it a witnessed transparency log. L1–L5 is not a universal taxonomy.
- It has **no legal recognition**; QES, QERDS and e-CMR are legally stronger.
- It does **not prove physical truth**, test adequacy, or production conformity.
- The four parcel cases are a **pattern, not a prevalence study**.
- The **no-audit-rights premise is false**; the gap is trust in the record holder, not access.
- A bundle cannot prove **completeness or non-equivocation offline** as one property; use the RFC 9943 tiers.
- The **RFC 9162 verifier gap is already filled** by `scitt-cose`; `microsoft/scitt-verifier` is CCF-only, but that is one tool.
- The **witness is not for completeness or time**; it is consistency/non-equivocation.
- The **VDS is not necessarily Merkle**; RFC 9162 is a choice.
- **Planning estimates are not measurements.**

---

## 12. Documents affected

| Document | Disposition |
| --- | --- |
| `CNP-001` | Historical: the roast object. Superseded by a new concept note for the automotive direction (to write). |
| `DISCUSSION-SUMMARY-003` | Historical: the parcel scenario search; not edited. |
| `BUILD-HANDOVER-003` | Historical: the roast commission; not edited. |
| `ANL-006` | Add Toyota Industries, Hino, Daihatsu; keep the parcel cases as method/contrast. |
| `ANL-007` | Re-cut by question; add registry/QERDS; keep the incumbent matrix. |
| `PRP-001a` | Method asset: the handover primitive maps to artefact → lab → result. |
| `PRP-001` v0.2.0 | Historical baseline; L1–L5 and the evaluation matrix are reusable as a method, not as the claim. |
| `PLN-002` | Re-cut the WBS for the automotive direction. |
| `BUILD-HANDOVER-004` | Current handover. |

---

## 13. One-paragraph summary

The parcel/adjudicated-delivery case is out. The roast showed that its claim (complete, non-equivocable, offline) was not a property of the bundle, its no-audit-rights premise was contradicted by its own anchors, and its baselines were rigged; the strongest alternative, a neutral registry or QERDS, is simpler and may be better. The new direction is automotive software/firmware **test-evidence integrity**: a thin attestation layer over build and test systems, with an accredited test lab as an issuer, an independent log operator, and the type-approval authority as auditor. The chain ends at the test result; production conformity is out of scope. The claim is narrow: the regulator can verify that the test result it relies on was produced for the exact artefact tested, by an accredited lab, and that the test-evidence set is complete and non-equivocal, without trusting the manufacturer’s compilation of the evidence. The pivot is a proposal, not a settled decision. The next step is a one-page options brief to the supervisor; do not build before that answer.
