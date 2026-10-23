# CNP-003 — Project Proposal: A Verifiable Accreditation-Scope Register

| Field | Value |
| --- | --- |
| Document ID | CNP-003 |
| Title | Project proposal — A Verifiable Accreditation-Scope Register |
| Version | 0.3.1 |
| Status | Draft for supervisor review. Direction not approved. No implementation has started. |
| Date | 2026-10-23 |
| Module | COMP4299 — final-year dissertation project (dissertation, demonstration and evaluation) |
| Related documents | Background only; this proposal is self-contained. `CNP-002`; `DEC-002`; `ANL-009`; `ANL-008`; `PRC-001`; `BUILD-HANDOVER-005`. |
| Purpose | Propose a direction for COMP4299: a verifiable register of laboratory accreditation scope, with offline proof of authority at issuance. |
| Supersedes | None. This proposal presents a new direction and is independent of earlier concept notes. |

**Decision requested.** (1) Approve the direction. (2) Confirm the module constraints in writing: a narrow mechanism with a measured boundary, a positive headline, and a plausible scenario as sufficient. (3) Confirm the ISO/IEC 17025 and Macau scope. (4) Confirm that no implementation begins before approval.

**Revision note for 0.3.0.** Version 0.3.0 responds to 2 independent audits of 0.2.0. It adds an issuance attestation that binds the certificate hash to a checkpoint and a TSA time, a freshness rule that bounds that time to the certificate date, per-event inclusion proofs at each first-inclusion checkpoint, and a deterministic replay order. It replaces the 6-state machine with 6 event types over 3 states and a state-transition table. The comparison now includes an embedded-snapshot baseline (B′), so the design claims 1 unique property, non-equivocation, not 2. It unifies the claim across §2.3, §4.7 and §4.8, corrects 3 facts, and fixes the outcome and attack lists. Sections 2–9 and the terms change.

**Revision note for 0.3.1.** Version 0.3.1 changes the status of the mandate. It was a risk in §9 and a precondition in §4.8 at the same time. It is now a condition of operation, stated once in §4.8, with its consequence in §4.6 and its plausibility monitored as R1 at Medium. §4.7 adds the condition to the claim. The direction, mechanism and schedule are unchanged.

## Terms

| Term | Meaning |
| --- | --- |
| **Accreditation** | Formal recognition by an accreditation body that a conformity assessment body is competent to perform specific activities. |
| **Accreditation body** | The organisation that assesses conformity assessment bodies and publishes their accreditation. Examples: UKAS (United Kingdom), CNAS (China), HKAS (Hong Kong). |
| **Conformity assessment body** | A body that performs testing, calibration, inspection, certification or verification. Here: a calibration laboratory. |
| **Scope of accreditation** | The activities a laboratory is accredited to perform: parameters, ranges, methods, uncertainties and locations. |
| **Schedule of accreditation** | The accreditation body's published record of a laboratory's scope, with an issue number and issue date. |
| **Calibration certificate** | The document a laboratory issues after a calibration, and on which a verifier relies. |
| **Verifier** | The party that relies on a calibration certificate and decides whether to accept it. |
| **Register log** | An append-only, verifiable record of scope events, with signed checkpoints. |
| **Merkle tree** | A hash tree that supports compact proofs of inclusion and consistency. |
| **Checkpoint** | A signed commitment to the state of the register log at a point in time, including the root of the scope map. |
| **Covering checkpoint** | The checkpoint at which the receipt proves the chain head. |
| **First-inclusion checkpoint** | The checkpoint that first includes a scope event in its tree. |
| **Scope entry** | The register's record of 1 laboratory, activity and method, identified by `entry_id`; it persists across events. |
| **Scope event** | 1 signed statement that changes a scope entry: grant, extension, reduction, suspension, reinstatement or withdrawal. |
| **Entry chain** | The hash-linked sequence of events for 1 scope entry, in which each event commits to its predecessor. |
| **Chain head** | The most recent event for a scope entry at a checkpoint. |
| **Scope map** | A Merkle map from `entry_id` to chain head, whose root appears in each checkpoint. |
| **Effective state** | The state of a scope entry on a given date, derived by replaying its chain up to that date. |
| **State** | `active`, `suspended` or `withdrawn`; the 6 event types move an entry between states. |
| **Backdating rule** | A grant, extension or reinstatement takes effect no earlier than its first-inclusion checkpoint; a reduction, suspension or withdrawal may take effect earlier. |
| **Timeliness** | The property that the register logs an event within 1 checkpoint of its effective date. |
| **Scope receipt** | A signed proof of a scope entry's chain and head at a checkpoint, bound to a certificate hash. |
| **Witness** | An independent party that cosigns a checkpoint. |
| **Fork** | 2 divergent histories presented by the same log operator to different verifiers. |
| **Non-equivocation** | The property that a fork is detectable. |
| **Time-stamping authority (TSA)** | A service that signs a token proving that a datum existed at a given time (RFC 3161). |
| **Status list** | A published list of the current status of many certificates or credentials (W3C Bitstring Status List; IETF Token Status List). |
| **COSE, CBOR, CWT** | Standards for signed messages (COSE, RFC 9052), binary encoding (CBOR) and token claims (CWT). |
| **ISO/IEC 17025** | The international standard for the competence of testing and calibration laboratories. |
| **ISO/IEC 17021** | The international standard for bodies that audit and certify management systems. |
| **IAF CertSearch** | The global database of accredited management-system certifications, mandated by IAF Mandatory Document 28. |
| **DCC** | Digital Calibration Certificate: a machine-readable, signed calibration certificate format developed under PTB (Physikalisch-Technische Bundesanstalt), the German national metrology institute. |
| **SCITT** | Supply Chain Integrity, Transparency and Trust: the IETF architecture for signed statements and transparency services (RFC 9943, RFC 9942). |
| **Certificate Transparency (CT)** | The register-log system for TLS certificates. Version 2 is defined in RFC 9162 and obsoletes RFC 6962, which the WebPKI logs deploy. Its Signed Certificate Timestamp (SCT) is a commitment to inclusion made at issuance. |
| **Multilateral Recognition Arrangement (MLA)** | The IAF/ILAC arrangement under which accreditation bodies recognise each other's accreditations. |
| **Evidence labels** | **V** = verified against a primary or official source; **P** = partial or secondary evidence; **A** = analytical or not yet supported by a documented case. |

---

## 1. Project title

> **A Verifiable Accreditation-Scope Register**
> *Offline proof that a laboratory was accredited for a test when it issued the certificate*

### 1.1 Definition

A register of the calibrations each laboratory is accredited to perform, held by the accreditation body, with a verifiable history.

### 1.2 Title rationale

The title names the object (accreditation scope), the artefact (register) and the capability (verifiable). It omits the standard number (ISO/IEC 17025, §6), the protocol stack (SCITT, RFC 9162, COSE) and the mechanism (Merkle log, witnesses, scope receipts), which belong in §4 and §6.

