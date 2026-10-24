# BUILD-HANDOVER-008 — Supervisor Package Ready, Comparison Ladder Settled

| Field | Value |
| --- | --- |
| Prepared | 2026-10-24 |
| For | The next BUILD agent continuing after the comparison and supervisor-package work |
| Repo | `/home/hj-core/project` |
| Branch | `tdd` |
| HEAD | `eabe67c` |
| Working tree | Clean at handover start. This handover and `DISCUSSION-SUMMARY-007` are untracked. **Do not stage or commit.** |
| Read first | `CNP-004` v0.6.4, then `DEC-003`, then `DEC-003a`, then this file, then `CNP-003` v0.3.1 |
| Status | **Direction proposed. Supervisor approval not recorded. No build started. Limitation check not run.** |
| Supersedes | `BUILD-HANDOVER-007` for the current state. -007 remains the record of the earlier proposal state. |

---

## 1. Status at a glance

| Item | State |
| --- | --- |
| Module | **COMP4299** — confirmed |
| Deliverable | Dissertation + demonstration + evaluation — confirmed |
| Current direction | **A Verifiable Issuance Register for Safety-Critical Certificates** (`CNP-004` v0.6.4) |
| Object | Certificate issuance: hashes, a subject index, receipts, witnesses |
| Claim | Offline proof of commitment at issuance, and archive proof of what is absent |
| Companion | `CNP-003` v0.3.1 — the accreditation-scope extension, future work |
| Comparison | A, B, C, D1, D2; D1 is the ablation, D2 the proposal |
| Unique property | D2 is the only arm with unconditional proofs on all five archive rows against an untrusted operator |
| Effort | 250 hours over 20 weeks |
| Build | **Not started** |
| Decision | **G1 and G2 open** |
| Limitation check | **Not run** |
| Supervisor package | **`DEC-003` and `DEC-003a` ready** |

---

## 2. The job

1. **Do not build.** `CNP-004` is a proposal, not an approved direction. G1 has not been recorded.
2. **Read `CNP-004` v0.6.4 first.** It is self-contained: problem, evidence, claim, comparison, scope, deliverables, schedule, risks.
3. **Take `DEC-003` and `DEC-003a` to the supervisor.** `DEC-003` is the two-page decision brief with the G1/G2 checkboxes and signature block. `DEC-003a` is the §5.1 comparison handout. Use `CNP-004` for the full argument.
4. **Ask for a written decision on G1 and G2.** G1 is the direction; G2 is the module constraints in writing. Record both in `DEC-003`.
5. **If approved, run the 2-day limitation check (G3/G4).** Confirm that no authority or industry body publishes a verifiable register of safety-critical certificates with a subject index, and that IAF CertSearch remains ISO/IEC 17021 only. Record the result in `CNP-004` §2.4 or a short note.
6. **If the limitation survives, start P1 (Design).** C1: leaf, checkpoint, allow-list, index, receipt and audit-attestation formats; the UTC period mapping; the visibility model; the exact checkpoint interval; the deadline-audit cost.
7. **Do not run another domain search.** Six domains have been tested; the filter in `BUILD-HANDOVER-005` §6 still applies.
8. **Do not merge `CNP-003` and `CNP-004` without approval.** The combined design is not in scope.

---

## 3. What happened since -007

`BUILD-HANDOVER-007` left `CNP-004` at v0.4.2 with a single central-registry baseline. This session turned it into a five-arm comparison, corrected two cryptographic errors, and produced the supervisor package.

| Step | Result |
| --- | --- |
| `CNP-004` v0.4.3 | Required-period schedule as a query input; maximum checkpoint interval; claim conditions 7 → 9 |
| `CNP-004` v0.4.4 | Subtitle tightened: commitment at issuance, checkpoint proof of absence |
| §5.1 self-audit | 10 blocking findings, including C undefined, issuance overclaim, time conflation, absence/proof contradiction, backfill window |
| `CNP-004` v0.5.0 | Single C baseline replaced by C-R (receipt) and C-L (unwitnessed log); Macau status-quo claim removed |
| External review 1 | 2-of-3 quorum error; fork prevention vs detection; C-L misclassified as existing practice; C-L over-credited → v0.6.0 |
| `CNP-004` v0.6.1 | Arms renamed C-R → C, C-L → D1, D → D2 |
| External review 2 | 1-of-3 audit attestation loophole; backfill condition inverted; C enumeration over-credited; C6 outcomes incomplete → v0.6.2 |
| `CNP-004` v0.6.3 | D1 defined forward: C plus log/index without witnesses; D2 = D1 + witness layer |
| External review 3 | D1 enumeration over-credited; deadline checkpoint unattested; deletion wording imprecise; cross-document drift → v0.6.4 |
| `DEC-003` | Two-page G1/G2 supervisor decision brief |
| `DEC-003a` | One-page §5.1 comparison handout |

