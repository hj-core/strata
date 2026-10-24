# DISCUSSION-SUMMARY-007 — Supervisor Package, Comparison Ladder and Audit Loop

| Field | Value |
| --- | --- |
| Document ID | DISCUSSION-SUMMARY-007 |
| Title | Session record — the comparison ladder, the supervisor package and the audit loop |
| Date | 2026-10-24 |
| Session | BUILD agent, with a new review subagent session |
| Review session | `ses_e92ef5aacffet6YYRucwgYN60t`, recorded in `.review-session` |
| Status | Record. No build. G1 and G2 open. |
| Related | `CNP-004` v0.6.4 (primary); `DEC-003` (decision brief); `DEC-003a` (comparison handout); `CNP-003` v0.3.1 (companion); `BUILD-HANDOVER-008` |

---

## 1. What this session was

The session began from `BUILD-HANDOVER-007` with `CNP-004` v0.4.2 presented as a proposal for supervisor review. It ended with `CNP-004` v0.6.4, a two-page decision brief (`DEC-003`) and a one-page comparison handout (`DEC-003a`), all committed and ready for the supervisor. No implementation started. G1 (supervisor approval) and G2 (module constraints in writing) remain open, and the 2-day limitation check has not run.

The previous review session (`ses_e973cde3cffegZlAAP3wZwwknd`) was not resumable from this BUILD session. A new standing review session was created: `ses_e92ef5aacffet6YYRucwgYN60t`, recorded in `.review-session`.

---

## 2. The arc

| Step | What happened | Result |
| --- | --- | --- |
| 1 | Read `BUILD-HANDOVER-007` and `CNP-004` v0.4.2 | State confirmed; G1/G2 open; no build |
| 2 | Established a new review session | `ses_e92ef5aacffet6YYRucwgYN60t` |
| 3 | `CNP-004` v0.4.3 | Required-period schedule as a query input; maximum checkpoint interval; claim conditions 7 → 9 |
| 4 | `CNP-004` v0.4.4 | Subtitle tightened to commitment at issuance and checkpoint proof of absence |
| 5 | Title discussion | "Who issued" is PKI's job; the contribution is commitment and absence |
| 6 | §5.1 self-audit | 10 blocking findings: C undefined, issuance overclaim, time conflation, commitment/time inconsistency, absence/proof contradiction, backfill window, 5-row overclaim, legal recognition, property-set drift, trust-table omissions |
| 7 | `CNP-004` v0.5.0 | Replaced the single C baseline with C-R (receipt) and C-L (unwitnessed log); removed the unverified Macau status-quo claim |
| 8 | Standing review of v0.5.0 | 4 blocking: C-R absence contradiction, † asymmetry, §7.2 rewrite attribution, backfill overclaim; all closed |
| 9 | External review 1 | 2-of-3 quorum error; fork prevention vs detection; C-L misclassified as existing practice; C-L over-credited; `log_entry_mismatch` missing from C6 → v0.6.0 |
| 10 | `CNP-004` v0.6.1 | Arms renamed globally: C-R → C, C-L → D1, D → D2 |
| 11 | External review 2 | 1-of-3 audit attestation loophole; backfill condition inverted; C enumeration over-credited; C6 outcomes incomplete → v0.6.2 |
| 12 | `CNP-004` v0.6.3 | D1 defined forward as C plus log/index without witnesses; D2 = D1 + witness layer |
| 13 | External review 3 | D1 enumeration still over-credited; deadline checkpoint unattested; deletion wording imprecise; cross-document version/gate drift → v0.6.4 |
| 14 | `DEC-003` | Two-page G1/G2 supervisor decision brief |
| 15 | `DEC-003a` | One-page §5.1 comparison handout |

---

## 3. Decisions taken