---

## 2. Problem statement

Accreditation is formal recognition by an accreditation body that a conformity assessment body is competent to perform specific activities. The scope of accreditation lists those activities: parameters, ranges, methods, uncertainties and locations. The accreditation body publishes the scope in a schedule of accreditation. A calibration certificate is the document a laboratory issues after a calibration, and the document on which a verifier relies.

The verifier must establish 3 things: whether the laboratory was accredited for this calibration, whether the accreditation covered the certificate date, and whether the record is the one the accreditation body published.

The certificate answers none of the 3. The laboratory's signature proves that the laboratory signed the certificate; it does not prove that the laboratory was authorised to sign it. The authorisation resides in a register held by the accreditation body.

The register exists, is public and is authoritative. It reports current state only.

### 2.1 The register as it operates today

| Property | Current practice | Consequence |
| --- | --- | --- |
| **Publication** | The accreditation body publishes a directory and a schedule of accreditation per laboratory, with an issue number and issue date | The register shows the current scope |
| **Revision** | Schedules "may be revised and reissued" in place | The previous version is not published |
| **Withdrawal** | UKAS states that "absence of a schedule … indicates that the accreditation is no longer in force" | The register signals withdrawal by deletion; the deletion is undated |
| **Expiry** | UKAS accreditation certificates "do not carry expiry dates"; the current schedule is the proof of validity | The laboratory's authority depends on a page that may no longer exist |
| **Granularity** | The scope is specific to parameter, range, method and location | The word "accredited" does not identify the calibration |
| **Partial sanction** | The accreditation body can suspend the whole scope or "a specific location or technical activity/scope" | A binary status record cannot represent a partial loss |

The register has 4 limitations:

1. **No history.** It cannot answer a query about the scope on 2024-03-03.
2. **No scope-level answer.** A laboratory can hold accreditation and lack accreditation for the calibration in the certificate. UKAS states that a body "may hold accreditation but it may not include the service … relevant to the product or service being provided."
3. **No reachability in all cases.** The laboratory can be foreign, defunct or unwilling; the accreditation body can be in another jurisdiction. The verifier's controls are a telephone call and a website.
4. **No non-revision.** The accreditation body can revise, reissue or delete the record after the verifier relies on the certificate. The verifier cannot determine whether the record is the one that existed at issuance.

### 2.2 Additional evidence

The sources for §2.1 are the UKAS FAQ and UKAS GEN-1. The table records evidence not stated in §2.1.

| Fact | Source | Status |
| --- | --- | --- |
| IAF CertSearch is mandated by IAF Mandatory Document 28 for ISO/IEC 17021-1 management-system certificates, with real-time status alerts and about 2.5 million certifications | IAF (International Accreditation Forum) | **V** |
| No equivalent global register covers ISO/IEC 17025 calibration certificates | IAF CertSearch scope; national directories | **V** (scope), **P** (absence) |
| UKAS publishes counterfeit-certificate warnings and records of false accreditation claims | UKAS counterfeit-certificates page | **V** |
| UKAS suspended all scopes of the BBA (British Board of Agrément) on 2026-02-26; certificates that the BBA issued under those scopes remain valid until their expiry dates | UKAS statement | **V** |
| TWI Certification's ISO/IEC 17024 accreditation was suspended and reinstated in 2026 | UKAS and TWI Certification statements | **V** |
| The Grenfell Tower Inquiry's Phase 2 report (2024-09-04) recorded that fire-test results were withheld from the certifier and recommended attaching supporting test results and the full testing history to certificates | Grenfell Tower Inquiry, Phase 2 | **V** |
| In the PIP (Poly Implant Prothèse) case, TÜV Rheinland suspended the certificates it had issued when fraud surfaced; liability litigation followed | Court and TÜV Rheinland statements | **V** |
| A documented adjudicated dispute on the scope date | Not found | **A** |

**Evidence labels.** **V** = verified against a primary or official source; **P** = partial or secondary evidence; **A** = analytical or not yet supported by a documented case.

### 2.3 The gap

> No method lets a verifier, offline and years later, establish that a calibration certificate fell inside the issuing laboratory's accreditation scope on the certificate date, and that the register's record of that scope cannot be altered or forked without detection.

The gap produces 2 costs. Over-trust accepts an authority claim the verifier cannot check. Under-trust rejects a valid certificate and repeats the calibration. Both carry safety and cost consequences.

---

## 3. Significance of the problem, with reference to Macau

### 3.1 General significance

Verifiers rely on calibration certificates years after issuance: an insurer assessing a building, a court apportioning liability, a regulator admitting a product, a buyer accepting a shipment. The verifier rarely commissioned the calibration and often has no relationship with the laboratory.

Without a check on authority, 1 of 2 outcomes follows. The verifier treats the certificate as self-proving, which rewards a laboratory that overstates its scope. Or the verifier rejects the certificate, which penalises a compliant laboratory and repeats the work. Both outcomes carry safety and cost consequences.

### 3.2 The Macau SAR context

Macau is a small, dense, high-rise jurisdiction with an ageing building stock and a construction and tourism economy that depends on third-party conformity assessment. 3 features make the verification problem more frequent here:

- **The laboratory is often elsewhere.** Laboratories accredited outside Macau test construction materials, fire-rated products, lifts and calibration artefacts. The verifier in Macau checks a foreign register it has no right to audit.
- **The reliance is years later.** Periodic building inspection, lift and escalator safety, fire safety and insurance renewal re-open certificates issued long before. The building changes owner; the insurer changes; the inspector leaves; the scope changes.
- **The authorities are not the issuer.** The public works, fire and municipal authorities, the courts and insurers are verifiers. They can require a document; they cannot audit the accreditation body.

3 illustrative scenarios (A: plausible, not documented as Macau cases):

1. **Lift maintenance.** A torque wrench used to verify lift brake torque is calibrated in 2021 by a calibration laboratory accredited outside Macau. In 2026 the laboratory's accreditation is suspended, the building changes insurer, and the insurer asks whether the 2021 calibration was inside the laboratory's scope. The laboratory is in another jurisdiction; the register shows current state; the technician who commissioned the calibration has left.
2. **Building safety.** A batch of fire-rated cladding is tested in 2022 by a laboratory accredited in another jurisdiction, and the report supports a building approval. In 2026 a fire-safety review asks whether the laboratory was accredited for that test method at the time. The register has been revised; the scope has changed; there is no dated history.
3. **Construction materials.** A concrete or steel test report from a Mainland laboratory supports a structural submission. The laboratory's accreditation is later reduced. The verifier must decide whether the report was inside the scope at issuance, without calling the laboratory and without access to the accreditation body's historical records.

The scenarios illustrate a general problem: the issuer, the register and the verifier are frequently in 3 jurisdictions. They are labelled A; the dissertation states that they are illustrative.