**The one-sentence version.** PKI proves who signed; the register proves the certificate was committed, cannot be withdrawn, cannot be backfilled, and that a required entry is missing — and D2 is the only arm that does so without trusting the operator.

---

## 4. The surviving direction

### 4.1 Problem

A safety-critical certificate can be signed, timestamped and never committed. A verifier can check the signature; an investigator cannot check the archive. After an accident, the record is a pile of PDFs: nobody holds the whole set, absence is indistinguishable from a lost file, and the laboratory may have dissolved.

The failure is not authenticity. It is **archive commitment**, and the consequence is that the record looks complete when it is not.

### 4.2 Evidence

| Fact | Status |
| --- | --- |
| Macau Law No. 14/2022 requires lift registration with DSSCU, annual inspection, and notification of every inspection declaration (Art. 6(6)); the declaration is valid for 1 year | **V** |
| No Macau instrument requires the ISO/IEC 17025 calibration certificates beneath those inspections to be registered | **P** |
| IAF CertSearch covers ISO/IEC 17021-1 only, about 2.5 million certifications | **V** |
| 121 ILAC MRA signatories; over 114,600 accredited laboratories | **V** |
| Aviation, medical devices, legal metrology and vehicle inspection mandate long-lived records | **V** |
| Grenfell Tower Inquiry Phase 2 report (2024-09-04) recommended attaching the full testing history to certificates | **V** |
| No verifiable register of safety-critical certificates with a subject index | **P** — the limitation check has not run |
| No adjudicated case of a laboratory denying a certificate it signed | **A** |
| Macau calibration-certificate practice | **Not verified**; scenarios are illustrative and labelled A |

### 4.3 Solution

The laboratory signs the certificate and submits its hash at issuance. The log appends a leaf: certificate hash, laboratory, key, certificate date, subject reference and certificate type. The checkpoint commits the log root and a subject index `(subject_ref, period)`, where `period` is the UTC calendar year of the certificate date.

The receipt carries the leaf, the inclusion proof, the checkpoint and the TSA token. The field verifier checks it offline. The investigator queries the index for a period range and receives inclusion, non-inclusion and deadline proofs against a supplied schedule.

Three witnesses cosign at 2-of-3, with **at least 2 honest** and each checking an RFC 9162 consistency proof before cosigning. The audit attestation is **2-of-3**, with each signer verifying the projection. The audit runs at each period-deadline checkpoint and at an incident. A witness refuses a checkpoint across a gap longer than the published maximum interval.

### 4.4 Claim and limits

> A verifier can establish, offline and years later, that a certificate was committed to the archive by the laboratory it names, within 1 checkpoint cadence of the certificate date; and an investigator can establish, from the archive, the full set of certificates the register holds for an asset and the absence of any entry required by a schedule supplied with the query — without the laboratory's cooperation and without trusting the operator's account of its own history.

The claim holds under 9 conditions in §4.6. The most important are: the laboratory logs at issuance; at least 2 of the 3 witnesses are honest and check consistency; the TSA is honest; the investigator holds an attested checkpoint at or after the range end plus 1 cadence, and an attested checkpoint at each deadline; the operator publishes at least every maximum interval.

The claim does not establish that the work happened, that the archive is complete, that the asset attribution is correct, that the supplied schedule is correct, that the individual signer was authorised, or that the register has legal recognition. It does not close the window between a period deadline and the first post-deadline checkpoint, and it assumes no key compromise. The full limits are in §4.5.

### 4.5 Evaluation

5 arms: **A**, current practice; **B**, a signed DCC under X.509 with an RFC 3161 token; **C**, a central registry with a signed receipt; **D1**, C plus log/checkpoints/index without witnesses; **D2**, D1 plus witnesses.

10 attacks: forged certificate, tampered certificate, backdated certificate, repudiation, deleted entry, rewrite, fork, missing witness, filled gap, backfill after incident. Metrics: detection, latency, proof size, trust-anchor count.

### 4.6 Scope and schedule

