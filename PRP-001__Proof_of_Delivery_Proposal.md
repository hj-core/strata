# PRP-001 — Proof of Delivery and Document Handover with a Witnessed Merkle Transparency Log

| Field | Value |
| --- | --- |
| Document ID | PRP-001 |
| Title | Proof of delivery and document handover with a witnessed Merkle transparency log |
| Version | 0.2.0 |
| Status | Draft — revised primary motivation; review pending |
| Last Updated | 2026-10-22 |
| Primary motivation | Document handover in universities and employers: transcripts, diplomas, employment letters, and contracts. The parcel delivery is the worked demo instance. |
| Track | COMP4299 system development (module fit to confirm, §10) |
| Related documents | ANL-002 (existing libraries and products); ANL-003 (status-quo comparison); DEC-001, ANL-001 (previous track, now related work); DISCUSSION-SUMMARY; Hasan & Salah (2018) as the level-5 comparison |
| Supersedes | None. The server-side cut proposal is closed; its analysis is retained as related work. |

---

## 1. Purpose

### 1.1 What this document is

This is a **proposal for examination**, not a design document. It states an idea, the conditions under which the idea is defensible, and the evidence that would kill it. It is written before any build so that the decision is made on requirements and threat model rather than on enthusiasm.

The primary motivation is **document handover in universities and employers**: a registry or HR function issues an official document — a transcript, diploma, employment letter, or contract — to a student or employee. The institution is both the issuer and, usually, the record keeper. The recipient needs evidence that the document was issued, in that version, at that time, and that the record cannot be silently changed.

The parcel scenario in §6.2a is the **worked demo instance**. It makes the handover visible and exercises the same protocol: two signed statements, a log, receipts, witnesses, and a timestamp. Where this document says "courier" or "carrier," read "the sender or its agent"; "parcel," read "document"; "shipment," read "transfer or document ID"; and "photo," read "content hash" for the primary case.

### 1.2 The question

> Can a proof-of-delivery and document-handover system built on a **witnessed Merkle transparency log** be a defensible COMP4299 system-development project, and what would have to be true for it to be one?

### 1.3 Short answer

**Conditionally yes.** The proposal is defensible if, and only if:

1. the supervisor's "blockchain" means a hash chain or a transparency log, not a consensus ledger (§10, G1);
2. a stakeholder with a real **document-handover or delivery workflow** can be named and reached this term (§10, G2);
3. the contribution is the **handover profile, offline time/revocation, and evaluation** built on the existing log/witness stack, not the stack itself (§7; ANL-002);
4. the project pre-registers its falsification tests and accepts a negative result (§8).

If any of (1)–(3) fails, the honest move is to stop or reframe before building (§8.6).

### 1.4 What this document does not claim

- It does not claim novelty. COMP4299 requires a system, not a novel result.
- It does not claim that a blockchain is necessary. The opposite is the hypothesis under test.
- It does not claim that a signature proves that a handover occurred. It proves an assertion; the physical event, and the truth of the document's content, are oracle problems (§3.4).
- It does not claim legal non-repudiation. That requires a qualified trust service or a contractual acceptance (§12).
- It does not present planning estimates as measurements. Every number is labelled.

---

## 2. Post-mortem: what went wrong last time

The previous proposal (a server-side geometric cut with exact aggregation) did not fail on engineering. It failed because the requirement that justified it (K3) was self-set and unanchored, no stakeholder imposed it, and the analysis showed no unconditional advantage. The project discovered this after the analysis, not before the build.

### 2.1 Failure modes and countermeasures

| Failure mode (previous) | Countermeasure (this proposal) |
| --- | --- |
| A self-set requirement justified the mechanism | Every requirement is anchored or explicitly labelled **assumed**. The baseline includes "no requirement". |
| No stakeholder | Stakeholder is a **gate**, not an assumption (§10, G2). |
| Mechanism first (the cut), problem second | Problem first (trust in handover records); the log is the **lowest-cost candidate**, not the premise. |
| Unmeasured constants presented as results | Measurement plan and labels from the start (§8). |
| A negative result would kill the project | The comparison is the contribution; a negative result is publishable (§7.3). |
| Novelty chase | System-development track; novelty is not required. |
| Build started before the anchor was fixed | Gates precede build; a spike is allowed only after G1–G2. |

### 2.2 Where this proposal is weak

| Weakness | Severity | Mitigation |
| --- | --- | --- |
| Domain novelty is modest — "Certificate Transparency for parcels and documents" | Medium | The contribution is handover semantics, offline time, revocation, disputes, and the evaluation; say so plainly. |
| Witnesses will be simulated | Medium | Separate processes, separate keys, documented governance; state that it is a simulation. |
| The oracle problem is unsolved | High | Scope to attestations; never claim physical or documentary truth; evaluate what the attestation is worth. |
| Legal recognition is external | High | Position eIDAS/MLETR as requirements and future work, not as achieved compliance. |
| No stakeholder yet | High | Gate G2. If it fails, stop. |
| The supervisor may want a consensus chain | Medium | Gate G1; level 4 is the hedge. |
| The hash chain alone is thin | Medium | Build level 3 (witnesses), not level 2 alone. |
| Most of the stack is commodity | High | ANL-002 estimates 70–80% by component count (assumed, not measured): SCITT, Tessera, Sigsum, Rekor, TSAs, and PoD apps cover most of it. The RFC 9162 receipt wrapper is the exception. Do not rebuild the rest; build the profile and the evaluation. |

### 2.3 The three hardest viva questions

1. **Why not a signed database?** Because the operator can rewrite or withhold. The project demonstrates this against level 2 and measures what witnesses add.
2. **Why not a blockchain?** Because PoD parties are known and contractual; consensus and tokens add cost without removing the oracle. The project prices level 4/5 rather than assuming.
3. **What does this prove about the parcel or document?** That a named party asserted a handover at a time, and that the assertion was included in a log that the operator cannot silently rewrite. Nothing more.

If the document cannot answer these three in one sentence each, it is not ready.

---

## 3. Problem

### 3.1 The workflow

A university registry or an employer's HR function hands an official document — a transcript, diploma, employment letter, or contract — to a student or employee. In the parcel variant, a seller, a carrier, and a recipient perform the same handover with goods. In both cases, the record is the evidence that the handover occurred, what was handed over, and when. Disputes are common: non-receipt, wrong version, altered record, disputed signature, lost paperwork.

### 3.1a Primary case and demo instance

| Role | University | Employer | Parcel demo |
| --- | --- | --- | --- |
| **Sender / issuer (document) / carrier (parcel)** | Registry | HR | Carrier, acting for the consignor |
| **Recipient / issuer** | Student | Employee | Consignee |
| **Subject** | Transcript or transfer ID | Letter or contract ID | `SHP-42` |
| **Content** | Document hash and metadata | Document hash and metadata | Items and photo hashes |
| **Channel** | Portal, email, in person | Portal, email, in person | Carrier |
| **Relying party** | Student, employer, auditor | Employee, auditor | Consignor, insurer |

In the document case, the registry or HR function is the issuer. In the parcel demo, the carrier is the issuer acting for the consignor. The protocol is identical: two signed statements linked by `prev`, a log, receipts, witnesses, and a TSA. The profile's subject is a transfer/document ID or a shipment ID; the content is a document hash or item/photo hashes.

### 3.2 The trust question

The record is usually held by the institution — the registry's student system, the HR system, or a portal — or, in the parcel case, the carrier's platform. That party can, in principle:

- **rewrite** history (alter a record after the fact),
- **truncate** history (remove a record),
- **withhold** a record from a verifier,
- **equivocate** (show different histories to different verifiers).

The institution is often both the issuer and the record keeper. That is the threat model, not an edge case: the party whose assertion is in question also controls the record.

Signatures alone do not detect (1)–(4). A hash chain detects (1) only relative to a known head. Independent witnesses detect (4); withholding (3) is detected by the issuer-signed sequence, not by gossip. This is the precise gap the project addresses.

### 3.3 Current practice

Paper transcripts, diplomas, employment letters, scanned signatures, PDFs by email, student and HR portals, and photographs. Digital-diploma and verifiable-credential platforms exist, but adoption is uneven; they typically provide an issuer-signed credential, not a transparency log. Electronic consignment notes (e-CMR) and electronic bills of lading exist for goods, but adoption is uneven and the trust model is usually a platform operator. The market evidence is mixed: TradeLens, the flagship blockchain trade platform, was discontinued in 2022–23 for lack of commercial viability. That is a caution about adoption, not about the technology.