### 3.3 Fit with the module

The module accepts a narrow mechanism with a measured boundary, requires a positive headline, and accepts a plausible scenario (stated 2026-10-23; written confirmation is decision (2)). The proposed register meets the 3 conditions: the mechanism is narrow, the result is a capability, and the Macau scenarios supply the setting without a recruited stakeholder.

---

## 4. Proposed solution

### 4.1 Object and artefact

The object is the accreditation-scope register: the accreditation body's authoritative list of which laboratory may perform which calibration, under which method, over which range, at which location and in which state.

The artefact is the scope receipt: a signed object issued when the laboratory creates a calibration certificate, which binds the certificate to the register's state at that moment. The scope receipt contains:

- the certificate hash, computed over the complete signed certificate, including the certificate date and the calibration values;
- the laboratory identifier, the scope entry and the certificate date;
- the issuance attestation: the register's signature over the certificate hash and the covering checkpoint, with an RFC 3161 token for the attestation;
- the entry chain: every event for that scope entry, from grant to chain head, and for each event its claimed first-inclusion checkpoint;
- an inclusion proof for each event at its claimed first-inclusion checkpoint;
- a scope-map proof: 1 path showing `entry_id → chain head` at the covering checkpoint;
- a Merkle inclusion proof of the chain-head event in the register log at the covering checkpoint;
- the signed checkpoint statements for the covering checkpoint and for each first-inclusion checkpoint, with witness cosignatures;
- an RFC 3161 timestamp token for each of those checkpoints;
- the verification policy: trust anchors, accepted algorithms, checkpoint cadence, freshness window and predicate rules.

This list is the canonical receipt definition; C4 and §7.2 refer to it.

The laboratory embeds the receipt in the certificate envelope at issuance. It does not add a receipt to a certificate that already exists: the issuance attestation binds the certificate hash to a checkpoint and a time, and the register signs neither retrospectively nor for a hash it did not receive.

The verifier accepts the receipt only when the covering checkpoint and the attestation token both fall in the interval from the certificate date to 1 cadence after it. A receipt outside that interval returns `receipt_stale`, not `in_scope_at_issuance`.

The pattern follows Certificate Transparency: a signed commitment at issuance, checked later. 2 differences follow from the object. The commitment covers accreditation, not the certificate. And a set-only log cannot state the effective state of 1 entry on 1 date, so the register also commits to a map from entry to chain head.

### 4.2 Flow

1. **Grant.** The accreditation body admits a scope event to the register: `entry_id`, sequence number, the hash of the previous event for that entry, the predicate (laboratory, activity, method, range, uncertainty and location), the effective date and the state. The body signs the event and appends it, then updates the scope map to the new chain head. The first checkpoint that includes the appended event becomes its first-inclusion checkpoint.
2. **Checkpoint.** The register log publishes a signed checkpoint at a defined cadence. The checkpoint carries the log root and the scope-map root. Independent witnesses cosign it.
3. **Issue.** The laboratory issues a calibration certificate and sends its hash to the register. The register returns an issuance attestation over the certificate hash and the current covering checkpoint, with a TSA token. The laboratory's tooling assembles the receipt from the attestation, the entry chain with an inclusion proof at each first-inclusion checkpoint, the scope-map proof, the inclusion proof of the chain head, the checkpoint statements and the TSA tokens. It embeds the receipt in the certificate envelope.
4. **Present.** The certificate and scope receipt travel to the verifier, possibly years later and possibly without the laboratory.
5. **Decide.** The verifier checks offline, at UTC day granularity: the certificate signature; the certificate hash recomputed over the complete signed certificate and matched to the attestation; the issuance attestation and its token against the freshness interval; the timestamps of the covering checkpoint and of each first-inclusion checkpoint; the freshness rule, which requires the covering checkpoint and the attestation token to fall in the interval from the certificate date to 1 cadence after it; the checkpoint signatures and the witness threshold; each event's inclusion proof at its claimed first-inclusion checkpoint; the scope-map proof; the inclusion of the chain-head event; every hash link in the chain; the backdating rule, which requires a grant, extension or reinstatement to take effect no earlier than its first-inclusion checkpoint; and the effective state, by replaying the chain in sequence order and applying every event whose effective date is at or before the certificate date. It applies the predicate to the calibration and returns a named result.
6. **Refresh (optional).** The verifier can query current accreditation separately. That query answers a different question and does not form part of the claim.

**States and transitions.** The register has 3 states — `active`, `suspended` and `withdrawn` — and 6 event types:

| Event | From | To |
| --- | --- | --- |
| Grant | absent | active |
| Extension | active | active |
| Reduction | active | active |
| Suspension | active | suspended |
| Reinstatement | suspended | active |
| Withdrawal | active or suspended | withdrawn |

5 rules complete the model. An extension or a reduction requires state `active`; an attempt on a suspended entry is rejected. A reduction to an empty predicate is rejected, and withdrawal is the correct event. A withdrawal closes the entry: a later grant opens a new `entry_id` and a new chain, and the old chain stays. A reinstatement restores the predicate the entry held at suspension; a change of predicate requires an extension after reinstatement. No event takes effect before the entry's grant: an event with an earlier effective date is rejected. The register has no correction event; a correction is a new event and obeys the backdating rule.

### 4.3 The 3 time questions

A verifier that merges 3 questions reaches the wrong result. The design keeps them separate:

| Time question | Question | Source |
| --- | --- | --- |
| **Scope on the certificate date** | Was this calibration inside the laboratory's scope on the certificate date? | The scope receipt and the register log |
| **Certificate lifetime** | Is this certificate within its own validity period, or has the laboratory revised or withdrawn it? | The certificate and the laboratory |
| **Current sanction** | Is the accreditation body's authorisation currently suspended, reduced or withdrawn? | A current status list, online |

The 3 questions have different answers. When UKAS suspended the BBA's accreditation, existing accredited certificates remained valid until expiry. A verifier that treats "body suspended" as "certificate invalid" reaches 1 wrong result; a verifier that ignores the suspension reaches the other.

### 4.4 Architecture

