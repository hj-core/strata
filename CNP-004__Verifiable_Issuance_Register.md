# CNP-004 — Project Proposal: A Verifiable Issuance Register for Safety-Critical Certificates

| Field | Value |
| --- | --- |
| Document ID | CNP-004 |
| Title | Project proposal — A Verifiable Issuance Register for Safety-Critical Certificates |
| Version | 0.6.2 |
| Status | Draft for supervisor review. Direction not approved. No implementation has started. |
| Date | 2026-10-24 |
| Module | COMP4299 — final-year dissertation project (dissertation, demonstration and evaluation) |
| Related documents | `CNP-003` proposes the scope extension and is the companion to this note. Background only: `CNP-002`, `DEC-002`, `ANL-009`, `BUILD-HANDOVER-006`. |
| Purpose | Propose a direction for COMP4299: a register of the certificates a safety decision rests on, with offline proof of commitment and checkpoint proof of what the record does not contain. |
| Supersedes | `CNP-004` v0.1.0, v0.2.0, v0.3.0, v0.4.0, v0.4.1, v0.4.2, v0.4.3, v0.4.4, v0.5.0, v0.6.0 and v0.6.1. |

**Revision history.**

| Version | Change |
| --- | --- |
| 0.6.2 | Requires a 2-of-3 witness threshold on the audit attestation; fetches the deadline state for every scheduled period and reports absent-at-deadline plus present-at-query as `backdated`; downgrades C's enumeration to partial; adds the archive failure outcomes to C6. |
| 0.6.1 | Renames the comparison arms: C-R becomes C, C-L becomes D1 and D becomes D2. D1 is D2 with the witness layer disabled. Earlier changelog entries use the old labels. |
| 0.6.0 | Corrects the 2-of-3 quorum assumption to an honest majority; separates fork prevention from fork detection; reclassifies C-L as an ablation of D rather than existing practice; downgrades C-L's post-incident proofs to conditional; adds `log_entry_mismatch` to C6. |
| 0.5.0 | Replaces the single central-registry baseline with C-R (signed receipt) and C-L (unwitnessed log); reframes D's unique property as non-equivocation; removes the unverified claim that the DSSCU register is the Macau baseline for this certificate class. |
| 0.4.4 | Tightens the subtitle to the 2 measured claims: commitment at issuance and checkpoint proof of absence. |
| 0.4.3 | Defines the required-period schedule as a query input; bounds the checkpoint interval and adds witness enforcement as a condition of the claim. |
| 0.4.2 | Anchors the audit attestation in the witness keys; separates storage deletion from tree rewrite; defines `period` as the UTC calendar year. |
| 0.4.1 | Separates the auditor (O(N) projection) from the investigator (O(log N) proofs); rewrites the withholding case; fixes the completeness row and the R2 wording. |
| 0.4.0 | Credits PKI with key binding and non-repudiation; reframes the gap as archive commitment; adds the 3 archive rows; treats the operator as untrusted. |
| 0.3.0 | Grounds the problem in Macau Law No. 14/2022; changes the object to installed sensors; corrects the baseline comparison. |

**Decision requested.** (1) Approve the direction. (2) Confirm the module constraints in writing: a narrow mechanism with a measured boundary, a positive headline, and a plausible scenario as sufficient. (3) Confirm the calibration-certificate scope and the Macau setting. (4) Confirm that no implementation begins before approval.

**Relationship to CNP-003.** `CNP-003` asks whether a laboratory was *authorised* to issue a certificate. This note asks whether the laboratory *issued* it, and what the archive says about an asset. The 2 questions are independent, and this note argues that the second is logically prior: until an investigator knows which laboratory issued a document, the scope question has no subject. The scope question is recorded as future work in §7.3.

## Terms

| Term | Meaning |
| --- | --- |
| **Accreditation** | Formal recognition by an accreditation body that a conformity assessment body is competent to perform specific activities. |
| **Accreditation body** | The organisation that assesses conformity assessment bodies. Examples: UKAS (United Kingdom), CNAS (China), HKAS (Hong Kong). |
| **Calibration certificate** | The document a laboratory issues after a calibration, and on which a verifier relies. |
| **Safety-critical certificate** | A certificate that a public-safety decision rests on, such as a lift inspection declaration, a fire test report or a calibration certificate. |
| **Verifier** | The party that relies on a certificate and decides whether to accept it. |
| **Investigator** | The party that reconstructs a record after an incident: a safety authority, an accident board, a court, an insurer. |
| **Issuance** | The act of producing and signing a certificate. Distinct from the work the certificate describes. |
| **Issuance register** | An append-only, verifiable record of certificate hashes, with signed checkpoints. |
| **Log operator** | The organisation that runs the register: admits leaves, publishes checkpoints and serves proofs. It may be the regulator or an accreditation body; integrity does not depend on its neutrality. |
| **Issuance leaf** | 1 entry in the register: certificate hash, laboratory, key, certificate date, subject reference and certificate type. |
| **Subject** | The asset a certificate supports: a lift, a building, a batch. The laboratory declares it, and the certificate carries it. |
| **Subject index** | A Merkle map from `(subject, period)` to a commitment over the leaves in that period, whose root appears in each checkpoint. `period` is the UTC calendar year of the certificate date. |
| **Gap** | An entry required by the supplied schedule that the register does not contain, proven by non-inclusion. |
| **Required-period schedule** | A declared list of `(subject_ref, period)` pairs that must hold a certificate, supplied with a subject query. Its source is named — a statutory interval, a manufacturer's calibration interval or an insurer's condition. The register proves presence or absence against it; it does not establish that the schedule is correct. |
| **Auditor** | The party that rebuilds the index from the log at a checkpoint and signs an audit attestation over the checkpoint and both roots. One of the 3 witnesses runs the audit, and at least 2 of the 3 sign the attestation after verifying the projection. |
| **Merkle tree** | A hash tree that supports compact proofs of inclusion and consistency. |
| **Checkpoint** | A signed commitment to the state of the register at a point in time, including the subject-index root. |
| **Maximum checkpoint interval** | The longest gap the policy permits between 2 checkpoints (the checkpoint cadence). Witnesses refuse to cosign across a longer gap. |
| **First-inclusion checkpoint** | The checkpoint whose tree first contains a given leaf. |
| **Issuance receipt** | A signed proof that a certificate hash was included in the register at a checkpoint. |
| **Witness** | An independent party that checks a consistency proof and cosigns a checkpoint. |
| **Fork** | 2 divergent histories presented by the same operator to different verifiers. |
| **Non-equivocation** | The property that an honest majority refuses to cosign a divergent checkpoint, and that a cosigned fork is detectable by comparison. |
| **Repudiation** | A laboratory's denial that it issued a certificate it in fact signed. |
| **Allow-list** | The signed list of laboratory keys admitted to the register, committed in each checkpoint. |
| **Retention** | The period for which the archive remains readable and provable, set against the life of the asset. |
| **Time-stamping authority (TSA)** | A service that signs a token proving that a datum existed at a given time (RFC 3161). |
| **COSE, CBOR, CWT** | Standards for signed messages (COSE, RFC 9052), binary encoding (CBOR) and token claims (CWT). |
| **ISO/IEC 17025** | The international standard for the competence of testing and calibration laboratories. |
| **DCC** | Digital Calibration Certificate: a machine-readable, signed calibration certificate format developed under PTB. |
| **SCITT** | Supply Chain Integrity, Transparency and Trust: the IETF architecture for signed statements and transparency services (RFC 9943, RFC 9942). |
| **Certificate Transparency (CT)** | The register-log system for TLS certificates. Version 2 is defined in RFC 9162 and obsoletes RFC 6962. |
| **Evidence labels** | **V** = verified against a primary or official source; **P** = partial or secondary evidence; **A** = analytical or not yet supported by a documented case. |

---

## 1. Project title

> **A Verifiable Issuance Register for Safety-Critical Certificates**
> *Offline proof that a certificate was committed at issuance, and checkpoint proof of what the record does not contain*

### 1.1 Definition

A register of the certificates that safety decisions rest on, held by a log operator, with a verifiable history, a receipt that travels with each certificate, and an index that lets an investigator reconstruct an asset's record and prove its gaps.

### 1.2 Title rationale

The title names the object (issuance of safety-critical certificates), the artefact (register) and the capability (verifiable). It omits the standard number (ISO/IEC 17025, §6), the protocol stack (RFC 9162, COSE) and the mechanism (Merkle log, subject index, witnesses, receipts), which belong in §4 and §6.