| Decision | Value | Where |
| --- | --- | --- |
| Primary proposal | `CNP-004` — a verifiable issuance register for safety-critical certificates | `CNP-004` §1 |
| Comparison arms | A, B, C, D1, D2 | `CNP-004` §5.1 |
| C | Central registry with a signed, TSA-timestamped receipt; no Merkle log | `CNP-004` §5.1 |
| D1 | C plus Merkle log, checkpoints and subject index, without witnesses; the ablation | `CNP-004` §5.1 |
| D2 | D1 plus 2-of-3 witnesses; the proposal | `CNP-004` §5.1 |
| Quorum | 3 witnesses, 2-of-3 threshold, **at least 2 honest**, each checking a consistency proof before cosigning | `CNP-004` §4.2, §4.4, §4.6 |
| Audit attestation | 2-of-3 witness signatures, each signer verifying the projection; audit at each period-deadline checkpoint and at an incident | `CNP-004` §4.2, C6 |
| Claim | Offline proof of commitment within 1 cadence; archive proof of absence relative to a supplied schedule; without the laboratory's cooperation and without trusting the operator's history | `CNP-004` §4.6 |
| Unique contribution | D2 is the only arm with unconditional proofs on all five archive rows against an untrusted operator; D1 isolates the witness layer | `CNP-004` §5.1 |
| Macau | The DSSCU register covers declarations, not calibration certificates; current calibration practice is not verified and is not claimed | `CNP-004` §5.2 |
| Supervisor package | `DEC-003` (decision brief) + `DEC-003a` (comparison handout) | `DEC-003`, `DEC-003a` |
| Review session | `ses_e92ef5aacffet6YYRucwgYN60t` | `.review-session` |

---

## 4. The discussion positions

- **PKI is not the gap.** A signed DCC under X.509 with an RFC 3161 token binds the key, fixes the signing time and gives non-repudiation. It cannot prove that the certificate was committed to an archive, nor anything about the population.
- **Absence forces a commitment to a set.** No signature answers "is anything missing?". A negative claim about a population requires a commitment to the set, a historical checkpoint and a proof of non-membership. That is the register.
- **Trusting the operator makes witnesses unnecessary — and changes the project.** If the operator is trusted, C is enough. The proposal is for the case where the operator may have an interest in the incident; that is why D2 carries an honest witness majority.
- **C's enumeration is a database assertion.** It can show what it holds, but it cannot prove the list is complete. D1 and D2 commit to a bucket; D2 makes that commitment non-equivocal.
- **D1's proofs are conditional.** They are Merkle proofs against an operator-signed checkpoint. A post-incident investigator who did not cache the checkpoint must trust the operator or run the O(N) audit. D2's witnesses make the proofs unconditional.
- **2-of-3 needs 2 honest, not 1.** With one honest and two colluding witnesses, the two colluders form a valid quorum. The honest majority is the correct assumption; a 3-of-3 threshold would tolerate one honest witness but lose liveness.
- **Fork prevention is not fork detection.** An honest majority prevents an un-cosigned fork by checking consistency before cosigning. A cosigned fork is detected by comparing checkpoints; a single checkpoint cannot reveal it.
- **The 1-of-3 audit attestation was a hole.** A malicious witness could sign a false index projection. The attestation is now 2-of-3, with each signer verifying the projection.
- **Backfill is a state comparison.** Absent at the deadline plus present at query is `backdated`; present at the deadline plus absent at query is a consistency break or a storage mismatch, not simple absence.
- **Macau is a setting, not a verified status quo.** Law No. 14/2022 is cited for the statutory-record pattern. The dissertation does not claim that the DSSCU register is the baseline for calibration certificates or that current calibration practice has been verified.

---

## 5. The audit loop

| Pass | Target | Found | Closed by |
| --- | --- | --- | --- |
| 1 | `CNP-004` v0.4.2, orientation | Standing role established; 3 supervisor risks | — |
| 2 | `CNP-004` v0.4.3 delta | 0 blocking; 6 non-blocking | v0.4.3 final |
| 3 | §5.1 self-audit | 10 blocking: C undefined, issuance overclaim, time conflation, commitment/time, absence/proof, backfill window, 5-row claim, legal recognition, §4.4 drift, trust-table omissions | v0.5.0 |
| 4 | `CNP-004` v0.5.0 | 4 blocking: C-R absence, † asymmetry, §7.2 rewrite, backfill overclaim | v0.5.0 final |
| 5 | External review 1 | Quorum math, fork prevention/detection, C-L framing, C-L over-credit, C6 outcome | v0.6.0 |
| 6 | `CNP-004` v0.6.0 | 0 blocking; 3 non-blocking | v0.6.0 final |
| 7 | `CNP-004` v0.6.1 rename | 0 blocking; 0 new | v0.6.1 final |
| 8 | External review 2 | 1-of-3 attestation, backfill logic, C enumeration, C6 outcomes | v0.6.2 |
| 9 | `CNP-004` v0.6.2 | 0 blocking; 1 non-blocking | v0.6.2 final |
| 10 | External review 3 | D1 enumeration, deadline attestation, deletion mechanics, cross-document sync | v0.6.4 |
| 11 | `CNP-004` v0.6.4 | 0 blocking; 2 non-blocking | v0.6.4 final |
| 12 | `DEC-003` | 1 blocking: claim conditions and "unconditionally" | `DEC-003` final |
| 13 | `DEC-003a` | 0 blocking; 2 non-blocking | `DEC-003a` final |