| Element | Choice | Rationale |
| --- | --- | --- |
| Register log | RFC 9162 Merkle tree, implemented with Tessera (an open-source register-log implementation) | Append-only; consistency proofs; existing implementation |
| Scope map | Sparse Merkle map from `entry_id` to chain head, with its root in each checkpoint | The map proof yields the chain head of 1 entry; the entry chain yields its state |
| Entry chain | Hash link from each scope event to its predecessor | Orders the events of 1 entry; an omitted event breaks the link |
| Scope events | COSE_Sign1 over `entry_id`, sequence, predecessor hash, predicate, effective date and state | 1 signed object per change; the chain link sits in the same object |
| First-inclusion record | The register records the checkpoint that first includes each event, and the receipt carries it | Makes the backdating rule checkable for every event |
| Issuer allow-list | Signed list of laboratory keys, committed in each checkpoint | Binds a certificate key to the laboratory |
| Scope receipts | RFC 9942 COSE receipts over the chain head and the scope-map proof | Offline inclusion and state verification |
| Statements | COSE_Sign1 (RFC 9052), canonical CBOR, CWT claims | Compact, signed, standardised |
| Time | RFC 3161 token per checkpoint | Independent time anchor |
| Certificate envelope | DCC v3.3.0 structure (PTB), with a scope-receipt extension | Reuses the deployed digital calibration certificate format |
| Current status | W3C Bitstring Status List or IETF Token Status List | Existing standard |
| Witnesses | 3 independent cosigners over checkpoints, threshold 2-of-3 | Non-equivocation without a public ledger |
| Backdating rule | Grants, extensions and reinstatements take effect no earlier than their first-inclusion checkpoint; reductions, suspensions and withdrawals may be backdated | Retroactive authority is the attack; retroactive removal is safe |
| Verifier | Offline command-line tool, plus a minimal web view for the demonstration | Named outcomes; reproducible |

SCITT (RFC 9943) supplies the architecture: signed statement, transparent statement, scope receipt and registration policy. The RFC scopes itself to software supply chains and states that the architecture applies to other supply-chain statements. A scope register is an extension.

RFC 9942 wraps a proof about 1 verifiable data structure. The receipt here covers 2: the RFC 9162 log and the scope map. Whether 1 receipt carries both proofs or 2 receipts travel together is a wire-format decision at C1.

### 4.5 Properties established

| Property | What the verifier obtains | Offline | Condition |
| --- | --- | --- | --- |
| **Inclusion** | The chain-head event is in the register log at the covering checkpoint | Yes | Merkle proof and checkpoint signature |
| **Effective state** | The scope entry's state on the certificate date | Yes | Issuance attestation, entry chain, per-event inclusion proofs, scope-map proof, freshness rule, backdating rule and the timeliness assumption |
| **Scope match** | The calibration falls inside the entry's predicate on that date | Yes | A scope predicate model |
| **Non-revision** | No event was removed or altered; every later change is an appended event | Yes, against the checkpoints in the bundle | Entry chain and consistency proofs |
| **Non-equivocation** | 2 verifiers comparing cosigned checkpoints detect a fork | Yes | At least 1 honest witness, and a verifier that checks cosignatures |
| **Self-containment** | The verifier needs neither the laboratory, the accreditation body nor a retained snapshot | Yes | A bundle that carries the attestation, the chain, the per-event proofs, the scope-map proof and the checkpoints |

**Self-containment** is shared with an embedded snapshot (arm B′); **non-equivocation** is unique to this design. Section 5.1 records the count, and §4.6 states what neither property covers.

### 4.6 Limits of the claim

- **Truth.** The register does not establish that the measurement is correct. A registered statement proves that an issuer produced it, not that it is true (RFC 9943 §9.2).
- **Completeness.** The scope map fixes the head of 1 scope entry, so the register proves the full event history of that entry. It does not prove that it contains every scope entry the laboratory holds, or that the laboratory logged every certificate. Omission of a whole entry stays undetectable (RFC 9943 §9.3).
- **Timeliness and freshness.** The chain fixes the order of events, not the promptness of logging. An event logged after the certificate date with an earlier effective date is visible to a later verifier and invisible to the offline verifier at decision time. The claim assumes that the accreditation body logs each event within 1 checkpoint of its effective date. The freshness rule bounds the other direction: the covering checkpoint and the attestation token fall in the interval from the certificate date to 1 cadence after it, and a receipt outside that interval returns `receipt_stale`.
- **Stability.** The backdating rule permits a reduction, suspension or withdrawal to take effect before the certificate date. A permitted removal therefore changes a result already reported: the same certificate returns `in_scope_at_issuance` from an issuance-time bundle and `out_of_scope` from a later bundle, with `history_rewritten` false throughout. A receipt answers as of its covering checkpoint.
- **Signer authority.** The register establishes the laboratory's accreditation, not that the individual signer was authorised by the laboratory.
- **Legal recognition.** The register carries no legal recognition. The accreditation body's register has regulatory weight; a self-hosted log does not.

The limits above apply inside the register. Outside the condition in §4.8, the register covers a self-selected population: a certificate that carries no receipt leaves no entry, and a result establishes nothing about it.

### 4.7 The claim

> A verifier can establish, at decision time and offline, without contacting the laboratory, the accreditation body or any archive, that on the certificate date the calibration fell inside the issuing laboratory's accredited scope, and that the register's record of that scope is internally consistent, with any later change appended and any fork detectable.

The claim holds under 6 conditions: the register covers the certificate, meaning the laboratory obtains a receipt and the verifier requires one (§4.8); the accreditation body logs each event within 1 checkpoint of its effective date; the verifier holds the public trust anchors; at least 1 witness is honest; the timestamping authority is honest; and fork detection has a verifier that holds a later checkpoint or compares checkpoints with another verifier. The limits of the claim are in §4.6.

### 4.8 Condition of operation

The Certificate Transparency model depends on 2 conditions: certificate authorities must log, and browsers refuse certificates without a Signed Certificate Timestamp. The proposed register requires the same pair:

1. **The accreditation body logs** every scope change, as a condition of its own accreditation or MLA signatory status.
2. **The verifier refuses** a calibration certificate without a scope receipt, through a procurement rule, a regulatory requirement or an insurer's condition.

The second condition is harder to satisfy. Without it, a laboratory can issue certificates without a scope receipt, and the register log does not cover them. The dissertation states this as a condition, not a result. IAF Mandatory Document 28 placed a comparable obligation on ISO/IEC 17021 certificates, requiring certification bodies to register them, which makes the move institutionally familiar, though no equivalent applies to ISO/IEC 17025. The number of potential loggers is bounded: 121 accreditation bodies are signatories to the ILAC MRA (ILAC, 2025), and each would run 1 register. The condition is absent today.

This section states the condition once. Section 4.6 states the consequence, §6.4 treats it as an assumption, and §9.1 monitors whether it can be made plausible.

The design also requires the laboratory to obtain an issuance attestation at issuance, because the register does not attest a hash retrospectively. Laboratories without network access need pre-issued receipts, whose validity window the verifier checks against the certificate date. Section 6.2 places that case out of scope.

### 4.9 Threat model