The subtitle states the 2 claims the dissertation measures, one per audience: offline proof that a certificate was committed at issuance, for the verifier, and checkpoint proof — a non-inclusion proof against a cosigned checkpoint — of what the record does not contain, for the investigator. It deliberately avoids "who issued", which PKI already establishes (§2.1), and "what is missing", which would assert more than a non-inclusion proof can support (§4.5; R6 in §9.1).

---

## 2. Problem statement

A safety-critical certificate is a signed assertion by a laboratory that it performed work, on an object, by a method, with a result. A lift certificate, a fire test report and a calibration certificate share 3 properties: a safety decision rests on them, they are relied on years after issuance, and the party that issued them may no longer exist when the decision is challenged.

The verifier must establish 4 things about 1 document:

1. that the document is the one the laboratory signed, unaltered;
2. that the laboratory named on it is the laboratory that signed it;
3. that it was signed at the time it claims;
4. that it was committed to the archive when it was issued.

The investigator must establish 3 things about a population:

1. which certificates the archive holds for this asset;
2. who issued each one, and when;
3. which required entries are absent.

Modern PKI answers the verifier's first 3 questions. A signed DCC under X.509 with an RFC 3161 token binds the key to the laboratory and fixes the time, and this proposal does not claim otherwise. What no signature answers is the fourth: a certificate can be signed, timestamped and never committed. Without it the archive is a voluntary collection, and it may be empty exactly when it is needed.

### 2.1 What a signature does not establish

| Question | Raw signature | PKI signature + timestamp | What is missing |
| --- | --- | --- | --- |
| Is the document unaltered? | Yes, given the key | Yes | — |
| Is the key the laboratory's? | No | Yes, through the CA | — |
| Did the laboratory sign it? | A key signed it | Yes | — |
| Was it signed at the time it claims? | A date written by the laboratory | Yes, through the TSA | — |
| Was it committed to the archive at issuance? | No | **No** | An inclusion proof against a checkpoint |
| What else exists for this asset? | No | No | An index by subject |
| Is anything missing? | No | No | A non-inclusion proof |

### 2.2 The 3 failures that remain

**Archive commitment.** A laboratory can sign a certificate and never log it. If the verifier accepts the signature, nothing records that the certificate exists, and the archive is a voluntary collection. The failure is not authenticity; it is the absence of a commitment.

**Withholding and backfill.** A certificate signed and timestamped in 2024 can be held out of the record at the deadline — to avoid a maintenance overhaul, a filing or a regulatory trigger — and quietly submitted after an incident in 2026. The signature and the token are genuine, and neither PKI nor a database without witnesses can prove that the certificate was absent at the 2024 deadline. Only a record of what was committed, and when, exposes the gap.

The same gap covers a pair of certificates: a laboratory that issues a failed and a passing certificate on the same day can withhold the failed one. A rule that requires every issued certificate to be logged makes the withheld certificate provable once the other surfaces; a rule that logs only the certificate presented does not.

**Survival and completeness.** The laboratory may be foreign, defunct or unwilling. The certificate is 5 years old, the company has been dissolved, and the verifier has no one to ask. The archive must outlive the issuer and prove what it does not contain.

The field verifier is the gate. If it accepts a signature alone, a laboratory can sign and never log, and the archive is empty after the accident. If it requires a receipt, the certificate must be committed before anyone relies on it. That is the mechanism that makes the archive complete enough to investigate.

### 2.3 The failure of the archive

After an accident, the question changes from "is this document genuine?" to "what does the record for this asset contain?". An investigator collects certificates from the owner, the insurer, the contractor and the maintenance file. The collection is incomplete by construction: nobody holds the whole set, absence is indistinguishable from a lost file, and the laboratory that could fill the gap may no longer exist.

Nothing in the current arrangement can prove that an entry is **missing**. A pile of PDFs proves only what it contains.

### 2.4 Additional evidence

| Fact | Source | Status |
| --- | --- | --- |
| IAF CertSearch is mandated by IAF Mandatory Document 28 for ISO/IEC 17021-1 management-system certificates, with about 2.5 million certifications | IAF | **V** |
| No equivalent global register covers ISO/IEC 17025 calibration certificates | IAF CertSearch scope; national directories | **V** (scope), **P** (absence) |
| UKAS publishes counterfeit-certificate warnings and records of false accreditation claims | UKAS counterfeit-certificates page | **V** |
| 121 accreditation bodies are signatories to the ILAC MRA; over 114,600 laboratories are accredited by them | ILAC, 2025 | **V** |
| Commercial aviation mandates maintenance records, with logbooks retained for the life of the aircraft | EASA Part-M; FAA 14 CFR 91.417 | **V** |
| Medical-device rules require unique device identification and traceability (EU MDR; US FDA UDI) | EU; US FDA | **V** |
| National legal-metrology law requires verification of measuring instruments and a record of it | National law; OIML R 76 as the model | **V** |
| The Grenfell Tower Inquiry's Phase 2 report (2024-09-04) recorded that test information was withheld from the certifier and recommended attaching the full testing history to certificates | Grenfell Tower Inquiry, Phase 2 | **V** |
| Macau Law No. 14/2022 requires owners to register lifts with the Lands and Urban Construction Bureau (DSSCU), requires an annual inspection by a registered inspection entity, and requires the entity to notify DSSCU of every inspection compliance declaration (Art. 6(6)); the declaration is valid for 1 year | Macau Official Gazette; DSSCU | **V** |
| No Macau instrument requires the ISO/IEC 17025 calibration certificates beneath those inspections to be registered | Not found | **P** |
| A documented adjudicated case of a laboratory denying a certificate it signed | Not found | **A** |

### 2.5 The gap

> No method lets a verifier, offline and years later, establish that a certificate was committed to an authoritative archive at issuance; and no method lets an investigator establish, from that archive, which certificates exist for an asset and which required entries are absent.

The gap produces 2 costs. A certificate that was never committed is indistinguishable from one that was, so the archive looks complete when it is not. And a record with a hole is treated as complete because no one can prove the hole.

---

## 3. Significance of the problem, with reference to Macau

### 3.1 General significance

Safety decisions rest on documents produced by parties with no continuing relationship to the decision. A lift is certified in 2019; the certifier dissolves in 2023; the building changes insurer in 2025; an incident in 2026 turns the certificate into evidence. The party that must decide — an insurer, a court, a safety authority — holds a document and no record.

### 3.2 The Macau SAR context

Macau is a small, dense, high-rise jurisdiction with an ageing building stock and a construction and tourism economy that depends on third-party conformity assessment. The lift regime is already statutory: Law No. 14/2022, in force from 2024-04-01, requires every lift to be registered with the Lands and Urban Construction Bureau (DSSCU), inspected annually by a registered inspection entity, and its inspection declaration notified to DSSCU. The register covers the declaration. It does not cover the calibration certificates of the instruments the inspection relies on.

3 features make the accountability problem more frequent here:

- **The laboratory is often elsewhere.** Laboratories accredited outside Macau certify lifts, fire-rated products and construction materials. The authority checks a foreign laboratory it has no right to audit and cannot easily contact.
- **The reliance is years later.** Periodic inspection, lift safety, fire safety and insurance renewal re-open certificates issued long before. Companies dissolve; staff leave; the issuing laboratory may no longer exist.
- **The authorities are not the issuer.** The public works, fire and municipal authorities, the courts and insurers hold the document. They can require it; they cannot confirm it or enumerate what else exists.

3 illustrative scenarios (A: plausible, not documented as Macau cases):

1. **Lift maintenance.** A brake-force sensor permanently installed on a lift is calibrated in 2022 by a laboratory accredited outside Macau. The certificate enters the maintenance file. In 2026 a brake incident occurs, the building changes insurer, and the laboratory has dissolved. The investigator holds the 2022 certificate and no way to know whether the 2020 calibration exists.
2. **Fire-rated products.** A test laboratory in another jurisdiction issues a report in 2022 that supports a building approval. In 2026 a fire-safety review asks whether the report is the one the laboratory issued, and whether a later version exists.
3. **Construction materials.** A concrete or steel test report from a Mainland laboratory supports a structural submission. The laboratory's accreditation is later reduced. The investigator must first establish that the report is genuine before asking anything else.

The scenarios illustrate a general problem: the issuer, the archive and the decision-maker are frequently in 3 jurisdictions. They are labelled A; the dissertation states that they are illustrative.

### 3.3 The order of the questions

An incident produces 3 questions in a fixed order:

1. **Is the document genuine?** Which laboratory signed it, and when?
2. **Was the issuer authorised?** Was the work inside the laboratory's accredited scope on that date?
3. **Is the result right?** Was the measurement or test correct?

The order is not a preference. The second question presupposes the first: until the investigator knows which laboratory issued the document, the scope question has no subject. The third is outside any documentary mechanism.

