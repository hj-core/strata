# DISCUSSION-SUMMARY-002 — PoD and Document-Handover Proposal

| Field | Value |
| --- | --- |
| Document ID | DISCUSSION-SUMMARY-002 |
| Title | Discussion summary — proof-of-delivery and document-handover proposal |
| Date | 2026-10-22 |
| Status | Working record — not a proposal, not a source of truth |
| Related documents | PRP-001 v0.2.0; ANL-002; ANL-003; PLN-002; TUT-001; REF-001; REVIEW-SESSION-POD |
| Supersedes | None. This file does not modify `DISCUSSION-SUMMARY.md`. |
| Purpose | Record the discussion, decisions, evidence, open questions, and next steps |

**Status note (2026-10-23).** This file remains the record of the v0.2.0 revision and is not rewritten. It is **superseded as the current discussion record** by `DISCUSSION-SUMMARY-003__PoD_Scenario_Search.md`, which records the subsequent scenario search and its outcome.

**How to read this.** The proposal is PRP-001 v0.2.0. This file records the discussion around it: what was decided, what is provisional, what was corrected, and what remains open. Where the two disagree, PRP-001 is the proposal and this file is the discussion history.

---

## 1. Starting point

The previous proposal failed for three reasons:

1. the requirement that justified the mechanism was self-set and unanchored;
2. no stakeholder imposed it;
3. no scenario showed an obvious advantage over existing solutions.

The new direction is deliberately different:

- a **thin end-to-end slice**, not a platform;
- the **RFC 9943 / SCITT architecture**, not a new protocol;
- the **contribution bar is lower**: a working standards-based slice plus an honest evaluation is sufficient;
- the **290 h plan is retained**, with a pre-authorized contingency plan (`PLN-002`);
- the **primary motivation is document handover in universities and employers**, with the SHP-42 parcel as the demo instance.

The project is system development, not protocol research. Novelty is not required; a defensible application profile and an honest evaluation are.

---

## 2. What the proposal is

A thin end-to-end document/parcel handover built on a **witnessed Merkle transparency log** (level 3):

| Layer | Standard / mechanism | What the project adds |
| --- | --- | --- |
| Signed statement | `COSE_Sign1` (RFC 9052/9053), CBOR (RFC 8949) | Handover payload, document hash, `prev`, `sequence` |
| Transparency service | RFC 9943 architecture | Registration policy, PoD/document profile |
| Receipts | RFC 9942 COSE receipts | RFC 9162 receipt wrapper (custom) |
| Log | RFC 9162 via Tessera | Configuration, checkpoint cadence |
| Time | RFC 3161 TSA | Offline-time policy |
| Witnesses | Two independent cosigners, 2-of-2 | Governance and independence assumptions |
| Verifier | RFC 9943 verifier role | Offline bundle, policy, named failures |
| Identity | X.509 CA | Role binding, revocation |

The two issuers are the **sender** and the **recipient**. In the document case, the registry/HR function is the sender and the student/employee is the recipient. In the parcel demo, the carrier acts for the sender.

**Non-claims:** it does not prove physical truth, documentary truth, legal non-repudiation, or that the device was uncompromised. It is evidence, not a qualified trust service, and not a blockchain.

---

## 3. Primary case and demo

### 3.1 Primary case

A university registry or an employer's HR function issues an official document — a transcript, diploma, employment letter, or contract — to a student or employee. The institution is both the issuer and, usually, the record keeper. The recipient needs evidence of what was issued, in which version, at what time, and that the record has not been silently changed.

### 3.2 Demo instance

The SHP-42 parcel remains the worked demo because it makes the handover visible. The mapping is explicit in PRP-001 §6.2a:

| Parcel demo | Primary case |
| --- | --- |
| `Acme` (consignor) | Registry/HR — the sender institution; in the demo it relies on the carrier rather than issuing |
| `Dana` / Northline (carrier, issuer) | The registry/HR clerk's signing key |
| `Sam` / BioLab (consignee, issuer) | Student or employee — the recipient |
| `SHP-42` | Transfer/document ID |
| Items and photo hashes | Document hash and metadata |
| Consignor/insurer | Student/employer/auditor relying party |

Acme is the sender institution in both cases. In the parcel demo it does not issue, so it verifies the carrier's statement; in the document case the registry or HR function issues directly, and the external relying party is the student, employer, or auditor.

### 3.3 Logical parcel vs legal parcel

At the protocol level, a document is a **logical parcel**: a subject with a content hash, a sender, a recipient, and a handover event. At the legal level, it is not a parcel: a transcript is not a consignment note, and an employment letter is not a bill of lading.