The review session is the default for future reviews. Resume it by session ID.

---

## 6. What was verified

| Fact | Status |
| --- | --- |
| Macau Law No. 14/2022: lift registration, annual inspection, notification of every declaration (Art. 6(6)); declaration valid for 1 year | **V**, Official Gazette and DSSCU |
| No Macau instrument requires the calibration certificates beneath those inspections to be registered | **P** |
| IAF CertSearch covers ISO/IEC 17021-1 only | **V**, IAF |
| 121 ILAC MRA signatories; over 114,600 accredited laboratories | **V**, ILAC 2025 |
| Grenfell Tower Inquiry Phase 2 report dated 2024-09-04 | **V** |
| No verifiable register of safety-critical certificates with a subject index | **P** — the limitation check has not run |
| No adjudicated case of a laboratory denying a certificate it signed | **A** |
| Macau calibration-certificate practice | **Not verified**; scenarios are illustrative and labelled A |

---

## 7. Open questions at handover

| # | Question | Why it matters |
| --- | --- | --- |
| **G1** | Does the supervisor approve the issuance-register direction? | No build before approval |
| **G2** | Are the module constraints confirmed in writing? | Shapes the claim and the evaluation |
| **G3** | Does any authority or industry body publish a verifiable register of safety-critical certificates with a subject index? | If yes, the limitation is removed and the project stops |
| **G4** | Does IAF CertSearch extend to ISO/IEC 17025 with history? | Incumbent risk R1 |
| **G5** | Which mandate route is plausible in Macau: statute, procurement or liability? | The condition in §4.7 |
| **G6** | Public leaves, or regulator-only leaves with public checkpoints? | Privacy and monitorability; C1 |
| **G7** | What is the maximum checkpoint interval, and who runs the 2-of-3 audit at each deadline? | The between-checkpoint and between-audit guarantee |
| **G8** | Does the dissertation combine `CNP-003` and `CNP-004` into one log? | Scope and effort |
| **G9** | Can the O(N) audit run at every period-deadline checkpoint within the pilot budget? | The deadline proofs become `unverifiable` without an attested deadline checkpoint |

---

## 8. What was not done

- No supervisor meeting; G1 and G2 are not recorded.
- No limitation check; G3 and G4 have not run.
- No implementation, no code, no live log.
- No Macau legal opinion; the law is cited, not interpreted.
- No decision on the visibility model, the exact checkpoint interval, the DCC envelope or the auditor role; all are C1 items.
- No review of `CNP-003` in this session.

---

## 9. One-paragraph summary

The session turned `CNP-004` from a proposal with a single central-registry baseline into a five-arm comparison — A, B, C, D1, D2 — with D1 as the unwitnessed ablation and D2 as the witnessed proposal. It corrected two cryptographic errors (an insufficient 2-of-3 honesty assumption and a 1-of-3 audit attestation), separated fork prevention from fork detection, fixed the backfill state machine, and made the deadline checkpoint carry its own attestation. It removed the unverified claim that the Macau DSSCU register is the calibration-certificate status quo. It produced a two-page supervisor decision brief (`DEC-003`) and a one-page comparison handout (`DEC-003a`). Both are committed with `CNP-004` v0.6.4 at HEAD `eabe67c`. The next agent's job is not to build: it is to take `DEC-003` and `DEC-003a` to the supervisor, obtain G1 and G2 in writing, and run the 2-day limitation check before any design work begins.