| Adversary | Capability | Mitigation | Residual risk |
| --- | --- | --- | --- |
| Malicious accreditation body | Fork the register log, delete an entry, backdate a scope change | Append-only log; entry chain; scope map; consistency proofs; witness cosignatures | A fork is detectable only if a verifier compares checkpoints |
| Accreditation body, retroactive | Append an extension or reinstatement with an effective date before the certificate date | Backdating rule: the verifier compares the effective date with the timestamp of the first-inclusion checkpoint | A late-logged event with an earlier effective date is a timeliness limit, not a detected attack |
| Malicious laboratory | Forge a certificate, issue outside scope, backdate a certificate, issue without logging | Signature; scope matching; required scope receipt; issuance attestation with a TSA token | An unlogged certificate leaves no entry in the register log; a verifier that accepts the certificate cannot detect the omission |
| Colluding witnesses | Cosign 2 histories | Independence criteria; threshold; cross-jurisdiction witnesses | Threshold collusion is not detectable |
| Compromised TSA | Roll back a timestamp sequence, or backdate an attestation | Multiple independent timestamps; checkpoints | A compromised or colluding TSA defeats the freshness rule |
| Compromised laboratory key | Issue after revocation | Current status list; short-lived keys | Depends on status freshness |

---

## 5. Comparison with existing practice

### 5.1 Capability and trust

Legend: ✓ yes · ◐ partial · ✗ no. The table states what each option provides from the verifier's perspective. **17021** marks a property that IAF CertSearch provides for ISO/IEC 17021 management-system certificates only. **B** is a signed scope snapshot, including the variant embedded in the certificate at issuance (B′); **C** is a central register with current status, event history and alerts; **D** is the proposed register. Row 7 asks whether the answer requires the laboratory's cooperation. Row 8 asks whether the answer survives deletion of the accreditation body's page.

| Question | Manual check and schedule | Accreditation-body e-certificate | IAF CertSearch | DCC or signed PDF | B | C | D |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Certificate authentic and unaltered | ◐ | ◐ | ✓ (17021) | ✓ | ✓ | ✓ | ✓ |
| Issued by an accredited laboratory | ◐ | ◐ | ✓ (17021) | ✗ | ✓ | ✓ | ✓ |
| Calibration inside scope on the certificate date | ◐ | ◐ | ◐ (17021) | ✗ | ✓ | ✓ | ✓ |
| Status at a past date, online | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ |
| Status at a past date, offline, from the certificate alone | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ | ✓ |
| Fork or alternative history detectable | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | **✓** |
| Answer does not require the laboratory's cooperation | ✗ | ◐ | ✓ (17021) | ✓ | ✓ | ✓ | ✓ |
| Answer survives deletion of the accreditation body's page | ✗ | ✗ | ✓ (17021) | ✓ (file only) | ✓ | ✗ | ✓ |
| Calibration is accurate | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| All calibrations disclosed (completeness) | ✗ | ✗ | ◐ | ✗ | ✗ | ◐ | ✗ |
| Legal recognition | ◐ | ◐ | ◐ | ◐ | ✗ | ◐ | ✗ |

**What the matrix shows.** D answers 1 row that no other option answers: detection of a fork or an alternative history. It matches the best alternative on 8 rows and trails on 2 — completeness and legal recognition — both outside the claim. Version 0.2.0 claimed 2 unique rows and no trailing rows; both counts were wrong. Arm B includes B′, the snapshot embedded in the certificate at issuance, which answers the past-state row on trust in the operator. The difference between B′ and D is the fork row.

**Trust assumptions.** Capability is 1 question and belief is another; the same ✓ rests on different assumptions:

| Option | What the verifier must believe |
| --- | --- |
| Manual check and schedule | The person who answers the telephone, and the page as it stands today |
| Accreditation-body e-certificate | The accreditation body and its vendor platform |
| IAF CertSearch | IAF and its operator |
| DCC or signed PDF | Nothing for integrity; the format makes no authority claim |
| B — snapshot, including B′ | The signing operator, and for a retained snapshot that the snapshot is the one that was published |
| C — central register with history | The register operator's account of its own history |
| D — proposed register | At least 1 witness is honest and the verifier checks cosignatures; the operator can equivocate, but not undetectably |

B′ and D answer the same rows. The fork row is what separates them, and it separates them in kind: B′ rests on the operator's signature, D on 1 honest witness and a second checkpoint.

### 5.2 Role of each existing practice

The descriptions draw on the public guidance of the bodies named. Facts carried in the §2.2 table keep that table's labels; the rest is **P**.

**Manual check and the accreditation-body schedule.** The current arrangement is authoritative, free and current. UKAS guidance is to check the IAF register, then the accreditation body's schedule, then confirm the certificate with the certification body or by direct contact. The arrangement works when the laboratory is reachable and honest. It does not work when the laboratory is defunct, foreign, or the person who answers the telephone confirms a false claim. Its status signal is deletion.

**The accreditation-body e-certificate.** The accreditation body's own certificate is verifiable online by QR code and a live database. The e-certificate covers the accreditation body's certificate to the laboratory, not the laboratory's certificate to the buyer. It reports current state and runs on a vendor platform. The claimed blockchain component is a vendor feature, not a public verifiable log.

**IAF CertSearch.** The most widely deployed alternative is mandated by IAF Mandatory Document 28 and operates within the IAF MLA. It answers the current-state question for ISO/IEC 17021 management-system certificates. It does not cover ISO/IEC 17025 calibration certificates. That scoping defines the gap.

**A central register with history.** Arm C is the closest in capability. It stores every scope change as an event, serves the state on any date and sends alerts, all from 1 operator. It answers the scope-date and past-status rows of the matrix. It does not carry the proof to the certificate, and it cannot show a verifier that its history is the history it published. The difference between C and D is the trust assumption, not the questions answered.

**A signed snapshot.** Arm B signs the scope state at a point in time, and B′ embeds that snapshot in the certificate at issuance. B′ answers the past-state question offline, from the certificate alone, and it is the strongest baseline in the comparison. It cannot show that the operator did not sign a different snapshot for another verifier. The design in this proposal differs from B′ in 1 property.

**DCC and signed PDFs.** DCC and signed PDFs provide authenticity and integrity; DCC adds machine-readability. A signature proves that the laboratory signed the certificate, not that the laboratory was accredited for the calibration. The proposal adopts DCC and does not replace it.

**The accreditation body's surveillance.** The annual assessment and 4-year reassessment are the current control. The control is periodic, the verifier does not see the assessment, and the party whose record is in dispute operates it. The proposed register makes the output dated and checkable; it does not replace the assessment.

### 5.3 Summary of the comparison

Existing practices answer the current-state question. A central register with history and a signed snapshot also answer the past-state question, on trust in the operator. The proposed register answers the past-state question from the certificate alone, and makes a fork or an alternative history detectable. That is the 1 property the comparison supports.

### 5.4 Existing-practice risk

IAF CertSearch is mandated, centralised and provides real-time status alerts. An append-only event history is an incremental product change, not a research contribution. The proposed register relies on a different trust model: a centralised history is trustworthy only if its operator is trusted; a witnessed log is checkable without that trust. This is a trust-model argument, not a market argument; the dissertation states it as such.

---

## 6. Scope of the project

### 6.1 In scope

