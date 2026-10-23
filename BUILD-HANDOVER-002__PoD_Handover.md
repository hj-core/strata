# BUILD-HANDOVER-002 — PoD and Document-Handover Proposal

| Field | Value |
| --- | --- |
| Prepared | 2026-10-22 |
| For | The next BUILD agent continuing the PoD / document-handover discussion |
| Repo | `/home/hj-core/project` |
| Branch | `tdd` |
| HEAD | `cdb5a93` (“Save”) |
| Working tree | `BUILD-HANDOVER.md` modified; **nine untracked files** (eight project documents plus this handover) |
| Read first | `DISCUSSION-SUMMARY-002__PoD_Handover_Proposal.md`, then `PRP-001__Proof_of_Delivery_Proposal.md` v0.2.0 |
| Status | Proposal revised; **review pending**; **no build started** |
| Supersedes | This file is the current handover for the PoD topic. `BUILD-HANDOVER.md` is a historical snapshot of the earlier 0.1.0 state; do not edit it. |

---

## 1. Status at a glance

| Item | State |
| --- | --- |
| Track | COMP4299 system development (module fit still to confirm, G3) |
| Topic | A thin end-to-end **document-handover / proof-of-delivery** slice on a witnessed Merkle transparency log using the RFC 9943 architecture |
| Primary motivation | **Document handover in universities and employers** — transcripts, diplomas, employment letters, contracts |
| Demo instance | **SHP-42 parcel** — two issuers, one log, two witnesses, TSA, verifier |
| Architecture | RFC 9943 roles, `COSE_Sign1` statements, RFC 9942 receipts, RFC 9162 Merkle log via Tessera, RFC 3161 time, X.509 CA |
| Issuers | Sender + recipient. Document case: registry/HR + student/employee. Parcel demo: carrier + recipient; consignor is a relying party |
| Trust anchors | Five: log key, witness A, witness B, TSA root, CA root. Issuer/recipient keys are leaves, not anchors |
| Plan | 290 h at 18 h/week over 16 weeks, with a four-tier contingency plan (`PLN-002`) |
| PRP-001 | **v0.2.0**, primary motivation revised; review pending |
| Build | **Not started** — do not start before G1 and G2 |
| Git | HEAD `cdb5a93`; do not stage or commit unless the user asks |

---

## 2. Repo and document state

- Branch: `tdd`. HEAD: `cdb5a93` (“Save”). Earlier: `74cdfa4`, `16469d0`, `5b1244f`.
- `BUILD-HANDOVER.md` is modified. It is the historical handover; leave it alone.
- The current working set is **untracked**:
  - `PRP-001__Proof_of_Delivery_Proposal.md` (v0.2.0)
  - `ANL-002__PoD_Existing_Solutions.md`
  - `ANL-003__PoD_Status_Quo_Comparison.md`
  - `PLN-002__Scope_Contingency.md`
  - `TUT-001__E2E_Slice_Learning_Guide.md`
  - `REF-001__PoD_Glossary.md`
  - `REVIEW-SESSION-POD.md`
  - `DISCUSSION-SUMMARY-002__PoD_Handover_Proposal.md`
  - `BUILD-HANDOVER-002__PoD_Handover.md` (this handover)
- Do **not** stage or commit. The user commits the documents.
- Old documents remain tracked and are **legacy/related work**, not the source of truth:
  - `docs/DEC-001__Decision_Options_Brief.md`
  - `docs/ANL-001__Existing_Solutions_Contrast.md`
  - `docs/PRD-001`, `docs/TDD-001`, `docs/PLN-001`
  - `DISCUSSION-SUMMARY.md` (historical record; the PoD discussion is in the new `-002` file)
- Do not bump versions unless asked. PRP-001 was bumped to 0.2.0 because the primary motivation and issuer mapping changed.

---

## 3. What the project is

A thin vertical slice of a **witnessed Merkle transparency log** applied to handovers:

- the **sender** issues a signed handover statement — a document hash in the primary case, items/photo hashes in the parcel demo;
- the **recipient** issues a separate receipt statement, linked by `prev` to the exact signed sender statement;
- a **Tessera-backed RFC 9162 log** appends both, returns RFC 9942 inclusion receipts, and produces signed checkpoints;
- **two independent witnesses** cosign checkpoints (2-of-2);
- an **RFC 3161 TSA** timestamps the statement hashes;
- a **verifier** checks signatures, inclusion, checkpoint, witnesses, `prev`, timestamps, revocation, and policy — offline;
- a **dispute bundle** carries the evidence and verifies without the operator.

**Contribution under the lower bar:** the handover profile, the RFC 9162 receipt wrapper, offline time/revocation choices, witness governance, privacy/linkage analysis, and the evaluation. The log, COSE, CBOR, TSA, and CA are reused. The RFC 9162 receipt wrapper is custom because `scitt-verifier` is CCF-only.

**Non-claims:** it does not prove physical truth, documentary truth, legal non-repudiation, or device integrity. It is evidence, not a qualified trust service, and not a blockchain.

---

## 4. What changed in PRP-001 v0.2.0

The proposal was re-framed from a parcel-first PoD to **document handover first**:

- **Primary motivation:** a university registry or employer HR function issues an official document to a student or employee; the institution is both issuer and record keeper.
- **SHP-42 parcel:** retained as the worked demo because it makes the handover visible.
- **Issuer mapping:** document case — registry/HR sender + student/employee recipient. Parcel demo — carrier (sender's agent) + recipient; consignor is a relying party.
- **§3.1a:** new primary-case/demo mapping table.
- **§6.1a:** actor/role table updated; counts become 3–4 workflow roles, 7 trust-path parties, five anchors.
- **§6.2a:** explicit demo-to-primary mapping.
- **§9.1a:** one document or parcel handover; document variant replaces items/photos with a document hash and metadata.
- **Appendix B:** handover schema with a document variant.
- **Related work:** added ANL-003 and W3C Verifiable Credentials/digital diplomas as adjacent work.
- **REVIEW-SESSION-POD:** records the v0.2.0 revision as **not yet reviewed**.

---

## 5. Decisions and status

### 5.1 Settled

1. Keep the **RFC 9943 / level-3 architecture** for planning; G1 still needs written confirmation from the supervisor.
2. **Primary motivation** is university/employer document handover; the parcel is the demo.
3. **Two issuers** — sender and recipient — linked by `prev`, not a multi-signature envelope.
4. **SHP-42 parcel** as the demo instance.
5. **290 h plan + `PLN-002`**: Tiers 1–2 pre-authorized; Tiers 3–4 need written sign-off.
6. **Five anchors**: log, two witnesses, TSA, CA root. Issuer/recipient keys are leaves.
7. **`prev` and `sequence` are profile fields**, not RFC 9943 fields.
8. **Document hash binds the content**: hash the exact bytes; preserve the original file.
9. **Build freeze**: no build before G1 and G2.
10. **Do not stage or commit** for the user.

### 5.2 Provisional

- **Minimal CBOR profile (option C)**; no EPCIS/e-cMR mapping unless a stakeholder requires interoperability.
- **2-of-2 witnesses**; 2-of-3 remains an evaluation option.
- **One CA root**; federation/multiple roots is open.
- **Selective disclosure** is required for employment/student records but is not implemented in the slice.

### 5.3 Open

- G1 written confirmation; G2 named stakeholder/design partner; G3 module; G4 deliverables; G6 ethics.
- RFC 9162 receipt wrapper spike.
- 2-of-2 vs 2-of-3 liveness/governance.
- Legal recognition and QTSP/ERDS complement.
- CA federation and key history.
- Cost quotes and adoption-cost model.
- Demand validation: no public PoD/university/employment deployment found.

---

## 6. Technical clarifications the new agent should not re-litigate

These were discussed at length; treat them as the current understanding unless the user reopens them.

### 6.1 RFC 9943 is an architecture, not a CA or log standard

- It defines roles, signed statements, registration policy, receipts, non-equivocation, replayability.
- It does **not** define a CA, certificate enrolment, revocation, the payload, the VDS, the API, or witness governance.
- It is content-agnostic. The software supply chain is the first scope, not the only one.
- "Third-party verifiable statements" is shorthand; the purpose is third-party verification of **publication, inclusion, and non-equivocation**, not truth.

### 6.2 The CA is outside the RFC's role model

- The CA verifies **who holds the key**, at enrolment; it never sees the statement.
- The verifier anchors at the **CA root**, not at the issuer's or recipient's key.
- Issuer and recipient keys are leaves under the CA; a federation root can collapse multiple institutional CAs.
- A CA compromise can mint identities; witnesses check consistency, not identity.

### 6.3 `prev` and `sequence`

- `prev` is the SHA-256 of the prior **signed `COSE_Sign1` bytes**; it links the receipt statement to the exact handover statement.
- `sequence` is a per-issuer signed counter; a gap is evidence of withholding.
- Neither is an RFC 9943 field; both are PoD profile fields.
- `sequence` is not the log's `log_index`; the log assigns the index and could renumber it, which is why the issuer signs its own sequence.

### 6.4 Witness model

- 2-of-2: both witnesses must cosign the head.
- One malicious witness cannot produce a valid fork if the other is honest, stateful, and the verifier requires both signatures; it can stall liveness or leak metadata.
- Both witnesses colluding can produce a valid fork; this is the binding assumption.
- The witness must remember its last cosigned head and refuse a conflicting head at the same size.
- If the verifier accepts 1-of-2, or the operator controls both witnesses, the non-equivocation claim collapses.

### 6.5 Single points of failure

- The log is a single point for new registrations and availability; existing bundles verify offline.
- The TSA is a single point for new time; existing tokens verify offline.
- The CA is a single point for identity.
- The 2-of-2 quorum is a single point for liveness, not safety.
- The verifier and anchor distribution are single points for the relying party's decision.
- The design removes the operator as a single point of **integrity** trust; it does not remove trust or availability dependence.

### 6.6 Privacy

- Hash-only is **data minimisation**, not anonymisation.
- Metadata — issuer, subject, event, time, sequence, size — can link records; small cohorts are easy to re-identify.
- A hash of a low-entropy document can be confirmed by hashing candidates; a per-document salt or HMAC helps, at the cost of sharing the salt.
- Immutability conflicts with erasure; a retention policy and a private/consortium log are required.
- Selective disclosure is essential for employment (salary) and student records (grades), but a single document hash is all-or-nothing.
- The honest claim: "more privacy-preserving than emailing full documents or granting portal access, less than a minimal trusted registry" — not "private by default."

### 6.7 Cost

- **Infrastructure is low:** Azure Confidential Ledger is about **$0.124/hour per instance** (~$90/month) plus **$0.16/GB-month** storage, first 100 GB included; open-source logs are free.
- **Supply-side cost is not low:** TSA tokens, CA/HSM, two independent witnesses, audits, integration, legal, and labour dominate.
- **Adoption cost is not infrastructure cost:** identity onboarding, SIS/HR integration, policy, training, legal, and change management are the real barriers.
- At 1M handovers/month, $300/month infrastructure is ~$0.0003/handover; at 10k/month it is ~$0.03/handover. The PRP target of <$0.01/handover is a marginal infrastructure target, not total cost of ownership.

---

## 7. Comparison and the candidate wedge

### 7.1 The trusted-operator boundary

If the operator is fully trusted — honest, competent, available, long-lived, and trusted by every relying party — the proposed system has little advantage; a signed database or portal is cheaper. The system is an **assurance purchase** for the case where the record keeper is not trusted or a third party needs independent proof.

- **L1 signed database:** sufficient if the operator is trusted and is the only relying party.
- **L2 hash chain + TSA:** adds tamper-evidence relative to a known head; sufficient if the operator is trusted not to equivocate.
- **L3 log + witnesses:** adds non-equivocation and withholding detection; justified only if the operator may equivocate or withhold, or multiple parties need a shared head.

### 7.2 The candidate wedge

> A record keeper that may equivocate or withhold, a relying party that cannot trust it, no trusted external registry that already provides independent verification, and a dispute value high enough to justify the fixed cost.

The strongest concrete framing is a **cross-institution credential consortium**:

- many universities/employers, many verifiers;
- no single trusted central registry;
- each institution issues signed statements;
- the log is run by a consortium, not by one institution;
- two external witnesses (another university, an auditor, or a public authority) cosign the head;
- the student/employee holds the receipt and the document;
- the employer/regulator verifies offline;
- selective disclosure protects salary and grades.

The employment variant is the higher-frequency commercial case: cross-border verification, former employers that are uncooperative or defunct, portable proof for mobile workers.

### 7.3 What would falsify it

- A trusted national registry/QTSP/ERDS already covers the need.
- The institution is trusted by every relying party; no split-view or withholding dispute exists.
- The witnesses are not independent, or the verifier accepts 1-of-2.
- The cost exceeds the dispute/fraud savings.
- Privacy/erasure rules rule out an immutable log.
- Recipients cannot use keys, and onboarding costs dominate.
- The physical-site engagement draft or another SCITT profile already owns the use case.

---

## 8. Demand-side evidence (public, as of 2026-10-22)

| Area | Status | Evidence |
| --- | --- | --- |
| Microsoft Signing Transparency | **Production**, scoped to Microsoft services | Azure services: MAA, Key Vault Managed HSM, Azure confidential ledger; GA 2026 |
| `scitt-ccf-ledger` | Open-source reference implementation | Runs Microsoft's service; CCF; MIT |
| `scitt-verifier` | Open-source offline verifier | **CCF receipts only; RFC 9162 receipts unsupported** |
| DataTrails | **Preview** | SCITT API, draft v10; no public price found |
| Software supply-chain use cases | **Requirements draft** | Signed builds, SBOMs, provenance, firmware, automotive |
| AI governance | **Drafts/profiles** | EU AI Act Article 50 receipts, AI refusal events, AI-agent action receipts |
| Algorithmic trading | **Draft/profile** | VeritasChain Protocol |
| Physical-site engagement | **Individual draft, Oct 2026** | Includes **delivery** as an engagement type; TEE + SCITT; closest to PoD |
| vCon / contact centers | **Prototype/pilot** | SCITT link; 211 pilot mentions SCITT-verified outcome reporting; no public production deployment claimed |
| PoD / university / employment | **No public production application found** | Open domain; demand unproven |

**Supply-heavy, demand-light.** The only production user is Microsoft's own software-signing workflow. The rest are previews, pilots, or drafts. The physical-site engagement draft is a warning that someone is already claiming the physical-world evidence space, including delivery.

---

## 9. Other applications discussed

- **Proof of employment:** a sub-case of the employer document handover. Employer signs, employee holds, bank/landlord/next employer verifies offline. Selective disclosure is essential. Google-scale verification is a trusted-intermediary process; the proposed system is not a first pitch to Google, but may fit a verification vendor, payroll provider, or cross-border network.
- **Tendering / commercial ledger:** technically applicable, but larger and more regulated. A tender is a multi-stage workflow with sealed bids, evaluation confidentiality, and procurement law. A commercial ledger can mean an evidence log (plausible) or a settlement/accounting ledger (not a fit; the log has no consensus or payment finality). Both are future work, not the 16-week slice.
- **ID card / driver's licence:** technically a credential-transparency use case, but not the ID itself. The primary technology should be an mDL (ISO/IEC 18013-5), eIDAS 2.0 wallet, or W3C Verifiable Credential. The log is an optional audit/non-equivocation layer; a public log of licence holders is a privacy problem. Not the primary case.
- **Physical-site engagement receipts:** the closest external draft to PoD. It uses a TEE, a site owner, an issuer, and a SCITT transparency service. It includes delivery as an engagement type. Treat it as related work and potential competition; differentiate on document/credential handover, two issuers, offline verification, privacy/selective disclosure, and no TEE dependency.

---

## 10. Open questions and gates

| Item | Status | Next action |
| --- | --- | --- |
| **G1** — does “blockchain” mean L3? | Verbal only | Get written confirmation that a witnessed Merkle log is acceptable |
| **G2** — named stakeholder | Open | Approach a registry, employer, verification vendor, or consortium |
| **G3** — module | Open | Confirm COMP4299 vs CSAI4299 and the rubric |
| **G4** — deliverables | Open | Confirm demo + report vs benchmark study |
| **G6** — ethics | Open | Synthetic data only, or user study with consent |
| **RFC 9162 receipt wrapper** | Spike needed | Prove RFC 9942 receipt encode/verify for an RFC 9162 leaf |
| **2-of-2 vs 2-of-3** | Open | Compare liveness/governance in the evaluation |
| **Selective disclosure** | Not implemented | Design for employment/student records |
| **Legal recognition** | Open | QTSP/ERDS complement; do not claim legal non-repudiation |
| **CA federation** | Open | One root vs multiple institutional roots |
| **Cost model** | Rough | Get quotes for TSA, managed platform, witnesses; add to `PLN-002`/ANL-003 |
| **Demand validation** | Open | Design-partner interviews; do not treat drafts as demand |
| **Review v0.2.0** | Pending | Resume `REVIEW-SESSION-POD`; the revision is not yet audited |

---

## 11. What the new agent should do

1. **Read** `DISCUSSION-SUMMARY-002__PoD_Handover_Proposal.md`, then PRP-001 v0.2.0, then ANL-003, `PLN-002`, and `REVIEW-SESSION-POD`. ANL-002, TUT-001, and REF-001 are supporting material.
2. **Check** `git log` and `git status`. Do not stage or commit.
3. **Continue the discussion.** Do not start coding. The user is deciding whether the proposal has a defensible scenario and a stakeholder.
4. **Do not start the build before G1 and G2.** G1 is now the written confirmation that a witnessed Merkle log is acceptable; G2 is a named stakeholder or design partner.
5. **If asked for next steps,** the safe analytical work is:
   - the strongest-alternative test (trusted registry, QTSP/ERDS, portal, paper);
   - the adoption-cost / who-pays matrix;
   - the RFC 9162 receipt spike;
   - the stakeholder/design-partner brief;
   - the supervisor question set for G1–G6.
6. **Resume the review session by default** for any audit: `ses_ea126d834ffeqbCczjZ77a9gAk`. PRP-001 v0.2.0 is **not yet reviewed**; do not treat it as audited.
7. **Preserve the conventions:**
   - every number measured, derived, or assumed, and labelled;
   - no weighted scores;
   - no strawman baselines;
   - two issuers, two witnesses 2-of-2, five trust anchors;
   - no “RFC compliant” without the conformance suite;
   - no “blockchain” for level 3; call it a witnessed Merkle transparency log;
   - no “trustless”;
   - no physical or documentary truth;
   - no legal non-repudiation;
   - the old DEC-001/ANL-001/PRD/TDD/PLN material is legacy.

---

## 12. Cautions

- **Do not call level 3 a blockchain.** It is a witnessed Merkle transparency log.
- **Do not claim RFC compliance.** Claim conformance to the implemented MUSTs and pass the suite.
- **Do not claim physical or documentary truth.** The system proves assertions, inclusion, time, and non-equivocation — not that the parcel arrived or that a transcript is genuine.
- **Do not claim legal non-repudiation.** A self-hosted log is evidence, not a qualified trust service.
- **Do not present planning estimates as measurements.** 290 h, 18 h/week, and the <$0.01/handover target are estimates/targets.
- **Do not rebuild the log, COSE, CBOR, TSA, or CA.** Reuse; the RFC 9162 receipt wrapper is the exception.
- **Do not start the build before G1 and G2.**
- **Do not stage or commit** for the user.
- **Do not present the five levels as a standard taxonomy.** They are an analytical device.
- **Do not let the statement schema balloon.** The slice needs a minimal profile.
- **Do not conflate the insurer with a witness.** They are separate roles.
- **Do not claim the 2-of-2 quorum is trustless.** It assumes both witnesses are honest.
- **Do not mistake drafts for demand.** RFC 9943 has one production user, one preview platform, and many profiles; no public PoD/university/employment deployment was found.
- **Do not assume Microsoft Signing Transparency is available to a university.** It is generally available but scoped to specific Microsoft services.

---

## 13. One-paragraph summary

The proposal is a thin end-to-end document-handover and proof-of-delivery slice built on the RFC 9943 / SCITT architecture: two signed statements linked by `prev`, an RFC 9162 Merkle log via Tessera, RFC 9942 receipts, two independent witnesses, RFC 3161 time, and an offline verifier. The primary case is a university or employer issuing an official document to a student or employee; the SHP-42 parcel is the demo instance. The 290 h plan is retained with a four-tier contingency plan. The system is not a universal improvement: it is a candidate for the case where the record keeper may equivocate or withhold, a third party must verify offline, and no trusted central registry exists. Public evidence shows a supply-heavy, demand-light RFC 9943 ecosystem — Microsoft Signing Transparency is in production but scoped to Microsoft services; DataTrails is a preview; the rest are drafts or pilots; no PoD, university, or employment application was found. The infrastructure is cheap; the cost of trust is not. The next gates are a written G1, a named G2 stakeholder, the RFC 9162 receipt spike, and a review of PRP-001 v0.2.0.

---

## 14. Assets and references

| Asset | Path / reference |
| --- | --- |
| Proposal | `PRP-001__Proof_of_Delivery_Proposal.md` (v0.2.0) |
| Discussion record | `DISCUSSION-SUMMARY-002__PoD_Handover_Proposal.md` |
| Status-quo comparison | `ANL-003__PoD_Status_Quo_Comparison.md` |
| Existing libraries/products | `ANL-002__PoD_Existing_Solutions.md` |
| Contingency plan | `PLN-002__Scope_Contingency.md` |
| Learning guide | `TUT-001__E2E_Slice_Learning_Guide.md` |
| Glossary | `REF-001__PoD_Glossary.md` |
| Review tracker | `REVIEW-SESSION-POD.md` |
| Review session | `ses_ea126d834ffeqbCczjZ77a9gAk` |
| Old handover | `BUILD-HANDOVER.md` (historical; do not edit) |
| Previous engine track | `docs/DEC-001`, `docs/ANL-001` (closed, related work) |
| Legacy design | `docs/PRD-001`, `docs/TDD-001`, `docs/PLN-001` (legacy references) |

**Standards:** RFC 9943 (SCITT architecture, June 2026), RFC 9942 (COSE receipts), RFC 9162 (CT v2 Merkle log), RFC 3161 (TSA), RFC 9052/9053 (COSE), RFC 8949 (CBOR); eIDAS, e-CMR, MLETR, GS1 EPCIS 2.0; ISO/IEC 18013-5 (mDL) for ID/licence comparisons. Verify current editions.

**Libraries / services:** Tessera, Trillian, Sigsum, `transparency-dev/witness`, Rekor v2, `microsoft/scitt-verifier`, `microsoft/scitt-ccf-ledger`, FreeTSA/`uts-server`, WebCrypto, `go-cose`, `fxamacker/cbor`; Microsoft Signing Transparency and DataTrails for supply-side reference.