### 3.4 Scope boundary

The system proves:

- **authenticity** — a key holder signed the statement;
- **integrity** — the statement has not changed;
- **time** — a trusted authority timestamped it (subject to offline limits, §6.6);
- **inclusion** — the statement is in the log;
- **non-equivocation** — independent witnesses prevent a silent split view.

It does **not** prove that the document is genuine or complete, that the institution's database is accurate, that the recipient read it, or, in the parcel case, that the goods arrived. It does not prove that the signer inspected the goods or that the device was uncompromised. Those are oracle and endpoint problems, and they are out of scope except where a sensor attestation is explicitly added.

### 3.5 Ethics and data protection

The slice processes sender, courier, and recipient identities, signatures, document hashes, and possibly location and photo hashes. Development and evaluation use **synthetic data**; no real student, employee, courier, or customer records are used. If a usability study recruits participants, it requires consent, the minimum data needed, a stated retention period, and the right to withdraw; no photos of people or addresses are retained. The payload store is encrypted and has a retention window; the log keeps hashes. An ethics approval or exemption is a gate alongside G3 (§10, G6).

---

## 4. Requirements

Status: **A** = anchored (external source), **D** = derived from the workflow, **S** = assumed (design choice, to be swept or justified).

### 4.1 Functional

| ID | Requirement | Status | Note |
| --- | --- | --- | --- |
| R1 | Create a handover statement with a subject (document/transfer or shipment), parties, time, optional place and condition, and a content hash | D | Schema in Appendix B; document and parcel variants |
| R2 | The sender signs the statement on the device; in the parcel case, the carrier signs for the sender | D | eIDAS/e-CMR motivate a signature; device signing is a design choice |
| R3 | The recipient co-signs at handover | D | The handover is the evidentiary moment |
| R4 | Obtain a trusted timestamp | A | RFC 3161 |
| R5 | Append the statement hash to an append-only Merkle log | D | CT-style (RFC 6962/9162) |
| R6 | Return an inclusion proof (receipt) | A | CT; IETF SCITT receipts |
| R7 | Verify signature, timestamp, inclusion offline | D | Dispute bundle |
| R8 | Produce and verify consistency proofs as the log grows | A | CT |
| R9 | Independent witnesses cosign tree heads and gossip | D | Split-view detection |
| R10 | Export a dispute bundle | D | Statement + signature + timestamp + proof |
| R11 | Revoke a key and mark statements after revocation | D | Offline revocation is a hard problem (§6.6) |
| R12 | Queue statements offline and upload later | D | Offices, depots, remote sites |
| R13 | Keep personal data off the log | S | GDPR motivates minimisation; the architecture is a design choice |
| R14 | Audit: who verified what, when | D | Operator and auditor views |
| R15 | Detect operator rewrite, truncation, withholding, equivocation | D | The core claim |

### 4.2 Non-functional

| ID | Requirement | Target | Status |
| --- | --- | --- | --- |
| N1 | Signing latency on a mid-range phone | < 300 ms | S |
| N2 | Verification latency (full bundle) | < 1 s | S |
| N3 | Append throughput | ≥ 50 statements/s | S |
| N4 | Inclusion proof size | O(log n), < 2 KB at 1M entries | D |
| N5 | Cost per handover | < $0.01 excluding device | S |
| N6 | Availability | 99.5% for the log; offline clients degrade gracefully | S |
| N7 | Usability | Sender completes a handover in < 60 s | S |

Targets are design goals, not measurements. They are swept in the evaluation (§8). N3 is headroom, not an S0 requirement: 1,000 deliveries/day is about 0.012 statements/s. N4's 1M-entry figure is **derived** (about 20 hash values × 32 B plus the envelope, under 2 KB), not measured; the evaluation measures 1k/10k/100k.

### 4.3 Anchors

| Anchor | Relevance | Status |
| --- | --- | --- |
| eIDAS (EU 910/2014; eIDAS 2.0, Reg. 2024/1183) | Qualified signatures, seals, timestamps, delivery | Regulation; verify current text |
| W3C Verifiable Credentials / digital diplomas | Verifiable document claims; issuer-signed credentials | Adjacent; verify editions |
| e-CMR Additional Protocol (2008) | Electronic consignment note | Protocol; verify adoption |
| UNCITRAL MLETR (2017) | Electronic transferable records | Model law; verify national adoption |
| RFC 3161 | Time-stamp protocol | Stable |
| RFC 6962 / RFC 9162 | Certificate Transparency v1/v2: Merkle log, inclusion, consistency | Stable |
| IETF SCITT | Transparent supply-chain statements and receipts | RFC 9943 (June 2026); RFC 9942 |
| GS1 EPCIS 2.0 | Supply-chain event data | Standard; verify edition |
| GDPR | Minimisation, erasure, lawful basis | Regulation |

**Caution.** Standards names and editions move. Verify each before citing; do not assume an RFC number.

---

## 5. Design space and the chosen point

### 5.1 The levels

| Level | Mechanism | Who can rewrite | Trust anchor | Cost | Role here |
| --- | --- | --- | --- | --- | --- |
| 1 | Signed records in a database | The operator | Operator + CA | Trivial | Baseline: signatures only |
| 2 | Hash chain + RFC 3161 timestamp | The operator, unless an old head is produced | Operator + TSA | Low | Baseline: the supervisor's model |
| 3 | Merkle log + independent witnesses + gossip | Only if witnesses collude | Operator + TSA + witness set | Medium | **Chosen** |
| 4 | Permissioned chain (known validators) | Validator threshold | Consortium | Medium–high | Optional comparison |
| 5 | Permissionless chain + collateral (the 2018 paper) | Economic majority | Validators + token market | High | Comparison, not built |

**Requirement-by-requirement contrast.** Appendix E sets the same options against the PoD requirements. It is qualitative, uses no weighted score, and is the table to take into the supervisor discussion.

**Status of the levels.** The five levels are an **analytical device for this comparison, not a standards taxonomy**. Each level corresponds to a real design family — signed database and portal (1), hash chain plus RFC 3161 time (2), Merkle log with receipts and witnesses (3), permissioned chain (4), permissionless chain with collateral (5) — but the numbering and the ordering are ours. The ordering follows the trust property each level adds: tamper-evidence (2), inclusion (3), non-equivocation (3), append-only consistency (3), consensus (4–5). Cite the designs, not the numbers; the designs are documented in Appendix E and ANL-002.

### 5.2 Why level 3

Level 2 is **necessary but not sufficient**. A hash chain makes tampering evident to someone who already knows the head; it does not stop the operator from recomputing the chain, truncating it, or showing different heads to different verifiers. Level 3 is the **lowest level that detects equivocation and withholding** without introducing a consensus ledger.

Level 4/5 are not justified for mainstream PoD because the parties are known, contracted, and insured; membership is not open; public verifiability is a privacy liability; and payment does not need a chain. The conditions that would move the choice up are stated in §5.3.

### 5.3 What would move the choice

| Condition | Moves to |
| --- | --- |
| Unknown or pseudonymous counterparties | 5 |
| Public verifiability by anyone, with no access grant | 5 |
| Trustless payment on delivery, no regulated intermediary | 4/5 |
| A consortium of mutually distrusting carriers needing shared ordering | 4 |
| A trusted operator plus an auditor | 2 |
| Signatures only, operator fully trusted | 1 |

### 5.4 The claim under test

> For a known-party handover workflow — a document transfer or a parcel delivery — with a trusted-but-auditable operator, level 3 is sufficient, level 2 is insufficient, and levels 4–5 are unnecessary.

This is falsifiable (§8.1). If level 2 plus an operator policy passes every red-team test, the witnesses add nothing and the claim fails.

---

## 6. Architecture

### 6.1 Components

| Component | Responsibility |
| --- | --- |
| Signing client (PWA) | Build the statement; sign with a device key; co-sign at handover; queue offline |
| Timestamp authority | RFC 3161 token over the statement hash |
| Log service | Append to a Merkle tree; return receipt and tree head |
| Witness (×2, 2-of-2) | Verify consistency; cosign the tree head; gossip with peers |
| Verifier | Check signature, timestamp, inclusion, tree-head signatures, consistency |
| Dispute service | Assemble and verify a bundle |
| Revocation service | Publish key status; short-lived keys where possible |
| Payload store | Encrypted off-chain statement body; log holds hashes only |