This proposal answers the first question, including commitment, for 1 document and for the archive. `CNP-003` answers the second. The third is out of scope in both.

### 3.4 The investigator's question

The field verifier and the investigator ask different things of the same record:

| | Field verifier | Investigator |
| --- | --- | --- |
| Holds | 1 certificate and its receipt | The archive, or a query against it |
| Asks | Is this genuine? | What exists for this asset, and what is missing from the archive? |
| Needs | Inclusion proof, offline | Enumeration, presence and absence proofs |
| Time | Minutes | Days, after an incident |
| Failure mode | Accepts an uncommitted certificate | Treats a hole as complete |

The second column is the one that motivates a mandate. An insurer can decline a certificate; a safety authority must reconstruct a record.

### 3.5 The regulatory path

Macau's lift regime already mandates a record. Law No. 14/2022 requires registration, annual inspection and notification of every inspection declaration to DSSCU, and the implementing regulation sets the procedure. The statutory register covers the declaration, not the calibration certificates beneath it. The proposal adds the calibration layer and a record that the authority cannot rewrite.

| Precedent | What is mandated |
| --- | --- |
| Lifts in Macau | Law No. 14/2022: registration, annual inspection, declaration notified to DSSCU |
| Aviation maintenance | Logbooks retained for the life of the aircraft |
| Medical devices | Unique identification and traceability |
| Legal metrology | Verification of instruments and a record of it |
| Vehicle inspection | Periodic inspection with a statutory record |
| Fire safety | Logbooks and inspection records |
| Construction | Permits and inspection records |

The proposal does not invent the idea of a statutory record. It adds **verifiability** to a pattern that already exists, for a class of certificate that currently has none.

3 routes can produce the mandate, and only the first is legislation:

1. **Statute.** A safety authority requires a receipt for certificates supporting a public-safety approval. The slowest route.
2. **Procurement.** A public works department requires a receipt as a condition of accepting a certificate. The fastest route, and it needs no new law.
3. **Liability.** An insurer requires a receipt as a condition of cover, because it lowers its own exposure after an incident.

Certificate Transparency is the precedent for the non-statutory route: no legislature mandated it, and browsers refusing unlogged certificates made logging the norm.

A mandate needs a cost bearer and an enforcement point. Here the laboratory bears the submission cost, and the state, a procuring authority or an insurer enforces. The dissertation states the 3 routes as a policy argument, not as a finding, and evaluates the mechanism under a stated mandate.

### 3.6 Fit with the module

The module accepts a narrow mechanism with a measured boundary, requires a positive headline, and accepts a plausible scenario (stated 2026-10-23; written confirmation is decision (2)). The proposed register meets the 3 conditions: the mechanism is a log, an index and a receipt; the result is a capability; and the Macau scenarios supply the setting without a recruited stakeholder.

---

## 4. Proposed solution

### 4.1 Object and artefact

The object is the issuance register: the append-only record of which laboratory issued which certificate, when, under which key, for which asset.

The artefact is the issuance receipt: a proof bundle assembled at the moment the certificate is created, which binds the certificate to a position in the register and a time. The receipt contains:

- the certificate hash, computed over the complete signed certificate, including the certificate date, the certified values, the subject reference and the certificate type;
- the laboratory identifier, the key identifier, the certificate date, the subject reference and the certificate type;
- the inclusion proof of the issuance leaf at the first-inclusion checkpoint;
- the signed checkpoint statement, with witness cosignatures;
- an RFC 3161 timestamp token for the checkpoint;
- the verification policy: trust-anchor identifiers, accepted algorithms, checkpoint cadence and freshness window.

A receipt is a proof bundle assembled from operator-signed and witness-signed parts. The laboratory assembles it; it cannot alter what was signed.

The certificate envelope carries a **subject reference**: the asset on which the calibrated instrument is installed, such as a lift or a building. The calibration item is the instrument; the asset reference records where it is installed. The laboratory declares it, the certificate signs it, and the leaf repeats it. A certificate for a portable instrument that names no installation is not admissible to the register.

The archive exposes 2 queries:

- **by receipt:** is this certificate included at this checkpoint?
- **by subject:** which leaves exist for this subject in a period range, and which periods in the supplied schedule are absent?

The subject query takes a **required-period schedule** as an input: a declared list of `(subject_ref, period)` pairs that must hold a certificate, with its source named — a statutory interval, a manufacturer's calibration interval or an insurer's condition. The register proves that each listed pair is present or absent. It does not establish that the schedule is the right one, and the report states the source beside the result. Without a declared schedule the query reports presence and enumeration only; it makes no gap claim.

### 4.2 Flow

1. **Admit.** The log operator publishes a signed allow-list binding each laboratory identifier to its certificate keys. The allow-list is committed in each checkpoint.
2. **Issue.** The laboratory creates the certificate and signs it. Its tooling submits the certificate hash, the laboratory identifier, the key identifier, the certificate date, the subject reference and the certificate type to the register, signed by the laboratory key.
3. **Log.** The operator checks the submission against the allow-list, appends the leaf, updates the subject index, and returns the receipt at the next checkpoint. The checkpoint is signed and cosigned by witnesses. Before cosigning, each witness verifies an RFC 9162 consistency proof from its last cosigned checkpoint and refuses a divergent history.
4. **Present.** The certificate and receipt travel to the verifier, possibly years later and possibly without the laboratory.
5. **Decide.** The verifier checks offline, at UTC day granularity: the certificate signature against the admitted key; the recomputed hash against the leaf; the leaf's laboratory, certificate date, subject reference and certificate type against the certificate; the inclusion proof; the checkpoint signature and witness threshold; the freshness rule; and, if the bundle carries a later checkpoint, the consistency proof and any fork.
6. **Investigate.** The investigator queries the subject index for a period range, supplying the required-period schedule. The archive returns each leaf with an inclusion proof at a checkpoint whose timestamp is at or after the end of the range plus 1 cadence. For every period in the supplied schedule, it also returns the state at the first checkpoint after that period's deadline: a non-inclusion proof if the period was empty, or an inclusion proof if it was not. The deadline state is returned for every scheduled period, not only those empty at query time, so a period that was empty at the deadline and holds a leaf at query time is reported as `backdated`. A period that was present at the deadline and is empty at query time is a deletion and returns `log_entry_mismatch`. The investigator verifies the proofs against the same pinned anchors and reports the timeline, the gaps, the schedule source and the named outcome. If no checkpoint at or after the range end is available, the result is `unverifiable`.

**Index projection.** Each checkpoint commits the log root and the index root. An **auditor** rebuilds the index from the log leaves at a checkpoint and compares the 2 roots. The rebuild is linear in the number of leaves (O(N)); it runs periodically and at an incident, not on every query. The result is a separate **audit attestation** over the checkpoint identifier, the log root and the index root, signed by at least 2 of the 3 witness keys; each signer verifies the projection against the log. The **investigator** does not download the log and does not rebuild the index: it verifies inclusion and non-inclusion proofs (O(log N)) against the cosigned index root, and checks the audit attestation for that root. The attestation may lag the checkpoint, and its timestamp is what the investigator reports. Between audits, the guarantee rests on the last audited root; the proof itself is against the current cosigned root. The attestation is reproducible: any party can rerun the audit, so a false attestation is detectable, not authoritative. This is the Index trust limit in §4.5. The attestation is part of D2. In the unwitnessed D1 ablation there is no witness to sign it, so the index root is operator-signed and a light investigator must run `--audit` or trust the operator.

### 4.3 Architecture

| Element | Choice | Rationale |
| --- | --- | --- |
| Register log | RFC 9162 Merkle tree, implemented with Tessera (an open-source register-log implementation) | Append-only; consistency proofs; existing implementation |
| Issuance leaves | COSE_Sign1 over `certificate_hash`, `laboratory_id`, `key_id`, `certificate_date`, `subject_ref`, `certificate_type` | 1 leaf per certificate; the submission is signed by the laboratory |
| Subject index | Sparse Merkle map from `(subject_ref, period)` to a commitment over the leaves in that period, with its root in each checkpoint | Presence and absence proofs; `period` is the UTC calendar year of `certificate_date`; a period may hold more than 1 leaf |
| Index projection | The checkpoint commits the log root and the index root; an auditor rebuilds the index from the log and compares | O(N), periodic and at an incident; at least 2 of the 3 witnesses sign the attestation, each verifying the projection against the log |
| Allow-list | Signed map from laboratory to admitted keys, committed in each checkpoint | Binds a key to a named laboratory |
| Receipts | RFC 9942 COSE receipts over the leaf and the checkpoint | Offline inclusion verification |
| Statements | COSE_Sign1 (RFC 9052), canonical CBOR, CWT claims, with a type discriminator per statement class | Compact, signed, standardised, not repurposable |
| Time | RFC 3161 token per checkpoint | Independent time anchor |
| Certificate envelope | DCC v3.3.0 structure (PTB), with an issuance-receipt extension | Reuses the deployed digital calibration certificate format |
| Witnesses | 3 independent cosigners over checkpoints, threshold 2-of-3; each verifies a consistency proof before cosigning | An honest majority prevents an un-cosigned fork; comparison detects a cosigned fork |
| Checkpoint cadence | A published maximum interval between checkpoints; a witness refuses to cosign across a longer gap | Bounds the freshness claim and gives the deadline proof a fixed target |
| Field verifier | Offline command-line tool, plus a minimal web view | Named outcomes; reproducible |
| Investigator query | An archive client with 2 modes: `--query` verifies compact proofs in O(log N) against the cosigned index root; `--audit` rebuilds the index in O(N) | The query never downloads the log; the audit is a separate role |