- **Certificate class:** ISO/IEC 17025 calibration certificates for permanently installed lift brake-force sensors. Overload and pressure sensors are boundary cases.
- **Artefact:** issuance receipt in a DCC-compatible envelope (subset, not full v3.3.0 validation).
- **Register:** Merkle log, subject index, allow-list, checkpoints, 3 witnesses at 2-of-3 with ≥2 honest, 2-of-3 audit attestation at deadline checkpoints.
- **Schedule:** 20 weeks, ~250 hours, P0–P5, gates G-A (design) and G-B (evaluation).
- **Demonstration:** BF-4471 on lift L-4471; 2020 and 2022 logged, 2024 missing, laboratory dissolved, insurer change 2025, incident 2026, deadline checkpoint CP-2500.

---

## 5. The comparison ladder

| Arm | Design | Role |
| --- | --- | --- |
| **A** | Signed certificate + telephone call | Current practice; generic, not a verified account of Macau practice |
| **B** | Signed DCC + X.509 + RFC 3161 | Strongest per-document baseline |
| **C** | Central registry with a signed, TSA-timestamped receipt; no Merkle log | Strongest non-log baseline |
| **D1** | C plus Merkle log, signed checkpoints and subject index, **without witnesses** | The ablation; not existing practice |
| **D2** | D1 plus 2-of-3 witnesses, ≥2 honest, consistency checks, 2-of-3 audit attestation | **The proposal** |

**The five archive rows:** commitment, absence, tamper evidence, backfill detection, non-equivocation.

**The grading rules a new agent must not accidentally undo:**

- C: commitment ◐†, enumeration ◐†, second certificate ◐†, absence ✗, tamper ◐†, backfill ✗, offline ✗, non-equivocation ✗.
- D1: commitment ◐†, enumeration ◐†, second certificate ◐†, absence ◐†, tamper ◐†, backfill ◐†, offline ✗, non-equivocation ✗.
- D2: ✓ on all five archive rows; offline verification without trusting operator history ✓.
- D2 is the only arm with **unconditional** proofs on all five archive rows **against an untrusted operator**, given an honest witness majority.
- Falsifier 2 compares **D2 to B + C**, not to D1. D1 is the internal ablation.
- C's enumeration is a database response, not a proof of completeness.
- D1's Merkle proofs are conditional because the checkpoint is operator-signed.

---

## 6. Decision state

| Item | State |
| --- | --- |
| **G1** | Supervisor approval of the direction — **not recorded** |
| **G2** | Module constraints in writing — **pending** |
| **G3** | No existing verifiable register with a subject index — **not checked** |
| **G4** | IAF CertSearch scope — **not checked this session** |
| **G5** | Mandate route: statute, procurement or liability — **open** |
| **G6** | Public leaves or regulator-only leaves with public checkpoints — **open; C1** |
| **G7** | Maximum checkpoint interval and who runs the 2-of-3 audit at each deadline — **open; C1** |
| **G8** | Combine `CNP-003` and `CNP-004` into one log — **not approved** |
| **G9** | Can the O(N) audit run at every deadline checkpoint within the pilot budget? — **open; C1** |

---

## 7. Next 5 actions

1. **Read `CNP-004` v0.6.4, `DEC-003` and `DEC-003a`.** Do not rely on this handover for the argument.
2. **Book the supervisor review.** Use `DEC-003` as the decision document and `DEC-003a` as the comparison handout. Ask for G1 and G2 in writing.
3. **If approved, run the 2-day limitation check (G3/G4).** Check national registers, IAF CertSearch scope, and any industry certificate register. Record the result.
4. **If the limitation survives, start P1.** C1: wire formats, period mapping, visibility model, exact checkpoint interval, deadline-audit cost.
5. **If the limitation is removed, stop.** Report to the supervisor. Do not re-scope without approval.

---

## 8. Document map and read order

**Current**

1. `CNP-004` v0.6.4 — the proposal. Read first.
2. `DEC-003` — the G1/G2 supervisor decision brief.
3. `DEC-003a` — the §5.1 comparison handout.
4. `DISCUSSION-SUMMARY-007` — this session's record.
5. `BUILD-HANDOVER-008` — this file.
6. `CNP-003` v0.3.1 — the companion scope proposal.

**Historical (do not edit)**

