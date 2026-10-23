# BUILD-HANDOVER-006 — Accreditation-Scope Register

| Field | Value |
| --- | --- |
| Prepared | 2026-10-23 |
| For | The next BUILD agent continuing after the accreditation-scope proposal |
| Repo | `/home/hj-core/project` |
| Branch | `tdd` |
| HEAD | `a9a5d40` |
| Working tree | **Uncommitted.** `CNP-003` and this handover are untracked. **Do not stage or commit.** |
| Read first | `CNP-003`, then this file |
| Status | **Direction proposed. Supervisor approval not recorded. No build started.** |
| Supersedes | `BUILD-HANDOVER-005` **for the current state**. -005 remains the record of the employment roast. |

**Update 2026-10-23, after review.** `CNP-003` is now v0.3.1; this handover describes v0.1.0. The direction is unchanged. Current state: 3 states and 6 event types, not 5 states; 9 attacks, not 6; 272 hours, not 262; the claim is scope on the certificate date plus non-equivocation, and the design claims 1 unique property against the embedded-snapshot baseline B′, not 5 unique answers; the receipt carries an issuance attestation, per-event inclusion proofs and a freshness rule. The mandate is a condition of operation in §4.8, with R1 at Medium. Read `CNP-003` for the argument. G1 and G2 remain open.

---

## 1. Status at a glance

| Item | State |
| --- | --- |
| Module | **COMP4299** — confirmed |
| Deliverable | Dissertation + demonstration + evaluation — confirmed |
| Current direction | **A Verifiable Accreditation-Scope Register** (`CNP-003`) |
| Object | The accreditation body's register of laboratory scope, instantiated on ISO/IEC 17025 torque calibration |
| Claim | A verifier can establish offline that a calibration certificate fell inside the laboratory's scope at issuance, and that the register's record has not changed since |
| Strongest alternatives | IAF CertSearch (ISO/IEC 17021 only), accreditation-body e-certificates, DCC/signed PDFs, central registers with history |
| Surviving cells | Issuance-time scope, non-revision, non-equivocation |
| Build | **Not started** |
| Decision | **Supervisor approval pending** |

---

## 2. The job

1. **Do not build.** The direction is a proposal, not an approved pivot. The supervisor has not recorded a decision.
2. **Read `CNP-003` first.** It is self-contained: problem, evidence, claim, comparison, scope, deliverables, schedule, risks, demonstration.
3. **Take `CNP-003` to the supervisor.** The decision is whether to approve this direction for COMP4299, with the module constraints confirmed in writing.
4. **If approved, run the 2-day limitation check.** Confirm that no accreditation body publishes a versioned, offline-verifiable ISO/IEC 17025 scope history, and that IAF CertSearch still covers ISO/IEC 17021 only. If the limitation is already removed, stop and report.
5. **If the limitation survives, start P1 (Design).** Build the scope predicate, the 5-state register model, the wire format and the first chapter draft.
6. **Do not run another domain search** without the filter in §6 of `BUILD-HANDOVER-005`. Six domains have now been tested; 5 failed or were occupied.

---

## 3. What happened since -005

`BUILD-HANDOVER-005` left the employment direction at one proposition: offline status at a past decision time. This session tested that proposition and replaced it.

| Step | Result |
| --- | --- |
| H2 analysis: does a signed status snapshot plus TSA answer the historical question offline? | **Yes, for irreversible revocation.** A later signed snapshot proves the record was not revoked earlier; a log cannot prove a negative. The log's residual is non-equivocation |
| Employment narrowed | The surviving cell is non-equivocation, which failed for the parcel case. Employment is on hold in `CNP-002` |
| Certificate issuance considered | CT and Sigstore own the deployed cases; no compulsion outside TLS. The field passes the filter but has no white space |
| Offline access control considered | The question is "is it valid now?", not "was it valid then?" Short-lived credentials and cached status lists already answer it |
| Lab and QC certificates considered | The register is current-state and signals withdrawal by **deletion**. No global register covers ISO/IEC 17025. This is the surviving direction |
| `CNP-003` written | 9 sections, 7,132 words; self-contained; tone-audited |
| User decisions | Keep the web view; drop the 3-week evidence phase; start the schedule at Week 1; add the demonstration design; cut optional deliverables; apply the tone rules |

**The one-sentence version.** The log cannot prove a negative, so it cannot own historical status; it can own the history of a register, and the only register in six domains whose normal operation is silent deletion is the accreditation-scope register.

---

## 4. The surviving direction

### 4.1 Problem

A verifier holds a calibration certificate and must establish 3 things: whether the laboratory was accredited for this calibration, whether the accreditation covered the certificate date, and whether the record is the one the accreditation body published.

The register answers the current-state question only. UKAS states that "absence of a schedule … indicates that the accreditation is no longer in force." Withdrawal is a deletion. Schedules are revised and reissued in place. The verifier cannot recover the state at issuance.