- **Object:** the accreditation-scope register of a national accreditation body.
- **Certificate class:** ISO/IEC 17025 torque calibration certificates. This is the only calibration family in the evaluation; other parameters and test certificates are boundary cases.
- **Artefact:** the scope receipt, embedded in a DCC-compatible certificate envelope (field names and structure, not full v3.3.0 schema validation).
- **Mechanism:** register log, entry chains, scope map, signed checkpoints, witness cosignatures, signed allow-list, backdating rule, offline verifier.
- **Evaluation:** simulated accreditation body, synthetic register data from public schedule structures, 9 core attacks, 3 baseline arms.
- **Scenario:** a Macau verifier, a foreign or defunct laboratory, a calibration certificate issued years earlier, and a register that has changed.

### 6.2 Out of scope

- ISO/IEC 17021 management-system certificates, which IAF CertSearch covers.
- ISO/IEC 17065 product certification and the EU NANDO database (the register of notified bodies).
- ISO 15189 medical laboratories.
- ISO/IEC 17020 inspection and ISO/IEC 17024 personnel certification.
- Test certificates and calibration parameters other than torque.
- Full DCC v3.3.0 schema validation.
- Issuance without network access: pre-issued receipts and the staleness they introduce.
- Multi-accreditation-body federation, privacy mode, closure statements and cost modelling.
- The truth of the measurement, the competence of the laboratory and the completeness of the register.
- The authority of the individual signer inside the laboratory.
- Legal recognition, regulatory acceptance and production deployment.
- A real accreditation body integration and a live public log.

### 6.3 Selection of ISO/IEC 17025

| Standard | Body | Scope granularity | Existing register | Assessment |
| --- | --- | --- | --- | --- |
| **17025** | testing and calibration laboratories | parameter, range, method | **none global** | **chosen** |
| 17021 | management-system certification bodies | standard and sector | IAF CertSearch, mandated (MD 28) | covered by a mandated register |
| 17065 | product certification bodies | product or scheme | NANDO (EU notified-body register), mandated | existing mandated register; coarser scope |
| 17024 | personnel certification bodies | scheme | none | adjacent; less document dependence |
| 17020 | inspection bodies | activity | none | viable alternative; thinner documentary evidence |
| 15189 | medical laboratories | test | national | clinical privacy |
| 17029 | validation and verification bodies | scheme | Verra, Gold Standard and other carbon-credit registries | different governance and market structure |

ISO/IEC 17025 has 3 properties that place it first. Its scope is the finest — parameter, range and method — so the authority question is specific. No mandated global register covers its certificates. The underlying fact is unreproducible and the certificate travels: a calibration is a moment in time, a destructive test consumes the sample, and a verifier relies on the certificate years later.

The standard number is a boundary label, not the contribution. The mechanism generalises; 17020 and 17024 are future work.

The evaluation uses torque calibration. Testing certificates and other calibration parameters are boundary cases: the mechanism applies, but the evaluation does not cover them.

### 6.4 Assumptions

1. A national accreditation body operates the register; the dissertation simulates one.
2. The project models the scope as a structured predicate — torque parameter, range, method, uncertainty and location — with a defined matching rule.
3. Witnesses can be sufficiently independent for the threat model; the evaluation tests collusion.
4. The condition in §4.8 holds for the evaluation: the accreditation body logs, and a stated mandate requires the verifier to refuse a certificate without a scope receipt.
5. Public schedule structures from UKAS, CNAS and HKAS are representative enough to construct synthetic data.
6. The accreditation body logs each event within 1 checkpoint of its effective date. A late-logged event is reported as a limitation, not as a detected attack.

### 6.5 Falsifiers and gates

Items 1, 3 and 5 stop the project. Items 2 and 4 narrow the claim, and the gates below decide whether the narrowed claim stands.

1. **The limitation is removed.** An accreditation body publishes an authoritative, versioned, offline-verifiable scope history, or IAF CertSearch extends to ISO/IEC 17025 with history.
2. **The scope model is intractable.** Parameter, range and method matching produces unacceptable false accepts or cannot be represented. The claim then covers register integrity only.
3. **The mechanism adds no property.** A signed snapshot embedded in the certificate at issuance (B′) answers the past-state question offline, from the certificate alone, at lower cost. The project stops unless the design shows 1 property that B′ lacks: a fork or an alternative history must be detectable in D and not in B′.
4. **No verifier refuses unlogged certificates.** The condition in §4.8 cannot be made plausible in a procurement or regulatory setting. The project re-scopes to the simulated setting, and the claim is limited to it.
5. **The scenario cannot be made plausible.** No public source supports an illustrative version of the scenario in the ISO/IEC 17025 domain.

**Gates.** G-A follows design and asks whether the scope model is tractable. G-B follows evaluation and asks whether the mechanism provides a property the baselines do not. A failed gate requires re-scoping or stopping; the project does not resolve it retrospectively. The project checks the limitation as a 2-day task at the start of design; it is not a separate gate.

---

## 7. Deliverables

The evaluation compares 4 arms: **A**, current practice; **B**, signed scope snapshots with timestamps, including the variant embedded in the certificate at issuance (B′); **C**, a central register with current status, event history and alerts; **D**, the proposed register with scope receipts.

### 7.1 Core deliverables

| ID | Deliverable | Acceptance criterion |
| --- | --- | --- |
| **C1** | Design basis: evidence note, requirements, threat model and scope-predicate model | The §2.2 evidence is confirmed; the limitation check is recorded; public schedule structures are collected; the scope model represents torque parameter, range, method and uncertainty with a matching rule; the entry chain, scope map, checkpoint format, issuance attestation, freshness window and backdating rule are specified; the state-transition table is defined; the threat model is documented |
| **C2** | Register log service | A scope event can be appended, checkpointed and proven included; the scope map commits `entry_id → chain head` in each checkpoint; a signed allow-list admits issuer keys; a signed issuance attestation binds a certificate hash to a checkpoint and a TSA token; 6 event types over 3 states are enforced by the transition table, and illegal transitions are rejected; the register records each event's first-inclusion checkpoint; backdated grants, extensions and reinstatements are rejected |
| **C3** | Witness cosigner | An independent process cosigns checkpoints; a threshold is enforced; a fork is detectable in the evaluation |
| **C4** | Scope-receipt issuer | A certificate receives the receipt defined in §4.1, with the issuance attestation, the freshness window and an inclusion proof at every first-inclusion checkpoint; the envelope is DCC-compatible (subset) |
| **C5** | Offline verifier | A command-line tool and a minimal web view return a report whose primary field is a named outcome: `in_scope_at_issuance`, `out_of_scope`, `scope_unknown`, `checkpoint_invalid`, `witness_insufficient`, `receipt_stale`, `entry_absent`, `history_rewritten`, `fork_detected`, `grant_backdated` and `certificate_tampered`. The current status is a separate field, `status_at_decision`, not an outcome. The report also carries `removal_after_issuance`, which separates a certificate that was outside scope on its date from one that a later, backdated removal excluded |
| **C6** | Evaluation harness and results | 4 arms against 9 core attacks — forged or tampered certificate, out-of-scope calibration, retroactive grant, retrospective reduction, stale receipt, deletion, rewrite, fork and missing witness — report detection, latency, bundle size and trust-anchor counts; remaining threats are reported as limitations |
| **C7** | Dissertation | Problem, evidence, design, implementation, evaluation, boundary, related work and conclusion |
| **C8** | Demonstration | The lift-maintenance scenario end to end: issue, change the register, verify offline years later, display the named outcome |
| **C9** | Reproducibility package | Code, synthetic data generator, scripts and README; 1-command reproduction without containerisation or CI |

