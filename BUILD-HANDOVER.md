# BUILD Handover — Proof-of-Delivery E2E Slice

| Field | Value |
| --- | --- |
| Prepared | 2026-10-21 |
| For | a new BUILD agent continuing the PoD proposal |
| Repo | `/home/hj-core/project` |
| Branch | `tdd` |
| HEAD | `cdb5a93` (“Save”) |
| Working tree | five **untracked** files (PRP-001, ANL-002, TUT-001, REF-001, REVIEW-SESSION-POD); old docs tracked |
| Read first | `PRP-001__Proof_of_Delivery_Proposal.md`, then `ANL-002__PoD_Existing_Solutions.md`, `TUT-001__E2E_Slice_Learning_Guide.md`, `REF-001__PoD_Glossary.md` |
| Status | Proposal drafted, audited, and fixed; **no build started** |
| First discussion | **Do we need to design the statement on our own?** (see §8) |

---

## 1. Status at a glance

| Item | State |
| --- | --- |
| Track | **COMP4299 system development** (module fit to confirm, G3). Novelty is not required; depth comes from the domain and the evaluation. |
| Topic | A thin end-to-end **proof-of-delivery (PoD)** slice using a witnessed Merkle transparency log: two issuers, RFC 9943/9942 receipts, RFC 9162 log, RFC 3161 time. |
| Previous track | The server-side geometric cut (DEC-001 / ANL-001) is **closed**. It is retained as related work, not as the source of truth. |
| Supervisor conversation | The topic changed after a supervisor meeting; the “blockchain” framing is being clarified (G1). |
| Candidate / anchor | **Open.** No stakeholder is named yet; G2 is a hard gate. |
| PRP-001 | v0.1.0, “Draft — for supervisor discussion and go/no-go,” 2026-10-21. |
| ANL-002 | v0.1.0, existing libraries and products; ~70–80% commodity **assumed**, not measured. |
| TUT-001 | v0.1.0, learning guide with 8 modules, exercises, 16-week plan. |
| REF-001 | v0.1.0, canonical glossary. |
| Review | Session `ses_ea126d834ffeqbCczjZ77a9gAk`; initial 3 critical / 8 high / 13 medium / 11 low; all critical and high **closed** on re-audit. |
| Plan | 290 h at **18 h/week over 16 weeks**; writing is a parallel track from week 2. |
| Main build | **Not started.** Do not start before G1 and G2. |
| Git | HEAD `cdb5a93`; do not stage or commit unless the user asks. |

---

## 2. Git and document state

- Branch: `tdd`. HEAD: `cdb5a93` (“Save”). Earlier: `74cdfa4`, `16469d0`, `5b1244f`.
- The five new files are **untracked**. The old documents are tracked and unchanged.
- Do **not** stage or commit. The user commits the documents.
- Old documents:
  - `docs/DEC-001__Decision_Options_Brief.md` (v0.4.0) — closed engine track, related work.
  - `docs/ANL-001__Existing_Solutions_Contrast.md` (v0.2.0) — closed engine track, related work.
  - `docs/PRD-001`, `docs/TDD-001`, `docs/PLN-001` — **legacy references**, not the source of truth.
  - `DISCUSSION-SUMMARY.md` — historical record; the PoD work is not yet appended to it.
- Do not bump versions unless asked.

---

## 3. What the project is

A business-to-business delivery has a **consignor** (sender), a **carrier** (the courier is its agent), and a **consignee** (recipient). The delivery record is the evidence of handover. A platform operator can rewrite, truncate, withhold, or equivocate. Signatures alone do not detect these; a hash chain detects rewriting only relative to a known head; independent witnesses detect equivocation; an issuer-signed sequence makes withholding detectable.

The project builds a **thin vertical slice**:

- **Two issuers:** the courier issues a delivery statement; the recipient issues a receipt statement about the same shipment (`sub = shipment:SHP-42`).
- **Transparency service:** Tessera-backed RFC 9162 Merkle log; RFC 9942 receipts; signed checkpoints.
- **Two witnesses, 2-of-2:** cosign checkpoints and gossip.
- **Trusted time:** RFC 3161 tokens from an external TSA.
- **Identity:** a development CA or self-signed certificates.
- **Verifier:** checks signatures, inclusion proofs, receipts, checkpoints, TSA tokens, and policy.
- **Dispute bundle:** portable evidence, verifiable offline.
- **Offline queue, revocation, conformance suite, red-team.**

**Contribution:** the PoD profile, offline trusted time, offline revocation, witness governance, privacy/linkage, and the evaluation — **not** a new log. ANL-002 estimates 70–80% of the code is commodity (assumed); the RFC 9162 receipt wrapper is custom because `scitt-verifier` is CCF-only.

