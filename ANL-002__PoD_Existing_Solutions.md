# ANL-002 — Proof of Delivery: Existing Libraries and Products

| Field | Value |
| --- | --- |
| Document ID | ANL-002 |
| Title | Proof of delivery: existing libraries and products |
| Version | 0.1.0 |
| Status | Draft — analysis annex to PRP-001 |
| Last Updated | 2026-10-21 |
| Related documents | PRP-001, RFC 9943, RFC 9942, RFC 9162, RFC 3161, eIDAS, e-CMR, MLETR |
| Purpose | Establish what is already covered before defending any build |

---

## 1. Purpose and method

### 1.1 The question

> Which parts of PRP-001 are already covered by standards, libraries, and products, and what is genuinely left to build?

This annex exists because the previous proposal failed partly on exploration: much of the intended work was already available, and the project discovered it late. The rule here is to find the coverage **before** committing.

### 1.2 Method

- Decompose PRP-001 into components: capture, statement, signature, timestamp, log, proofs, witnesses, receipts, verification, policy, revocation, dispute, privacy, legal, UI, evaluation.
- Search official documentation, repositories, and standards for each.
- Label every claim: **verified** (official docs or repo), **vendor claim** (marketing), **unverified** (not yet checked).
- Treat negative claims ("no product does X") as weak evidence. They are the easiest to falsify and the most expensive to be wrong about.

### 1.3 Scope

This is a component and product scan, not an exhaustive market study. It covers the level 1–3 stack in PRP-001 and the adjacent standards. It does not price vendors or assess procurement.

---

## 2. Verdict

**An estimated 70–80% of the code in PRP-001 is commodity** (assumed, by component count; not measured). Signatures, timestamps, Merkle logs, inclusion and consistency proofs, receipts, offline verification, capture UI, and storage all have mature libraries or products. The exception is the RFC 9162 receipt path: the reference offline verifier is CCF-only, so that wrapper is custom work. The standards layer is now complete enough to build against:

- **RFC 9943** — *An Architecture for Trustworthy and Transparent Digital Supply Chains* (SCITT), Standards Track, June 2026.
- **RFC 9942** — COSE receipts (inclusion and consistency proofs).
- **RFC 9162** — Certificate Transparency v2 (Merkle log).
- **RFC 3161** — Time-Stamp Protocol.

What is **not** packaged is the PoD-specific layer:

1. a PoD **profile** of SCITT (statement schema, handover state machine, multi-leg, condition);
2. **offline trusted time** (RFC 3161 assumes connectivity);
3. **offline revocation** and key compromise at the edge;
4. **witness governance** for a delivery consortium;
5. **privacy/linkage** analysis of a delivery log;
6. the **evaluation** (level 1 vs 2 vs 3, red-team, trust anchors).

**Consequence.** A project that builds the stack is a reproduction. A project that builds the profile plus offline time/revocation and evaluates it is defensible, but narrow. The contribution must be the profile and the evaluation, not the log.

---

## 3. Coverage map

| Component | Covered by | Representative | Coverage | What remains |
| --- | --- | --- | --- | --- |
| Capture: photo, signature, location | PoD apps and SDKs | Onfleet, Detrack, Track-POD, Scandit | Commodity | Co-sign UX, evidence bundle |
| Statement format | EPCIS 2.0, COSE, SCITT | GS1 EPCIS, RFC 9052, RFC 9943 | Commodity | **PoD profile** |
| Event identity / hashing | EPCIS event hash | OpenEPCIS, GS1 sandbox | Commodity | PoD fields |
| Signature | WebCrypto, libsodium, COSE | Ed25519, P-256 | Commodity | Key lifecycle at the edge |
| Trusted time (online) | RFC 3161 | FreeTSA, DigiCert, OpenSSL `ts`, uts-server | Commodity | **Offline time** |
| Append-only Merkle log | Trillian, Tessera, Rekor, immudb, Azure Confidential Ledger, Azure SQL ledger | Tessera v1.0, Trillian | Commodity | PoD indexing |
| Inclusion / consistency proofs | RFC 9162, Tessera, Rekor | CT lineage | Commodity | — |
| Independent witnesses | Sigsum, transparency-dev/witness, CT gossip | Sigsum quorum policies | Partial | **PoD governance** |
| Receipts and offline verifier | SCITT | RFC 9942/9943, `microsoft/scitt-verifier`, `scitt-ccf-ledger` | **Partial** — CCF only; build the RFC 9162 receipt wrapper | PoD policy and RFC 9162 proofs |
| Policy and authorization | OPA, XACML, cloud IAM | OPA, eIDAS | Commodity | — |
| Revocation | PKI, OCSP, CRL, short-lived certs | — | Partial | **Offline revocation** |
| Dispute bundle | SCITT transparent statement | — | Partial | **PoD bundle format** |
| Offline trusted time | — | — | **Gap** | Design and evaluate |
| Privacy / selective disclosure | W3C VC, BBS+, SD-JWT | — | Partial | **Linkage analysis** |
| Legal recognition | eIDAS, e-CMR, MLETR | QTSPs, TransFollow | External | Not buildable |
| UI and audit view | Web | — | Commodity | Minimal audit UI |
| Evaluation and red-team | — | — | **Gap** | The contribution |

