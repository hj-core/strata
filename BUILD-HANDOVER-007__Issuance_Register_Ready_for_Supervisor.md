# BUILD-HANDOVER-007 — Issuance Register, Ready for the Supervisor

| Field | Value |
| --- | --- |
| Prepared | 2026-10-24 |
| For | The next BUILD agent continuing after the issuance-register proposal |
| Repo | `/home/hj-core/project` |
| Branch | `tdd` |
| HEAD | `e9d45c2` |
| Working tree | **Clean.** `CNP-004` v0.4.2 and `CNP-003` v0.3.1 are committed. This handover and `DISCUSSION-SUMMARY-006` are untracked. **Do not stage or commit.** |
| Read first | `CNP-004`, then this file, then `CNP-003` |
| Status | **Direction proposed. Supervisor approval not recorded. No build started.** |
| Supersedes | `BUILD-HANDOVER-006` for the current state. -006 remains the record of the employment and scope work. |

---

## 1. Status at a glance

| Item | State |
| --- | --- |
| Module | **COMP4299** — confirmed |
| Deliverable | Dissertation + demonstration + evaluation — confirmed |
| Current direction | **A Verifiable Issuance Register for Safety-Critical Certificates** (`CNP-004` v0.4.2) |
| Object | Certificate issuance: hashes, a subject index, receipts, witnesses |
| Claim | Offline proof of commitment at issuance, and archive proof of what is absent |
| Companion | `CNP-003` v0.3.1 — the accreditation-scope extension, future work |
| Unique properties | 5 archive properties: commitment, absence, deletion detection, backfill detection, fork detection |
| Effort | 250 hours over 20 weeks |
| Build | **Not started** |
| Decision | **G1 and G2 open** |
| Limitation check | **Not run** |

---

## 2. The job

1. **Do not build.** `CNP-004` is a proposal, not an approved direction. G1 has not been recorded.
2. **Read `CNP-004` first.** It is self-contained: problem, evidence, claim, comparison, scope, deliverables, schedule, risks.
3. **Take `CNP-004` to the supervisor.** Ask for a written decision on G1 (direction) and G2 (module constraints). Use §1 (title), §4.6 (claim), §6 (scope) and §8 (schedule).
4. **If approved, run the 2-day limitation check (G3/G4).** Confirm that no authority or industry body already publishes a verifiable register of safety-critical certificates with a subject index, and that IAF CertSearch still covers ISO/IEC 17021 only. Record the result in `CNP-004` §2.4 or a short note.
5. **If the limitation survives, start P1 (Design).** C1: leaf, checkpoint, allow-list, index, receipt and audit-attestation formats; the UTC period mapping; the visibility model; the audit cadence.
6. **Do not run another domain search.** Six domains have been tested; the filter in `BUILD-HANDOVER-005` §6 still applies.

---

## 3. What happened since -006

`BUILD-HANDOVER-006` left a scope register at v0.1.0 with no approval. This session tightened it to v0.3.1, then pivoted.

| Step | Result |
| --- | --- |
| Read -006 and `CNP-003` v0.1.0 | The claim proved inclusion, not coverage at a date |
| `CNP-003` v0.2.0 | Entry chain, scope map, effective state, backdating rule, embedded-snapshot baseline B′ |
| 5 review passes | 10 blockers, then 3, then 1; all closed. v0.3.1 has 1 unique property and a condition of operation |
| Pivot to issuance | The user changed the question: not "what can the laboratory certify?" but "what has it certified?" |
| `CNP-004` v0.1.0 | A log of certificate hashes, receipts, witnesses |
| External critique 1 | The torque wrench could not name the lift; Baseline B was strawmanned; Macau law was missed |
| `CNP-004` v0.3.0 | Installed brake-force sensor; Law No. 14/2022; PKI credited |
| External critique 2 | §2 contradicted §5; the table undercounted; the operator assumption was self-defeating |
| `CNP-004` v0.4.0 | Archive commitment framing; 5 archive properties; operator untrusted |
| External critique 3 | O(N) audit inside the O(log N) query; withholding motive; completeness row |
| `CNP-004` v0.4.1 | Auditor and investigator separated; `--query` and `--audit` |
| External critique 4 | Audit attestation unanchored; deleted entry vs rewrite; period undefined |
| `CNP-004` v0.4.2 | Audit attestation in the witness keys; storage deletion separated; period = UTC calendar year |
| Copy-edit pass | Changelog table, numbering, duplicate property, terminology |

**The one-sentence version.** PKI proves who signed; the log proves the certificate was committed, cannot be withdrawn, cannot be backfilled, and that a required entry is missing.

---

## 4. The surviving direction