SCITT (RFC 9943) supplies the architecture: signed statement, transparent statement, receipt and registration policy. The RFC scopes itself to software supply chains and states that the architecture applies to other supply-chain statements. A certificate register is an extension.

### 4.4 Properties established

| Property | What the verifier or investigator obtains | Offline | Condition |
| --- | --- | --- | --- |
| **Authenticity** | The certificate is the one whose hash is in the register | Yes | Signature and recomputed hash |
| **Key binding** | The signing key was admitted for this laboratory at the checkpoint | Yes | Allow-list commitment |
| **Commitment** | The leaf is included at its first-inclusion checkpoint, which falls within 1 cadence of the certificate date | Yes | Inclusion proof and checkpoint signature |
| **Non-repudiation** | The laboratory cannot deny the submission without contradicting a cosigned checkpoint | Yes | Operator signature and witness cosignatures |
| **Enumeration** | The full set of leaves the register holds for a subject in a period range | Yes, against a local snapshot | Subject index, inclusion proofs and the projection audit |
| **Provable absence** | No leaf exists for a scheduled subject-period pair | Yes, against a local snapshot | The schedule supplied with the query, a non-inclusion proof against a cosigned index root at or after the range end plus 1 cadence, and a deadline proof at the first checkpoint after the period deadline |
| **Backfill detection** | A scheduled period was empty at its deadline and holds a leaf at the query checkpoint | Yes, against a snapshot | An inclusion proof at the query checkpoint, paired with a non-inclusion proof at the first checkpoint after the period's deadline; the result is `backdated`. A submission between the deadline and the first post-deadline checkpoint is not distinguishable (§4.5) |
| **Non-revision** | No leaf was removed or altered; every later addition is consistent | Yes, against the checkpoints held | Merkle log and consistency proofs |
| **Non-equivocation** | An honest majority refuses to cosign a divergent checkpoint; 2 parties comparing cosigned checkpoints detect a cosigned fork | Yes | At least 2 of 3 honest witnesses, each checking consistency before cosigning |
| **Self-containment** | The field verifier needs neither the laboratory nor the operator | Yes | A bundle that carries the leaf, the proof and the checkpoint |

Section 5.1 marks 5 archive rows. D1 provides conditional proofs on 4 of them — commitment, absence, deletion or alteration, and backfill — but only against an operator-signed checkpoint, and it cannot prevent a fork; a post-incident investigator who did not cache the checkpoint must trust the operator or run the O(N) audit. D2 is the only option that answers all 5 unconditionally against an untrusted operator, because an honest witness majority prevents an un-cosigned fork. B + C answers every non-archive row that any option answers, with C's enumeration marked partial because it is an unproven database response. Performance and completeness are answered by none; legal recognition depends on a mandate and is not a property of the mechanism.

### 4.5 Limits of the claim

- **Performance.** The register proves that the laboratory submitted a certificate, not that the work happened. A laboratory can log a certificate for work it did not perform, and a logged leaf can fill a gap that no real work supports. A failed calibration that never becomes a certificate is outside the register, so suppression of a result is invisible unless the result was issued.
- **Completeness.** The register does not prove that it contains every certificate the laboratory issued. It proves that the archive has no indexed entry for a scheduled period. Whether that means the work did not happen, was not logged, was logged under another subject, or never became a certificate is a question for the investigator.
- **Schedule authority.** The register proves absence relative to a schedule supplied with the query. It does not establish that the statutory, manufacturer or insurer interval is the right one, and a wrong schedule produces a wrong gap. The report names the schedule source, and the dissertation treats that source as an input outside the claim.
- **Attribution.** The subject reference is declared by the laboratory and covered by its signature. The register proves that the declared asset was named, not that the asset is correct. A certificate for 1 lift can be filed under another, and the archive cannot tell.
- **Index trust.** A light investigator verifies a non-inclusion proof against a cosigned index root. The root is a projection of the log only if the projection audit runs. Between audits, the guarantee is as of the last audit attestation, and a party that skips the attestation trusts the operator's index.
- **Timeliness and freshness.** The sequence of checkpoints fixes order, not promptness. The claim assumes that the laboratory logs each certificate within 1 checkpoint of its date; a later submission with an earlier date returns `backdated`. It also assumes that the operator publishes a checkpoint at least every declared maximum interval. That bound is enforced only by the witnesses' refusal to cosign a longer gap, so a colluding witness threshold can still stall the record. One window stays open even with the bound: a certificate submitted after the period deadline but before the first post-deadline checkpoint is indistinguishable, at that checkpoint, from a punctual submission. The freshness rule catches submissions later than 1 cadence after the certificate date, not this window.
- **Signer authority.** The register proves that an admitted laboratory key signed, not that the individual who used it was authorised by the laboratory.
- **Key compromise.** A compromised laboratory key can sign and log certificates the laboratory never issued. Revocation is prospective and cannot erase what the log already contains.
- **Legal recognition.** The register carries no legal recognition. The operator's record has no regulatory weight by itself.
- **Privacy.** The index links certificates to assets. A public archive exposes building addresses, lift identifiers and inspection history. A regulator-only archive limits that exposure and weakens third-party monitoring.
- **Retention.** The archive must remain readable and provable for the life of the asset, which exceeds the life of the operator, the laboratory and the certificate format.

### 4.6 The claim

> A verifier can establish, offline and years later, that a certificate was committed to the archive by the laboratory it names, at the time it claims, within 1 checkpoint cadence; and an investigator can establish, from the archive, the full set of certificates the register holds for an asset and the absence of any entry required by a schedule supplied with the query — without the laboratory's cooperation and without trusting the operator's account of its own history.

The claim holds under 9 conditions: the laboratory logs the certificate at issuance (the condition in §4.7); the laboratory logs within 1 checkpoint of the certificate date; the verifier and investigator hold the public trust-anchor identifiers; at least 2 of the 3 witnesses are honest, each verifies a consistency proof before cosigning, and at least 2 sign the audit attestation after verifying the projection; the timestamping authority is honest; the investigator's snapshot holds a checkpoint at or after the range end plus 1 cadence; the operator publishes a checkpoint at least every declared maximum interval, and the witnesses refuse to cosign a longer gap; the schedule supplied with the query states the applicable requirement; and a cosigned fork is detected by a party that holds a later checkpoint or compares checkpoints with another party. The limits of the claim are in §4.5.

### 4.7 Condition of operation

The Certificate Transparency model depends on 2 conditions: certificate authorities must log, and browsers refuse certificates without a Signed Certificate Timestamp. The proposed register requires the same pair:

1. **The laboratory logs** every safety-critical certificate it issues, as a condition of the certificate being admissible for a public-safety decision.
2. **The decision-maker refuses** a certificate without an issuance receipt, through a procurement rule, a regulatory requirement or an insurer's condition.

The pair is not sufficient on its own. The operator must also publish a checkpoint at least every declared maximum interval, and the witnesses must refuse to cosign across a longer gap. Without that bound, a delayed checkpoint lets a withheld certificate enter the log before the first post-deadline checkpoint, and the deadline proof in §4.2 loses its force. The bound does not close the window between the deadline and the first post-deadline checkpoint; §4.5 states that residual. The interval is a published policy term, decided at design time (deliverable C1), not a per-query choice.

The first condition has a lever that the scope register lacks: the accreditation body can make logging a condition of accreditation. The second needs an authority, a procuring body or an insurer to refuse a certificate without a receipt, which is the harder one. Without it, a laboratory can issue without logging, and the register does not cover the certificate.

3 routes can produce the condition (§3.5), and the dissertation states them as a policy argument, not as a finding. The number of potential operators is bounded: 121 accreditation bodies are signatories to the ILAC MRA (ILAC, 2025), and each already operates a directory.

### 4.8 Threat model

