# DEC-003 — G1/G2 Supervisor Decision Brief

> *Offline proof that a certificate was committed at issuance, and checkpoint proof of what the record does not contain*

| Field | Value |
| --- | --- |
| Document ID | DEC-003 |
| For | Supervisor, COMP4299 |
| From | [Student] |
| Date | 2026-10-24 |
| Module | COMP4299 — final-year dissertation (dissertation, demonstration and evaluation) |
| Proposal | `CNP-004` v0.6.4 — A Verifiable Issuance Register for Safety-Critical Certificates |
| Companion | `CNP-003` v0.3.1 — the accreditation-scope extension, future work |
| Status | **Decision requested. Direction not approved. No implementation started.** |

## 1. The two decisions requested

**G1 — Direction.** Approve the proposed direction: a verifiable issuance register for safety-critical certificates, evaluated on ISO/IEC 17025 calibration certificates for permanently installed lift brake-force sensors, in a Macau setting.

**G2 — Module constraints, in writing.** Confirm that the module accepts:

- a narrow mechanism with a measured boundary;
- a positive headline — a capability, not a critique;
- a plausible, labelled scenario rather than a recruited stakeholder;
- the calibration-certificate scope and the Macau setting;
- no implementation before G1 and G2 are recorded.

## 2. The direction in one paragraph

A safety-critical certificate can be signed, timestamped and never committed to an archive. The signature proves who signed. It does not prove that the certificate was logged, that the archive holds the full set for an asset, or that a required entry is absent. The proposal is a witnessed append-only register. The laboratory submits the certificate hash at issuance. The operator appends a leaf and commits a log root and a subject index in each checkpoint. Three witnesses cosign at 2-of-3, with at least 2 honest and each checking consistency before cosigning. The receipt travels with the certificate. A verifier checks it offline. An investigator queries the subject index and receives inclusion, non-inclusion and deadline proofs against a supplied calibration schedule.

## 3. The claim, and its limits in the same section

> A verifier can establish, offline and years later, that a certificate was committed to the archive by the laboratory it names, within 1 checkpoint cadence of the certificate date; and an investigator can establish, from the archive, the full set of certificates the register holds for an asset and the absence of any entry required by a schedule supplied with the query — without the laboratory's cooperation and without trusting the operator's account of its own history.

**The claim holds under the conditions in `CNP-004` §4.6–§4.7 — in particular, that the laboratory logs each certificate at issuance and that the decision-maker refuses a certificate without a receipt.** Without the second condition, a laboratory can issue without logging, and the archive does not cover the certificate.

**The claim does not establish:** that the work was performed; that the archive contains every certificate issued; that the asset reference is correct; that the supplied calibration schedule is correct; that the individual signer was authorised; or that the register has legal recognition. It also does not close the window between a period deadline and the first post-deadline checkpoint, and it assumes no key compromise and an honest TSA. The full limits are in `CNP-004` §4.5.

## 4. What the register does, and does not do

| It does | It does not |
| --- | --- |
| Offline proof of commitment at a checkpoint | Prove the calibration was performed |
| Proof of absence relative to a supplied schedule | Prove the archive is complete |
| Detection of deletion, alteration and post-deadline backfill | Prove the asset reference is correct |
| Fork prevention by an honest witness majority; fork detection by comparison | Establish signer authority or legal recognition |

## 5. The comparison

| Arm | Design | What it adds |
| --- | --- | --- |
| **A** | Current practice: signed certificate + telephone call | Baseline |
| **B** | Signed DCC + X.509 + RFC 3161 | Authenticity, key binding, signing time, non-repudiation |
| **C** | Central registry with a signed receipt | Trusted commitment and unproven database enumeration |
| **D1** | C plus the log, checkpoints and subject index; no witnesses — the ablation | Merkle proofs, but conditional on the operator |
| **D2** | D1 plus 2-of-3 witnesses — the proposal | Unconditional archive proofs against an untrusted operator, given an honest witness majority |