**What it does not prove:** that the goods arrived, that the signer inspected them, or that the device was uncompromised. It proves assertions and inclusion, not physical truth.

---

## 4. Decisions already made

1. **Two issuers** — courier (delivery) and recipient (receipt). The consignor is a relying party, not an issuer; adding it later is a third statement linked by subject.
2. **Level 3, not a blockchain** — a witnessed Merkle transparency log. The supervisor’s “blockchain” means a hash chain or a transparency log; confirm at G1. Level 4/5 are comparisons, not the build.
3. **RFC 9162 SHA-256 via Tessera** as the VDS; **RFC 9942** inclusion and consistency receipts.
4. **Two witnesses, 2-of-2**, independent processes and keys; independence is simulated on one host if necessary.
5. **Offline time:** timestamp on reconnect plus recipient co-signature time. The token proves existence at token time, not signing time; the compromise window is indeterminate.
6. **Revocation:** signed status list; check at the TSA time; state the indeterminate window.
7. **18 h/week over 16 weeks**, 290 h plan; writing starts at week 2; no slack, the cut ladder is the buffer.
8. **Defaults** live in PRP-001 Appendix F; they are provisional, not RFC requirements.
9. **Do not stage or commit** for the user.
10. **Review session** is tracked in `REVIEW-SESSION-POD.md` and resumes by default.

Do not re-open these unless the user does.

---

## 5. Current scope and numbers

- **290 h** total: 235 h build/evaluation/docs, 55 h dissertation. The WBS sums exactly; 18 × 16 = 288, with the 2 h difference disclosed.
- **16 weeks:** M0 week 1, M1 2–3, M2 4–5, M3 6–8, M4 9–10, M5 11–12, M6 13–14, M7 15–16.
- **Parties:** 3 workflow (consignor, carrier, consignee); 7 trust-path (carrier, consignee, log, witness A, witness B, TSA, CA); 9 distinct organisations if every role is separate, including an insurer/auditor; 5 project-operated; 5 verifier trust anchors (log, 2 witnesses, TSA, CA).
- **Two issuers, two statements, two leaves.** The receipt statement’s `prev` is the SHA-256 of the courier’s signed `COSE_Sign1` bytes.
- **Targets (not measurements):** signing < 300 ms; verification < 1 s; proof < 2 KB at 1M; cost < $0.01/delivery; usability < 60 s. The evaluation uses 1k/10k/100k entries; the 1M figure is derived.
- **Evaluation:** 10 red-team attacks; conformance matrix; latency, proof size, cost; usability n = 8; levels 1/2/3 compared, level 5 as a cost model.
- **Status of numbers:** all planning figures are estimates; nothing is measured yet.

---

## 6. Document map

Read in this order:

1. **PRP-001** — the proposal.
   - §2 post-mortem and countermeasures; §2.2 weaknesses; §2.3 the three viva questions.
   - §3 problem, trust model, scope boundary, ethics; §4 requirements and anchors.
   - §5 levels 1–5 (analytical device, not a taxonomy); §5.4 claim under test.
   - §6 architecture; **§6.1a actors/parties**; §6.2 protocol; **§6.2a SHP-42 worked scenario**; §6.3 data model; §6.5 trust anchors; §6.6 offline time; §6.7 privacy.
   - §7 commodity vs contribution; §8 evaluation, hypotheses, red-team, kill criteria.
   - §9 feasibility, **§9.1a E2E slice**, §9.2 WBS, §9.4 cut ladder.
   - §10 gates G1–G6; §11 related work; §12 non-claims; §13 summary.
   - Appendices: A threat model, **B statement schema**, C evaluation matrix, D terminology, **E requirements × options**, **F default decisions**.
2. **ANL-002** — what already ships: RFC 9943/9942/9162/3161, Tessera, Sigsum, Rekor, `scitt-verifier`, e-CMR/eBL, PoD apps, EPCIS. The ~70–80% figure is assumed; the RFC 9162 wrapper is custom.
3. **TUT-001** — the learning guide: 8 modules, the SHP-42 scenario, exercises with expected outcomes, the 16-week plan, reading order, tooling.
4. **REF-001** — the canonical glossary; PRP Appendix D and TUT §15 are subsets.
5. **REVIEW-SESSION-POD.md** — the review session ID and status.

Old documents (DEC-001, ANL-001, PRD-001, TDD-001, PLN-001, DISCUSSION-SUMMARY) are **legacy/related work**, not the source of truth for this topic.

---

## 7. Review status

- Session: `ses_ea126d834ffeqbCczjZ77a9gAk`.
- Initial audit: 3 critical, 8 high, 13 medium, 11 low.
- Re-audit: **all critical and high closed**; one new medium (falsifier logic) and the low nits were fixed.
- Open critical/high: **none**.
- The next review should **resume this session by default**, not start a new one. A subagent session can only be resumed from the BUILD session that spawned it; if a new BUILD session cannot resume it, start a fresh review and port `REVIEW-SESSION-POD.md`.