| Adversary | Capability | Mitigation | Residual risk |
| --- | --- | --- | --- |
| Malicious laboratory | Issue without logging; backdate; deny a logged certificate; log a certificate for work not done | Required receipt; freshness rule; cosigned checkpoints | An unlogged certificate leaves no entry; a false claim about the work is the performance limit |
| Forger | Fabricate a certificate with a copied letterhead | Signature against the admitted key; hash must match a logged leaf | A forged certificate with no leaf fails; one with a copied signature fails the signature check |
| Malicious operator | Delete a leaf; rewrite a checkpoint; answer a query with a fabricated list | Append-only log; consistency proofs; proofs rather than assertions | An honest majority prevents an un-cosigned fork; a cosigned fork is detectable only if a party compares checkpoints |
| Operator, index omission | Leave a leaf in the log but out of the index, so a query returns `absent` | Index projection audit rebuilds the index from the log, and at least 2 of 3 witnesses sign the attestation | An investigator that skips the audit trusts the index |
| **The state as operator** | Rewrite the record after an incident it failed to prevent | Witness cosignatures across independent stakeholders; the operator need not be trusted for history (§6.4 assumption 1) | Collusion between the operator and 2 of the 3 witnesses is not preventable; a cosigned fork is detectable by comparison |
| Colluding witnesses | 2 of 3 witnesses cosign 2 histories | Independence criteria; at least 2 honest; cross-jurisdiction witnesses | Collusion of 2 witnesses is not preventable; a cosigned fork is detectable by comparison |
| Compromised TSA | Roll back a timestamp sequence | Multiple checkpoints; pinning | A compromised or colluding TSA defeats the freshness rule |
| Compromised laboratory key | Log certificates the laboratory never issued | Allow-list revocation; short-lived keys | The log keeps what was logged; revocation is prospective |

### 4.9 What this design does not answer

The register says nothing about whether the laboratory was *accredited* for the work. A certificate can be genuinely issued by a laboratory that had no authority to issue it. That is the question `CNP-003` answers, and §7.3 records how the same log can carry it.

---

## 5. Comparison with existing practice and the ablation

The section compares existing or non-log practice (A, B, C) with the proposal (D2). D1 is not existing practice: it is D2 with the witness cosigner disabled, included as an ablation to isolate the witness layer. C is a comparison arm; D1 and D2 are configurations of the proposal. None is a deliverable ID; the deliverables are C1–C10 in §7.1.

### 5.1 Capability and trust

Legend: ✓ yes · ◐ partial · ✗ no · † conditional on the operator not equivocating, or on the party having cached a checkpoint before the event being proved. C and D1 require † on the affected rows; D2 replaces the operator-trust condition with an honest witness majority. D1 and D2 both require the party to hold a checkpoint; the notes state it.

**A** is current practice: a signed certificate and a telephone call. It is a generic description, not a verified account of Macau practice. **B** is a signed DCC under X.509 with an RFC 3161 token; the CA binds the key and the signature gives non-repudiation. **C** is a central registry that stores the certificate hash, a key directory and a subject index, requires a signed submission, and issues a signed, TSA-timestamped receipt at submission, but has no Merkle log. **D1** is C plus a Merkle log, signed checkpoints and a subject index, with the witness cosigner disabled; it is an ablation of D2, not existing practice. **D2** is D1 plus 2-of-3 witnesses, with at least 2 of the 3 assumed honest and each checking a consistency proof before cosigning. C, D1 and D2 assume B's PKI and TSA.

| Question | A | B | C | D1 (ablation) | D2 |
| --- | --- | --- | --- | --- | --- |
| Certificate authentic and unaltered | ◐ | ✓ | ✓ | ✓ | ✓ |
| Key belongs to the named laboratory | ✗ | ✓ | ✓ | ✓ | ✓ |
| The laboratory's key signed this certificate | ◐ | ✓ | ✓ | ✓ | ✓ |
| Signing time independent of the laboratory | ✗ | ✓ | ✓ | ✓ | ✓ |
| The laboratory cannot deny issuance | ✗ | ✓ | ✓ | ✓ | ✓ |
| Commitment: offline proof the certificate entered the archive within 1 cadence | ✗ | ✗ | ◐† | ◐† | ✓ |
| Enumerate the certificates for an asset | ✗ | ✗ | ◐† | ✓† | ✓ |
| Show a second certificate for the same asset-period | ✗ | ✗ | ◐† | ✓† | ✓ |
| Provable absence: O(log N) proof a scheduled entry is absent | ✗ | ✗ | ✗ | ◐† | ✓ |
| Tamper evidence: detect deletion or alteration of a logged entry | ✗ | ✗ | ◐† | ◐† | ✓ |
| Backfill detection: a period empty at its deadline holds a leaf at query time | ✗ | ✗ | ✗ | ◐† | ✓ |
| Offline verification without trusting operator history | ✗ | ✓ (sig only) | ✗ | ✗ | ✓ |
| Non-equivocation: prevent an un-cosigned fork; detect a cosigned fork | ✗ | ✗ | ✗ | ✗ | ✓ |
| Survives deletion of the registry page | ✗ | ✓ (file only) | ✓ (receipt only) | ✓ | ✓ |
| The work was performed | ✗ | ✗ | ✗ | ✗ | ✗ |
| All certificates disclosed (completeness) | ✗ | ✗ | ✗ | ✗ | ✗ |
| Legal recognition | ◐ | ◐ | ✗ | ✗ | ✗ |

**Notes on the conditional marks.** † marks a capability that depends on trusting the operator not to equivocate, or on the party having cached a checkpoint before the event being proved. **C** issues a signed, TSA-timestamped receipt, but the receipt is an assertion of receipt, not an inclusion proof: commitment is ◐†, absence cannot be proven, and deletion or backfill is detectable only for an entry the party holds a receipt for. C's enumeration is a database response: it can show what it holds, but it cannot prove that the list is complete, which is why the enumeration and second-certificate rows are ◐†. **D1** adds Merkle inclusion, non-inclusion and consistency proofs, but they are against an operator-signed checkpoint. There is no witness to sign the audit attestation, so a light investigator who did not cache the checkpoint must trust the operator or run the O(N) `--audit`; its 4 archive rows are therefore ◐†, and offline verification against operator history is ✗. **D2** replaces operator trust with an honest witness majority: each witness checks a consistency proof before cosigning, so a divergent checkpoint cannot obtain 2 cosignatures, and a verifier holding a single checkpoint can rely on it. A cosigned fork is detected by comparing checkpoints. D1 and D2 both require the party to hold a checkpoint or snapshot; the difference is that D2's checkpoint does not need to have been cached before the event. Every ✓ assumes no key compromise and an honest TSA; signer authority is a limit for every option (§4.5).

**What the matrix shows.** Columns A, B and C are existing or non-log practice. D1 and D2 are the proposed register; D1 is D2 with the witness layer disabled, so the D1-to-D2 delta isolates the witness contribution. C adds trusted commitment and a database enumeration to the registry pattern, but the enumeration is not a proof of completeness, absence and backfill cannot be proven, and tamper evidence is limited to receipted entries. D1 adds the log, the checkpoints and the subject index; its proofs are conditional because the checkpoint is operator-signed, and a post-incident investigator cannot tell a fork from a consistent history without a cached checkpoint or the O(N) audit. D2 adds the witness layer: with at least 2 of 3 honest witnesses checking consistency before cosigning, and at least 2 signing the audit attestation after verifying the projection, a divergent checkpoint cannot obtain 2 cosignatures, so a single checkpoint can be relied on, and a cosigned fork is detectable by comparison. D2 is the only arm that answers all 5 archive rows unconditionally against an untrusted operator. B + C answers every non-archive row that any option answers, with C's enumeration marked partial because it is an unproven database response. Performance and completeness are answered by none; legal recognition depends on a mandate and is absent from C, D1 and D2 because none is enacted, while A and B have some existing legal weight as signed documents.

**Trust assumptions.**

| Option | What the party must believe |
| --- | --- |
| A — signed certificate and a call | The person who answers the telephone |
| B — signed DCC under PKI | The CA bound the key, the key was not compromised before revocation, and the TSA is honest |
| C — registry with a signed receipt | The operator's account of its own contents; the operator does not omit an entry; the TSA is honest |
| D1 — unwitnessed log (ablation) | The operator does not equivocate, or the party cached the relevant checkpoint before the event; the index attestation is operator-signed; the TSA is honest |
| D2 — proposed register | At least 2 of the 3 witnesses are honest and each checks a consistency proof before cosigning; the party holds the relevant checkpoint or snapshot; the TSA is honest; the 2-of-3 audit attestation is checked. A cosigned fork is detectable by comparison, not by inspecting a single checkpoint |

