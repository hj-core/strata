# PRC-001 — Declared-Set Evidence Process

| Field | Value |
| --- | --- |
| Document ID | PRC-001 |
| Title | Declared-set evidence process: specification and scandal walkthrough |
| Version | 0.1.0 |
| Status | Historical — domain assessment. Retained as the record of why the automotive direction was dropped; not a project plan. |
| Last Updated | 2026-10-23 |
| Related documents | DISCUSSION-SUMMARY-004; BUILD-HANDOVER-004; ANL-006; ANL-007; PRP-001a |
| Purpose | Define the process that the automotive direction would build, and trace seven documented scandals through it to show which step flags which failure and which failures remain outside the process |
| Supersedes | Nothing. It refines the "thin layer" of BUILD-HANDOVER-004 §6 into an operable process. |

**How to read this.** §2–§6 define the process: six steps, the declaration it depends on, the attestations it consumes, the checks it runs, and the named outcomes it reports. §7 states who must be honest for what. §8 states the five structural limits. §9 is the empirical part: seven scandals traced step by step, each ending in a verdict of *flagged*, *partially flagged* or *not flagged*. §10 aggregates the verdicts. The aggregate is deliberately unflattering; it is the boundary of the claim, and the claim is written to fit it.

---

## 1. The problem the process addresses

Every manufacturer in §9 already held a certified quality system, signed test reports and a type approval. The failures did not occur outside certification; they occurred inside it.

The distinguishing feature is narrower than "fraud":

> **The set of things tested and the set of things declared were both selected by the party being certified, and no party outside could enumerate either set or compare the tested artefact with the declared one.**

A signature over a self-selected record proves that the record was not altered after signing. It does not prove that the record is the whole record, that the artefact it describes is the artefact that was tested, or that the same record was shown to every relying party.

### 1.1 What the process is not

- Not a prevention system. It makes particular manipulations **visible and attributable**; it does not stop them.
- Not a truth oracle. A signature over fabricated data is a signature over fabricated data.
- Not a blockchain. It is a process using signatures, an append-only log and a declared set.
- Not a substitute for market surveillance or independent measurement.

---

## 2. Roles

| Role | Party | Function |
| --- | --- | --- |
| **Authority** | Type-approval authority (e.g. Kestrel) | Declares the required evidence set; audits; enforces |
| **Applicant** | OEM (Aurelia) | Integrates, builds, submits, bears the approval |
| **Tier** | Tier 1 / Tier 2 suppliers (NordECU, Voltparts) | Builds components; attests; is also a **relying party** |
| **Test issuer** | Accredited test laboratory or authority test house | Performs the test; signs the result |
| **Log operator** | Independent operator | Appends attestations; issues receipts; publishes checkpoints |
| **Witness** | Independent cosigner (optional) | Cosigns checkpoints; detects forks |
| **Time authority** | RFC 3161 TSA | Fixes existence in time |
| **Auditor** | Authority, or a party it appoints | Runs the checks in §5 |

Note the double role of the tier. It is a producer of evidence **and** a relying party that wants proof its evidence was not misrepresented upward. This is the process's answer to the incentive problem: the party that bears the burden of producing evidence is also the party that benefits from its being independently recorded.

---

## 3. The six steps

| # | Step | Owner | Input | Output | What it removes |
| --- | --- | --- | --- | --- | --- |
| 1 | **Declare** | Authority | Vehicle type, variants, calibrations | **Declaration of Required Evidence (DRE)** | The manufacturer choosing which evidence is required |
| 2 | **Attest at source** | Tier, OEM build systems | Build, source, SBOM, calibration set | `component` and `build` attestations | Evidence reconstructed after the fact |
| 3 | **Test independently** | Accredited lab | Artefact as received | `test` attestation | The manufacturer being the sole author of its own test result |
| 4 | **Register** | Independent log operator | Signed attestations | Inclusion receipts, checkpoints | A single party holding the only copy |
| 5 | **Audit the set** | Authority; each tier | DRE + registered set | Audit report with named outcomes | Selective presentation to one relying party |
| 6 | **Measure production** | Authority | Production units | COP / RXSWIN record, independent measurement | The manufacturer's account of what it ships |