### 6.1a Actors, parties, and roles

The system has three layers that are often conflated: the **physical workflow** (who does what), the **legal parties** (who is liable), and the **key holders** (whose device signs). They are not the same entity. In the primary case, the registry or HR function is responsible for issuing the document; a clerk's device holds the key; the student or employee receives. In the parcel case, the carrier is liable; the courier's phone holds the key; the consignee receives; a receptionist may sign on its behalf.

#### Workflow parties

| Party | Also called | Legal role | Key holder | In the slice |
| --- | --- | --- | --- | --- |
| Sender / issuer (document case) | Registry, HR | Hands over the document; issuer | Registry/HR key | Issuer |
| Carrier / issuer (parcel demo) | Transport company | Moves goods; acts for the consignor; issuer | Courier | Issuer |
| Recipient / issuer | Student, employee, consignee | Receives the document or goods | Recipient | Issuer |
| Relying party | Consignor, student, employer, insurer, auditor | Uses the evidence | None | Verifier |

#### Trust parties

| Party | SCITT role | Signs | Trust anchor | In the slice |
| --- | --- | --- | --- | --- |
| Log operator | Transparency Service | Receipts, checkpoints | Checkpoint key | Operated |
| Witness A, B | Checkpoint cosigner | Checkpoints | Two witness keys | Operated, independent |
| TSA | Time authority | RFC 3161 tokens | TSA key | External |
| CA / IdP | Identity | Certificates, enrolment | CA root | External or development |
| Insurer / auditor | Relying Party | Nothing | None | External, verifier only |

#### Counts

| Count | Number | Who |
| --- | --- | --- |
| Workflow parties | 3–4 roles | Sender or carrier (issuer), recipient (issuer), and relying party; the carrier is an optional fourth role in the parcel demo |
| Trust-path parties | 7 | The two issuers, the log, witness A, witness B, the TSA, and the CA |
| Signing keys | 7 | Sender or carrier, recipient, log, witness A, witness B, TSA, CA |
| Verifier trust anchors | 5 | Log checkpoint key, two witness keys, TSA key, CA root |
| Distinct organisations, all separate | 7–9 | The two issuers, the log, the two witnesses, the TSA, and the CA; add a relying party, a carrier, or a separate log operator as they are separated |
| Project-operated | 5 | Issuer A (sender or carrier), recipient, log, witness A, witness B |

The insurer, auditor, student, or employer is a **relying party**, not a witness. It verifies the dispute bundle and never cosigns a checkpoint. Keeping the two roles separate protects the independence of the witness set.

**Two issuers, two statements.** The sender issues the handover statement; the recipient issues the receipt statement about the same subject. In the parcel demo, the carrier acts for the sender and issues the delivery statement; the consignor is a relying party. They are linked by subject, not by a multi-signature envelope. Adding a third issuer later is a third statement linked by subject.

**What the count means.** Trust depends on independence, not headcount. Two witnesses run by the same operator are one party; a log run by the institution or carrier is the adversary, so the witnesses must be operationally independent. The dissertation should state the trust path, the operated set, and the independence assumptions, not just the total.

**Not parties.** The transfer, document, or shipment is the subject; the verifier is a role that the sender, recipient, employer, or auditor can fill; the software components in §6.1 are not parties.

### 6.2 Protocol sequence

1. The sender creates the handover statement, signs it, and records optional condition and location. In the parcel demo, the carrier does this for the sender.
2. The recipient verifies the statement and issues its own receipt statement about the same subject; its `prev` field is bound to the sender's signed statement.
3. The client sends the statement hashes to the TSA, then submits the signed statements to the log.
4. The log verifies, appends, and returns a receipt (inclusion proof) and a checkpoint.
5. Witnesses check consistency against the previous checkpoint and cosign.
6. The verifier checks the bundle; both issuers keep a copy.
7. On dispute, the bundle is exported and verified independently.
8. On key compromise, revocation is published and statements are checked at the TSA time.

### 6.2a Worked scenario — SHP-42 (parcel demo)

#### Demo instance and primary-case mapping

This worked scenario is the **parcel demo**, not the primary case. It is the primary case with a carrier in the middle and a physical item to photograph. Read it as follows:

| Demo | Primary case |
| --- | --- |
| `Acme` (consignor) | Registry or HR — the sender institution; in the parcel demo it relies on the carrier's statement rather than issuing one |
| `Dana` / Northline (carrier, issuer) | The registry or HR clerk's signing key; the carrier is the sender's agent in the parcel variant |
| `Sam` / BioLab (consignee, issuer) | Student or employee — the recipient |
| `SHP-42` | Transfer or document ID, e.g. `TR-2026-0042` |
| Items and photo hashes | Document hash and metadata: media type, size, capture time |
| Place and condition | Optional; a document transfer may have neither |

Acme is the sender institution in both cases. In the parcel demo it does not issue, so it verifies the carrier's statement; in the primary case the registry or HR function issues directly, and the external relying party is the student, employer, or auditor. The protocol is identical: two signed statements linked by `prev`, a log, receipts, witnesses, and a TSA. The payload changes, not the envelope.

#### Cast

| Actor | Party | Key | Role |
| --- | --- | --- | --- |
| Dana | Northline Logistics (carrier) | Courier device key | Issuer: delivery statement |
| Sam | BioLab (consignee) | Recipient device key | Issuer: receipt statement |
| Acme Instruments | Consignor | None | Relying party in the demo; sender institution in the primary case |
| ClearLog | Independent log operator | Checkpoint key | Transparency Service |
| UniAudit, Aegis Audit | Two witnesses | Witness keys | Cosign checkpoints |
| InsureCo | Insurer | None | Relying party: verifies the dispute bundle |
| FreeTSA | Timestamp authority | TSA key | Trusted time |
| UniCA | Certification authority | CA root | Binds keys to parties |

Shipment `SHP-42`: one calibrated spectrometer, Acme to BioLab.

#### Phase 0 — Enrolment

UniCA issues device certificates binding Dana's and Sam's public keys to their parties and roles. ClearLog generates its checkpoint key; the two witnesses generate theirs. Acme configures five trust anchors: ClearLog's checkpoint key, the two witness keys, FreeTSA's key, and UniCA's root. ClearLog publishes its registration policy: accepted issuers (`carrier:northline:*`, `consignee:biolab:*`), accepted subjects (`shipment:*`), accepted algorithms, maximum statement size, and the requirement that receipt statements carry `prev`. In the document case, the policy would accept `university:registry:*` and `student:*` (or `employer:hr:*` and `employee:*`), with `transfer:*` as the subject, and the student, employer, or auditor would hold the anchors rather than the registry.

#### Phase 1 — Dispatch

Acme creates `SHP-42` and sends the manifest to Northline. No statement is issued in this slice. The shipment ID is the subject that links every later statement. If Acme were an issuer, it would sign a dispatch statement here. In the primary document case, the registry or HR function would sign this dispatch statement; the student or employee would issue the receipt. The parcel demo omits the dispatch statement because the carrier issues the delivery statement instead.

#### Phase 2 — Delivery statement (courier)

Dana opens the PWA, selects `SHP-42`, records condition, captures a photo, and records location and device time. The app builds `S_c` in canonical CBOR:

```
{
  statement_id: "st-001",
  sequence:     1,
  shipment_id:  "SHP-42",
  issuer:       "carrier:northline:courier:dana",
  subject:      "shipment:SHP-42",
  event:        "delivery",
  time:         "2026-10-21T14:02:10Z",
  place:        { lat: ..., lon: ..., accuracy_m: 8 },
  items:        [ { sku: "SPEC-1", qty: 1, condition: "sealed" } ],
  photos:       [ "sha256:..." ],
  prev:         null,
  nonce:        "..."
}
```

Dana signs it:

```
COSE_Sign1 = 18([
  protected:   { 1: -7, 4: dana_kid,
                 15: { 1: "carrier:northline:courier:dana",
                       2: "shipment:SHP-42" } },
  unprotected: {},
  payload:     S_c,
  signature:   Sign(dana_priv, ["Signature1", protected, h'', S_c])
])
```

The signed statement is stored locally and queued if offline. It proves authorship and integrity of the assertion. It does not prove the goods arrived intact.

#### Phase 3 — Receipt statement (recipient)