### 5.2 Role of each existing practice

**Signed certificate and telephone call.** Free, and works when the laboratory is reachable and honest. It fails when the laboratory is defunct, foreign or denying, and it leaves no record for the next party.

**DCC and signed PDFs.** They provide authenticity and integrity, and DCC adds machine-readability. Under PKI they also bind the key to the laboratory and give non-repudiation. They cannot answer a question about a population: what else exists, and what is missing.

**Central registry.** The closest alternative, and the one the proposal must beat. With a signed receipt (C), a registry can evidence receipt of a certificate and can enumerate what it holds; the enumeration is a database response, not a proof of completeness, and absence cannot be proven. Deletion or backfill is detectable only for an entry the party holds a receipt for. With a Merkle log and checkpoints (D1), it can produce inclusion, non-inclusion and consistency proofs, but they are against an operator-signed checkpoint: a party who did not cache the checkpoint must trust the operator or run the O(N) audit, and the operator can still equivocate. The log is a product change; the witnesses are the trust change.

**Certificate Transparency.** The deployed mechanism for TLS. It logs certificates, binds a timestamp to issuance and makes forks detectable through monitors and gossip. The mechanism is not novel; the object, the subject index and the profile are. D1 is CT without the monitor layer; D2 replaces that layer with a small witness set.

**When a central registry is enough.** A statutory registry with manual verification is sufficient where the record keeper is trusted, disinterested and the only party that verifies. It is cheaper to build, and it needs no witnesses. The proposal is aimed at the case where at least 1 condition fails: the keeper has an interest in the incident, the verifier is in another jurisdiction, or the question is what the archive does not contain. Outside those cases the log is overhead, and the dissertation says so rather than arguing for the mechanism everywhere. The DSSCU register under Law No. 14/2022 is a statutory register of lift inspection declarations, not a register of the calibration certificates beneath them. It illustrates the statutory-record pattern; the dissertation does not treat it as a verified baseline for this certificate class or as a description of current calibration-certificate practice in Macau.

### 5.3 Summary of the comparison

Existing practice answers "a key signed this document". C answers "the registry received it", if the registry is trusted. D1 adds inclusion and non-inclusion proofs, but against an operator-signed checkpoint: the proofs are only as good as the operator's willingness not to equivocate, and a post-incident investigator who did not cache the checkpoint must trust the operator or run the O(N) audit. D2 makes the checkpoint non-equivocal: with at least 2 of 3 honest witnesses checking consistency before cosigning, a divergent checkpoint cannot obtain 2 cosignatures, and a cosigned fork is detectable by comparison. The contribution is the object, the subject index with absence proofs made trustworthy by an honest witness majority, the profile and the measurement, not the log, the receipt, the timestamp or the witness mechanism alone.

### 5.4 Existing-practice risk

A central registry with a Merkle log is an incremental product change; D1 is that change, and it is an ablation of D2, not existing practice. A "CT for certificates" framing is a domain transfer; D2 is that transfer plus a witness layer. The proposal relies on the measurement: how much of the provenance and accountability questions each option answers, at what proof size, with how many anchors. The dissertation states this, and does not claim novelty for the log, the receipt, the timestamp or the witnesses. D1 is the ablation that isolates the witness contribution: it is D2 with the witness check disabled.

---

## 6. Scope of the project

### 6.1 In scope

- **Object:** the issuance register of a log operator, which may be the regulator or an accreditation body.
- **Certificate class:** ISO/IEC 17025 calibration certificates for permanently installed lift brake-force sensors (the evaluated family), with overload and pressure sensors as boundary cases.
- **Artefact:** the issuance receipt, embedded in a DCC-compatible envelope (field names and structure, not full v3.3.0 schema validation).
- **Mechanism:** Merkle log, subject index, signed allow-list, signed checkpoints at a published maximum interval, witness cosignatures, offline verifier and archive query. The required-period schedule is a query input, not a register output.
- **Evaluation:** simulated operator, synthetic certificate data, 10 core attacks, 3 baseline arms (A, B, C), the D1 ablation and the proposal D2.
- **Scenario:** a Macau investigator after a lift incident, a foreign or defunct laboratory, certificates from 2020 to 2024, and a missing year.

### 6.2 Out of scope

- The accreditation-scope question. It is the subject of `CNP-003` and is recorded as future work in §7.3.
- Fire test reports, lift inspection certificates and structural reports as evaluated objects.
- ISO/IEC 17021, 17065, 17020, 17024 and 15189 certificates.
- Calibration parameters and sensor types other than brake force.
- Full DCC v3.3.0 schema validation.
- Certificate contents in the archive. The register holds hashes and metadata.
- Privacy-preserving publication beyond the stated visibility model.
- Key transparency and algorithm agility.
- The truth of the work, the competence of the laboratory, signer authority and legal recognition.
- A real operator integration and a live public log.

### 6.3 Why calibration certificates

| Standard | Object | Existing register | Assessment |
| --- | --- | --- | --- |
| **17025 calibration** | certificate, parameter, range | **none global** | **chosen** |
| 17021 | management-system certificate | IAF CertSearch, mandated | covered by a mandated register |
| 17065 | product certificate | NANDO (EU) | mandated, coarser |
| 17020 | inspection report | none | viable; thinner document trail |
| 15189 | medical result | national | clinical privacy |

ISO/IEC 17025 has 3 properties that place it first. Its certificates travel across borders and are relied on years later. No mandated global register covers them. The issuing laboratory is often small, foreign or defunct, which is exactly when the record is needed.

An installed brake-force sensor is the instance because the asset is fixed, the record is simple and the certificate is short. The mechanism generalises; the evaluation does not.

### 6.4 Assumptions

1. The log operator, whether the regulator or an accreditation body, is treated as untrusted for historical integrity. Safety rests on at least 2 of the 3 witnesses being honest and each checking a consistency proof before cosigning, not on the operator's neutrality. The dissertation simulates 1 operator and 3 witnesses; D1 is the same simulation with the witness check disabled.
2. Laboratories can submit a leaf at issuance; the demonstration uses a synthetic submission.
3. Witnesses can be sufficiently independent; the evaluation tests collusion.
4. A mandate can be produced by 1 of the 3 routes in §3.5; the proposal states this as a condition, not a fact.
5. Public DCC and certificate structures are representative enough to construct synthetic data.
6. The visibility model is chosen at design time (deliverable C1): public leaves, or regulator-only leaves with public checkpoints.
7. The required-period schedule is supplied with the query and its source is named. The register proves absence relative to it and does not establish it.

### 6.5 Falsifiers and gates

Items 1, 2, 3 and 5 stop the project. Item 4 narrows the claim. The gates decide which.

1. **The limitation is removed.** An authority or industry body already publishes a verifiable, append-only register of safety-critical certificates with a subject index.
2. **The mechanism adds no property.** A signed DCC under PKI with a timestamp plus a central registry with signed receipts (C) answers the same questions at lower cost. The project stops: no narrower claim survives. D1 is an ablation, not a competitor; if D2 matches D1 on every archive row, the witness layer adds nothing and R1 fires.
3. **The archive cannot be populated.** No route in §3.5 produces a mandate, and unlogged certificates dominate. The scenario has no setting.
4. **The object is unworkable.** Certificate volumes or retention requirements make the log or the index impractical.
5. **Provable absence misleads.** A gap in the archive is routinely read as "the work did not happen" when it means "the work was not logged", "the result was logged under another subject" or "the result never became a certificate", and the design cannot correct the inference.

**Gates.** G-A follows design and asks whether the receipt, the index and the verifier are tractable. G-B follows evaluation and asks whether the mechanism provides a property the baselines do not. A failed gate requires re-scoping or stopping.

---

## 7. Deliverables

The evaluation compares 5 arms: **A**, current practice; **B**, a signed DCC under X.509 with an RFC 3161 token; **C**, a central registry with an index and a signed receipt; **D1**, C plus a Merkle log, checkpoints and a subject index, with the witness cosigner disabled; **D2**, the proposed register. The C and D1 arms are configurations of the same implementation; the D1 arm runs with the witness check disabled.

### 7.1 Core deliverables