Steps 2–4 are the build. Step 5 is the build plus the authority. Steps 1 and 6 are institutional prerequisites that the project models but does not own. **Without step 1 the required set is self-set; without step 6 there is no independent side to compare against.**

---

## 4. Step 1 — the declaration

The DRE is published per type and is the unit the audit enumerates against. It states:

| Field | Meaning |
| --- | --- |
| `required_attestations` | Which attestation classes must exist (component, build, test) |
| `required_variants` | The calibration/variant set that must each be evidenced |
| `required_tests` | Test cases (by identifier) that must each have a `test` attestation |
| `identity_fields` | Artefact hash algorithm, calibration ID format, RXSWIN binding |
| `independence` | Whether the test issuer must be legally independent of the applicant |
| `retention` | How long raw data and attestations are retained |
| `validity` | The window within which evidence supports the approval |

The DRE is the process's most important artefact and its weakest point: **the process can only check what the DRE declares.** This is stated again in §8 as a structural limit, not a caveat.

---

## 5. Steps 2–5 — attestations, registration, audit

### 5.1 Attestation classes

| Class | Issuer | Commits to |
| --- | --- | --- |
| `component` | Tier build system | Component artefact hash, source hash, SBOM hash, calibration set, test result hash, `sequence` |
| `build` | Applicant or integrating tier | Input attestation hashes (`prev`), output artefact hash, **test calibration ID**, **production calibration ID**, both hashes, test report hash, `sequence` |
| `test` | Accredited lab | Artefact hash as received, calibration ID tested, test case ID, raw-data hash, result, facility, accreditation ID, `sequence` |

All three carry `subject` (the vehicle type), `issuer`, `time`, `sequence` and a TSA token. The log stores leaves; payloads may be off-chain and encrypted.

### 5.2 Audit checks and named outcomes

The audit enumerates the set for a type and reports the first failure by name. It does not report "invalid."

| Check | Named outcome | Meaning |
| --- | --- | --- |
| Every required attestation class present | `set_incomplete` | The set does not cover the DRE |
| Every required variant evidenced | `variant_undeclared` | A calibration in production evidence has no DRE entry |
| Issuer sequence continuity per subject | `sequence_gap` | An issuer's own counter has a hole |
| Test artefact hash = build artefact hash | `artifact_mismatch` | The result is not bound to the artefact claimed |
| Test calibration = declared test calibration | `calibration_mismatch` | The tested calibration is not the declared one |
| Production calibration = COP record | `calibration_mismatch` | The declared production calibration is not what was built |
| Every required test case attested | `test_missing` | A required test has no result |
| Raw-data hash present | `raw_data_missing` | The result cannot be traced to data |
| Issuer accreditation valid at test time | `issuer_unaccredited` | The lab was not accredited for that test then |
| Test issuer independent of applicant | `self_issued_test` | The DRE required independence and did not get it |
| Individual credential valid at signing time | `inspector_unqualified` | The signer was not qualified then |
| Checkpoints consistent across relying parties | `split_view` | Two histories, or a set that differs by audience |
| Evidence inside approval validity | `stale_evidence` | Evidence does not support the approval period |

`split_view` is the only outcome that a single-custodian registry cannot produce, because it requires two relying parties to compare independently held checkpoints. Every other outcome is available to a registry that the relying parties are willing to trust.

---

## 6. The process as a state machine

```
 1  DECLARE      authority publishes DRE
        │
        ▼
 2  ATTEST       tier/OEM build emits component/build  ──┐
        │                                                │
        ▼                                                │
 3  TEST         accredited lab emits test  ─────────────┤
        │                                                │
        ▼                                                ▼
 4  REGISTER     log appends, issues receipts, publishes checkpoints
        │
        ▼
 5  AUDIT        enumerate vs DRE → named outcomes
        │
        ├── set complete, bound, consistent ──▶ evidence supports approval
        └── outcome fired ────────────────────▶ authority investigates
        │
        ▼
 6  MEASURE      authority tests production units; compares to declared
```

A type approval under this process is supported by an **audit report that names what was checked**, not by a submission that asserts what was done.

---

## 7. The trust boundary

The process does not eliminate trust. It relocates it and makes some betrayals detectable. The table states exactly who must be honest for what, and whether the process notices when they are not.