### 4.1 Problem

A safety-critical certificate can be signed, timestamped and never committed. A verifier can check the signature; an investigator cannot check the archive. After an accident, the record is a pile of PDFs: nobody holds the whole set, absence is indistinguishable from a lost file, and the laboratory may have dissolved.

The failure is not authenticity. It is **archive commitment**, and the consequence is that the record looks complete when it is not.

### 4.2 Evidence

| Fact | Status |
| --- | --- |
| Macau Law No. 14/2022 requires lift registration with DSSCU, annual inspection and notification of every inspection declaration (Art. 6(6)); the declaration is valid for 1 year | **V** |
| No Macau instrument requires the ISO/IEC 17025 calibration certificates beneath those inspections to be registered | **P** |
| IAF CertSearch covers ISO/IEC 17021-1 only, about 2.5 million certifications | **V** |
| 121 ILAC MRA signatories; over 114,600 accredited laboratories | **V** |
| Aviation, medical devices, legal metrology and vehicle inspection mandate long-lived records | **V** |
| Grenfell Tower Inquiry Phase 2 report (2024-09-04) recommended attaching the full testing history to certificates | **V** |
| No adjudicated case of a laboratory denying a certificate it signed | **A** |

### 4.3 Solution

The laboratory signs the certificate and submits its hash at issuance. The log appends a leaf: certificate hash, laboratory, key, certificate date, subject reference and certificate type. The checkpoint commits the log root and a subject index `(subject_ref, period)`, where `period` is the UTC calendar year of the certificate date.

The receipt carries the leaf, the inclusion proof, the cosigned checkpoint and the TSA token. The field verifier checks it offline. The investigator queries the index for a period range and receives inclusion proofs, non-inclusion proofs and a deadline proof at the first checkpoint after the period deadline.

The auditor rebuilds the index in O(N) and signs an audit attestation over the checkpoint and both roots. The investigator verifies O(log N) proofs against the cosigned index root and checks the attestation. One of the 3 witnesses holds the auditor role.

### 4.4 Claim and limits

> A verifier can establish, offline and years later, that a certificate was committed to the archive by the laboratory it names, at the time it claims, within 1 checkpoint cadence; and an investigator can establish, from the archive, the full set of certificates the register holds for an asset and the absence of any required entry — without the laboratory's cooperation and without trusting the operator's account of its own history.

The claim does not establish that the work happened, that the archive is complete, that the individual signer was authorised, or that the register has legal recognition. It also does not establish the asset attribution beyond the laboratory's declared subject reference. These limits are in §4.5.

### 4.5 Evaluation

4 arms: **A**, current practice; **B**, a signed DCC under X.509 with an RFC 3161 token; **C**, a central registry with an index; **D**, the proposed register.

10 attacks: forged certificate, tampered certificate, backdated certificate, repudiation, deleted entry, rewritten history, fork, missing witness, filled gap, backfill after incident. Metrics: detection, latency, proof size, trust-anchor count.

The 5 unique rows are archive properties: offline commitment, provable absence, deletion or alteration detection, post-deadline backfill detection, and fork detection. Baseline B+C answers every other row that any option answers.

### 4.6 Scope and schedule

- **Certificate class:** ISO/IEC 17025 calibration certificates for permanently installed lift brake-force sensors. Overload and pressure sensors are boundary cases.
- **Artefact:** issuance receipt in a DCC-compatible envelope (subset, not full v3.3.0 validation).
- **Register:** Merkle log, subject index, allow-list, checkpoints, 3 witnesses at 2-of-3, audit attestation.
- **Schedule:** 20 weeks, ~250 hours, 5 phases P0–P5, gates G-A (design) and G-B (evaluation).
- **Demonstration:** brake-force sensor BF-4471 on lift L-4471, 2020 and 2022 logged, 2024 missing, laboratory dissolved, insurer change in 2025, incident in 2026, deadline proof at CP-2500.

---

## 5. The companion

`CNP-003` v0.3.1 answers the next question: was the laboratory **authorised** on the certificate date? It adds a scope event leaf type, a per-entry hash chain, a scope map and a backdating rule. It has 1 unique property against its strongest baseline: non-equivocation.

The 2 proposals share the log, the checkpoints, the witnesses and the verifier. A combined design carries both leaf types under one receipt. That combination is not approved and is not in scope for `CNP-004`.

---

## 6. Decision state