- `BUILD-HANDOVER-007` — the state before this session; superseded by this file.
- `BUILD-HANDOVER-006` — the scope state; its update block is correct, its body describes v0.1.0.
- `CNP-002`, `ANL-009`, `DEC-002`, `ANL-008`, `PRC-001`, `BUILD-HANDOVER-005`, `DISCUSSION-SUMMARY-005` — employment and parcel records.
- `CNP-001`, `BUILD-HANDOVER-003`, `-004`, `DISCUSSION-SUMMARY-003`, `-004`, `PRP-001`, `PRP-001a`, `ANL-004`–`ANL-007` — parcel-era records.

**Standards and references**

- RFC 9162 (CT v2), RFC 9942 (COSE receipts), RFC 9943 (SCITT), RFC 3161 (TSA), RFC 9052 (COSE).
- W3C Bitstring Status List; IETF Token Status List.
- DCC v3.3.0 (PTB); IAF Mandatory Document 28; ISO/IEC 17025; ISO/IEC 17011; ISO/IEC 17021.
- Macau Law No. 14/2022; the implementing regulation appears to be 11/2023.
- Tessera; `transparency-dev/witness`.

---

## 9. Git and document state

- Branch `tdd`; HEAD `eabe67c`; working tree clean at handover start.
- `CNP-004` v0.6.4, `DEC-003`, `DEC-003a` and `.review-session` are committed at HEAD.
- This handover and `DISCUSSION-SUMMARY-007` are untracked. The user commits.
- `.review-session` holds `ses_e92ef5aacffet6YYRucwgYN60t`. Resume that session for reviews; the old session is retired.
- `BUILD-HANDOVER-007` records HEAD `e9d45c2`; that state is stale.

---

## 10. Cautions

- **Do not build before G1 and G2 are recorded.**
- **Do not call the design a blockchain.** Call it a register log or a witnessed append-only log.
- **Do not claim completeness, truth of the work, signer authority or legal recognition.** State the limits in the same section as the claim.
- **Do not claim novelty for Certificate Transparency, Sigstore, COSE receipts, timestamps or witnesses.** The contribution is the object, the subject-index absence proofs, the profile and the measurement.
- **The 2-of-3 threshold requires at least 2 honest witnesses, not 1.** With one honest and two colluding witnesses, the two colluders form a valid quorum. Do not revert to "at least 1 honest".
- **The audit attestation is 2-of-3, not 1-of-3.** Each signer must verify the projection. A single malicious witness must not be able to sign a false index root.
- **The audit runs at each period-deadline checkpoint.** Without an attested deadline checkpoint, the deadline proof is `unverifiable`.
- **Fork prevention is not fork detection.** An honest majority prevents an un-cosigned fork; a cosigned fork is detected by comparing checkpoints, not by inspecting a single one.
- **D1 is an ablation, not existing practice.** Falsifier 2 compares D2 to B + C. Do not describe D1 as a baseline.
- **C's enumeration is a database assertion.** It can show what it holds; it cannot prove the list is complete.
- **The Macau register is not the calibration-certificate status quo.** Law No. 14/2022 is cited for the statutory-record pattern. Current calibration practice is not verified.
- **The 2-day limitation check has not run.** The premise that no verifiable register exists is **P**, not **V**.
- **Do not merge `CNP-003` and `CNP-004` without approval.**
- **Use the forward definition:** D1 = C plus log/index without witnesses; D2 = D1 plus the witness layer.
- **Deletion is two failures, not one.** Removing a leaf from a later checkpoint breaks consistency (`history_rewritten`); deleting it from storage while keeping the checkpoint fails against the root (`log_entry_mismatch`).

---

## 11. One-paragraph summary

The session began with `CNP-004` v0.4.2 and a single central-registry baseline. It ended with `CNP-004` v0.6.4, a five-arm comparison — A, B, C, D1, D2 — and a two-page supervisor decision brief plus a one-page comparison handout. It corrected an insufficient 2-of-3 honesty assumption, closed a 1-of-3 audit-attestation loophole, separated fork prevention from fork detection, fixed the backfill state machine, required an attestation at each deadline checkpoint, and removed the unverified claim that the Macau DSSCU register is the calibration-certificate status quo. D2 is the only arm with unconditional proofs on all five archive rows against an untrusted operator; D1 is the ablation that isolates the witness layer. Both the proposal and the supervisor package are committed at HEAD `eabe67c`. The next agent's job is not to build: it is to take `DEC-003` and `DEC-003a` to the supervisor, obtain G1 and G2 in writing, and run the 2-day limitation check before any design work begins.