| Party | Must be honest about | Detectable? | By what |
| --- | --- | --- | --- |
| Tier build system | Artefact hash, calibration list, sequence | **Partly** | Cross-tier comparison; contradiction with a production record; its own sequence |
| Test issuer (lab) | That the test ran as attested, on the artefact received, with the full result | **No** | Accreditation surveillance, whistleblower, re-analysis of retained raw data |
| Log operator | Append-only, one history | **Yes** | Witnesses; auditor comparing checkpoints |
| Time authority | Time of existence | **Partly** | Witness cosignature timing |
| Authority | The required set, and enforcement | **No** | Outside the process |
| Individual inspector | That they performed the inspection | **Partly** | Credential binding removes the stamping channel; key custody remains |

The single most important row is the second. **The process cannot detect a lab that fabricates a result, and it cannot detect a manufacturer that fabricates data before the first registration.** Any claim that it does is false.

---

## 8. Five structural limits

1. **Event-creation control.** A sequence proves consistency, not completeness. A manufacturer that never creates an attestation for a failed test leaves no gap. Completeness is bounded by *who decides whether the event is created*: lab-created events are controlled by the DRE; in-house events are not. This confirms the finding already recorded in `DISCUSSION-SUMMARY-003`: a sequence number does not guarantee completeness.
2. **Fabrication at source.** A signed hash of fabricated data proves the fabrication was internally consistent. Attribution is all that is gained.
3. **Prototype representativeness.** Crash and durability testing use non-production artefacts by design. Artefact binding cannot establish that a prototype represents production.
4. **Behaviour versus record.** A production artefact that behaves differently under test conditions than in service can pass every check, because every record about it is true. This is the VW case and it is outside the process.
5. **Enforcement.** The process can surface a contradiction. Only the authority can act on it.

Limits 1–4 are why the process is scoped to **software and calibration evidence**, where the tested artefact and the production artefact are the same binary and identity is meaningful. The scope cut in `BUILD-HANDOVER-004` §6.3 is therefore not only a schedule decision; it is the condition under which the claim is true.

---

## 9. The walkthrough

Seven documented cases. For each: what was false, where it sits in the process, what the audit would see, the verdict, and what still requires a human.

Status labels follow `ANL-006`: **V** = adjudicated primary source; **R** = regulatory finding or company admission; **P** = public reporting not independently verified in this review. No case is presented as a court finding unless it is one.

### 9.1 Toyota Industries (2024) — test ECU differed from production ECU

**What was false.** During horsepower certification, the engines were tested with ECUs running software different from mass production. The results appeared smoother and less variable than production units would have produced. Three diesel engine models and ten vehicle models were affected; MLIT issued a correction order in February 2024. **Status: R** — regulatory finding and company admission.