Sam inspects the pallet, confirms the item and condition, and captures a photo. Sam's app obtains Dana's signed statement by QR code, NFC, or link; verifies Dana's signature offline; and computes `prev = SHA256(canonical COSE_Sign1 bytes of S_c)`. It then builds `S_r`:

```
{
  statement_id: "st-002",
  sequence:     1,
  shipment_id:  "SHP-42",
  issuer:       "consignee:biolab:recipient:sam",
  subject:      "shipment:SHP-42",
  event:        "receipt",
  time:         "2026-10-21T14:05:30Z",
  items:        [ { sku: "SPEC-1", qty: 1, condition: "sealed" } ],
  photos:       [ "sha256:..." ],
  prev:         "sha256(canonical COSE_Sign1 bytes of S_c)",
  nonce:        "..."
}
```

Sam signs it with the recipient key. The `prev` field is what turns two statements into one handover; without it, the log holds two unrelated facts about the same shipment.

#### Phase 4 — Trusted time

Each app sends only `SHA256(signed_statement)` to FreeTSA. The TSA returns an RFC 3161 token whose `messageImprint` is that hash and whose `genTime` is the TSA's clock. The token is stored beside the statement, not inside it: the token is over the statement, so it cannot be part of what is hashed.

The token proves the signed statement existed at the TSA time. It does not prove the handover happened at the device time. If Dana is offline, she signs at 14:02 and timestamps at 16:40; the token then proves existence at 16:40. Sam's co-signature at 14:05 is the second attestation of handover time; the verifier must know which time is which.

#### Phase 5 — Registration and receipts

The app submits both signed statements to ClearLog. ClearLog checks the COSE signature and certificate chain, the presence and policy match of `iss` and `sub`, that the protected header's `iss`/`sub` equal the body's `issuer`/`subject`, the issuer's authority for that subject, the algorithm, the statement size, the `prev` requirement for receipts, and the continuity of the issuer's signed `sequence`. It computes the leaves:

```
L_c = SHA256(0x00 || signed_S_c)
L_r = SHA256(0x00 || signed_S_r)
```

It appends both, computes the root at the new size `n`, and builds an inclusion proof for each leaf. It issues a receipt per statement:

```
18([
  protected:   { 1: -7, 395: 1 },
  unprotected: { 396: { -1: [ <<[n, i_c, path_c]>> ] } },
  payload:     nil,          // detached; the root is computed
  signature:   Sign(log_priv, ["Signature1", protected, h'', root])
])
```

In the CDDL, each proof is a `bstr` containing `[tree_size, leaf_index, inclusion_path]`; the `<<...>>` above is that wrapped CBOR. A consistency receipt has the same envelope with `vdp: { -2: [ <<[tree_size_1, tree_size_2, consistency_path]>> ] }`. The examples are schematic; the CDDL in RFC 9942 is authoritative.

The app now holds a transparent statement: the signed statement plus its receipt. ClearLog needs the signed statements to verify them; it does not need the photo bytes, and the policy may discard the payload after verification.

#### Phase 6 — Checkpoint and witnesses

ClearLog produces `CP = { origin: "clearlog", size: n, root }`, signs it, and sends it with a consistency proof from the previous checkpoint to UniAudit and Aegis Audit. Each witness verifies consistency against the last checkpoint it cosigned and returns a cosignature. The witnesses gossip the cosigned checkpoint.

The receipt proves inclusion at size `n`; the checkpoint proves `root` is the log's head at size `n`; the witness signatures prove independent parties saw the same head. Without witnesses, the log can show different heads to different verifiers.

If the checkpoint is at size `m > n`, the verifier needs a consistency proof from `n` to `m`, or a fresh receipt at size `m`. The slice issues receipts and checkpoints at the same size to avoid the gap; the evaluation tests the gap explicitly.

#### Phase 7 — Verification at handover

Sam's app runs eight checks in order:

1. Dana's signature over `S_c`, using Dana's certificate.
2. `L_c` from the signed statement; apply `path_c`; recompute `root`.
3. ClearLog's receipt signature over `root`.
4. `root` equals the checkpoint root at size `n`.
5. The checkpoint carries two valid witness signatures.
6. Sam's own signature over `S_r`; `prev == SHA256(signed_S_c)`.
7. `L_r`; apply `path_r`; same `root`; verify the receipt.
8. TSA tokens match the hashes and fall inside the handover window.

A failure names the check that failed, not "invalid."

#### Phase 8 — Verification by the consignor

Acme's portal fetches the transparent statements for `SHP-42` and runs the same checks, then applies its policy: both statements present and sharing the subject; `prev` matches; issuers are the expected carrier and consignee; TSA times are inside the window; the witness quorum is met; condition fields agree or the discrepancy is flagged. Acme trusts the witnesses and the cryptography, not ClearLog's honesty; it needs ClearLog only for availability.

#### Phase 9 — Dispute

The insurer receives a bundle: both signed statements, both receipts, both TSA tokens, the checkpoint with two witness signatures, and the CA certificates. It verifies offline, with no access to ClearLog. The bundle establishes who asserted what, when, and that neither statement changed since signing. It does not establish who damaged the pallet. The system preserves evidence; the humans and the insurer adjudicate.

#### Phase 10 — Revocation

If Dana's phone is stolen at 16:00 and UniCA revokes her certificate at 16:30, the verifier checks revocation at the TSA time, not the current time. A token **issued before revocation** proves the signed statement existed before revocation; it does not prove it was **signed** before revocation. Statements signed in the window between compromise and revocation are therefore indeterminate: they may be innocent (queued offline) or forged (stolen key). A statement signed before compromise but timestamped after revocation is also indeterminate for the same reason: the token proves existence at the later time, not the earlier signing. The design states this rather than hiding it; short-lived keys or witness freshness (§6.6) narrow the window.

#### Phase 11 — Audit

The auditor replays ClearLog from its checkpoints, verifies consistency and a sample of inclusion proofs, and checks each issuer's **signed** `sequence`. The sequence is inside the statement, so the leaf commits to it: a missing entry leaves a gap in that issuer's sequence, and a verifier can detect the gap even though it cannot know what the missing statement said. A log that drops an entry and renumbers log metadata cannot repair the issuer's signed sequence. Completeness therefore rests on the issuer-signed sequence plus gossip, not on log metadata; a deliberately skipped sequence is indistinguishable from a lost statement, which is why the auditor also compares against the issuer's published submission record when one exists.

#### Who sees what

| Actor | Sees | Does not see |
| --- | --- | --- |
| Dana | Her statement, receipt, checkpoint | Sam's private key |
| Sam | Both statements, both receipts, checkpoint | Dana's private key |
| ClearLog | Signed statements (to verify), hashes | Photo bytes if the payload is off-chain |
| FreeTSA | Hashes only | The statements |
| Witnesses | Checkpoints and consistency proofs | The statements |
| Acme / insurer | The dispute bundle | Private keys |
| Auditor | Log and checkpoints | Off-chain payloads unless granted |

#### Why the order is sign → timestamp → register → checkpoint

The issuer signature is independent of the log; the TSA token is independent of the log; the receipt is independent of the TSA. Each covers a different claim: authorship, time, inclusion, and non-equivocation. Reordering them changes what is proved. In particular, a timestamp obtained after reconnect proves existence at reconnect, not at handover; the design must say so rather than hide it.

### 6.3 Data model (log entry)

The log stores only:

```
{ statement_hash, issuer_key_id, log_index }
```

`log_index` is the leaf's position in the tree. The issuer's `sequence` and the handover `prev` are **inside the signed statement**, so the leaf commits to them; they are not log metadata. A log that renumbers `log_index` cannot change the issuer's signed sequence.

The statement body (names, addresses, items, condition, photos) stays off-chain, encrypted. This keeps the log free of personal data and limits linkage.

### 6.4 Cryptography

| Choice | Candidate | Note |
| --- | --- | --- |
| Signature | Ed25519 (RFC 8032) or ECDSA P-256 (FIPS 186-5) | Device support decides |
| Hash | SHA-256 (FIPS 180-4) | Merkle tree and statement hash |
| Encoding | CBOR (RFC 8949) + COSE (RFC 9052/9053) or JWS (RFC 7515) | COSE is the SCITT/CT lineage |
| Merkle tree | RFC 6962-style | Inclusion and consistency proofs |
| Timestamp | RFC 3161 | TSA trust anchor |