| Item | State |
| --- | --- |
| Primary direction | **Proposed** — issuance register |
| Title | **Proposed** — A Verifiable Issuance Register for Safety-Critical Certificates |
| Claim | **Proposed** — commitment at issuance and archive absence |
| Scope | **Proposed** — installed brake-force sensors, Macau setting |
| Deliverables | **Proposed** — C1–C10 |
| Schedule | **Proposed** — 20 weeks, 250 hours |
| Supervisor approval | **Not recorded** |
| Module constraints in writing | **Pending** |
| Limitation check | **Not run** |
| Build | **Not started** |

---

## 7. Open questions and gates

| # | Question | Why it matters |
| --- | --- | --- |
| **G1** | Does the supervisor approve the issuance-register direction? | No build before approval |
| **G2** | Are the module constraints confirmed in writing? | Shapes the claim and the evaluation |
| **G3** | Does any authority or industry body publish a verifiable register of safety-critical certificates with a subject index? | If yes, the limitation is removed and the project stops |
| **G4** | Does IAF CertSearch extend to ISO/IEC 17025 with history? | Incumbent risk R1 |
| **G5** | Which mandate route is plausible in Macau: statute, procurement or liability? | The condition in §4.7 |
| **G6** | Public leaves, or regulator-only leaves with public checkpoints? | Privacy and monitorability; C1 decision |
| **G7** | Audit cadence, and who holds the auditor role? | The between-audit guarantee |
| **G8** | Does the dissertation combine `CNP-003` and `CNP-004` into one log? | Scope and effort |

---

## 8. Next 5 actions

1. **Read `CNP-004`.** Do not rely on this handover for the argument.
2. **Book the supervisor review.** One-page brief from §1, §4.6, §6 and §8; ask for a written decision on G1 and G2.
3. **If approved, run the 2-day limitation check (G3/G4).** Check national registers, IAF CertSearch scope, and any industry certificate register. Record the result.
4. **If the limitation survives, start P1.** C1: wire formats, period mapping, visibility model, audit cadence; draft the problem and evidence chapter.
5. **If the limitation is removed, stop.** Report to the supervisor. Do not re-scope without approval.

---

## 9. Document map and read order

**Current**

1. `CNP-004` v0.4.2 — the proposal. Read first.
2. `BUILD-HANDOVER-007` — this file.
3. `DISCUSSION-SUMMARY-006` — the session record.
4. `CNP-003` v0.3.1 — the companion scope proposal.

**Historical (do not edit)**

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

## 10. Git and document state

- Branch `tdd`; HEAD `e9d45c2`; working tree clean.
- `CNP-004` v0.4.2 and `CNP-003` v0.3.1 are committed.
- This handover and `DISCUSSION-SUMMARY-006` are untracked. The user commits.
- `.review-session` is committed and holds `ses_e973cde3cffegZlAAP3wZwwknd`. Resume that session for reviews.
- `BUILD-HANDOVER-006` records HEAD `a9a5d40`; that state is stale.

---

## 11. Cautions

- Do not build before G1 and G2 are recorded.
- Do not call the design a blockchain. Call it a register log or a witnessed append-only log.
- Do not claim completeness, truth of the work, signer authority or legal recognition. State the limits in the same section as the claim.
- Do not claim novelty for Certificate Transparency, Sigstore, COSE receipts or timestamps. The contribution is the object, the subject index with absence proofs, the profile and the measurement.
- The field verifier is the enforcement gate. Without the refusal rule, the archive is a voluntary collection.
- The operator need not be trusted for history. Do not reintroduce an "independent operator" assumption; integrity rests on the 2-of-3 witnesses.
- The subject reference is the installation of a fixed sensor. Do not put a lift identifier inside a portable instrument's calibration certificate.
- Cite Macau Law No. 14/2022. The implementing regulation appears to be 11/2023, not 39/2023. Article 37 is unverified; cite the law and Art. 6(6).
- Do not present the Macau scenarios as documented cases. They are labelled A.
- The 2-day limitation check has not run. The premise that no verifiable register exists is **P**, not **V**.
- Do not merge `CNP-003` and `CNP-004` without approval; the combined design is not in scope.

---

## 12. One-paragraph summary

The session began with a scope register and ended with an issuance register. `CNP-003` was tightened to 1 unique property and a condition of operation; `CNP-004` was built through 4 external critiques and 6 audit passes into a proposal with 5 archive properties that PKI and a central registry cannot provide. The object is a permanently installed brake-force sensor on a lift; the setting is Macau under Law No. 14/2022; the mechanism is a witnessed log with a subject index, non-inclusion proofs and an audit attestation; the operator is untrusted and the witnesses are the mechanism. The claim is offline proof of commitment and archive proof of absence. Both proposals are committed and neither is approved. The next agent's job is not to build. It is to take `CNP-004` to the supervisor, obtain G1 and G2 in writing, and run the 2-day limitation check before any design work begins.