---

## 8. The first question: do we need to design the statement on our own?

**Short answer: yes, a PoD profile is required — but do not invent a vocabulary from scratch.** This is the first discussion, not a settled decision. Present the trade before choosing.

### Why a profile is required

- **RFC 9943 is content-agnostic.** It defines the roles, the Signed Statement envelope (`COSE_Sign1`), `iss`/`sub`, registration policy, receipts, non-equivocation, and replayability. It does **not** define a delivery payload.
- **RFC 9942** defines receipts (proofs over a VDS), not the statement body.
- **RFC 9162** defines the Merkle tree and proofs.
- So the project must choose **what a delivery statement contains**. That choice is C1 in PRP-001 §7.2, and it is part of the contribution.

### The options

| Option | What it is | Strengths | Costs |
| --- | --- | --- | --- |
| **A. Adopt EPCIS 2.0** | Use GS1 EPCIS events (ObjectEvent, AggregationEvent, TransactionEvent, etc.) as the payload | Standard, interoperable, event vocabulary, sensor elements | Heavy; XML/JSON-LD; CBOR/COSE mapping is custom; PoD fields (condition, receipt, `prev`, `sequence`) need extensions |
| **B. Adopt e-CMR fields** | Use the consignment-note data elements (consignor, carrier, consignee, goods, places, dates, reservations) | Legal alignment; familiar to the domain | Paper-oriented; no event model; no CBOR/COSE; incomplete for digital receipts and offline time |
| **C. Minimal PoD profile** | The current Appendix B schema: `statement_id`, `sequence`, `shipment_id`, `issuer`, `subject`, `event`, `time`, `place`, `items`, `photos`, `prev`, `nonce` | Small; tailored; easy to sign and test; fits the slice | Custom; needs justification; may miss legal/interop fields |
| **D. Hybrid** | EPCIS/e-CMR field names inside a minimal CBOR profile, with a PoD extension | Reuses vocabulary without the full standard; keeps the slice small | Two sources of truth unless the mapping is explicit; needs a profile document |

### A provisional lean (for discussion, not a decision)

**D or C.** Define a **minimal CBOR profile** (C) but borrow field names and semantics from EPCIS 2.0 and e-CMR where they fit (D). Treat the profile as C1, with a short decision record:

- the subject (`shipment` vs `consignment` vs `item`);
- the event types (`pickup`, `delivery`, `receipt`);
- condition representation (enum vs free text);
- photo binding (hash + media type + capture time);
- `prev` (hash of the prior signed `COSE_Sign1`);
- `sequence` (per issuer, signed, for withholding detection);
- which legal fields (if any) are in scope;
- what is deliberately out of scope.

### What to ask before choosing

1. Does the stakeholder need **EPCIS interoperability** (e.g., an existing WMS/ERP)?
2. Does the jurisdiction require **e-CMR fields** or a qualified signature?
3. Is the slice judged on **interoperability** or on the **transparency properties**? (The latter needs less vocabulary.)
4. Can the profile be **extended later** without invalidating old bundles? (Version the profile.)
5. Does the statement need to carry **personal data** (names, addresses), or only hashes/IDs? (Privacy and retention.)

### What not to do

- Do not invent a large schema. The slice needs the minimum that supports the claims.
- Do not put the full photo in the statement; hash it and keep the payload off-chain.
- Do not confuse the **statement** with the **receipt**; the statement is the assertion, the receipt is the proof of inclusion.
- Do not use JSON for the signed bytes; use deterministic CBOR and sign the bytes.

**The new agent should open with this trade, not with a recommendation.** The user wants to discuss it.

---

## 9. Other open questions

1. **G1 terminology:** does the supervisor mean a hash chain, a transparency log, or a consensus ledger? If consensus, the project is a kill or a 4-week extension.
2. **G2 stakeholder:** which delivery workflow, and who owns it? No build without a named person or unit.
3. **G3 module:** COMP4299 or CSAI4299? If CSAI4299 only, this proposal does not fit; do not force it.
4. **G6 ethics:** synthetic data only, or a user study with consent?
5. **Statement profile:** EPCIS, e-CMR, minimal, or hybrid (§8).
6. **RFC 9162 receipt wrapper:** can the spike produce and verify an RFC 9942 inclusion receipt for an RFC 9162 leaf? If not, change the VDS or the verifier.
7. **Witness quorum:** 2-of-2 for the slice; is 2-of-3 needed for the evaluation?
8. **Offline time:** is timestamp-on-reconnect acceptable, or is a depot TSA needed?
9. **Review session:** can the new BUILD session resume `ses_ea126d834ffeqbCczjZ77a9gAk`? If not, port the tracker.