- **e-CMR** is for road-transport consignment notes.
- **eBL** is for transferable documents of title.
- **QTSP/ERDS** (qualified seals, registered delivery), **digital-diploma/verifiable-credential platforms**, and **academic registries** are the document analogs.
- None of them automatically provides independent non-equivocation across operators; that is the proposed layer.

### 3.4 Employment variant

Proof of employment is a sub-case of the employer document handover:

- employer signs an employment statement with a document hash;
- employee holds the bundle;
- bank, landlord, next employer, or visa authority verifies offline;
- selective disclosure is essential (role/dates without salary);
- the employer must issue and keep its key; the employee's receipt is a separate signature;
- Google-scale verification is a trusted-intermediary process; the proposed system is not a first pitch to Google, but may fit a verification vendor, payroll provider, or cross-border network.

---

## 4. Decisions and status

| Decision | Status | Note |
| --- | --- | --- |
| Keep the RFC 9943 / level-3 architecture | **Settled for planning** | G1 still needs written confirmation from the supervisor |
| Primary motivation: university/employer document handover | **Settled in PRP-001 v0.2.0** | Parcel retained as demo |
| Two issuers: sender + recipient | **Settled** | Carrier is the sender's agent in the parcel demo |
| SHP-42 parcel as demo instance | **Settled** | Document variant replaces items/photos with a document hash |
| 290 h plan + `PLN-002` contingency | **Settled** | Tiers 1–2 pre-authorized; Tiers 3–4 need sign-off |
| 2-of-2 witnesses | **Default** | 2-of-3 remains an evaluation option |
| Five verifier anchors: log, 2 witnesses, TSA, CA root | **Settled** | Issuer/recipient keys are leaves, not anchors |
| `prev` and `sequence` are PoD profile fields | **Settled** | Not defined by RFC 9943 |
| Document hash binds the content | **Settled** | Hash the exact bytes; preserve the original file |
| Minimal CBOR profile (option C) | **Provisional** | No EPCIS/e-CMR mapping required; no formal decision record yet |
| Selective disclosure | **Required for employment; not implemented** | Open design/extent |
| Build not started | **Settled** | Do not build before G1 and G2 |
| Do not stage or commit | **Settled** | The user commits |

---

## 5. Technical clarifications

### 5.1 RFC 9943 is an architecture, not a CA or log standard

- It defines roles, signed statements, registration policy, receipts, non-equivocation, and replayability.
- It does **not** define a CA, certificate enrolment, revocation, the payload, the VDS, the API, or witness governance.
- It is content-agnostic; the software supply chain is the first scope, not the only one.
- "Third-party verifiable statements" is a shorthand; the purpose is third-party verification of **publication, inclusion, and non-equivocation**, not truth.

### 5.2 The CA is outside the RFC's role model

- The CA verifies **who holds the key**, at enrolment; it does not see the statement.
- The verifier anchors at the **CA root**, not at the issuer's or recipient's key.
- Issuer and recipient keys are leaves under the CA; a federation root can collapse multiple institutional CAs.
- A CA compromise can mint identities; witnesses check consistency, not identity.

### 5.3 `prev` and `sequence`

- `prev` is the SHA-256 of the prior **signed `COSE_Sign1` bytes**; it links the receipt statement to the exact handover statement.
- `sequence` is a per-issuer signed counter; a gap is evidence of withholding.
- Neither is an RFC 9943 field; both are PoD profile fields.
- `sequence` is not the log's `log_index`; the log assigns the index and could renumber it, which is why the issuer signs its own sequence.

### 5.4 Witness model

- 2-of-2: both witnesses must cosign the head.
- One malicious witness cannot produce a valid fork if the other is honest, stateful, and the verifier requires both signatures; it can stall liveness or leak metadata.
- Both witnesses colluding can produce a valid fork; this is the binding assumption.
- The witness must remember its last cosigned head and refuse a conflicting head at the same size.
- If the verifier accepts 1-of-2, or the operator controls both witnesses, the non-equivocation claim collapses.

### 5.5 Single points of failure

- The log is a single point for new registrations and availability; existing bundles verify offline.
- The TSA is a single point for new time; existing tokens verify offline.
- The CA is a single point for identity.
- The 2-of-2 quorum is a single point for liveness, not safety.
- The verifier and anchor distribution are single points for the relying party's decision.
- The design removes the operator as a single point of **integrity** trust; it does not remove trust or availability dependence.

### 5.6 Privacy