---

## 4. The standards stack

| Standard | What it fixes | Status | Relevance |
| --- | --- | --- | --- |
| **RFC 9943** | SCITT architecture: signed statements, transparency service, receipts, non-equivocation, append-only, replayability | Standards Track, June 2026 | The level-3 architecture, standardized |
| **RFC 9942** | COSE receipts; inclusion and consistency proofs | Referenced by RFC 9943 | The receipt format |
| **RFC 9162** | Certificate Transparency v2: Merkle log, inclusion, consistency | Stable | The log proofs |
| **RFC 3161** | Time-Stamp Protocol | Stable | Trusted time (online) |
| **RFC 9052/9053** | COSE | Stable | Statement signing |
| **RFC 8949** | CBOR | Stable | Encoding |
| **eIDAS** (EU 910/2014) | Qualified signatures, seals, timestamps, delivery | Regulation | Legal recognition |
| **e-CMR** | Electronic consignment note | Protocol; adoption varies | The transport document |
| **MLETR** | Electronic transferable records | Model law; national adoption | eBL recognition |
| **EPCIS 2.0** | Supply-chain event data | GS1 standard, 2022 | Event vocabulary |

**Implication.** RFC 9943 scopes its use cases to the software supply chain and states that the architecture is applicable to other supply chains; the specific extension to hardware, manufacturing, and food is stated in the RFC's introduction but should be verified before citing it as a domain claim. A PoD system is an **application profile** of SCITT, not a new architecture. Profiles are a legitimate contribution, but they are judged on the profile's requirements and evaluation, not on the log.

---

## 5. The library stack

| Project | Language / form | What it gives | Status | Note |
| --- | --- | --- | --- | --- |
| **Trillian** | Go, open source | Merkle tree log and map; CT lineage | Mature | The original general transparency server |
| **Tessera** | Go library | Tile-based transparency logs; v1.0 | Current | Recommended successor to Trillian v1 |
| **Rekor** | Go, open source | Signature transparency log; v2 returns inclusion proof + checkpoint, uses RFC 3161 for time | Current | v2 drops the signed entry timestamp; witness support is evolving — verify the current docs |
| **Sigsum** | Go, open source | Transparency log with cosigning witnesses and quorum policies | Current | Closest thing to level 3 out of the box |
| **transparency-dev/witness** | Go | Witness binary for cosigning checkpoints | Current | Separate from the log |
| **scitt-ccf-ledger** | Microsoft, open source | SCITT transparency service on CCF | Current | Receipt profile CCF |
| **scitt-verifier** | Rust, MIT | Offline verification of SCITT statements and receipts with relying-party policy | Current | CCF receipts only; RFC 9162 receipts not supported |
| **immudb** | Go, open source | Immutable database with Merkle proofs and audit logging | Current | Level 2/3 storage, not a witness network |
| **Azure Confidential Ledger** | Managed service (CCF) | Tamper-evident append-only ledger, receipts | Current | Managed; confidential computing |
| **Azure SQL ledger** | Database feature | Tamper-evident tables, digests | Current | Privileged operators can still tamper at the storage layer; weaker than a witnessed log |
| **Guardtime KSI** | Commercial | Hash-based integrity, time, provenance; keyless signatures | Current | Industrial, government use; vendor claims |
| **OpenTimestamps** | Python/JS/Java | Bitcoin-anchored timestamp proofs | Current | Level 5 timestamping; confirmation delay |
| **uts-server** | C | Self-hosted RFC 3161 TSA | Current | Simple; good for a lab |
| **OpenSSL `ts`** | CLI/library | RFC 3161 client and server | Stable | The default tool |
| **DataTrails / RKVST** | Commercial | SCITT-based provenance and receipts | Vendor claim | Verify current API |
| **Conserver** | Commercial/analysis | SCITT transparency service commentary | Vendor claim | Verify |
| **FaxSeal** | Commercial | Signed delivery attestation + independent transparency log + offline verification | Vendor claim | **Adjacent pattern, not parcel PoD** |
| **nucleo** | Go, alpha | Local C2SP-compatible transparency log with offline receipts | Alpha | Small; useful as a reference |

