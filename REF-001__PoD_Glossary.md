# REF-001 — Proof-of-Delivery Glossary

| Field | Value |
| --- | --- |
| Document ID | REF-001 |
| Title | Proof-of-delivery glossary |
| Version | 0.1.0 |
| Status | Reference — companion to PRP-001 and TUT-001 |
| Last Updated | 2026-10-21 |
| Related | PRP-001, ANL-002, TUT-001, RFC 9943, RFC 9942, RFC 9162, RFC 3161 |

**Canonical glossary.** This document is the single source for terminology. PRP-001 Appendix D and TUT-001 §15 are short subsets and defer to this file.

---

## A. Delivery domain

| Term | Definition |
| --- | --- |
| **Consignor** | The party that sends the goods (also sender, shipper). In the document case, the registry or HR function is the sender. Not necessarily the seller; a 3PL can act for the seller. |
| **Carrier** | The legal entity that contracts to transport the goods and bears liability. The courier is its agent. There may be no carrier in a digital document handover. |
| **Courier** | The person who performs a parcel handover and holds the signing key. In the document case, the registry or HR clerk holds the key. One of the two issuers. |
| **Consignee** | The party that receives the goods or document (also receiver, recipient). A student or employee in the document case; a receptionist may sign on the organisation's behalf. |
| **Shipment** | The subject of a parcel statement, identified by `shipment_id`. The document case uses a transfer/document ID, or `subject` directly. |
| **Handover** | The physical or digital transfer of custody of goods or a document. In the system, a pair of statements (handover + receipt) linked by `prev`. |
| **Document handover** | A transfer of an official document — a transcript, diploma, employment letter, or contract — from an institution to a student or employee. The primary case in PRP-001. |
| **Proof of delivery (PoD)** | Evidence that a handover was asserted and acknowledged. It is not proof that the goods arrived, or that a document is genuine or complete. |
| **Consignment note** | The legal transport document (CMR). The electronic form is e-CMR. |
| **Electronic bill of lading (eBL)** | A transferable document of title. Different object from PoD; do not conflate. |
| **Chain of custody** | The sequence of custody transfers. Each handover is a statement. |
| **Condition** | The recorded state of the goods at handover (sealed, damaged, short, etc.); optional for a document. |
| **Dispute** | A disagreement about a handover. The system preserves evidence; it does not adjudicate. |
| **Revocation** | Invalidating a key or certificate. A timestamp token issued before revocation proves the statement existed before revocation, not that it was signed before revocation; statements in the compromise window are indeterminate. |
| **Authority** | The right to sign for a party. It comes from enrolment and role, not from the signature. |
| **Oracle problem** | The gap between a digital assertion and the physical event. No ledger closes it. |

---

## B. Trust, roles, and log properties

| Term | Definition |
| --- | --- |
| **Issuer** | An entity that signs a statement. In the slice: the courier and the recipient. |
| **Subject** | The artifact or shipment a statement is about (`sub = shipment:SHP-42`). |
| **Transparency Service (TS)** | The service that verifies statements, appends them to a verifiable data structure, and issues receipts. |
| **Relying Party** | An entity that verifies transparent statements and applies its own policy. |
| **Auditor** | A relying party that replays the log and checks consistency. |
| **Witness** | An independent party that checks and cosigns checkpoints. |
| **Trust anchor** | A key or root the verifier is configured to trust: CA root, TSA key, log checkpoint key, witness keys. |
| **Registration policy** | The rules a TS applies before appending a statement (accepted issuers, subjects, algorithms, size). |
| **Transparency** | The property that statements are accessible to authorised parties, auditable, and cannot be silently changed. |
| **Non-equivocation** | The property that no two conflicting histories are accepted. |
| **Equivocation** | Showing different histories to different parties. |
| **Split view** | The observable form of equivocation: two verifiers see different checkpoints. |
| **Withholding** | Omitting a statement from the log or from a verifier. |
| **Append-only** | Entries cannot be modified, deleted, or reordered. |
| **Replayability** | The VDS includes enough information for authorised actors with access to check that each statement was correctly registered; an auditor can reconstruct the log and check inclusion and consistency. |
| **Completeness** | Whether every statement an issuer submitted is present. It rests on the issuer-signed `sequence` plus a published submission record; a third party cannot know what it never saw. |
| **Trust assumption** | What must be true for the guarantee to hold: CA honest, TSA honest, witness quorum honest, device uncompromised. |

---

## C. Cryptography and proofs