- Hash-only is **data minimisation**, not anonymisation.
- Metadata — issuer, subject, event, time, sequence, size — can link records; small cohorts are easy to re-identify.
- A hash of a low-entropy document can be confirmed by hashing candidates; a per-document salt or HMAC helps, at the cost of sharing the salt.
- Immutability conflicts with erasure; a retention policy and a private/consortium log are required.
- Selective disclosure is essential for employment (salary) and student records (grades), but a single document hash is all-or-nothing.
- The honest claim: "more privacy-preserving than emailing full documents or granting portal access, less than a minimal trusted registry" — not "private by default."

### 5.7 Cost

- The **infrastructure** is low: Azure Confidential Ledger is about **$0.124/hour per instance** (~$90/month) plus **$0.16/GB-month** storage, first 100 GB included; open-source logs are free.
- The **supply-side cost** is not low: TSA tokens, CA/HSM, two independent witnesses, audits, integration, legal, and labour dominate.
- The **adoption cost** is not the same as infrastructure: identity onboarding, SIS/HR integration, policy, training, legal, and change management are the real barriers.
- At 1M handovers/month, $300/month infrastructure is ~$0.0003/handover; at 10k/month it is ~$0.03/handover. The PRP target of <$0.01/handover is a marginal infrastructure target, not total cost of ownership.

---

## 6. Comparison with alternatives

| Approach | Strength | Weakness | When it wins |
| --- | --- | --- | --- |
| **Paper signed copies** | Cheap, legally familiar, no infrastructure, independent copies | Copies can diverge; no trusted time, inclusion, or non-equivocation; manual retrieval | Low-value, local, cooperative handovers |
| **Scanned PDF / portal** | Integrated UX, onboarding, support, payload storage | Operator controls integrity, availability, audit, and often the only copy | Trusted platform, routine digital records |
| **e-CMR / eBL** | Legal recognition for transport documents | Wrong document for transcripts/employment letters; platform/registry trust | The legal consignment note or transferable document is the object |
| **QTSP / ERDS / digital-diploma registry** | Legal recognition, qualified seals, registered delivery, existing identity | Provider-dependent; typically no independent non-equivocation across operators | The requirement is legal recognition or registered delivery |
| **Proposed L3** | Shared head, inclusion, withholding/equivocation detection, offline third-party verification | High fixed/governance cost; privacy/erasure tension; requires employer/institution issuance | The record keeper may equivocate or withhold, a third party must verify offline, and no trusted central registry exists |

### 6.1 Trusted-operator boundary

If the operator is fully trusted — honest, competent, available, long-lived, and trusted by every relying party — the proposed system has little advantage; a signed database or portal is cheaper. The system is an **assurance purchase** for the case where the record keeper is not trusted or a third party needs independent proof.

The level boundary:

- **L1 signed database:** sufficient if the operator is trusted and is the only relying party.
- **L2 hash chain + TSA:** adds tamper-evidence relative to a known head; sufficient if the operator is trusted not to equivocate.
- **L3 log + witnesses:** adds non-equivocation and withholding detection; justified only if the operator may equivocate or withhold, or multiple parties need a shared head.

### 6.2 Demand-side evidence (public, as of 2026-10-22)

| Area | Status | Evidence |
| --- | --- | --- |
| Microsoft Signing Transparency | **Production**, scoped to Microsoft services | Azure services: MAA, Key Vault Managed HSM, Azure confidential ledger; GA 2026 |
| `scitt-ccf-ledger` | Open-source reference implementation | Runs Microsoft's service; CCF; MIT |
| `scitt-verifier` | Open-source offline verifier | CCF receipts only; **RFC 9162 receipts unsupported** |
| DataTrails | **Preview** | SCITT API, draft v10; no public price found |
| Software supply-chain use cases | **Requirements draft** | IETF draft; signed builds, SBOMs, provenance, firmware, automotive |
| AI governance | **Drafts/profiles** | EU AI Act Article 50 receipts, AI refusal events, AI-agent action receipts |
| Algorithmic trading | **Draft/profile** | VeritasChain Protocol |
| Physical-site engagement | **Individual draft, Oct 2026** | Includes **delivery** as an engagement type; TEE + SCITT; closest to PoD |
| vCon / contact centers | **Prototype/pilot** | SCITT link; 211 pilot mentions SCITT-verified outcome reporting; no public production deployment claimed |
| PoD / university / employment | **No public production application found** | Open domain; demand unproven |

**Supply-heavy, demand-light.** The only production user is Microsoft's own software-signing workflow. The rest are previews, pilots, or drafts. The physical-site engagement draft is a warning that someone is already claiming the physical-world evidence space, including delivery.

---

## 7. The candidate wedge

The proposed system does **not** beat a trusted registry, QTSP, ERDS, portal, or e-CMR platform in general. The defensible scenario is narrower:

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