`D2` is the only arm that answers all five archive rows — commitment, absence, deletion or alteration, backfill and non-equivocation — unconditionally against an untrusted operator, given an honest witness majority. `D1` isolates the witness contribution; `C` is the strongest non-log baseline. The contribution is the object, the subject index with absence proofs, the profile and the measurement, not the log, receipt, timestamp or witness mechanism alone.

The full matrix, the trust assumptions and the reading notes are in **`DEC-003a`**.

## 6. Scope, evaluation and demonstration

**In scope:** issuance register; ISO/IEC 17025 calibration certificates for permanently installed lift brake-force sensors; DCC-compatible receipt; Merkle log, subject index, allow-list, checkpoints, 2-of-3 witnesses, 2-of-3 audit attestation, offline verifier and archive query; simulated operator and synthetic data; 5 arms and 10 attacks.

**Out of scope:** the accreditation-scope question (`CNP-003`, future work); other certificate classes; certificate contents; full DCC schema validation; legal recognition; a live public log; a real operator integration.

**Evaluation:** 5 arms (A, B, C, D1, D2) against 10 attacks — forged, tampered, backdated, repudiated, deleted, rewritten, fork, missing witness, filled gap and backfilled. Metrics: detection, latency, proof size and trust-anchor count.

**Demonstration:** brake-force sensor BF-4471 on lift L-4471; 2020 and 2022 logged; 2024 absent; the laboratory dissolved; an insurer change in 2025; an incident in 2026; the investigator proves the missing year. All names are fictional.

## 7. Schedule

| Phase | Weeks | Effort | Output |
| --- | --- | --- | --- |
| P0 — Approval | W1 | ~8 h | Signed-off proposal (M0) |
| P1 — Design | W2–W5 | ~54 h | C1; Gate G-A |
| P2 — Implementation | W6–W12 | ~84 h | C2–C6 |
| P3 — Evaluation | W13–W15 | ~34 h | C7; Gate G-B |
| P4 — Writing and demonstration | W16–W19 | ~62 h | C8, C9 |
| P5 — Finalisation | W20 | ~8 h | C10 |
| **Total** | **20 weeks** | **~250 h** | |

**G-A** asks whether the receipt, the index and the verifier are tractable. **G-B** asks whether D2 provides a property the baselines do not.

## 8. Evidence status

| Fact | Status |
| --- | --- |
| Macau Law No. 14/2022: lift registration, annual inspection, notification of every declaration (Art. 6(6)) | **V** |
| No Macau instrument requires calibration certificates to be logged | **P** |
| IAF CertSearch covers ISO/IEC 17021-1 only | **V** |
| No verifiable register of safety-critical certificates with a subject index | **P** — the limitation check has not run |
| No adjudicated case of a laboratory denying a certificate it signed | **A** |
| Macau calibration-certificate practice | **Not verified**; the scenarios are illustrative and labelled A |

## 9. Principal risks

| Risk | Mitigation |
| --- | --- |
| The increment over B + C is too small | Measure B + C → D2 and D1 → D2 on the five archive rows; if either delta is zero, R1 fires |
| The mandate is not adopted | Argue the statute, procurement and liability routes; evaluate under a stated mandate |
| A proven gap is read as "the work did not happen" | Label `absent` separately from `not performed` in every output |
| Privacy: the index links certificates to assets | Choose the visibility model at C1; regulator-only leaves with public checkpoints |

## 10. Decision record

**G1 — Direction**

☐ Approved   ☐ Approved with changes   ☐ Not approved

**G2 — Module constraints**

☐ Confirmed in writing   ☐ Partly confirmed   ☐ Not confirmed

**Comments**

______________________________________________________________________________

______________________________________________________________________________

Supervisor: ______________________________   Date: ______________

## 11. If approved

1. Record G1 and G2, and any required changes, in this document.
2. Run the 2-day limitation check (G3: no authority or industry body publishes a verifiable register with a subject index; G4: IAF CertSearch remains ISO/IEC 17021 only). Record the result in `CNP-004` §2.4 or a short note.
3. If the limitation survives, start P1 (Design, C1). If it is removed, stop and report. Do not re-scope without approval.

---

*This brief records a decision request. It does not record approval and does not authorise implementation.*