**Implication.** A competent team can assemble levels 1–3 from these libraries in days, not months. The build effort moves to the PoD profile, the edge cases (offline time, revocation), and the evaluation.

---

## 6. The product landscape

### 6.1 PoD capture

| Product | What it does | Gap for this project |
| --- | --- | --- |
| Onfleet | Signatures, photos, barcodes, ID checks, geostamping | No cryptographic transparency log |
| Detrack | ePOD, geofencing, bulk proof, vehicle checks | Same |
| Track-POD | Offline mode, photos, custom forms | Same |
| Descartes / SAP TM / Oracle OTM | Enterprise delivery and proof | Same; platform-held records |

These cover the capture UX, which is the commodity half. None of them, on the public documentation, offers an independently verifiable, witnessed delivery log. That is the gap — but it is a feature gap a vendor could close, not a research gap.

### 6.2 Legal transport documents

| Product | What it does | Note |
| --- | --- | --- |
| TransFollow | e-CMR platform; recommended by IRU | The legal consignment note |
| Docuten, Cargofy, Dashdoc | e-CMR integrations | National adoption varies |
| CargoX | eBL on public Ethereum; blockchain document transfer | Commercial level 5 for documents |
| WaveBL | Blockchain-based eBL | Commercial |
| Bolero | eBL, SWIFT-backed | Commercial |

e-CMR and eBL already solve the **legal document** problem, including signature and transfer. A PoD project must not claim to replace them; it can complement them with a transparency layer.

### 6.3 Supply-chain integrity and traceability

| Product / project | What it does | Note |
| --- | --- | --- |
| OpenEPCIS | Open-core EPCIS 2.0 platform | Event capture, not transparency |
| GS1 EPCIS event hash | Stable hash per event | Identity and duplicate detection |
| Morpheus.Network | Blockchain supply-chain middleware | Vendor claim; broad platform |
| TradeLens | Blockchain trade platform | **Discontinued 2022–23**; market caution |
| IBM Food Trust | Permissioned food traceability | Consortium model |

### 6.4 Adjacent proof-of-delivery patterns

- **FaxSeal** (vendor claim): signed fax delivery attestation, logged to an independent append-only transparency log, verifiable offline with OpenSSL. This is the level-3 pattern applied to a different delivery medium. It shows the architecture is already productized in an adjacent domain.
- **Offline-verifiable accountability for cross-organisation agent messaging** (arXiv 2608.28542): signed receipts and transparency logs for message delivery. Adjacent, not parcel PoD.
- **Sigstore/Rekor**: the pattern at scale for software artifacts; v2 uses RFC 3161 plus checkpoints.

---

## 7. What is genuinely left

| ID | Contribution | Type | Effort (planning) | Risk |
| --- | --- | --- | --- | --- |
| C1 | **PoD profile** of SCITT: statement schema, handover state machine, co-signature, multi-leg, condition, dispute fields | Build + design | 35 h | Profile may be judged trivial |
| C2 | **Offline trusted time**: depot TSA, co-signature time, or secure time source; evaluated | Design + build | 35 h | Hardware dependency |
| C3 | **Offline revocation**: short-lived keys, witness freshness, re-signing after reconnect | Design + build | 25 h | PKI complexity |
| C4 | **Witness governance** for a delivery consortium: who, threshold, audit, collusion | Analysis + design | 15 h | Simulated witnesses |
| C5 | **Privacy/linkage** analysis of a delivery log: hashes, timing, selective disclosure | Analysis | 10 h | Hard to prove |
| C6 | **Evaluation and red-team**: level 1 vs 2 vs 3, cost, latency, detection, trust anchors | Build + analysis | 60 h | Requires a real workload |
| — | Integration, test harness, write-up | Build | 60 h | Scope |

**Total:** approximately 240 h for the contribution components at the top end. This is an **overlapping scope estimate**, not a second budget: PRP-001 §9.2 is the authoritative 290 h plan and includes evaluation execution, artifact documentation, and the dissertation. The line items are indicative; where they differ from PRP-001 §9.2, the PRP is authoritative.

### 7.1 Build versus reuse

| Area | Reuse | Build |
| --- | --- | --- |
| Log, proofs, receipts, verifier | Tessera/Trillian, SCITT, `scitt-verifier` | RFC 9162 receipt wrapper |
| Timestamp (online) | RFC 3161 TSA | — |
| Signatures, encoding | WebCrypto/COSE | — |
| Capture UI | PWA, existing SDKs | Co-sign flow |
| PoD profile | — | **C1** |
| Offline time | — | **C2** |
| Offline revocation | — | **C3** |
| Witness governance | Sigsum/witness primitives | **C4** |
| Privacy | VC/BBS+ primitives | **C5** |
| Evaluation | — | **C6** |