The employment variant is the higher-frequency commercial case: cross-border verification, former employers that are uncooperative or defunct, and portable proof for mobile workers.

### 7.1 What must be true

1. The institution is the record keeper, and the relying party cannot fully trust its portal.
2. No trusted central registry/QTSP/ERDS already provides the same verification at acceptable cost.
3. The witnesses, TSA, CA, and verifier are independent of the institution.
4. The privacy design supports selective disclosure and a private/consortium log.
5. The dispute value or volume justifies the fixed cost.
6. The recipient can use a key or an SSO-issued credential.

### 7.2 What would falsify it

- A trusted national registry/QTSP/ERDS already covers the need.
- The institution is trusted by every relying party; no split-view or withholding dispute exists.
- The witnesses are not independent, or the verifier accepts 1-of-2.
- The cost exceeds the dispute/fraud savings.
- Privacy/erasure rules rule out an immutable log.
- Recipients cannot use keys, and onboarding costs dominate.
- The physical-site engagement draft or another SCITT profile already owns the use case.

---

## 8. Open questions and gates

| Item | Status | Next action |
| --- | --- | --- |
| **G1** — does "blockchain" mean L3? | Verbal only | Get written confirmation that a witnessed Merkle log is acceptable |
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

## 9. Next steps

1. **Get G1 and G2 in writing.** Do not start the build before both.
2. **Run the two-week spike** on the RFC 9162 receipt wrapper; if it fails, apply the `PLN-002` Tier 3 wrapper fallback.
3. **Write the strongest-alternative test** — trusted registry, QTSP/ERDS, portal, paper — and confirm the proposed system wins only where the operator is not trusted and no trusted registry exists.
4. **Approach a design partner** (registry, employer, verification vendor, consortium). Ask what they use today, what it costs, and whether they have had a split-view or withholding dispute.
5. **Implement the adoption-cost / who-pays matrix** for the university case, not just the infrastructure cost.
6. **Resume the review session** for PRP-001 v0.2.0; the revision changed the primary motivation and issuer mapping.
7. **Do not overclaim.** No "trustless," no "RFC compliant," no "proves the document is genuine," no "replaces a QTSP."

---

## 10. One-paragraph summary

The proposal is a thin end-to-end document-handover and proof-of-delivery slice built on the RFC 9943 / SCITT architecture: two signed statements linked by `prev`, an RFC 9162 Merkle log via Tessera, RFC 9942 receipts, two independent witnesses, RFC 3161 time, and an offline verifier. The primary case is a university or employer issuing an official document to a student or employee; the SHP-42 parcel is the demo instance. The earlier 290 h plan is retained with a four-tier contingency plan. The system is not a universal improvement: it is a candidate for the case where the record keeper may equivocate or withhold, a third party must verify offline, and no trusted central registry exists. Public evidence shows a supply-heavy, demand-light RFC 9943 ecosystem — Microsoft Signing Transparency is in production but scoped to Microsoft services; DataTrails is a preview; the rest are drafts or pilots; no PoD, university, or employment application was found. The infrastructure is cheap; the cost of trust is not. The next gates are a written G1, a named G2 stakeholder, the RFC 9162 receipt spike, and a review of PRP-001 v0.2.0.

---

## Evidence consulted

- RFC 9943, *An Architecture for Trustworthy and Transparent Digital Supply Chains*, Proposed Standard, June 2026: <https://www.rfc-editor.org/info/rfc9943/>
- Microsoft Signing Transparency Ledger: <https://learn.microsoft.com/en-us/azure/confidential-ledger/about-microsoft-signing-transparency-ledger>
- `scitt-ccf-ledger`: <https://github.com/microsoft/scitt-ccf-ledger>
- `scitt-verifier`: <https://github.com/microsoft/scitt-verifier>
- DataTrails SCITT API (preview): <https://docs.datatrails.ai/developers/developer-patterns/scitt-api/>
- SCITT specifications and implementations: <https://scitt.io/scitt-specs.html>, <https://scitt.io/implementations.html>
- IETF SCITT documents and profiles: <https://datatracker.ietf.org/wg/scitt/documents/>
- Physical-site engagement receipts draft: <https://datatracker.ietf.org/doc/draft-wilder-scitt-physical-site-engage-receipt/>
- vCon / SCITT patterns: <https://www.conserver.io/use-cases-studies/patterns>
- Azure Confidential Ledger pricing: <https://azure.microsoft.com/en-us/pricing/details/confidential-ledger/>
- FreeTSA: <https://freetsa.org/index_en.php>

All prices are public list prices or estimates as of 2026-10-22, not quotes. All deployment statuses are based on public evidence; absence of evidence is not proof of absence.