| ID | Deliverable | Acceptance criterion |
| --- | --- | --- |
| **C1** | Design basis: evidence note, requirements, threat model and wire format | The §2.4 evidence is confirmed; the limitation check is recorded; the leaf, checkpoint, allow-list, index, receipt and audit-attestation formats are specified with type discriminators; the freshness rule, the maximum checkpoint interval, the required-period schedule input and the visibility model are defined; the retention requirement is stated |
| **C2** | Register log service | A leaf can be submitted, admitted against the allow-list, appended and proven included; the subject index updates and its root enters each checkpoint; a deleted or altered leaf breaks a consistency proof |
| **C3** | Witness cosigner | An independent process verifies a consistency proof and cosigns checkpoints; a 2-of-3 threshold is enforced with at least 2 of 3 honest witnesses assumed; an un-cosigned fork is prevented and a cosigned fork is detected by comparison in the evaluation |
| **C4** | Receipt assembler | A certificate receives a receipt with the leaf, the inclusion proof, the cosigned checkpoint and the TSA token; the envelope is DCC-compatible (subset) |
| **C5** | Field verifier | A command-line tool and a minimal web view return a report whose primary field is a named outcome: `issued_and_logged`, `not_logged`, `log_entry_mismatch`, `backdated`, `checkpoint_invalid`, `witness_insufficient`, `history_rewritten`, `fork_detected`, `key_not_admitted` and `certificate_tampered` |
| **C6** | Archive query and gap analysis | An investigator tool has 2 modes. `--query` takes a required-period schedule with a named source and returns the leaves for a subject and period range, each with an inclusion proof at a checkpoint at or after the range end plus 1 cadence. For every period in the supplied schedule it also returns the state at the first checkpoint after that period's deadline: a non-inclusion proof if the period was empty, or an inclusion proof if it was not. It verifies the proofs in O(log N) against the cosigned index root, checks the 2-of-3 audit attestation for that root (signed by at least 2 of the 3 witness keys), and reports `present`, `absent`, `backdated` or `unverifiable`, with `not required` as a schedule attribute and the schedule source named. An empty period at the deadline that holds a leaf at query time is reported as `backdated`. It returns `log_entry_mismatch` when a returned leaf or the stored log fails to verify against the cosigned checkpoint root. When a checkpoint, proof or backfilled entry fails verification, it returns the corresponding named failure from C5: `backdated`, `checkpoint_invalid`, `witness_insufficient`, `history_rewritten` or `fork_detected`. Without a supplied schedule it reports presence and enumeration only and makes no gap claim. `--audit` rebuilds the index from the log in O(N), compares the roots and produces the projection result; at least 2 of the 3 witnesses sign the attestation, each verifying the projection. The query never downloads the log and returns no unproven assertion |
| **C7** | Evaluation harness and results | 5 arms against 10 core attacks — forged certificate, tampered certificate, backdated certificate, repudiation, deleted entry, rewrite, fork, missing witness, filled gap and backfill — report detection, latency, proof size and trust-anchor counts. The C and D1 arms are configurations of the same implementation; the D1 arm runs with the witness check disabled, so the D1-to-D2 delta isolates the witness layer. Remaining threats are reported as limitations |
| **C8** | Dissertation | Problem, evidence, design, implementation, evaluation, boundary, related work and conclusion |
| **C9** | Demonstration | The lift incident end to end: issue and log 2 years, skip the next, dissolve the laboratory, investigate in 2026, verify each receipt and prove the missing year |
| **C10** | Reproducibility package | Code, synthetic data generator, scripts and README; 1-command reproduction without containerisation or CI |

### 7.2 Demonstration design

**Purpose.** The demonstration shows 1 case end to end. C7 measures 5 arms and 10 attacks.

**Scenario and cast.**

| Role | In the demonstration |
| --- | --- |
| Log operator | A simulated operator with a fictional name, which may be the regulator or an accreditation body; integrity rests on the witnesses, not on the operator |
| Calibration laboratory | "Pearl River Calibration" (fictional), accredited outside Macau and dissolved in 2024 |
| Holder | A lift-maintenance company that commissioned the calibrations |
| Investigator | A Macau safety authority after a brake incident in 2026 |
| Field verifier | An insurer's risk engineer, offline, checking 1 certificate |
| Witnesses | 3 independent cosigners drawn from the regulator, an insurer consortium and an accreditation body, threshold 2-of-3, each verifying a consistency proof before cosigning |
| Auditor | At least 2 of the 3 witnesses: one runs `C6 --audit`, and both verify the projection before signing the 2-of-3 attestation |
| Time authority | A TSA issuing RFC 3161 tokens |

**The case.** A brake-force sensor BF-4471 is permanently installed on lift L-4471. The required-period schedule supplied to the query is the manufacturer's 2-year calibration interval for this sensor, anchored at the 2020 commissioning calibration and stated as the pairs 2020, 2022 and 2024; it is a declared input, not a finding of the register. Each certificate names BF-4471 as the calibration item and L-4471 as the subject reference; both are covered by the certificate hash. The laboratory logs the 2020 and 2022 calibrations. Under the declared schedule the 2024 calibration is due, but the owner skips it and no certificate is issued; the laboratory dissolves later that year. In 2025 the building changes insurer. The new insurer's engineer checks the 2022 receipt offline, flags the missing 2024 certificate, and binds the policy with a condition to produce it within 30 days. The owner claims the certificate was lost in the handover and stalls. In 2026 a brake incident occurs, and the investigator queries the archive for L-4471.

**The bundle.** The field verifier receives 3 items:

1. the calibration certificate, DCC-compatible, signed by the laboratory;
2. the issuance receipt, defined in §4.1;
3. the trust-anchor identifiers; the verifier resolves them against a locally pinned set.

The investigator holds the same anchors and queries the archive.

**Happy path.**

1. **2020–2022 — issue and log.** Each certificate is signed, submitted and logged. Checkpoints CP-2144 and CP-2387 commit the leaves and the subject index.
2. **2024 — no certificate.** Under the declared schedule the calibration is due, the owner skips it, and no certificate is issued. Checkpoint CP-2500, dated 2025-01-05, records the state at the deadline.
3. **2026 — investigate.** The authority queries L-4471 for 2019–2025. The archive returns 2 leaves with inclusion proofs, a non-inclusion proof for 2024 at query time against CP-3100, and the deadline state at CP-2500 (non-inclusion). Each proof is checked against the pinned anchors.

The expected output:

```text
subject                   L-4471
period                    2019-01-01 to 2025-12-31
schedule                  manufacturer interval, 24 months (supplied)
query_checkpoint          CP-3100, 2026-01-15, witnesses 2/3
2019                      not required
2020                      present, CP-2144, witnesses 2/3
2021                      not required
2022                      present, CP-2387, witnesses 2/3
2023                      not required
2024                      absent, non-inclusion at CP-2500 (2025-01-05) and CP-3100
2025                      not required
gap                       2024
laboratory                Pearl River Calibration, dissolved 2024
```

The output separates 3 results: the certificate is present and genuine; the certificate is absent; the archive cannot decide. `not required` is a schedule attribute, not a result. The `schedule` line names the source, so the gap is read against a declared requirement rather than an assumed one. The absence is proven, which is what a pile of PDFs cannot do.

**Attack variants.** Each variant is a scripted change to the same archive or bundle.

| Variant | Change | Expected outcome |
| --- | --- | --- |
| **Forged certificate** | A document with the same letterhead, signed by an unadmitted key | `key_not_admitted` |
| **Tampered certificate** | 1 certified value is edited | `certificate_tampered` |
| **Backdated certificate** | A certificate created and signed in 2026 claims a 2022 calibration date | `backdated`; B detects this through the TSA token |
| **Repudiation** | The laboratory denies issuance; the receipt is presented | `issued_and_logged`, with the checkpoint as the answer |
| **Deleted entry** | The operator deletes a logged leaf from its storage while keeping the cosigned checkpoint | `log_entry_mismatch`, on the query path |
| **Rewritten history** | The operator publishes a later checkpoint with a different root | `history_rewritten` |
| **Fork** | 2 checkpoints at the same tree size with different roots, both cosigned | `fork_detected` |
| **Missing witness** | 1 cosignature is removed | `witness_insufficient` |
| **Filled gap** | The laboratory logs a 2024 certificate for work it did not do | `present`, with a note that the performance limit applies |
| **Backfill after incident** | A certificate genuinely signed and token-stamped in 2024 is withheld, then submitted in 2026 | The CP-2500 non-inclusion and the CP-3100 inclusion together return `backdated`. B accepts it; C records a late receipt but cannot prove the archive was empty at the deadline; D1 can, if the investigator cached the deadline checkpoint; D2 can, because the checkpoint is cosigned |

The fork variant is the case where the witness layer is decisive: an honest majority prevents an un-cosigned fork, and a cosigned fork is detected by comparison. The rewritten-history and backfill variants are detectable by D1 only when the party cached the relevant checkpoint before the incident; D2 does not require a pre-event cache, because the checkpoint is cosigned. The demonstration allocates most of its time to these 3 variants.

**What the demonstration does not show.**