### 4.2 Evidence

| Fact | Status |
| --- | --- |
| Presence of a schedule means current accreditation; absence means it is no longer in force; certificates carry no expiry | **V** — UKAS FAQ |
| IAF CertSearch is mandated by IAF Mandatory Document 28 for **ISO/IEC 17021-1** certificates, with real-time alerts and about 2.5 million certifications | **V** — IAF |
| No equivalent global register covers **ISO/IEC 17025** calibration certificates | **V** (scope), **P** (absence) |
| UKAS suspended all BBA scopes on **2026-02-26**; existing certificates remain valid until expiry | **V** — UKAS |
| TWI Certification's ISO/IEC 17024 accreditation was suspended and reinstated in 2026 | **V** — UKAS; TWI |
| The Grenfell Tower Inquiry recommended attaching supporting test results and the full testing history to certificates | **V** — Phase 2 |
| PIP/TÜV Rheinland suspended certificates when fraud surfaced; liability litigation followed | **V** — court and TÜV statements |
| An adjudicated dispute on the **scope date** | **A** — not found |

### 4.3 Solution

The accreditation body appends scope statements to a register log (RFC 9162, Tessera). Checkpoints are cosigned by independent witnesses. A calibration certificate carries a **scope receipt**: certificate hash, scope entry, Merkle inclusion proof, cosigned checkpoint, RFC 3161 timestamp and verification policy. The verifier checks the receipt offline and returns a named outcome.

### 4.4 Claim and limits

> A verifier can establish, at decision time and without contacting the laboratory or the accreditation body, that a calibration certificate was issued by a laboratory whose accreditation scope covered the calibration when the certificate was issued, and that the register's record of that scope has not changed since.

The claim does **not** establish truth of the measurement, completeness of the register, authority of the individual signer, or legal recognition. These limits are in `CNP-003` §4.6.

### 4.5 Adoption precondition

The design requires 2 conditions: the accreditation body logs every scope change, and the verifier refuses a calibration certificate without a scope receipt. The second condition is absent today. The dissertation states it as a precondition, not a result. The number of potential loggers is small: about 50 national accreditation bodies.

### 4.6 Evaluation

4 arms:

- **A** — current practice: manual check and accreditation-body schedule.
- **B** — signed scope-snapshot chain with timestamps.
- **C** — central register with current status, event history and alerts.
- **D** — the proposed register with scope receipts.

6 core attacks: forged certificate, out-of-scope calibration, retrospective reduction, deletion, rewrite, fork. Remaining threats are reported as limitations.

### 4.7 Scope and schedule

- **Certificate class:** ISO/IEC 17025 **torque calibration** only. Other parameters and test certificates are boundary cases.
- **Artefact:** scope receipt in a DCC-compatible envelope (subset, not full v3.3.0 validation).
- **Register:** 5-state machine — grant, reduction, suspension, reinstatement, withdrawal; signed allow-list.
- **Schedule:** 20 weeks, ~262 hours, 12–15 hours per week, 5 phases P0–P5, gates G-A (design) and G-B (evaluation).
- **Writing:** chapter drafts in P1–P3; full draft to the supervisor by end W17; revisions W18–W19; submission W20.
- **Demonstration:** lift-maintenance torque calibration, 2021; scope reduced 2024; laboratory suspended 2025; insurer verifies offline in 2026.

---

## 5. Decision state

| Item | State |
| --- | --- |
| Domain | **Proposed** — accreditation-scope register |
| Title | **Proposed** — A Verifiable Accreditation-Scope Register |
| Claim | **Proposed** — offline scope at issuance and non-revision |
| Scope | **Proposed** — ISO/IEC 17025 torque calibration, Macau setting |
| Deliverables | **Proposed** — C1–C9 |
| Schedule | **Proposed** — 20 weeks, 262 hours |
| Supervisor approval | **Not recorded** |
| Module constraints in writing | **Pending** |
| Build | **Not started** |

The old `D8` (proceed or stop) and `G1` (supervisor approval) remain the decision. `CNP-003` replaces the D8 framing with a concrete proposal. `G1` is still open.

---

## 6. Open questions and gates

| # | Question | Why it matters |
| --- | --- | --- |
| **G1** | Does the supervisor approve the accreditation-scope direction? | No build before approval |
| **G2** | Are the module constraints confirmed in writing — narrow mechanism, measured boundary, positive headline, plausible scenario? | Shapes the claim and the evaluation |
| **G3** | Does any accreditation body publish a versioned, offline-verifiable ISO/IEC 17025 scope history? | If yes, the limitation is removed and the project stops |
| **G4** | Does IAF CertSearch extend to ISO/IEC 17025 with history? | Incumbent risk R2 |
| **G5** | Can procurement or regulation require a verifier to refuse unlogged certificates? | The log only covers logged certificates |
| **G6** | Is torque calibration the right single family? | Scope and evaluation cost |
| **G7** | Are the Macau legal and institutional references correct? | Accuracy; no law numbers are cited yet |
| **G8** | Is there an adjudicated scope-date case? | Evidence is A; the project does not gate on it |
| **G9** | DCC subset or full v3.3.0 schema? | C4 effort; the proposal uses a subset |
| **G10** | What is the module calendar? | P4 writing and submission |