| Term | Definition |
| --- | --- |
| **Hash function** | A one-way function mapping arbitrary bytes to a fixed-size digest. |
| **Collision resistance** | It is hard to find two different inputs with the same hash. |
| **Preimage resistance** | It is hard to recover an input from its hash. |
| **Digital signature** | A private-key operation that anyone with the public key can verify. |
| **Key pair** | A private key for signing and a public key for verification. |
| **Certificate** | A signed binding between a public key and an identity. |
| **CA (certification authority)** | The entity that issues certificates; a trust anchor. |
| **Enrolment** | The process of binding a key to a party and a role. |
| **Non-repudiation** | A legal and evidential property: a signer cannot credibly deny signing. It needs a trust anchor and time; a bare signature does not provide it. |
| **Canonical encoding** | A unique byte representation of a data structure, so that signing and verification see the same bytes. |
| **Deterministic CBOR** | CBOR with sorted map keys, shortest integers, and definite lengths. |
| **Domain separation** | Prefixing hash inputs by type: `0x00` for a leaf, `0x01` for an internal node. |
| **Merkle tree** | A hash tree: leaves hash data, internal nodes hash their children, the root commits to all leaves. |
| **Leaf hash** | `SHA256(0x00 \|\| data)`. |
| **Node hash** | `SHA256(0x01 \|\| left \|\| right)`. |
| **Root** | The top hash, committing to all leaves at a given tree size. |
| **Inclusion proof** | The sibling path that lets a verifier recompute the root from one leaf. |
| **Consistency proof** | A proof that an older tree is a prefix of a newer one. |
| **Checkpoint (signed tree head)** | A signed statement of `{origin, size, root}` at a point in time. |
| **Audit path** | The sibling hashes in an inclusion proof. |
| **EUF-CMA** | Existential unforgeability under chosen-message attack: the standard signature security notion. |
| **TSA (timestamp authority)** | A trusted service that signs a token over a hash and a time. |
| **Timestamp token** | An RFC 3161 signed token proving a hash existed at a time. |
| **Message imprint** | The hash carried in a timestamp request and token. |
| **Nonce (TSA)** | A value used once, to prevent replay in an RFC 3161 timestamp exchange. |
| **Nonce (statement)** | A per-statement value that makes a replay of an old statement detectable. |

---

## D. Standards and protocol

| Term | Definition |
| --- | --- |
| **CBOR** | Concise Binary Object Representation; a binary encoding with a deterministic profile. |
| **COSE** | CBOR Object Signing and Encryption. |
| **COSE_Sign1** | A single-signer COSE structure, tagged 18. |
| **Protected header** | The signed metadata of a COSE structure. |
| **Unprotected header** | Metadata that is not covered by the signature. |
| **Sig_structure** | The byte structure actually signed in COSE: `["Signature1", protected, external_aad, payload]`. |
| **CWT claims** | CBOR Web Token claims in COSE header 15: `iss` and `sub`. |
| **SCITT** | Supply Chain Integrity, Transparency and Trust; the IETF architecture for transparent supply-chain statements. |
| **RFC 9943** | The SCITT architecture (Standards Track, June 2026). |
| **RFC 9942** | COSE Receipts: inclusion and consistency proofs in COSE. |
| **Receipt** | A signed proof that a statement is included in a verifiable data structure. |
| **Transparent statement** | A signed statement augmented with its receipt. |
| **VDS** | Verifiable Data Structure, e.g. an RFC 9162 Merkle tree (`vds = 1`). |
| **VDP** | Verifiable Data Structure Proof: `-1` inclusion, `-2` consistency. |
| **RFC 9162** | Certificate Transparency v2: the Merkle tree, inclusion, and consistency definitions. |
| **CT** | Certificate Transparency, the original transparency-log deployment. |
| **RFC 3161** | Time-Stamp Protocol. |
| **eIDAS** | EU regulation on electronic identification and trust services. |
| **MLETR** | UNCITRAL Model Law on Electronic Transferable Records. |
| **EPCIS** | GS1 standard for supply-chain event data. |
| **e-CMR** | The electronic consignment note under the CMR convention. |
| **Conformance** | Implementing the MUSTs of a specification; established by tests. |
| **Compliance** | Meeting a legal or regulatory obligation; not the same as conformance. |

---

## E. System and evaluation