### 7.2 Demonstration design

**Purpose.** The demonstration shows 1 case end to end. C6 measures 4 arms and 9 attacks.

**Scenario and cast.**

| Role | In the demonstration |
| --- | --- |
| Accreditation body | A simulated national accreditation body with a fictional name, operating the register log |
| Calibration laboratory | "Pearl River Calibration" (fictional), accredited outside Macau and later suspended |
| Holder | A lift-maintenance company that commissioned the calibration |
| Verifier | An insurer's risk engineer in Macau in 2026, with no network access |
| Witnesses | 3 independent cosigners over checkpoints, with a 2-of-3 threshold |
| Time authority | A TSA issuing RFC 3161 tokens |

**The case.** In 2021-03 the laboratory calibrates torque wrench TW-4471 at 50 N·m, within a scope of 20–200 N·m under ISO 6789-2. The certificate enters the lift-maintenance file. In 2024 the accreditation body reduces the scope to 20–100 N·m. In 2025 the laboratory is suspended. In 2026 the building changes insurer, and the insurer must decide whether the 2021 calibration was inside the laboratory's scope at issuance.

**The bundle.** The verifier receives 4 items:

1. the calibration certificate, DCC-compatible, signed by the laboratory;
2. the scope receipt, defined in §4.1;
3. the trust anchors: accreditation body key, witness keys and TSA certificate;
4. optionally, a current status list for the current-state question.

The verifier needs no network and makes no call to the laboratory or the accreditation body.

**Happy path.**

1. **2021 — grant.** The accreditation body appends the grant event for the scope entry (torque, 20–200 N·m, ISO 6789-2, uncertainty, location, effective date 2021-03-01, state `active`). Checkpoint CP-1042 first includes the event and is signed and cosigned.
2. **2021 — issue.** The laboratory issues the certificate, sends its hash to the register, and receives the issuance attestation with a TSA token. The receipt carries the attestation, the entry chain (the grant) with its inclusion proof at CP-1042, the scope-map proof and the inclusion proof of the chain head.
3. **2024 — reduction.** The body appends a scope reduction to 20–100 N·m. The change is a new event; the 2021 entry is not overwritten.
4. **2025 — suspension.** The body appends a suspension. The 2021 entry remains in the register log.
5. **2026 — verify.** The insurer runs the offline verifier. It checks the certificate signature, the issuance attestation and its token against the freshness interval (CP-1042 and the attestation token fall in the interval from 2021-03-14 to 1 cadence after it), the checkpoint signatures and 2-of-3 witness cosignatures, each event's inclusion proof at its first-inclusion checkpoint, the scope-map proof, the inclusion of the chain head, every hash link in the chain, the grant's effective date against its first-inclusion checkpoint, the effective state on 2021-03-14 (the grant, state `active`) and the scope match (50 N·m inside 20–200 N·m). If the bundle carries a later checkpoint, it checks the consistency proof, re-runs the chain checks including the backdating rule, and reports the later state.

The expected output:

```text
outcome                   in_scope_at_issuance
scope_entry               torque / 20–200 N·m / ISO 6789-2
effective_on              2021-03-14, state active
checkpoint                CP-1042, 2021-03-14, witnesses 2/3
status_at_decision        suspended_after_issuance
receipt_stale             false
history_rewritten         false
fork_detected             false
certificate_tampered      false
```

The output separates the scope on the certificate date (in scope) from the current sanction (suspended).

**Attack variants.** Each variant is a scripted change to the same bundle.

| Variant | Change | Expected outcome |
| --- | --- | --- |
| **Forged or tampered certificate** | The certificate signature fails, or one calibration value is edited | `certificate_tampered` |
| **Out of scope** | Certificate for 150 N·m issued in 2025, after the reduction to 20–100 N·m | `out_of_scope` |
| **Retroactive grant** | The body appends an extension with an effective date before the certificate date, logged in 2026 | `grant_backdated` |
| **Retrospective reduction** | A later checkpoint carries a reduction with an effective date between 2021-03-01 and 2021-03-14 | `out_of_scope`, with `removal_after_issuance` true |
| **Stale receipt** | A certificate dated 2021 carries a receipt obtained in 2026 | `receipt_stale` |
| **Deleted entry** | The body removes the entry from the scope map at a later checkpoint | `entry_absent` |
| **Rewritten history** | The body publishes a later checkpoint with a different chain head for the same entry | `history_rewritten` |
| **Fork** | 2 checkpoints at the same tree size with different roots, both with valid cosignatures | `fork_detected` |
| **Missing witness** | One cosignature is removed | `witness_insufficient` |

The fork, rewrite and retroactive-grant variants are the cases where D detects what B′ and C do not. The retrospective-reduction and stale-receipt variants exercise the 2 limits in §4.6. The demonstration allocates most of its time to them.

**What the demonstration does not show.**

- That the calibration is accurate; only that the laboratory was authorised.
- That the register is complete; only that this entry has not been rewritten.
- That the individual signer was authorised by the laboratory.
- That the certificate has legal recognition.

**Staging and success criteria.**

- The demonstration runs with the network disabled.
- 2 surfaces consume the same bundle: the command-line verifier for evidence and the minimal web view for legibility.
- All names are fictional; no real laboratory, insurer or accreditation body is named.
- A single `demo.sh` provides the subcommands `issue`, `verify`, `tamper`, `backdate`, `reduce`, `stale`, `delete`, `rewrite`, `fork` and `witness`, so any variant runs in 1 command.
- Duration: 8–10 minutes live, or 4 minutes recorded.
- Success criteria: the happy path returns the expected outcomes in under 1 second; each attack returns its specific named outcome; the run uses only the bundle and the public trust anchors.

**Relationship to the evaluation.** The demonstration is 1 case; C6 is the measurement.

### 7.3 Future work

The project does not deliver the following; it records them as future work: multi-accreditation-body federation (Macau, Hong Kong and the Mainland); extension to ISO/IEC 17020 or ISO/IEC 17024; closure statements for declared completeness; a privacy mode with hash-only, salted commitments; a cost and adoption model; and full DCC v3.3.0 schema validation. The project defers them to protect the core evaluation.

### 7.4 Exclusions

