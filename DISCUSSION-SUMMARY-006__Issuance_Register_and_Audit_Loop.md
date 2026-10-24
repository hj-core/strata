# DISCUSSION-SUMMARY-006 — Issuance Register, Audit Loop and the Pivot from Scope

| Field | Value |
| --- | --- |
| Document ID | DISCUSSION-SUMMARY-006 |
| Title | Session record — the pivot to an issuance register, the audit loop and the state at handover |
| Date | 2026-10-24 |
| Session | Build agent, with a review subagent resumed across the session |
| Review session | `ses_e973cde3cffegZlAAP3wZwwknd`, recorded in `.review-session` |
| Status | Record. No build. G1 and G2 open. |
| Related | `CNP-004` v0.4.2 (primary); `CNP-003` v0.3.1 (companion); `BUILD-HANDOVER-007` |

---

## 1. What this session was

The session began from `BUILD-HANDOVER-006` with the scope register proposed and no approval recorded. It ended with a different primary proposal — an issuance register for safety-critical certificates — a companion scope proposal, 11 review passes, and both documents ready for the supervisor.

No implementation started. G1 (supervisor approval) and G2 (module constraints in writing) remain open. The 2-day limitation check has not run.

---

## 2. The arc

| Step | What happened | Result |
| --- | --- | --- |
| 1 | Read `BUILD-HANDOVER-006` and `CNP-003` v0.1.0 | The scope register was a proposal, not an approved direction |
| 2 | Internal critique of `CNP-003` | The claim proved inclusion, not coverage at a date; the premise was unevidenced; the matrix encoded trust as capability |
| 3 | `CNP-003` v0.2.0 | Entry chain, scope map, effective state, backdating rule, embedded-snapshot baseline B′ |
| 4 | Review of v0.2.0 | 10 blocking findings; all closed in v0.3.0 |
| 5 | `CNP-003` v0.3.0 | Issuance attestation, freshness rule, per-event inclusion proofs, 3 states and 6 events, 1 unique property |
| 6 | Review of v0.3.0 | 3 blocking findings on time binding; all closed |
| 7 | `CNP-003` v0.3.1 | Mandate reframed as a condition of operation; R1 demoted; claim carries 6 conditions |
| 8 | Pivot discussion | The user's question changed from "what can the laboratory certify?" to "what has it certified?" |
| 9 | `CNP-004` v0.1.0 | Issuance register: certificate hashes, receipts, witnesses |
| 10 | External critique 1 | Subject reference was unrealistic; Baseline B was strawmanned; Macau law was missed |
| 11 | `CNP-004` v0.3.0 | Installed sensor, Law No. 14/2022, PKI credited, unique rows reduced |
| 12 | External critique 2 | §2 contradicted §5; the table undercounted; the operator assumption was self-defeating |
| 13 | `CNP-004` v0.4.0 | Archive commitment framing, 5 archive properties, operator untrusted |
| 14 | External critique 3 | O(N) audit inside the O(log N) query; withholding motive; completeness row; R2 wording |
| 15 | `CNP-004` v0.4.1 | Auditor and investigator separated, `--query` and `--audit`, narrative closed |
| 16 | External critique 4 | Audit attestation unanchored; deleted entry vs rewrite; period undefined |
| 17 | `CNP-004` v0.4.2 | Audit attestation in the witness keys; storage deletion separated; period = UTC calendar year |
| 18 | Copy-edit pass | Changelog table, numbering, duplicate property, terminology, date |

---

## 3. Decisions taken

| Decision | Value | Where |
| --- | --- | --- |
| Primary proposal | `CNP-004` — a verifiable issuance register for safety-critical certificates | `CNP-004` §1 |
| Companion | `CNP-003` — the scope extension, recorded as future work | `CNP-004` §7.3 |
| Order of questions | Commitment → authority → accuracy | `CNP-004` §3.3 |
| Claim | Offline proof that a certificate was committed by the laboratory it names, and archive proof of what is absent | `CNP-004` §4.6 |
| Unique rows | 5 archive properties: commitment, absence, deletion detection, backfill detection, fork detection | `CNP-004` §5.1 |
| Operator | Untrusted for history; integrity rests on 2-of-3 witnesses | `CNP-004` §4.8, §6.4 |
| Object | Permanently installed brake-force sensor on a lift | `CNP-004` §6.1, §7.2 |
| Mandate | 3 routes: statute, procurement, liability | `CNP-004` §3.5 |
| Effort | 250 hours over 20 weeks | `CNP-004` §8 |
| Review session | `ses_e973cde3cffegZlAAP3wZwwknd`, reused for every pass | `.review-session` |

---

## 4. The discussion positions

The session tested several arguments. The conclusions that survived:

- **Scope versus issuance.** A scope register answers "was the laboratory authorised?"; an issuance register answers "did it commit this certificate?". The second is logically prior: until the investigator knows which laboratory issued the document, the scope question has no subject.
- **PKI is not the gap.** A signed DCC under X.509 with an RFC 3161 token binds the key and gives non-repudiation. The gap is **archive commitment**: a certificate can be signed, timestamped and never logged.
- **The field verifier is the gate.** If verifiers accept a signature alone, laboratories can sign without logging and the archive is empty after the accident. The refusal rule makes commitment necessary.
- **Absence is provable only against a checkpoint.** A non-inclusion proof must be taken against a checkpoint at or after the query range plus 1 cadence, and a deadline proof at the first checkpoint after the period deadline. Without both, an operator can serve an old root.
- **The index must be a projection of the log.** The checkpoint commits both roots; an auditor rebuilds the index and signs an attestation; the investigator verifies O(log N) proofs and never downloads the log.
- **The operator need not be trusted.** Assuming a disinterested operator undermines the case against a central registry. Witnesses are the mechanism, not the operator's neutrality.
- **A central registry is enough in some settings.** Where the keeper is trusted, disinterested and the only verifier, a database is cheaper. The proposal is aimed at the case where at least 1 condition fails.
- **The mandate is a proposal, not a finding.** The class of statutory record is established; Macau's lift regime already mandates one; the calibration layer beneath it does not exist.

---

## 5. The audit loop

The review subagent was resumed in the same session for every pass. Each pass returned blockers; all were closed before the next pass.

| Pass | Target | Found | Closed by |
| --- | --- | --- | --- |
| 1 | `CNP-003` v0.2.0 | 10 blocking: time binding, per-event proofs, claim duplication, matrix count, Macau, state model | v0.3.0 |
| 2 | `CNP-003` v0.3.0 | 3 blocking: receipt creation time, first-inclusion claims, interval wording | v0.3.0 final |
| 3 | `CNP-003` v0.3.0 | None; ready | — |
| 4 | `CNP-003` v0.3.1 | 1 blocking: falsifier 4 contradicted R1 severity | v0.3.1 |
| 5 | `CNP-003` v0.3.1 | Closed | — |
| 6 | `CNP-004` v0.2.0 | 5 blocking: stale checkpoint, index vs log, subject binding, count, falsifier | v0.3.0 |
| 7 | `CNP-004` v0.3.0 | 2 blocking: §5 prose, year-granularity gap proof | v0.3.0 final |
| 8 | `CNP-004` v0.3.0 | None; ready | — |
| 9 | `CNP-004` v0.4.0 | 2 blockers: operator independence leftovers | v0.4.0 final |
| 10 | `CNP-004` v0.4.1 | 3 gaps: audit signature, deleted entry, period | v0.4.2 |
| 11 | `CNP-004` v0.4.2 | Closed | — |

The audit session is the default for future reviews. Resume it by session ID rather than starting a new one.

---

## 6. What was verified

| Fact | Status |
| --- | --- |
| Macau Law No. 14/2022: lift registration with DSSCU, annual inspection, notification of every declaration (Art. 6(6)), 1-year validity | **V**, Official Gazette and DSSCU |
| Implementing regulation | Appears to be 11/2023; article 37 unverified; cite the law, not the article number |
| 121 ILAC MRA signatories; over 114,600 accredited laboratories | **V**, ILAC 2025 |
| IAF CertSearch covers ISO/IEC 17021-1 only, about 2.5 million certifications | **V**, IAF |
| Grenfell Tower Inquiry Phase 2 report dated 2024-09-04 | **V** |
| No Macau instrument requires calibration certificates to be logged | **P** |
| No adjudicated case of a laboratory denying a certificate | **A** |

---

## 7. Open questions at handover

| # | Question | Why it matters |
| --- | --- | --- |
| G1 | Does the supervisor approve `CNP-004`? | No build before approval |
| G2 | Are the module constraints confirmed in writing? | Shapes the claim and the evaluation |
| G3 | Does any authority or industry body already publish a verifiable register with a subject index? | If yes, the limitation is removed |
| G4 | Does IAF CertSearch extend to ISO/IEC 17025 with history? | Incumbent risk |
| G5 | Which mandate route is plausible in Macau? | The condition in §4.7 |
| G6 | Public leaves or regulator-only leaves with public checkpoints? | Privacy and monitorability |
| G7 | Audit cadence and who holds the auditor role? | The between-audit guarantee |
| G8 | Combine `CNP-003` and `CNP-004` into one log with two leaf types? | Scope and effort |

---

## 8. What was not done

- No supervisor meeting; G1 and G2 are not recorded.
- No limitation check; G3 and G4 have not run.
- No implementation, no code, no live log.
- No Macau legal opinion; the law is cited, not interpreted.
- No decision on the visibility model or the audit cadence; both are C1 items.

---

## 9. One-paragraph summary

The session began with a scope register and ended with an issuance register. The scope proposal was tightened through 5 audit passes to 1 unique property and a condition of operation. The user then changed the question to "what has been certified?", which produced `CNP-004`: a witnessed log of certificate hashes, a subject index with non-inclusion proofs, an issuance receipt, and an investigator query. Four external critiques and 6 audit passes moved it from a strawmanned comparison to 5 archive properties that PKI and a central registry cannot provide: commitment, absence, deletion detection, backfill detection and fork detection. The operator is untrusted, the witnesses are the mechanism, and the field verifier is the gate that makes commitment necessary. Both proposals are committed; neither is approved. The next agent takes `CNP-004` v0.4.2 to the supervisor, obtains G1 and G2 in writing, and runs the 2-day limitation check before any design work begins.