### 6.5 Trust anchors (count them)

| Anchor | What it can do | Mitigation |
| --- | --- | --- |
| Log operator | Withhold, truncate, equivocate | Witnesses + gossip |
| Witnesses | Cosign a bad head; collude | Independence; 2-of-3 on revisit |
| TSA | Backdate or mis-time | Reputable TSA; multiple tokens |
| CA / key issuer | Issue a key to the wrong party | eIDAS/QTSP or contractual identity |
| Device | Sign under malware; key theft | Secure element; revocation; biometrics |
| Payload store | Lose or alter the body | Hash binding; replication |

The claim is **detectable misbehaviour**, not trustlessness. The project should say this explicitly.

### 6.6 Offline and time (the hard problem)

A sender's phone or a clerk's desktop in a basement or a remote office cannot obtain a trusted timestamp. Options:

1. **Local TSA** — a timestamp service at the office or depot; trusted by contract.
2. **Recipient co-signature time** — the handover time is attested by two parties, not a TSA.
3. **Timestamp on reconnect** — proves upload time, not handover time; weaker.
4. **Secure time source** — GNSS or a trusted gateway; hardware-dependent.

The project must choose one and evaluate it. This is the most interesting engineering decision in the system, and it should be a named contribution, not a footnote.

### 6.7 Privacy and erasure

- The log holds hashes, not personal data.
- The payload is encrypted and deletable; the hash remains as proof of existence.
- Selective disclosure: reveal the minimum fields needed for a dispute.
- Residual risk: timing and linkage can still reveal a handover; state it.

### 6.8 Failure modes

| Failure | Detection | Recovery |
| --- | --- | --- |
| Lost device key | Revocation; missing statements | Re-issue key; re-sign with witness |
| Log fork | Witness refuses to cosign; gossip | Freeze; publish evidence; re-anchor |
| Witness collusion | Independence analysis; audit | 2-of-3 on revisit; disclose |
| TSA compromise | Multiple tokens; audit | Re-timestamp; flag affected window |
| Clock skew | Co-signature mismatch | Use co-signature time; flag |
| Replay | Nonce + sequence | Reject duplicates |
| Payload loss | Hash mismatch | Recover from replica |

---

## 7. Commodity, contribution, and the minimum result

### 7.1 Commodity (do not rebuild)

ANL-002 decomposes the system and estimates that 70–80% of the code is already covered (assumed, by component count; the RFC 9162 receipt wrapper is custom work):

| Area | Use | Do not build |
| --- | --- | --- |
| Merkle log, inclusion/consistency proofs | Tessera, Trillian, Rekor | A new log |
| Receipts and offline verification | SCITT (RFC 9943/9942), `scitt-verifier`, `scitt-ccf-ledger` | A new receipt format |
| Witnesses | Sigsum, `transparency-dev/witness` | A new witness protocol |
| Trusted time (online) | RFC 3161 TSA (FreeTSA, DigiCert, `uts-server`) | A TSA |
| Signatures and encoding | WebCrypto, COSE, CBOR | Crypto primitives |
| Capture UX | PWA, existing SDKs | A native app |

The architecture is standardized: **RFC 9943** (SCITT, June 2026) defines signed statements, a transparency service, receipts, non-equivocation, and replayability. A handover or PoD system is an **application profile** of that architecture, not a new one.

The exception is the RFC 9162 proof encoding: `scitt-verifier` supports CCF receipts only, so the RFC 9942 wrapper for RFC 9162 leaves is custom work (25 h in §9.2), not reuse.

### 7.2 Contribution

1. **Handover profile** — statement schema for documents and parcels, handover state machine, co-signature, multi-leg, condition, dispute fields, mapped to SCITT (C1). The primary case is document handover; the parcel demo exercises the same profile with items and photos.
2. **Offline trusted time** — the §6.6 decision and its evaluation (C2).
3. **Offline revocation** — short-lived keys, witness freshness, re-signing after reconnect (C3).
4. **Witness governance** — who they are, what collusion breaks, how the set is audited (C4).
5. **Privacy/linkage** — what a hash-only log still reveals over time (C5).
6. **The evaluation** — level 1 vs 2 vs 3 (vs 4/5) on the same workload, with trust-anchor counts and red-team results (C6).

The contribution is the profile and the evaluation, not the log. If the profile reduces to a schema, the project is a reproduction.

### 7.3 The minimum publishable result

Even if the system underperforms, the project yields:

- a threat model for handover trust anchors;
- a requirements-to-level mapping;
- measured cost, latency, and proof sizes;
- evidence on whether witnesses are worth their cost;
- a negative or positive answer to "is a chain needed?"

That is a valid system-development dissertation. It does not depend on the system winning.

---

## 8. Evaluation

### 8.1 Hypotheses and falsification

| ID | Hypothesis | Falsified if |
| --- | --- | --- |
| H1 | Level 3 detects equivocation that level 2 misses; its receipts make each issuer's signed sequence third-party visible and enumerable, so a verifier can detect a gap, while at level 2 only the submitter can | Level 2 + operator policy detects every red-team case, including equivocation |
| H2 | Level 3 stays within 2× of the latency and cost targets | p99 verification > 2 s, or signing > 600 ms, or cost > $0.02/handover |
| H3 | A single compromised witness cannot produce an undetected fork; the 2-of-2 quorum is the binding assumption for non-equivocation | One compromised witness produces an undetected fork, or a TSA/CA compromise alone produces equivocation |
| H4 | Offline time is the weakest link | Offline cases verify as strongly as online ones |
| H5 | Level 5 adds no benefit for this threat model | A red-team case requires consensus |
| H6 | Senders and recipients complete the flow within 120 s and under 20% task failure | Task time > 120 s, or ≥ 20% task failure |

**This table is canonical.** TUT-001 §11 cites these IDs and adds measures; it does not define a second set.

### 8.2 Method