The project does not deliver a production accreditation service, a live public log, a legal opinion, regulatory acceptance or a replacement for the accreditation body's assessment. The scope exclusions in §6.2 apply.

---

## 8. Schedule

### 8.1 Assumptions

Week 1 begins when supervisor approval is recorded. The plan uses weeks; no calendar dates are assumed. Week 1 covers approval and set-up; no implementation begins before approval.

The evidence in §2.2 is sufficient for the proposal. A limitation check and the collection of public schedule structures form part of P1 (Design), not a separate phase.

**Effort.** The plan uses 12–15 hours per week on average, with peak weeks at 16–18 hours, for about 250–280 hours in total (the plan uses 272 hours). These are planning estimates, not measurements. If the module's credit weighting implies a different load, the phase durations scale; the sequence and gates do not change.

### 8.2 Plan

| Phase | Weeks | Indicative effort | Activities | Output | Milestone |
| --- | --- | --- | --- | --- | --- |
| **P0 — Approval** | W1 | ~8 h | Confirm direction, module constraints, scope and ethics | Signed-off proposal | **M0 — approval recorded** |
| **P1 — Design** | W2–W5 | ~60 h | Limitation check; schedule structures; requirements; threat model; scope predicate; entry chain, scope map and backdating rule; register state machine; wire format; chapter draft | C1 | **M1 / Gate G-A** |
| **P2 — Implementation** | W6–W12 | ~88 h | Register log; scope map; witness cosigner; scope-receipt issuer; offline verifier; demonstration data; chapter draft | C2–C5 | **M2 — end-to-end slice** |
| **P3 — Evaluation** | W13–W15 | ~36 h | 9 attacks; 4 arms; metrics; analysis; chapter draft | C6 | **M3 / Gate G-B** |
| **P4 — Writing and demonstration** | W16–W19 | ~70 h | Integrate and revise the dissertation; demonstration script; draft to supervisor by end W17; revisions in W18–W19 | C7, C8 | **M4 — revised draft and demonstration ready** |
| **P5 — Finalisation** | W20 | ~10 h | Final corrections; reproducibility package; submission | C9 | **M5 — submission** |
| **Total** | **20 weeks** | **~272 h** | — | — | — |

G-A and G-B are the gates defined in §6.5.

### 8.3 Critical path and dependencies

**Critical path.** Scope model → entry chain and scope map → scope receipt → verifier → evaluation. The witness cosigner can run in parallel with the verifier; the scope receipt cannot be finalised before the scope model, the chain and the map are stable.

**Buffers.** 2 weeks are held within P2 and P4. If P2 slips by more than 2 weeks, the future-work items in §7.3 are deferred.

**Dependencies.** Supervisor availability at M1, M3 and the W17 draft review; public schedule structures for synthetic data; a decision on the DCC envelope by M1.

---

## 9. Risk assessment

### 9.1 Principal risks

| # | Risk | Likelihood | Impact | Mitigation | Trigger |
| --- | --- | --- | --- | --- | --- |
| **R1** | The condition in §4.8 cannot be made plausible in the dissertation's setting | Medium | Medium | Cite the IAF Mandatory Document 28 precedent; evaluate under a stated mandate; position the work as a reference implementation | No plausible mandate by M1 |
| **R2** | IAF CertSearch extends to ISO/IEC 17025 with history | Medium | High | Focus on non-equivocation, not current status; measure the difference; monitor developments | Announcement before M3 |
| **R3** | Scope matching (parameter, range, method) is intractable | High | Medium | Restrict the parameter set; use a structured predicate; build a test suite; report false accepts | More than 10% false accepts in the pilot |
| **R4** | No adjudicated scope-date case | Medium | Medium | Use official suspension records (BBA, TWI Certification); label the evidence A or P; do not gate on an adjudicated case | No public source supports the scenario |
| **R5** | Witnesses are not independent | Medium | High | Define independence criteria; simulate collusion; report the limitation | Cosigners are not independent in the demonstration |
| **R6** | No usable register data | Medium | Medium | Construct synthetic data from public schedule structures; publish the generator | No usable public data by W3 |
| **R7** | Timeline and scope growth | Medium | High | Fix the feature set; prioritise core deliverables; defer future-work items | M2 slips by more than 2 weeks |
| **R8** | Legal or regulatory overclaim | Low | High | Include the boundary section; make no legal-recognition claim; review Macau references | A reviewer identifies a legal claim |
| **R9** | Privacy or personal-data exposure | Low | Medium | Use hash-only entries; place no personal data in the register | A scenario requires personal data |
| **R10** | Direction risk: approval not recorded | High | High | This proposal is the decision document; no implementation before approval | Approval not recorded by the end of W1 |
| **R11** | The timeliness assumption fails, or the freshness window is too wide to be useful | Medium | High | State the assumption; measure the detection window against the checkpoint cadence; report late-logged events and stale receipts as limitations | A backdated event is logged after the certificate date in the pilot |

### 9.2 Principal risks in detail

**R1 — plausibility of the condition.** The condition in §4.8 may not be makeable in the dissertation's setting: no procurement rule, regulatory requirement or insurer condition may be available. Technical validity does not depend on adoption, given the conditions in §4.7: the evaluation runs under a stated mandate. What the trigger changes is the scope of the claim, not its correctness. The mitigation is the IAF Mandatory Document 28 precedent, a stated mandate in the evaluation and a reference implementation. The trigger executes falsifier 4: the project re-scopes to the simulated setting and the claim is limited to it.

**R2 — existing-practice extension.** IAF CertSearch holds the mandate and the alerts. If it extends to ISO/IEC 17025, the limitation becomes a feature of the central register. The proposal relies on the trust model and the measured comparison. If the extension occurs before M3, the project narrows the claim to non-equivocation and reports the register as a supporting mechanism.

**R3 — scope matching.** The scope is a predicate over parameter, range, method, uncertainty and location, not a boolean. If the model cannot represent the cases without false accepts, the claim covers only register integrity.

**R11 — timeliness and freshness.** The claim depends on the accreditation body logging each event within 1 checkpoint of its effective date, and on the receipt's covering checkpoint and attestation token falling in the interval from the certificate date to 1 cadence after it. A body that logs late defeats the offline verifier at decision time, because the bundle holds the state before the event and cannot see the event. A wide freshness window admits a backdated certificate. The mitigation is to measure the cadence against both windows and to report the limits. The design cannot remove them: a log records what was logged, not what was omitted.

### 9.3 Ethical and legal notes

- The register holds no personal data; certificates and scope receipts are the object.
- The Macau scenarios are illustrative and labelled A; they are not allegations against a laboratory or accreditation body.
- The dissertation offers no legal opinion and claims no regulatory acceptance.
- The author checks references to Macau's building, lift and fire regimes with the relevant public authorities before submission.

---

*End of CNP-003 v0.3.1. This document proposes a direction. It does not record approval and does not authorise implementation.*