---

## 7. Next 5 actions

1. **Read `CNP-003`.** Do not rely on this handover for the argument; it records state.
2. **Book the supervisor review.** Use `CNP-003` §1 (title), §4.7 (claim), §6 (scope) and §8 (schedule). Ask for a written decision on G1 and G2.
3. **If approved, run the 2-day limitation check (G3/G4).** Check UKAS, CNAS and HKAS schedule versioning; check IAF CertSearch scope; check any national 17025 history. Record the result in `CNP-003` §2.2 or a short note.
4. **If the limitation survives, start P1.** Build the scope predicate, the 5-state model and the wire format; draft the problem and evidence chapter.
5. **If the limitation is removed, stop.** Report the finding to the supervisor. Do not re-scope without approval.

---

## 8. Document map and read order

**Current**

1. `CNP-003` — the proposal. Read first.
2. `BUILD-HANDOVER-006` — this file.

**Historical (do not edit)**

- `CNP-002` — employment concept note, on hold.
- `ANL-009` — employment roast; the two fatal findings.
- `DEC-002` — automotive drop and employment pivot.
- `ANL-008` — employment anchors.
- `PRC-001` — automotive process and 7-case trace.
- `BUILD-HANDOVER-005` — the employment state and the four-question filter.
- `DISCUSSION-SUMMARY-005` — the session record for the employment pivot.
- `CNP-001`, `BUILD-HANDOVER-003`, `-004`, `DISCUSSION-SUMMARY-003`, `-004`, `PRP-001`, `PRP-001a`, `ANL-004`–`ANL-007` — parcel-era records.

**Standards and references**

- RFC 9943 (SCITT architecture), RFC 9942 (COSE receipts), RFC 9162 (CT v2 log), RFC 3161 (TSA), RFC 9052 (COSE).
- W3C Bitstring Status List; IETF Token Status List.
- DCC v3.3.0 (PTB); IAF Mandatory Document 28; ISO/IEC 17025; ISO/IEC 17011; ISO/IEC 17021.
- Tessera; `scitt-cose`; `transparency-dev/witness`.

---

## 9. Git and document state

- Branch `tdd`; HEAD `a9a5d40`.
- `CNP-003` is untracked. This handover is untracked.
- The user commits the documents. **Do not stage or commit.**
- `BUILD-HANDOVER-005` records HEAD `8246f7f` and an uncommitted tree; that state is stale.

---

## 10. Cautions

- Do not call the design a blockchain. Call it a register log or a witnessed append-only log.
- Do not claim truth of the measurement, completeness, signer authority or legal recognition. State the limits in the same section as the claim.
- Do not claim novelty for Certificate Transparency, SCITT, signed certificates or revocation. The contribution is the object (accreditation scope), the profile and the measurement.
- Do not claim "no one is doing this." IAF CertSearch, DCC and accreditation-body e-certificates exist.
- Do not present the Macau scenarios as documented cases. They are labelled A.
- Do not cite Macau law numbers without checking the current instruments.
- Do not describe RFC 9943 as requiring witnesses, tiers or a public log. It does none of the three.
- Do not use `should` or `we` in project documents. Follow the tone rules applied to `CNP-003`: present tense, active voice, ISO dates, digits for quantities, 1 name per thing.
- Do not treat the ISO/IEC 17025 number as the title. It is a boundary label.
- Do not build before G1 and G2 are recorded.

---

## 11. One-paragraph summary

The employment direction ended at one proposition, and the H2 analysis showed that a signed status snapshot plus a TSA answers the historical-status question offline; the log cannot prove a negative. The search then tested certificate issuance, offline access control and laboratory certificates. The first 2 are occupied or answer a "now" question. The third produced the surviving direction: the accreditation body's scope register is current-state and signals withdrawal by deletion, no global register covers ISO/IEC 17025, and a verifier cannot recover the scope at issuance. `CNP-003` proposes a register log with scope receipts, witness cosignatures and an offline verifier, instantiated on torque calibration in a Macau setting. The claim is offline proof of scope at issuance and non-revision; the limits are truth, completeness, signer authority and legal recognition; the precondition is that accreditation bodies log and verifiers refuse unlogged certificates. The next agent's job is not to build. It is to take the proposal to the supervisor, obtain a written decision on the direction and the module constraints, and run the 2-day limitation check before any design work begins.