- **Testbed:** one log service, two witnesses on separate processes (separate hosts if available), a TSA (or a local RFC 3161 service), a PWA, a verifier, a payload store.
- **Workload:** 1,000 handovers/day for 30 simulated days, 10 senders, 3 offices or depots, 10% offline, 1% disputes.
- **Baselines:** level 1 (signed DB), level 2 (hash chain + TSA), level 5 (the paper's contracts, for cost comparison only).
- **Protocol:** the same statement set through each arm; pre-register the metrics and the oracle.

### 8.3 Metrics

| Category | Metric | Method |
| --- | --- | --- |
| Correctness | Inclusion/consistency proofs verify; tamper detected | Unit + property tests |
| Security | Red-team cases detected/undetected | Scripted attacks |
| Performance | Sign, append, prove, verify latency; throughput; proof size | Instrumented runs |
| Cost | $/handover; storage; bandwidth | Derived from measured usage |
| Usability | Task completion, time, errors | Small study (n = 8) |
| Trust | Number of anchors an attacker must compromise | Analysis |

### 8.4 Red-team suite

1. Operator rewrites a record after the fact.
2. Operator truncates the log.
3. Operator withholds a record from one verifier.
4. Operator shows two different heads (equivocation).
5. Sender or courier denies signing (key repudiation).
6. Stolen device signs a false statement.
7. Replay of an old statement.
8. Clock skew / offline timestamp forgery.
9. Witness collusion (threshold).
10. TSA compromise / backdating.

**This is the canonical list** (ten attacks). TUT-001 Module 6 cites it and must not add a second count.

### 8.5 Comparison protocol

Run the same 30-day workload through levels 1, 2, 3 and a cost model for 5. Report per-handover cost, p50/p99 latency, proof size, detection rate, and trust-anchor count. No weighted score; name the trades.

### 8.6 Kill criteria

These are the domain-side criteria; ANL-002 §9 adds the supply-chain criteria. Stop, or extend by formal approval (G1), if any of these holds at the gate:

1. No stakeholder or workflow can be named by the gate date (G2).
2. The supervisor requires a consensus ledger and will not accept level 3 (G1).
3. The deployment legally requires a qualified trust service, making a self-hosted log non-compliant.
4. Level 2 + an operator policy passes every red-team case, so witnesses add nothing.
5. The build exceeds the term after the spike, with no cut that preserves the claim.
6. The chosen domain has no disputes, so the threat model is contrived.

---

## 9. Feasibility

### 9.1 MVP scope

One tenant; two roles (sender, recipient); a PWA; two simulated witnesses (2-of-2); one depot or registry; no payment; no IoT sensors (optional single sensor attestation); self-signed development certificates; one log service; one verifier. In the parcel demo, the carrier acts for the sender; in the document case, the registry or HR function is the sender.

### 9.1a The E2E slice

The build is a **vertical slice**, not a product: one document or parcel handover from capture to independent verification, one log, one receipt type, one dispute bundle. Every component is either reused or supports a named claim; anything that does neither is cut.

The demo instance is the SHP-42 parcel. The document variant replaces `items` and `photos` with a document hash and metadata, and makes `place` and `condition` optional; the envelope, log, receipts, witnesses, and verifier are unchanged.

**Issuer scope: two issuers.** The sender issues the handover statement; the recipient issues the receipt statement about the same subject. In the parcel demo, the carrier acts for the sender and issues the delivery statement; the consignor is a relying party. They are two `COSE_Sign1` statements linked by subject, not two signatures on one envelope. Adding a third issuer later is a third statement linked by subject, not a change to the envelope.

| Component | Disposition | Claim it supports |
| --- | --- | --- |
| Sender PWA: statement, content hash, signature | Build thin | Usability, capture |
| Recipient co-sign | Build thin (second tab or device) | Handover assertion |
| Signing | Reuse WebCrypto / COSE library | Authenticity |
| Trusted time | Reuse RFC 3161 TSA | Time |
| Log and receipts | Reuse Tessera; build the RFC 9162 receipt wrapper | Inclusion, non-equivocation |
| Verifier | Build thin (CLI or web) | Offline verification |
| Dispute bundle | Build | Accountability |
| Offline queue | Build (IndexedDB) | Offline time |
| Revocation | Build minimal (key status list) | Key compromise |
| Audit list | Build minimal | Demo only |
| Auth / SSO | **Cut** — development accounts | None |
| Native apps | **Cut** — PWA | None |
| Maps, routing, notifications | **Cut** | None |
| Multi-tenant, billing, HA | **Cut** | None |

**Milestones.**

| Milestone | Outcome | Target week |
| --- | --- | --- |
| M0 | Statement + signature + verify (script) | 1 |
| M1 | Merkle tree + inclusion and consistency proofs (script) | 2–3 |
| M2 | RFC 9162 receipt in COSE (RFC 9942) + verify | 4–5 |
| M3 | Tessera-backed TS + checkpoint + witnesses | 6–8 |
| M4 | PWA capture + recipient co-sign + receipt | 9–10 |
| M5 | Offline queue + TSA + dispute bundle + revocation | 11–12 |
| M6 | Red-team + evaluation | 13–14 |
| M7 | Write-up and viva | 15–16 |

**Pace: 18 h/week over 16 weeks.** That is 288 h against a 290 h plan; the two-hour difference is inside the estimate's error. If the deadline is shorter, apply the cut ladder in §9.4; if the sustainable pace is below 18 h/week, plan for 18–20 weeks rather than compressing.

**The walking skeleton is M0–M3.** The UI comes after the crypto works; otherwise the budget goes into screens and the receipt format is discovered late. **Writing starts at week 2 (M1), not at M7.** From week 2 onward, draft the chapter that matches the milestone (problem, threat model, requirements, design); the final two weeks are for results, discussion, and revision.

### 9.2 Work breakdown (planning estimates, not measurements)

| Work package | Hours |
| --- | --- |
| Requirements, threat model, standards review | 20 |
| Handover profile, statement schema, signing client | 35 |
| Integrate log and build the RFC 9162 receipt wrapper (Tessera/SCITT) | 25 |
| Witness integration and governance | 15 |
| Verifier and dispute bundle | 15 |
| Offline time and revocation | 45 |
| Privacy layer and payload store | 5 |
| Test harness and red-team suite | 30 |
| Evaluation execution | 30 |
| Artifact documentation (README, conformance, red-team, benchmark) | 15 |
| Dissertation writing | 55 |
| **Total** | **290** |

The total is lower than the first draft on the build side because the log, receipts, and verifier are reused (ANL-002). It is higher on the writing side because the earlier 50 h line conflated evaluation execution, artifact documentation, and the dissertation. The 55 h dissertation line is the middle of the 50–60 h range; the privacy line is 5 h because the privacy analysis is the first cut-ladder item.

**The receipt wrapper is custom work.** ANL-002 labels receipt verification "partial", not commodity: `scitt-verifier` supports CCF receipts only, and this slice uses RFC 9162 receipts. The 25 h line includes writing and testing the RFC 9942 wrapper; it is not reuse.

**Learning is inside the build lines, not additional.** The ~82 h of module learning (TUT-001 §3) is absorbed into the work packages: Modules 0–1 into the profile/schema/client and test harness; Module 2 into the receipt wrapper and verifier; Module 3 into the policy and verifier; Module 4 into witness integration; Module 5 into the offline queue and PWA; Module 6 into the red-team; Module 7 into the evaluation. The lines are sized for a newcomer; the 290 h is not a second budget on top of the learning.

**Writing overlap.** Artifact documentation is written with the code. The dissertation chapters that do not depend on results — problem, background, requirements, architecture, threat model, design decisions — are drafted from **week 2 (M1)** onward. Only results, discussion, and revision are left for the final weeks.

At 18 h/week, the 290 h plan is 16 weeks (288 h plus rounding). The total sits at the upper end of a one-term project. The cut-down ladder is §9.4.

### 9.3 Risks

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Supervisor expects a chain | Medium | High | G1; level 4 hedge |
| No stakeholder | High | High | G2; use a university registry, HR, or estates partner |
| Crypto implementation errors | Medium | High | Use libraries; test vectors; no custom primitives |
| Scope creep (payments, IoT, mobile apps) | High | Medium | MVP list; ladder |
| Legal overclaim | Medium | Medium | Non-claims; contractual framing |
| Witness simulation criticised | High | Medium | Separate processes and keys; state that independence is simulated on one host |
| Ethics or consent not approved | Medium | High | Synthetic data; G6 before any user study; retention and withdrawal stated |

### 9.4 Cut-down ladder

1. Drop selective disclosure and the privacy study.
2. Drop the sensor attestation.
3. Drop the Fabric/level-4 arm; keep the cost model.
4. Drop the usability study; keep a heuristic walkthrough. If cut, H6 is reported as **untested**, not confirmed.
5. Reduce the workload to 7 simulated days; keep the red-team suite.

Do not cut the witness protocol or the red-team suite; they carry the claim.

---

## 10. Gates

| Gate | Question | Decision needed | If it fails |
| --- | --- | --- | --- |
| G1 — terminology | Does "blockchain" mean level 1, 2, 3, or 4? | Supervisor confirms the target level | If level 4/5 is required, this is a kill or a 4-week extension; do not absorb a consensus arm silently |
| G2 — stakeholder | Which workflow, and who owns it? | A named person or unit, with access to a document-handover or delivery process | Stop; do not build |
| G3 — module | COMP4299 or CSAI4299? | Programme confirm | If CSAI4299 is the only option, this proposal does not fit; do not force it |
| G4 — deliverables | Dissertation only, or demo + benchmarks? | Supervisor confirms | Adjust §8 and §9 |
| G5 — spike | Does a 2-week spike hit the proof-of-concept? | Go/no-go after G1–G2 | Reframe or stop |
| G6 — ethics | Is synthetic data sufficient, or is a user study planned? | Ethics approval or exemption before any study | Use synthetic data only; drop the usability study and report H6 as untested |

**Recommended sequence:** G1 and G2 first; a two-week spike (log + witnesses + one red-team case) before the hardened build; then G5. G6 applies before any user study, not before the build.

---

## 11. Related work and standards

| Work | Relevance | Status |
| --- | --- | --- |
| Hasan & Salah, *Blockchain-Based Solution for Proof of Delivery of Physical Assets*, ICBC 2018 | Level-5 comparison: permissionless Ethereum, collateral, arbitration | Verified (DOI `10.1007/978-3-319-94478-4_10`) |
| Hasan & Salah, *…With Single and Multiple Transporters*, IEEE Access 6:46781–46793, 2018 | Extended journal version; open access | Verified (DOI `10.1109/ACCESS.2018.2866512`) |
| RFC 6962 / RFC 9162 | Certificate Transparency: Merkle log, inclusion, consistency | Standard |
| **RFC 9943** (SCITT) | Transparency services, signed statements, receipts, non-equivocation | Standards Track, June 2026 |
| **RFC 9942** | COSE receipts; inclusion and consistency proofs | Referenced by RFC 9943 |
| Tessera, Trillian | Tile-based Merkle logs; the log layer | Current |
| Sigsum, transparency-dev/witness | Witness cosigning and quorum policies | Current |
| Rekor v2 | Transparency log; RFC 3161 time + checkpoint | Current |
| `scitt-verifier`, `scitt-ccf-ledger` | Offline SCITT verification; CCF transparency service | Current |
| Guardtime KSI, Azure Confidential Ledger | Commercial/managed integrity and ledger services | Vendor claims |
| eIDAS, e-CMR, MLETR | Legal and transport anchors | Verify current text/adoption |
| W3C Verifiable Credentials / digital diplomas | Verifiable document claims; issuer-signed credentials | Adjacent: typically no transparency log or witness; could supply the identity or payload layer |
| TransFollow, CargoX, WaveBL | e-CMR and eBL platforms | Vendor claims |
| GS1 EPCIS 2.0 / OpenEPCIS | Supply-chain event vocabulary and event hashing | Standard/repo |
| FaxSeal | Adjacent: signed delivery attestation + transparency log | Vendor claim |
| TradeLens shutdown (Maersk/IBM, 2022–23) | Market caution for blockchain logistics | Verified |

The 2018 paper is the **treatment** (level 5), not the blueprint. The project builds level 3 and prices level 5; it does not reimplement the paper.

---

## 12. Non-claims and cautions

- **Not a blockchain.** It is a witnessed Merkle transparency log. Say so; define terms in the dissertation.
- **Not physical or documentary truth.** A signature proves an assertion, not a handover, and not that a transcript or letter is genuine or complete.
- **Not a substitute for the institution's official record.** The log is evidence around the document; the registry or HR system remains the authoritative source.
- **Not legal non-repudiation.** Without a QTSP or a contractual trust anchor, a signer can claim key theft.
- **Not novel.** The contribution is the domain integration and the evaluation.
- **Not production-grade PKI.** Development certificates and simulated witnesses must be declared.
- **Not privacy-preserving by default.** Hash-only logging reduces exposure; it does not eliminate linkage.
- **Do not present planning estimates as measurements.** Label every number.
- **Do not let the build start before G1 and G2.**

---

## 13. Summary

The proposal is a **document-handover and proof-of-delivery** system built on a witnessed Merkle transparency log (level 3). Its primary case is a university registry or an employer's HR function issuing an official document — a transcript, diploma, employment letter, or contract — to a student or employee, where the institution is both the issuer and the record keeper. The parcel scenario is the demo instance. The system is evaluated against a plain hash chain (level 2), a signed database (level 1), and the 2018 permissionless-chain paper (level 5). The claim is narrow and falsifiable: for known-party handovers with a trusted-but-auditable operator, level 3 detects equivocation that level 2 cannot, the issuer-signed sequence makes withholding detectable, and levels 4–5 are unnecessary.

ANL-002 changes the build case. An estimated 70–80% of the stack is already covered (assumed, by component count) — RFC 9943 standardizes the architecture, and Tessera, Trillian, Sigsum, Rekor, and RFC 3161 TSAs supply the log, witnesses, receipts, and time. The exception is the RFC 9162 receipt wrapper: `scitt-verifier` is CCF-only, so that path is custom work. The contribution is therefore the **handover profile**, **offline trusted time**, **offline revocation**, **witness governance**, **privacy/linkage**, and the **evaluation** — not a new log. A project that builds the stack is a reproduction; a project that profiles and evaluates it is defensible, but narrow.

The project is defensible if the supervisor's "blockchain" means a hash chain or a transparency log, a stakeholder with a real document-handover or delivery workflow is reachable, and the evaluation is allowed to return a negative result. The gates in §10 and the kill criteria in §8.6 exist so that the same mistake as the previous proposal — building on an unanchored requirement, or rebuilding what already ships — is caught before the build, not after.

---

## Appendix A — Threat model

| Actor | Capability | Attack | Mitigation | Residual |
| --- | --- | --- | --- | --- |
| Log operator | Full control of the log | Rewrite, truncate, withhold, equivocate | Witnesses (equivocation), issuer-signed sequence (withholding), receipts | Collusion; availability |
| Witness | Cosign heads | Cosign a bad head | Independence, audit; 2-of-3 on revisit | Collusion |
| Sender / courier | Device and key | Sign a false statement; deny later | Co-signature, secure element, revocation | Collusion with recipient |
| Recipient | Key | Deny receipt; sign under duress | Co-signature, audit | Duress |
| TSA | Time | Backdate | Multiple TSAs, audit | TSA compromise |
| Network | Intercept/replay | Replay, drop | Nonce, sequence, TLS | DoS |
| Insider | Payload store | Alter body | Hash binding, replication | Store compromise |
| Outsider | Public data | Linkage | Hash-only log, encryption | Timing/linkage |

## Appendix B — Handover statement schema (indicative)

```
{
  "statement_id": "uuid",
  "sequence": 1,
  "shipment_id": "string",
  "issuer": "carrier:...:courier:... | university:registry",
  "subject": "shipment:... | transfer:...",
  "event": "dispatch" | "pickup" | "delivery" | "receipt",
  "time": "ISO-8601",
  "place": { "lat": 0.0, "lon": 0.0, "accuracy_m": 0 },
  "items": [ { "sku": "...", "qty": 0, "condition": "..." } ],
  "photos": [ "sha256:..." ],
  "prev": "sha256(canonical COSE_Sign1) | null",
  "nonce": "string"
}
```

`sequence` is per issuer and is signed, so the leaf commits to it; a gap is evidence of withholding. `prev` is the hash of the prior **signed** statement, not of the body. The body is encrypted off-chain; the log stores its hash.

**Document variant.** Replace `shipment_id` with `transfer_id` (or use `subject` as the canonical identifier), set `event` to `dispatch` or `receipt`, replace `items` and `photos` with `document: { "hash": "sha256:...", "media_type": "application/pdf", "size": 0, "capture_time": "ISO-8601" }`, and make `place` and `condition` optional. The envelope, `prev`, `sequence`, and signature are unchanged.

## Appendix C — Evaluation matrix (summary)

| Test | Level 1 | Level 2 | Level 3 | Level 5 |
| --- | --- | --- | --- | --- |
| Tamper after the fact | Undetected | Detected with a known head | Detected | Detected |
| Truncation | Undetected | Detected with an old head | Detected | Detected |
| Withholding | Undetected | Partial (submitter) | Yes (issuer seq) | Yes (issuer seq) |
| Equivocation | Undetected | Undetected | Detected | Detected |
| Cost per handover | Lowest | Low | Medium | Highest |
| Trust anchors (configured keys) | 2 | 3 | 5 | 5+ |
| Offline | Easy | Hard (time) | Hard (time, revocation) | Hard (gas, keys) |

## Appendix D — Terminology

This is a short subset; **REF-001 is the canonical glossary**.

| Term | Definition |
| --- | --- |
| Hash chain | Hash-linked records, single writer; tamper-evident relative to a known head |
| Anchored log | Head timestamped or published externally; a rewrite is detectable |
| Transparency log | Merkle tree with inclusion/consistency proofs and independent witnesses; detects split views; withholding is detected by issuer-signed sequences |
| Blockchain | Replicated, consensus-ordered, with a membership rule |
| DLT | Replicated shared ledger, not necessarily a chain |
| Receipt | Proof that a statement is included in the log at a given tree head |

## Appendix E — Requirements × options

Qualitative contrast; no weighted score. Columns are the two status-quo systems plus levels 1–5 from §5.1. **L3 is the chosen point.** A cell states what the option provides, not how well it is implemented.

| Requirement | Paper/manual | Central portal | L1 signed DB | L2 chain + TSA | L3 log + witnesses | L4 permissioned | L5 permissionless |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **Evidentiary** | | | | | | | |
| Handover assertion (who, what, when, where) | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| Sender / courier signature | Wet ink | Scan / login | Yes | Yes | Yes | Yes | Yes |
| Recipient co-signature | Wet ink | Sometimes | Yes | Yes | Yes | Yes | Yes |
| Trusted time | No | Server clock | Server clock | TSA | TSA + co-sign | Block time | Block time |
| Integrity after signing | No | DB policy | DB policy | Hash chain | Merkle + witnesses | Chain | Chain |
| Inclusion proof | No | No | No | Head receipt | Merkle proof | Yes | Yes |
| Withholding detection | No | No | No | Partial (submitter) | Yes (issuer seq) | Partial | Yes |
| Equivocation detection | No | No | No | No | Yes | Partial | Yes (economic) |
| Independent offline verification | Manual | No | Yes (sig) | Yes (sig + token) | Yes (sig + token + proof) | Partial (light client) | Partial (light client) |
| Dispute bundle | Paper file | Export | Records + sig | + chain proof | + inclusion proof | Chain query | Chain query |
| **Operational** | | | | | | | |
| Offline operation | Yes | No | Yes (queue) | Yes (weak time) | Yes (weak time) | No | No |
| Key revocation / recovery | n/a | Account | PKI | PKI | PKI + witness | Identity layer | Wallet |
| Multi-leg / multi-carrier | Paper | Yes | Yes | Yes | Yes | Yes | Yes (paper) |
| Integration (ERP/TMS/EPCIS/SIS) | Manual | Strong | Strong | Medium | Medium | Medium | Weak |
| Audit trail (who verified what) | Paper | Logs | DB logs | Chain + TSA | Log + receipts | Ledger | Public ledger |
| Availability / scale | n/a | High | High | High | High | Medium | Low–medium |
| Privacy / erasure | Paper | Controlled | Controlled | Hashes | Hashes + linkage | Permissioned | Public (worst) |
| **Economic / legal** | | | | | | | |
| Legal recognition | Yes (paper) | Contract | eIDAS if QTSP | Evidence | Evidence (not QTSP) | Varies | Weak / varies |
| Cost per handover | Low | Low | Lowest | Low | Low–medium | Medium | High (gas + collateral) |
| Latency | Hours | Seconds | ms | ms + TSA | ms + proof | Seconds | Seconds–minutes |
| Trust anchors (configured keys) | n/a (2 parties) | n/a (1 party) | 2 | 3 | 5 | 4+ | 5+ |
| Membership | Known | Known | Known | Known | Known | Permissioned | Open |
| Physical truth (parcel arrived) | No | No | No | No | No | No | No |

**Notes on the table.**

1. **Trust anchors** counts the keys the verifier must configure: the log/operator key, the TSA key, the witness keys, and the CA root (plus validators or arbiters at levels 4–5). It is a count, not a score; independence matters more than the number. The L3 value is five: log checkpoint key, two witness keys, TSA key, CA root. For paper and central portal the cell is the number of trusted **parties**, not keys.
2. **Legal recognition** depends on the trust service and jurisdiction. A self-hosted log is evidence, not a qualified service under eIDAS; MLETR recognition is national.
3. **Cost and latency** are orders of magnitude, not measurements. Gas, TSA, and witness hosting prices vary.
4. **L2 assumes an RFC 3161 timestamp; L3 assumes the same plus independent witnesses.** Without witnesses, L3 collapses to L2. This is the load-bearing distinction in the whole proposal.
5. **The last row is a property of the problem, not a defect of an option.** Every option proves assertions, not the physical event; the oracle remains outside the ledger.

| Option | Role in PRP-001 |
| --- | --- |
| Paper/manual | Status-quo reference |
| Central portal | Status-quo reference |
| L1 signed DB | Baseline: signatures only |
| L2 chain + TSA | Baseline: the supervisor's model |
| L3 log + witnesses | **Chosen** |
| L4 permissioned | Hedge if a chain is required |
| L5 permissionless | Comparison: the 2018 paper |

## Appendix F — Default decisions

These are the starting choices for the slice. They are **provisional**: each is recorded in the decision log, tested in the spike or the evaluation, and changed only with evidence. They are not RFC requirements; RFC 9943 leaves the API, the VDS, and the policy open, and RFC 9942 permits several verifiable data structures.

| # | Decision | Default | Why | Revisit if |
| --- | --- | --- | --- | --- |
| 1 | Verifiable data structure | RFC 9162 SHA-256 via Tessera | Fills the verifier gap; CT lineage; Go library | Tessera's API blocks the receipt profile |
| 2 | Stack | Go backend + TypeScript PWA | Tessera and COSE on the server; WebCrypto in the browser | You already know one stack well; do not learn two new languages at once |
| 3 | Receipt scope | RFC 9942 inclusion first; consistency by M2 | Smallest useful proof; consistency follows the same tree | Consistency proves harder than expected |
| 4 | Witnesses | Two independent processes, 2-of-2 | Demonstrates non-equivocation with one assumption | One witness is unreliable; move to 2-of-3 |
| 5 | TSA | FreeTSA online; local `uts-server` for offline tests | Free, RFC 3161 | FreeTSA is unreliable; switch to a commercial TSA |
| 6 | CA | Self-signed for the spike; `step-ca` or OpenSSL later | Fast start | Certificate-chain complexity dominates |
| 7 | Offline time | Timestamp on reconnect plus recipient co-signature time | No hardware; honest limitation | The stakeholder needs a stronger time claim |
| 8 | Revocation | Signed status list checked by the verifier | Simple and offline-checkable | Short-lived keys become necessary |
| 9 | `prev` | SHA-256 of the sender's signed statement (the courier's, in the parcel demo) | Strong binding to the exact handover | Statement size becomes a QR/transfer problem |
| 10 | Payload | Off-chain SQLite; the log stores hashes | Privacy and simplicity | Multi-host deployment |
| 11 | Policy | JSON validator: issuers, subjects, algorithms, size, `prev` | No OPA dependency | Policy grows beyond a page |
| 12 | PWA | One codebase, two roles; plain TypeScript or Svelte | A boring UI is a feature | — |
| 13 | Authentication | Development accounts | SSO is out of scope | Real deployment |
| 14 | Deployment | Docker Compose on one host | Reproducible demo | Multi-host or cloud deployment |
| 15 | Evaluation | Ten red-team cases, conformance matrix, latency and proof size, eight users | Fits the claims and the budget | — |