**Where it sits.** Step 2 (the applicant's declaration of what was built) and step 6 (the production record). The test itself was real.

**Trace.**

| Step | What is produced | What the audit sees |
| --- | --- | --- |
| 1 Declare | DRE requires test calibration, production calibration, both hashes, and the COP reference | — |
| 2 Attest | `build{TICO, engine, TC-17, PC-04, h(TC-17), h(PC-04)}` | Both calibrations declared |
| 3 Test | `test{lab, engine, TC-17, h(raw), result}` | Result bound to TC-17 |
| 4 Register | Receipts for both | — |
| 5 Audit | `test.calibration` vs `build.production_calibration`; `build.h(PC-04)` vs COP record | **`calibration_mismatch`** if the declared production calibration is not the one in the COP record; `set_incomplete` if the production calibration field is omitted |

**Verdict: partially flagged.** If the manufacturer records both calibrations honestly, the audit shows that the test used TC-17 while production uses PC-04 — the fact is **visible and non-repudiable**, but the process does not judge whether TC-17 is representative. That judgement needs step 6. If the manufacturer declares TC-17 as both test and production calibration, the contradiction appears only when compared against the authority's independently obtained COP record.

**Residual.** The choice of a non-representative test object. A manufacturer may honestly declare a test configuration that the authority then has to reject. The process moves the question from "trust us" to "here is the configuration; rule on it."

### 9.2 Hino Motors (2022–2025) — falsified test data and undisclosed software functions

**What was false.** Emissions and fuel-economy test data were altered or fabricated; software functions affecting emission controls were not disclosed; 105,000+ non-conforming engines; over **$1.6B** US resolution in January 2025. **Status: R and criminal plea/settlement** — not a Japanese court adjudication.

**Where it sits.** Two distinct places, with different outcomes.

**(a) Fabricated data — step 3.** If the data were fabricated before the result was first registered, the lab signs a hash of fabricated data. The `test` attestation is internally consistent and no outcome fires. If the testing was performed in-house and the DRE requires an independent issuer, the evidence is absent and `self_issued_test` or `set_incomplete` fires — but that only catches the *arrangement*, not the data. An accredited in-house facility defeats it.

**(b) Undisclosed software functions — step 1 and step 2.** A function that is compiled into the image but absent from the declaration does not create a hash mismatch, because the image hash matches the tested image. It is detectable only by two routes: the DRE requires a function-manifest attestation per emissions-relevant function, and independent analysis of the production image enumerates the functions actually present. Hashing cannot do it; the process records the declaration and makes it **attributable after the fact**.

**Verdict: not flagged.** Hino is the clearest case where the process does nothing at the point of failure. Its contribution is retrospective: a later finding can be tied to a specific false declaration, signed, at a known time.

**Residual.** A whistleblower, a third-party committee, or independent measurement. In Hino's case, all three.

### 9.3 Daihatsu (2023–2024) — falsified crash-safety data

**What was false.** Crash-test data were manipulated across dozens of models; a door trim was modified before testing so that a side-airbag deployment indication would not appear. 64 models and 174 items were reported affected. **Status: R** — regulatory and company admission.

**Where it sits.** Step 3, and it exposes limit 3.

**Trace.** Crash testing uses prototypes, not production vehicles. The tested artefact is therefore *not* the production artefact by design, so the `artifact_mismatch` check has nothing to bind. If the DRE required the prototype's configuration to be declared against the production configuration, the door-trim modification would either be declared — visible, and a matter for engineering judgement — or concealed, in which case no hash changes and nothing fires. The data manipulation is the Hino (a) case again.

**Verdict: not flagged.** This is the sharpest demonstration of the scope limit. The process's core property does not map onto prototype-based physical testing.

**Residual.** Engineering review and the third-party committee. The process has no purchase here, and the dissertation should say so rather than stretch the scenario to cover it.

### 9.4 Volkswagen (2015) — defeat device

**What was false.** Production software detected the emissions test cycle and enabled full after-treatment only during the test; on-road NOx was up to roughly 40 times the standard. About 11 million vehicles were affected. Detection came from independent on-road measurement commissioned by the ICCT and performed by West Virginia University, not from certification. **Status: V** — criminal plea and civil settlements.

**Where it sits.** Outside the process. Every record was true: the tested artefact was the production artefact, the result was the result of the test, and the software was present in the image.

**Verdict: not flagged.** This is limit 4, and it is the best-known case in the set. Including it and failing it is what makes the boundary credible.

**Residual.** Independent real-driving measurement. The process can make the *software-function declaration* complete and attributable, which supports a later finding; it cannot detect behavioural divergence.

### 9.5 Nissan (2017) — final inspection by unqualified staff

**What was false.** Final vehicle inspections were performed by staff without the required qualification, and certificates were then stamped by qualified inspectors. About 1.2 million vehicles were recalled in Japan. **Status: R** — regulatory finding.

**Where it sits.** Step 3, at the *individual* level.

**Trace.** The DRE requires each inspection attestation to be signed by the person who performed it, with a credential binding: qualification identifier and validity at signing time. An unqualified performer cannot produce a valid signature, so the inspection either fails to be attested — `inspector_unqualified` — or is attested by someone else, which is an attributable false statement.

**Verdict: partially flagged.** The process removes the **stamping channel**, which was the specific enabler. It does not prevent a qualified person from signing for work they did not do; that is key custody, and it remains a residual.

**Residual.** Identity binding and supervision. Short-lived credentials narrow the window.

### 9.6 Kobe Steel (2017) — falsified materials inspection data

**What was false.** Inspection data on aluminium, copper and steel products were falsified; products shipped with certificates that did not match their properties. Around 500 customers across automotive, rail and aerospace were affected, and the immediate harm was that **nobody could establish which batches were affected**. **Status: R** — company admission.

**Where it sits.** Steps 2 and 5, and it is the best fit in the set.

**Trace.**

| Check | What it produces |
| --- | --- |
| Per-batch `component` attestation with certificate hash and raw-data hash | Each shipment's claim is bound and attributable |
| Supplier sequence per customer | A batch with no attestation is visible to the buyer |
| Buyer's goods-receipt set vs supplier's attested set | `set_incomplete` where a received batch has no attestation |
| Two customers compare checkpoints | **`split_view`** if the supplier tells them different affected sets |

**Verdict: partially flagged.** The harm was enumeration and equivocation, not the truth of the test data. The process makes the affected set enumerable and prevents the supplier from giving different customers different accounts. It does not make the data true.

**Residual.** The truth of the inspection data. Detection of that requires the buyer's incoming test — which the process supports but does not perform.

**Why this case matters.** It is a supplier case with a tier structure, an enumeration harm, and a demonstrated cross-customer equivocation risk. It fits the scenario better than Hino does, and it should be the primary anchor for the evaluation.

### 9.7 Takata — falsified and selectively reported inflator test data

**What was false.** Inflator test results were manipulated and some failing results were omitted from reports; rupturing inflators led to the largest automotive recall on record, in the order of 100 million inflators. **Status: V** — criminal plea and civil settlements.

**Where it sits.** Step 3, and it exposes limit 1 precisely.

**Trace.** If the tests were performed in-house and the failing results were never registered, the manufacturer's own sequence has no gap, because the manufacturer assigns the sequence. **Selective reporting from a self-assigned sequence is invisible.** If the DRE requires an independent lab to attest *every* test it performs, and the lab has its own sequence and its own accreditation interest, then a suppressed result leaves a gap in the **lab's** sequence, not the manufacturer's.

**Verdict: partially flagged — conditional on event-creation control sitting with the lab.** This is the clearest illustration of limit 1 and the strongest argument for decision 5 in `BUILD-HANDOVER-004` (accredited lab as issuer).

**Residual.** Field failures. The process cannot see an event that was never created.

---

## 10. The aggregate

| Case | Manipulation site | Step that can see it | Named outcome | Verdict | What still catches it |
| --- | --- | --- | --- | --- | --- |
| Toyota Industries | Declared vs production calibration | 2, 5, 6 | `calibration_mismatch`, `set_incomplete` | **Partial** | COP measurement; authority judgement |
| Hino | Fabrication at source; omission | 1, 2 | `self_issued_test` (arrangement only) | **Not flagged** | Whistleblower; third-party committee |
| Daihatsu | Prototype configuration; data | — | none | **Not flagged** | Engineering review; committee |
| VW | Production behaviour | — | none | **Not flagged** | Independent on-road measurement |
| Nissan | Individual qualification | 3 | `inspector_unqualified` | **Partial** | Credential custody; supervision |
| Kobe Steel | Batch enumeration; equivocation | 2, 5 | `set_incomplete`, `split_view` | **Partial (best fit)** | Buyer incoming test |
| Takata | Selective reporting | 3 | `sequence_gap` (lab sequence only) | **Partial, conditional** | Field failures |

**Nothing is fully caught. Three are partially caught. Three are not caught at all.**

This is the honest result, and it defines the claim. The process addresses the **record class** of failure: enumeration, artefact binding, attribution, and cross-party equivocation. It does nothing for the **artefact-truth class**: fabrication at source, behavioural divergence, and prototype representativeness.

The two classes are not equally served by existing practice. For the record class there is no incumbent that gives an outside party enumeration *and* cross-party non-equivocation; a registry gives the first if the relying parties trust the operator, and nothing else gives the second. For the artefact-truth class there are incumbents — whistleblower channels, third-party committees, market surveillance, real-driving measurement — and they are the right instruments.

### 10.1 What this changes in the project

1. **Scope narrows to software and calibration evidence.** Daihatsu and VW are outside by construction; that is the boundary, not a gap to be closed.
2. **The primary anchor is Kobe Steel, with Toyota Industries as the identity case.** Hino and Daihatsu become the contrast that motivates the boundary.
3. **The lab-as-issuer decision is load-bearing, not stylistic.** It is the only thing that moves event-creation control outside the manufacturer, and therefore the only thing that makes `sequence_gap` mean anything.
4. **The evaluation metric becomes measurable.** For each case: does the process flag it without a whistleblower or external measurement? Status quo scores 0 of 7. The question is whether the process moves that to 3 of 7, and the answer is testable by replay.
5. **The claim must be rewritten to fit the matrix.** Proposed:

> The process makes the test-evidence set for a type enumerable against an authority-declared requirement, binds each result to the artefact and calibration tested, and makes a party that presents different sets to different relying parties detectable — without any relying party holding the manufacturer's data. It does not establish that the artefact behaves as declared, that fabricated source data is true, or that a prototype represents production.

---

## 11. Evidence register

| # | Source | Claim used | Status |
| --- | --- | --- | --- |
| 1 | Toyota Industries report to MLIT, Jan 2024; MLIT correction order, Feb 2024 | Test ECUs ran software different from mass production; 3 engine models, 10 vehicle models | **R** |
| 2 | Hino Motors admissions, Aug 2022; US DOJ resolution, Jan 2025 | Altered/fabricated emissions and fuel data; undisclosed software functions; 105,000+ engines; >$1.6B | **R / plea** |
| 3 | Daihatsu third-party committee report, Dec 2023 | Crash-test data manipulation; door trim modification; 64 models, 174 items | **R** |
| 4 | US EPA notice of violation, Sept 2015; ICCT/WVU PEMS study, 2014 | Defeat device; ~40× NOx; ~11 million vehicles; detected by independent measurement | **V** |
| 5 | Japanese MLIT inspection findings, 2017 | Final inspection by unqualified staff; certificates stamped; ~1.2 million recalled | **R** |
| 6 | Kobe Steel disclosure, Oct 2017 | Falsified inspection data; ~500 customers; affected-batch enumeration failure | **R** |
| 7 | US DOJ plea, 2017; recall record | Manipulated and selectively omitted inflator test results; ~100 million inflators | **V** |
| 8 | `ANL-006` | The four parcel cases, as method and contrast | **V / A** as labelled |

**Status key.** V = adjudicated primary source; R = regulatory finding or company admission; P = public reporting not independently verified; A = allegation.

---

## 12. Open questions

1. **Who writes the DRE, and at what granularity?** The process's value is bounded by the declaration. This is the supervisor's most likely objection and needs an answer in `CNP-002`.
2. **Does the authority already hold a production record independent of the manufacturer?** G6 in `BUILD-HANDOVER-004` is not optional; without it, Toyota Industries is visible but unjudgeable.
3. **Is the lab an issuer or an auditor?** This specification makes it an issuer, because issuer status is what moves event-creation control. Confirm against how accreditation actually works in the target jurisdiction.
4. **Do tiers sign?** If a tier refuses, the cross-tier completeness property collapses and only the OEM's own set is auditable.
5. **Which jurisdiction?** UNECE R155/R156 and ISO 24089 supply the vocabulary; none supplies the DRE. The project should model a plausible DRE, not claim one exists.
6. **Raw-data retention.** `raw_data_missing` is only useful if the raw data can later be re-examined; the retention rule is a policy decision with privacy consequences.

---

## 13. One-paragraph summary

The process has six steps: the authority declares the required evidence set; tiers and the OEM attest at build; an accredited lab attests the test bound to the artefact it received; an independent log registers the set; the authority and each tier audit the set against the declaration; and the authority measures production independently. Traced against seven documented scandals, nothing is fully caught, three are partially caught, and three are not caught at all. Toyota Industries becomes a visible calibration contradiction; Kobe Steel becomes an enumerable, non-equivocal affected set; Nissan loses the stamping channel; Takata is caught only if the lab, not the manufacturer, controls whether a test is recorded. Hino, Daihatsu and VW are outside the process, because the fabrication happened before the record existed, the test object was a prototype, or the production artefact behaved differently from the tested one. The contribution is therefore not scandal prevention but a characterised boundary plus a working mechanism for the record class: the set is enumerable, the result is bound to the artefact tested, and a party that shows different sets to different relying parties is detectable, without any relying party holding the manufacturer's data.