---

## 10. What the new agent should do

1. Read PRP-001, ANL-002, TUT-001, REF-001, and the review tracker.
2. Check `git log` and `git status`; do not stage or commit.
3. Answer the first question (§8) as a **trade**, with the options and the questions above; do not start coding.
4. Do not start the build before **G1 and G2**.
5. Preserve the conventions:
   - every number measured, derived, or assumed, and labelled;
   - no weighted scores;
   - no strawman baselines;
   - two issuers, two witnesses 2-of-2, five trust anchors;
   - no “RFC compliant” without the conformance suite;
   - no “blockchain” for level 3; call it a witnessed Merkle transparency log;
   - the oracle and legal boundaries are stated;
   - the old DEC-001/ANL-001 material is legacy.
6. Resume the review session by default for any audit; do not recreate a full review record under the project root (the tracker is minimal by design).
7. If asked for next steps, the safe analytical work is:
   - the statement profile decision record (§8);
   - the RFC 9162 receipt spike;
   - the supervisor question set (G1–G6);
   - the stakeholder/anchor search.

---

## 11. Assets and references

| Asset | Path / reference |
| --- | --- |
| Proposal | `PRP-001__Proof_of_Delivery_Proposal.md` |
| Existing libraries/products | `ANL-002__PoD_Existing_Solutions.md` |
| Learning guide | `TUT-001__E2E_Slice_Learning_Guide.md` |
| Glossary | `REF-001__PoD_Glossary.md` |
| Review tracker | `REVIEW-SESSION-POD.md` |
| Review session | `ses_ea126d834ffeqbCczjZ77a9gAk` |
| Previous engine track | `docs/DEC-001`, `docs/ANL-001` (closed, related work) |
| Legacy design | `docs/PRD-001`, `docs/TDD-001`, `docs/PLN-001` (legacy references) |

**Standards:** RFC 9943 (SCITT architecture, June 2026), RFC 9942 (COSE receipts), RFC 9162 (CT v2 Merkle log), RFC 3161 (TSA), RFC 9052/9053 (COSE), RFC 8949 (CBOR); eIDAS, e-CMR, MLETR, GS1 EPCIS 2.0. Verify current editions.

**Libraries:** Tessera, Trillian, Sigsum, `transparency-dev/witness`, Rekor v2, `microsoft/scitt-verifier`, `scitt-ccf-ledger`, FreeTSA/`uts-server`, WebCrypto, `go-cose`, `fxamacker/cbor`.

---

## 12. Cautions

- **Do not call level 3 a blockchain.** It is a witnessed Merkle transparency log. Define the terms.
- **Do not claim RFC compliance.** Claim conformance to the implemented MUSTs and pass the suite.
- **Do not claim physical truth.** The system proves assertions, inclusion, time, and non-equivocation — not that the parcel arrived.
- **Do not claim legal non-repudiation.** A self-hosted log is evidence, not a qualified trust service.
- **Do not present planning estimates as measurements.** 290 h, 18 h/week, and 70–80% are estimates.
- **Do not rebuild the log, COSE, CBOR, TSA, or CA.** Reuse; the RFC 9162 receipt wrapper is the exception.
- **Do not start the build before G1 and G2.**
- **Do not stage or commit** for the user.
- **Do not present the five levels as a standard taxonomy.** They are an analytical device.
- **Do not let the statement schema balloon.** The slice needs a minimal profile.
- **Do not conflate the insurer with a witness.** They are separate roles.
- **Do not claim the 2-of-2 quorum is trustless.** It assumes both witnesses are honest; say so.

---

## 13. One-paragraph summary

The project is a thin end-to-end proof-of-delivery slice: a courier and a recipient each issue a `COSE_Sign1` statement about the same shipment, a Tessera-backed RFC 9162 log returns RFC 9942 receipts, two independent witnesses cosign checkpoints, an RFC 3161 TSA supplies time, and a verifier checks the bundle offline. It is evaluated against a signed database (level 1), a hash chain plus TSA (level 2), and a cost model of the 2018 permissionless-chain paper (level 5). The contribution is the PoD profile, offline trusted time, offline revocation, witness governance, privacy/linkage, and the evaluation — not the log. ANL-002 estimates 70–80% of the code is commodity (assumed); the RFC 9162 receipt wrapper is custom. The plan is 290 h at 18 h/week over 16 weeks, with writing from week 2. The supervisor’s “blockchain” terminology (G1) and a named stakeholder (G2) are the gates. The review session closed all critical and high findings; no build has started. The first question for the new agent is whether the statement must be designed from scratch — the answer is a profile, but the choice between EPCIS 2.0, e-CMR, a minimal schema, and a hybrid is the discussion.