### F.1 Decision log

Every default and every change is recorded in one table. The log becomes the design chapter, the viva answers, and the limitations section.

| Field | Meaning |
| --- | --- |
| Decision | What was chosen |
| Alternatives | What was rejected |
| Reason | Why this choice, given the requirements |
| Consequence | What it costs, limits, or makes harder |
| Evidence | The test, measurement, or source that supports it |
| Residual risk | What remains after the choice |

**Default versus assumption.** A *default* is provisional and testable. An *assumption* is a belief the project cannot test, such as "both witnesses in the 2-of-2 quorum are honest" or "SHA-256 is collision-resistant." The dissertation defends defaults with evidence and states assumptions as limits. Do not let a default silently become an assumption.

**Receipt profile note.** The examples omit `iss` and `typ` from the receipt header. If the verifier must identify the transparency service, add `iss` to the protected header and record it in the decision log; it is a profile choice, not an RFC default.

### F.2 Two-week spike

The spike is the first test of the defaults, not a throwaway:

| Days | Task | Gate |
| --- | --- | --- |
| 1–2 | Tessera append; inclusion proof for a leaf | Proof verifies against the root |
| 3–4 | RFC 9942 receipt: encode, sign, verify in Go | Receipt binds the leaf to the root |
| 5 | COSE sign/verify in the PWA; cross-check with Go | Same bytes produce the same result |
| 6–7 | One statement, one receipt, one verification, end to end | M2 reached |
| 8–10 | Checkpoint and one witness | The verifier rejects a bad checkpoint |

If the gate at day 7 is not met, do not proceed to the UI. Apply the cut ladder in §9.4 or change the default that is blocking — for example, use a simpler VDS or a local verifier wrapper. If the gate at day 10 is not met, the witness/default combination needs revision before M3.

**The spike is not an extra block.** It is the first pass through M0–M3 (minimal): day 7 reaches a minimal M2, and the milestone table's week 4–5 is the hardened version of the same milestone. The two-week spike therefore occupies weeks 1–2, not a separate two weeks before them.

### F.3 What not to change mid-stream

- **The VDS** after the receipt encoder is written; it changes the proof format and the verifier.
- **The `prev` binding** after statements exist; it invalidates old bundles.
- **The witness quorum** after the evaluation starts; it changes the trust claim.
- **The stack** after the PWA and the backend share test vectors; it doubles the encoding bugs.

Change a default only with a decision-log entry, and re-run the conformance suite after the change.