**Ratio:** an estimated 70–80% of the code is reuse (assumed, by component count); roughly 20–30% is build, and most of the intellectual contribution is in C1–C6, not in the code volume. The RFC 9162 receipt wrapper is the main custom-code exception.

---

## 8. The "so what" test

| Framing | Verdict |
| --- | --- |
| "Build a PoD system with a hash chain" | **Not defensible.** The stack is commodity; a point-in-time GitHub search returned 342 repositories matching "proof of delivery" (not reproducible, and quality varies); commercial apps cover capture; SCITT covers the architecture. |
| "Build a PoD system with a witnessed transparency log" | **Weak.** Sigsum, Tessera, Rekor, and SCITT already provide the log and witnesses. |
| "Define and evaluate a SCITT PoD profile, with offline time and revocation, against levels 1–3" | **Defensible, but narrow.** The profile and the evaluation are the contribution. |
| "Measure whether a transparency log is worth it for PoD" | **Defensible as a comparison study.** The result can be negative. |

The minimum publishable result is the coverage map, the profile, the offline-time/revocation design, and the evaluation. If those are thin, the project is a reproduction.

---

## 9. Kill criteria (extends PRP-001 §8.6)

Stop or reframe if:

1. A mainstream PoD vendor ships a witnessed transparency log before the build.
2. SCITT (or another standards body) publishes a PoD profile that covers the intended contribution.
3. An existing product solves offline trusted time at the edge (e.g., a device-resident TSA).
4. The evaluation cannot distinguish levels 1–3 (all pass the red-team).
5. The PoD profile reduces to a schema with no non-trivial semantics.
6. The stakeholder's workflow turns out to require a qualified trust service, making a self-hosted log non-compliant.

---

## 10. Sources and status

| Source | Claim | Status |
| --- | --- | --- |
| RFC 9943 (June 2026) | SCITT architecture; receipts; non-equivocation | Verified |
| RFC 9942 | COSE receipts | Referenced; verify text |
| RFC 9162 | CT v2 Merkle log | Verified standard |
| RFC 3161 | TSP | Verified |
| `transparency-dev/tessera` | Tile-based log library; v1.0 | Verified (repo/blog) |
| `google/trillian` | Merkle log/map | Verified (repo) |
| `sigstore/rekor` | Transparency log; v2 RFC 3161 + checkpoint | Verified (docs) |
| Sigsum | Witnesses, quorum policies | Verified (site/docs) |
| `transparency-dev/witness` | Witness binary | Verified (repo) |
| `microsoft/scitt-verifier` | Offline verifier; CCF receipts only | Verified (README) |
| `microsoft/scitt-ccf-ledger` | SCITT TS on CCF | Referenced; verify |
| Azure Confidential Ledger | CCF ledger, receipts | Verified (Microsoft docs) |
| Azure SQL ledger | Tamper-evident tables | Verified (Microsoft docs) |
| AWS QLDB | Retired 2025-07-31 | Verified |
| immudb | Immutable DB, Merkle proofs | Verified (repo/docs) |
| Guardtime KSI | Integrity/timestamping | Vendor claim |
| OpenTimestamps | Bitcoin-anchored timestamps | Verified (site) |
| e-CMR / TransFollow | Electronic consignment note | Vendor/industry sources |
| CargoX, WaveBL, Bolero | eBL platforms | Vendor claims |
| Onfleet, Detrack, Track-POD | PoD capture | Vendor claims |
| OpenEPCIS / GS1 event hash | EPCIS 2.0, event hashing | Verified (repo/sandbox) |
| Morpheus.Network | Blockchain supply chain | Vendor claim |
| TradeLens shutdown | Discontinued 2022–23 | Verified (Maersk) |
| FaxSeal | Signed delivery attestation + transparency log | Vendor claim |
| arXiv 2608.28542 | Agent messaging receipts | Preprint; adjacent |
| GitHub search | 342 repos match "proof of delivery" | Verified count; quality varies |

---

## 11. Summary

The level-3 architecture is no longer a research gap: RFC 9943 standardizes it, and Tessera, Trillian, Rekor, Sigsum, `scitt-verifier`, and `scitt-ccf-ledger` implement it. Commercial PoD apps cover capture; e-CMR and eBL cover the legal documents; EPCIS covers event data. A project that builds the stack is a reproduction and will be marked as one.

What remains is the **PoD profile**, **offline trusted time**, **offline revocation**, **witness governance**, **privacy/linkage**, and the **evaluation**. That is a narrower and more honest contribution, and it is the only version of PRP-001 that survives the coverage test.