- That the work was performed or the result is accurate.
- That the archive contains every certificate the laboratory issued.
- That the individual signer was authorised by the laboratory.
- That the certificate or the archive has legal recognition.

**Staging and success criteria.**

- The field verification runs with the network disabled; the archive query uses a local snapshot of the archive.
- 2 surfaces consume the same anchors: the command-line verifier and the investigator report.
- All names are fictional; no real laboratory, insurer or authority is named.
- A single `demo.sh` provides the subcommands `issue`, `verify`, `forge`, `tamper`, `backdate`, `deny`, `delete`, `rewrite`, `fork`, `witness`, `gap` and `backfill`, so any variant runs in 1 command.
- Duration: 8–10 minutes live, or 4 minutes recorded.
- Success criteria: the happy path returns `issued_and_logged` in under 1 second; the gap case returns a verified non-inclusion proof; each attack returns its specific outcome; the run uses only the bundle, the snapshot and the pinned anchors.

### 7.3 Future work: the scope extension

The same log can carry a second leaf type, `scope_event`, with a per-entry hash chain and a map from entry to chain head, as `CNP-003` specifies. That extension answers the second question in §3.3: whether the laboratory was authorised on the certificate date.

It is deferred for 3 reasons:

1. It requires the accreditation body to operate the log itself, a different actor from the log operator proposed here.
2. It adds a predicate model, a state machine and a backdating rule, which is the larger half of the design.
3. It depends on a limitation check of its own: whether any accreditation body already publishes a versioned scope history.

The 2 proposals share the log, the checkpoints, the witnesses and the verifier. A combined design would carry both leaf types under one receipt.

### 7.4 Exclusions

The project does not deliver a production register, a live public log, a legal opinion, regulatory acceptance, or a certificate-content registry. The scope exclusions in §6.2 apply.

---

## 8. Schedule

### 8.1 Assumptions

Week 1 begins when supervisor approval is recorded. The plan uses weeks; no calendar dates are assumed. Week 1 covers approval and set-up; no implementation begins before approval.

**Effort.** The plan uses 12–15 hours per week on average, with peak weeks at 16–18 hours, for about 230–270 hours in total (the plan uses 250 hours). These are planning estimates, not measurements.

### 8.2 Plan

| Phase | Weeks | Indicative effort | Activities | Output | Milestone |
| --- | --- | --- | --- | --- | --- |
| **P0 — Approval** | W1 | ~8 h | Confirm direction, module constraints, scope and ethics | Signed-off proposal | **M0 — approval recorded** |
| **P1 — Design** | W2–W5 | ~54 h | Limitation check; requirements; threat model; leaf, index and receipt formats; allow-list; freshness rule; maximum checkpoint interval; required-period schedule; visibility model; chapter draft | C1 | **M1 / Gate G-A** |
| **P2 — Implementation** | W6–W12 | ~84 h | Log; subject index; allow-list; witness cosigner; receipt issuer; field verifier; archive query; demonstration data; chapter draft | C2–C6 | **M2 — end-to-end slice** |
| **P3 — Evaluation** | W13–W15 | ~34 h | 10 attacks; 5 arms (the C and D1 arms are configurations); gap case; metrics; analysis; chapter draft | C7 | **M3 / Gate G-B** |
| **P4 — Writing and demonstration** | W16–W19 | ~62 h | Integrate and revise the dissertation; demonstration script; draft to supervisor by end W17; revisions in W18–W19 | C8, C9 | **M4 — revised draft and demonstration ready** |
| **P5 — Finalisation** | W20 | ~8 h | Final corrections; reproducibility package; submission | C10 | **M5 — submission** |
| **Total** | **20 weeks** | **~250 h** | — | — | — |

### 8.3 Critical path and dependencies

**Critical path.** Leaf, index and receipt format → log and index → field verifier and archive query → evaluation. The witness cosigner runs in parallel.

**Buffers.** 2 weeks are held within P2 and P4. If P2 slips by more than 2 weeks, the web view and the retention analysis are deferred.

**Dependencies.** Supervisor availability at M1, M3 and the W17 draft review; a decision on the DCC envelope and the visibility model by M1; synthetic certificate data by W5.

---

## 9. Risk assessment

### 9.1 Principal risks

| # | Risk | Likelihood | Impact | Mitigation | Trigger |
| --- | --- | --- | --- | --- | --- |
| **R1** | The increment over existing practice (B + C) is too small, or the witness layer adds nothing over the D1 ablation | Medium | High | Measure B + C → D2 on absence, tamper, backfill and non-equivocation; use D1 to isolate the witness layer (D1 ◐† → D2 ✓) | D2 matches B + C on every row, or D2 matches D1 on every archive row |
| **R2** | The mandate is not adopted within the project horizon | Medium | High | Argue the 3 routes; use the precedent class; evaluate under a stated mandate | No route is plausible by M1 |
| **R3** | Certificate volume makes the log or index impractical | Low | Medium | Restrict to installed safety sensors; model volume; report checkpoint and index cost | Volume exceeds the pilot estimate by an order of magnitude |
| **R4** | Privacy: the index links certificates to assets | Medium | High | Choose the visibility model at design time (deliverable C1); regulator-only access with public checkpoints; state the limit | The scenario requires public asset data |
| **R5** | Novelty: "CT for certificates" | Medium | High | Frame the contribution as the object, the index, the profile and the measurement; cite CT | A reviewer identifies the mechanism as the contribution |
| **R6** | A proven gap is read as proof that the work did not happen | Medium | Medium | Distinguish `absent` from `not performed` in every output and in the dissertation | An examiner reads the gap output as a finding of fact |
| **R7** | Key compromise or a key that is never revoked | Medium | Medium | Allow-list revocation; short-lived keys; state the limit | A demo attack succeeds through a stale key |
| **R8** | Legal or regulatory overclaim | Low | High | Include the boundary; make no legal claim; review Macau references | A reviewer identifies a legal claim |
| **R9** | Timeline and scope growth | Medium | High | Fix the feature set; defer future work | M2 slips by more than 2 weeks |
| **R10** | Direction risk: approval not recorded | High | High | This proposal is the decision document; no implementation before approval | Approval not recorded by W1 |

### 9.2 Principal risks in detail

**R1 — the increment.** The strongest objection is that existing non-log practice (B + C) already answers most of the question: authenticity, key binding, signing time, non-repudiation and enumeration (as a trusted assertion). What it cannot do is prove absence, prove that an enumerated list is complete, provide tamper evidence beyond receipted entries, backfill, or non-equivocation. D1 is the ablation: it adds Merkle proofs, but they are against an operator-signed checkpoint, and without a witness-signed audit attestation a post-incident investigator must trust the operator or run the O(N) audit. D2 adds the witness layer: at least 2 of 3 honest witnesses check consistency before cosigning, and at least 2 sign the audit attestation after verifying the projection, so a divergent checkpoint cannot obtain 2 cosignatures and a single checkpoint can be relied on. The evaluation measures B + C → D2 (the archive proof layer) and D1 → D2 (the witness layer). If either delta is zero, R1 fires.

**R2 — the mandate.** No Macau instrument requires logging calibration certificates today. The proposal argues 3 routes and a precedent class, and evaluates under a stated mandate. If no route is plausible, the project re-scopes to the simulated setting and the claim is limited to it; falsifier 3 applies only if unlogged certificates dominate in that setting.

**R4 — privacy.** A public index of assets is a map of buildings, lifts and inspection history. The visibility model is a design-basis decision (deliverable C1): public leaves for monitorability, or regulator-only leaves with public checkpoints. The dissertation states the choice and its cost.

**R5 — novelty.** Certificate Transparency is deployed and Sigstore extends it to software artefacts. The dissertation does not claim the mechanism. The contribution is the object, the subject index with absence proofs made trustworthy by an honest witness majority, the profile and the measurement. The log, the receipt, the timestamp and the witnesses are existing mechanisms; D1 is the ablation that isolates the witness layer, and the dissertation says so.

**R6 — the gap inference.** A non-inclusion proof says the archive has no entry. It does not say the calibration did not happen. Every output labels the 3 states separately, and the dissertation states the inference limit in the same section as the gap result.

### 9.3 Ethical and legal notes

- The register holds no certificate contents; leaves are hashes with a laboratory, a subject reference and a date.
- The index links certificates to assets, which is personal data in some jurisdictions. The visibility model and the retention policy address it at design time (deliverable C1).
- The Macau scenarios are illustrative and labelled A; they are not allegations against a laboratory or an authority.
- The dissertation offers no legal opinion and claims no regulatory acceptance.
- The author checks references to Macau's building, lift and fire regimes with the relevant public authorities before submission.

---

*End of CNP-004 v0.6.2. This document proposes a direction. It does not record approval and does not authorise implementation.*