| Term | Definition |
| --- | --- |
| **PWA** | Progressive web app; a browser app that can work offline and be installed. |
| **WebCrypto** | The browser cryptography API, including non-extractable keys. |
| **IndexedDB** | Browser storage used for the offline queue. |
| **Offline queue** | Signed statements held locally until connectivity returns. |
| **Payload store** | The off-chain, encrypted statement body; the log holds hashes. |
| **Log service** | The wrapper around Tessera that appends, proves, and issues receipts. |
| **Verifier** | The component that checks signatures, proofs, receipts, checkpoint, and policy. |
| **Dispute bundle** | A portable package of statements, receipts, tokens, checkpoint, and certificates, verifiable offline. |
| **Conformance suite** | Positive and negative tests for the implemented RFC MUSTs. |
| **Red-team** | A scripted set of attacks with expected detections. |
| **Threat model** | The adversaries, their capabilities, and what they can and cannot do. |
| **Residual risk** | What remains after the mitigations: colocation, key theft, oracle, legal status. |
| **Trust-anchor count** | The number of keys the verifier must configure (log/operator, TSA, witnesses, CA root, plus validators at higher levels). A count, not a score. |
| **p50 / p99** | Median and 99th-percentile latency. |
| **Proof size** | The bytes in an inclusion or consistency proof. |
| **Throughput** | Entries appended per second. |
| **Level 1–5** | The project's analytical spectrum: signed DB, hash chain + TSA, Merkle log + witnesses, permissioned chain, permissionless chain. Not a standard taxonomy. |
| **Thin E2E slice** | One handover — a document or parcel — from capture to independent verification; the build scope. |
| **Two issuers** | The sender (registry/HR, or the carrier in the parcel demo) and the recipient (student/employee or consignee). |
| **`prev`** | The hash of the canonical `COSE_Sign1` bytes of the prior statement; it links the receipt to the handover statement and makes the pair a handover. |
| **`sequence`** | A per-issuer counter inside the signed statement; the leaf commits to it, so a gap is evidence of withholding. |
| **SHP-42** | The worked parcel demo in PRP-001 §6.2a and TUT-001 §2.5. |

---

## F. Commonly confused pairs

| Pair | Distinction |
| --- | --- |
| **Authentication vs authorization** | Authentication proves who signed; authorization decides whether that signer is allowed to. The certificate does the first; policy does the second. |
| **Authentication vs non-repudiation** | Authentication is a technical check at verification time. Non-repudiation is a legal/evidential property that also needs a trust anchor and trusted time. |
| **Hash chain vs Merkle tree** | A hash chain links records linearly. A Merkle tree hashes leaves into a root and supports logarithmic inclusion proofs. |
| **Merkle tree vs blockchain** | A Merkle tree is a data structure. A blockchain adds replication, consensus, and a membership rule. |
| **Transparency log vs blockchain** | A transparency log is append-only and witnessed, usually with a single writer. A blockchain has multi-party consensus on ordering. |
| **Inclusion vs consistency** | Inclusion proves one entry is in the tree. Consistency proves an older tree is a prefix of a newer one. |
| **Equivocation vs withholding** | Equivocation shows different histories to different parties. Withholding omits an entry. Witnesses address the first; receipts and issuer-side sequences address the second. |
| **Issuer vs subject vs relying party** | The issuer signs; the subject is what the statement is about; the relying party verifies. |
| **Signature vs receipt** | The issuer signature covers the statement. The receipt signature covers the Merkle root and proves inclusion. |
| **Verification vs validation** | Verification checks the cryptography and proofs. Validation checks that the content is acceptable under policy. |
| **PoD vs proof of existence** | PoD asserts a handover. Proof of existence asserts that a record existed at a time. A TSA gives the latter, not the former. |
| **Conformance vs compliance** | Conformance is implementing the RFC's MUSTs and passing tests. Compliance is meeting a legal obligation. |
| **Canonical encoding vs serialization** | Serialization is any encoding of an object. Canonical encoding is the one unique byte form that signatures rely on. |
| **Trust anchor vs trust assumption** | A trust anchor is a key you configure. A trust assumption is a belief you cannot test, e.g. that a quorum of witnesses is honest. |
| **Oracle problem vs data integrity** | Data integrity is whether bytes changed. The oracle problem is whether the bytes correspond to reality. |
| **Audit vs monitoring** | Monitoring watches operations in real time. Auditing reconstructs and checks history after the fact. |

---

## G. How to use this glossary

- In the dissertation, define each term at first use and cite the section of RFC 9943/9942/9162 that defines it.
- Do not use "blockchain" for levels 1–3. Use "hash chain," "transparency log," or "witnessed log."
- Do not say "proves delivery." Say "records an assertion of delivery, with inclusion and time."
- Do not say "non-repudiation" without naming the trust anchor and the timestamp.
- Do not say "compliant with RFC 9943." Say "conforms to the implemented MUSTs and passes the conformance suite."
