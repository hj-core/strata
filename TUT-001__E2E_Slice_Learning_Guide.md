# TUT-001 — Learning Guide: Building the PoD E2E Slice

| Field | Value |
| --- | --- |
| Document ID | TUT-001 |
| Title | Learning guide for the proof-of-delivery E2E slice |
| Version | 0.1.0 |
| Status | Draft — teaching companion to PRP-001 §9.1a |
| Last Updated | 2026-10-21 |
| Audience | A student new to transparency logs, COSE, and Merkle proofs |
| Related | PRP-001, ANL-002, RFC 9943, RFC 9942, RFC 9162, RFC 3161 |

---

## 1. How to use this guide

The slice in PRP-001 §9.1a is one handover — a document transfer or a parcel delivery — from capture to independent verification. This guide teaches the knowledge needed to build it, one layer at a time. Each module has:

- **Concept** — the idea, in plain language.
- **In the slice** — where it appears.
- **Exercise** — something you build or break.
- **Checkpoint** — what you should be able to explain before moving on.
- **Reading** — the exact RFC sections, not the whole document.

Do not read all the RFCs first. Read the section the module needs, do the exercise, then move on. The RFCs are reference manuals, not textbooks.

**Rule zero:** do not implement cryptographic primitives. Use a library for SHA-256, Ed25519/ECDSA, CBOR, and COSE. Your work is the protocol, the proof handling, and the evaluation.

---

## 2. The slice in one page

### 2.1 Actors

| Actor | Role in SCITT | In the slice |
| --- | --- | --- |
| Sender / issuer | Issuer | Registry or HR function in the primary case; the courier, acting for the sender, in the parcel demo |
| Recipient | Issuer (second) | Student or employee in the primary case; the consignee in the parcel demo |
| Relying party | Relying Party | Student, employer, or auditor; the consignor or insurer in the parcel demo |
| Carrier (parcel demo only) | — | Moves the goods; acts for the sender |
| Log service | Transparency Service | Appends, proves, issues receipts |
| Auditor / insurer | Relying Party | Verifies offline |
| Witness | Checkpoint cosigner | Cosigns checkpoints; does not see the statements |

**Primary case.** PRP-001's primary motivation is **document handover in universities and employers**: the registry or HR function is the sender and issuer; the student or employee is the recipient and issuer. The parcel scenario in this guide is the **demo instance**; the protocol is identical, with a document hash and metadata in place of the item and photo hashes.

**Two issuers, two statements.** The sender and the recipient each issue a `COSE_Sign1` about the same `sub` (the shipment or transfer). They are linked by subject, not by a multi-signature envelope. This keeps each statement single-signer and extends naturally to a third issuer later.

### 2.2 The flow

1. The courier fills the form. The statement is **canonical CBOR** (deterministic encoding). Call its bytes `S`.
2. The courier signs `S` in a `COSE_Sign1` structure. The issuer signature proves authorship.
3. The recipient issues its own receipt statement about the same subject (`sub = SHP-42`), in a second `COSE_Sign1`. It proves the handover was witnessed.
4. Each side sends only `SHA256(signed_statement)` to the TSA; the token is stored beside the statement.
5. Both signed statements go to the log service.
6. The log verifies the signatures and its registration policy, computes the **leaf hash** of each signed statement, appends them to a Merkle tree, and returns a **receipt**: an inclusion proof plus a signature over the tree root.
7. The recipient receives the **transparent statement**: the signed statement plus its receipt.
8. A verifier checks, in order: the issuer signature, the leaf hash, the inclusion proof to the root, the receipt signature over the root, the checkpoint, the TSA tokens, and the policy.
9. On a dispute, the bundle (both signed statements, timestamps, receipts, checkpoint) is verified offline.

### 2.3 The two signatures

This is the part beginners usually miss:

- The **issuer signature** is over the statement. It proves *who said it*.
- The **receipt signature** is over the **tree root**, not the statement. It proves *the statement is in the log*.

Neither alone is enough. A signed statement can be withheld; a receipt without a valid issuer signature proves inclusion of an unsigned blob. The verifier needs both, plus the proof that binds them.

### 2.4 The chain of hashes

```
statement bytes S
   │  issuer signs
   ▼
COSE_Sign1 (signed statement)  ── hash ──► leaf = SHA256(0x00 || signed_statement_bytes)
   │                                            │
   │                                            ▼
   │                                     Merkle tree
   │                                            │
   │                                     root at size n
   │                                            │
   │                                     receipt signs root
   ▼                                            ▼
verifier: signature OK, leaf recomputed, proof → root, receipt signature OK
```

If any byte of the statement changes, the issuer signature fails. If any leaf or path changes, the root changes and the receipt signature fails. If the log shows a different root to another verifier, the witnesses catch it.

---

## 2.5 Worked scenario — SHP-42 (parcel demo)

This is the **parcel demo**. For the primary document case, read Dana/Northline as the registry or HR sender, Sam/BioLab as the student or employee, `SHP-42` as the transfer ID, and the items and photo hashes as the document hash and metadata. The protocol is identical.

### Cast

| Actor | Party | Key | Role |
| --- | --- | --- | --- |
| Dana | Northline Logistics (carrier) | Courier device key | Issuer: delivery statement |
| Sam | BioLab (consignee) | Recipient device key | Issuer: receipt statement |
| Acme Instruments | Consignor | None | Relying party |
| ClearLog | Independent log operator | Log checkpoint key | Transparency Service |
| UniAudit, Aegis Audit | Two witnesses | Witness keys | Cosign checkpoints |
| InsureCo | Insurer | None | Relying party: verifies the dispute bundle |
| FreeTSA | Timestamp authority | TSA key | Trusted time |
| UniCA | Certification authority | CA root | Binds keys to parties |

Shipment `SHP-42`: one calibrated spectrometer, Acme to BioLab. The two issuers are Dana (delivery) and Sam (receipt); Acme is a verifier.

### Phase 0 — Enrolment (before the day)

1. UniCA issues device certificates to Dana and Sam, binding each public key to its party and role.
2. ClearLog generates its checkpoint key; UniAudit and Aegis Audit generate witness keys. InsureCo, the insurer, is a relying party and holds no witness key.
3. Acme configures five trust anchors: ClearLog's checkpoint key, the two witness keys, FreeTSA's key, and UniCA's root.
4. ClearLog publishes its registration policy: accepted issuers (`carrier:northline:*`, `consignee:biolab:*`), accepted subjects (`shipment:*`), accepted algorithms (ES256, Ed25519), maximum statement size, and the requirement that a receipt statement carries `prev`.

### Phase 1 — Dispatch (09:00, consignor to carrier)

Acme creates `SHP-42` in its ERP and sends the manifest to Northline. No statement is issued in this slice; the shipment ID is the subject that links everything. If Acme were an issuer, it would sign a dispatch statement here.

### Phase 2 — At the doorstep (14:02, courier)

1. Dana opens the PWA, selects `SHP-42`, records condition, captures a photo, and takes the device location and time.
2. The app builds statement `S_c` and encodes it as deterministic CBOR:

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

3. Dana signs it:

```
COSE_Sign1 = 18([
  protected:   { 1: -7, 4: dana_kid, 15: { 1: "carrier:northline:courier:dana", 2: "shipment:SHP-42" } },
  unprotected: {},
  payload:     S_c,
  signature:   Sign(dana_priv, ["Signature1", protected, h'', S_c])
])
```

4. The app stores the signed statement locally. If offline, it queues it.

**What Dana has produced:** a signed assertion that she delivered `SHP-42` in sealed condition at 14:02. It proves authorship and integrity. It does **not** prove the pallet arrived intact; that is the recipient's job.

### Phase 3 — Handover (14:05, recipient)

1. Sam inspects the pallet, confirms the item and condition, captures a photo.
2. Sam's app builds `S_r`, the receipt statement:

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

3. Sam signs it with the recipient key.

**Why `prev` matters:** it binds the receipt to the exact delivery statement Dana signed. Without it, the log would hold two statements about the same shipment with no cryptographic link; with it, the pair is a handover.

### Phase 4 — Trusted time (14:06)

1. Each app sends `SHA256(signed_statement)` to FreeTSA.
2. FreeTSA returns an RFC 3161 token whose `messageImprint` is that hash and whose `genTime` is the TSA's clock.
3. The token is stored beside the statement, not inside it. Putting the token inside the statement would create a circular dependency: the token is over the statement, so it cannot be part of what is hashed.

**What the token proves:** the signed statement existed at 14:06 by a trusted clock. **What it does not prove:** that the delivery happened at 14:02. Dana's device time is an assertion; the TSA time is evidence of existence, not of the event.

**Offline variant:** if Dana is offline, she signs at 14:02, queues, and timestamps at 16:40 on reconnect. The token then proves existence at 16:40, not 14:02. The recipient's co-signature at 14:05 is a second attestation of the handover time. The verifier must know which time is which.

### Phase 5 — Registration (14:07, log)

1. The app submits the two signed statements to ClearLog.
2. ClearLog checks, in order:
   - the COSE signature and the certificate chain to UniCA;
   - `iss` and `sub` are present and match the policy;
   - the protected header's `iss`/`sub` equal the body's `issuer`/`subject`;
   - the issuer is allowed for that subject;
   - the algorithm is allowed;
   - the statement size is within the limit;
   - for `event: receipt`, `prev` is present;
   - the issuer's signed `sequence` is continuous (no gap).
3. ClearLog computes the leaves:

```
L_c = SHA256(0x00 || signed_S_c)
L_r = SHA256(0x00 || signed_S_r)
```

4. It appends both, computes the root at the new size `n`, and builds an inclusion proof for each leaf.
5. It issues a receipt for each:

```
18([
  protected:   { 1: -7, 395: 1 },
  unprotected: { 396: { -1: [ <<[n, i_c, path_c]>> ] } },
  payload:     nil,          // detached; the root is computed
  signature:   Sign(clearlog_priv, ["Signature1", protected, h'', root])
])
```

Each proof is a `bstr` containing `[tree_size, leaf_index, inclusion_path]`; the `<<...>>` is that wrapped CBOR. A consistency receipt uses `vdp: { -2: [ <<[tree_size_1, tree_size_2, consistency_path]>> ] }`.

6. It returns the receipt to the app. The app now holds a **transparent statement**: the signed statement plus its receipt.

**What ClearLog sees:** the signed statements (to verify them) and their hashes. It does not need the photo bytes or the full payload; if the policy says so, it discards the payload after verification and keeps the hash.

### Phase 6 — Checkpoint and witnesses (14:08)

1. ClearLog produces a checkpoint `CP = { origin: "clearlog", size: n, root }` and signs it.
2. It sends `CP` plus a consistency proof from the previous checkpoint to UniAudit and Aegis Audit.
3. Each witness verifies the consistency proof against the last checkpoint it cosigned, then returns a cosignature over `CP`.
4. The witnesses gossip the cosigned checkpoint to each other and to verifiers.

**Why this step exists:** the receipt proves inclusion at size `n`; the checkpoint proves that `root` is the log's head at size `n`; the witness signatures prove that independent parties saw the same head. Without witnesses, the log can show different heads to different verifiers.

**Size reconciliation:** if the checkpoint is at size `m > n`, the verifier needs a consistency proof from `n` to `m`, or a fresh receipt at size `m`. The slice issues receipts and checkpoints at the same size to avoid the gap; the evaluation should test the gap explicitly.

### Phase 7 — Verification at handover (14:09, recipient)

Sam's app verifies the bundle it holds:

1. Dana's signature over `S_c`, using Dana's certificate.
2. `L_c` from the signed statement; apply `path_c`; get `root`.
3. ClearLog's receipt signature over `root`.
4. `root` equals the checkpoint root at size `n`.
5. The checkpoint has two valid witness signatures.
6. Sam's own signature over `S_r`; `prev` equals `SHA256(signed_S_c)`.
7. `L_r`; apply `path_r`; get the same `root`; verify the receipt.
8. The TSA tokens match the statement hashes and their times are within the delivery window.

If all eight pass, the app shows "delivery verified." If any fails, it shows which check failed, not a generic error.

### Phase 8 — Verification by the consignor (14:30, Acme)

Acme's portal fetches the transparent statements for `SHP-42` and runs the same checks, plus its own policy:

- both statements present, same subject;
- the receipt statement's `prev` matches the delivery statement;
- the issuers are the expected carrier and consignee;
- the TSA times are within the delivery window;
- the witness quorum is met (2 of 2 in the slice);
- the condition fields agree, or the discrepancy is flagged.

Acme does not need to trust ClearLog's honesty; it trusts the witnesses and the cryptography. It does need ClearLog for availability.

### Phase 9 — Dispute (two weeks later)

The pallet is found damaged. Acme and BioLab disagree about when the damage occurred.

The verifier exports a **dispute bundle**:

- `signed_S_c` + `receipt_c` + `ts_c`;
- `signed_S_r` + `receipt_r` + `ts_r`;
- the checkpoint `CP` + two witness signatures;
- the CA certificates.

The insurer verifies the bundle offline, with no access to ClearLog. The bundle establishes:

- Dana's device time was 14:02 and the TSA token was issued at 14:06 (or 16:40 if offline); the two times are distinct claims;
- Sam asserted "sealed" at 14:05;
- both statements are in the same log, in the same checkpoint, witnessed by two independent parties;
- neither statement has been altered since signing.

It does **not** establish who damaged the pallet. The system preserves evidence; the humans and the insurer adjudicate. That boundary must be stated in the dissertation.

### Phase 10 — Revocation (if Dana's phone is stolen at 16:00)

1. Northline reports the compromise; UniCA revokes Dana's certificate at 16:30.
2. The verifier checks revocation **at the TSA time**, not at the current time. A token **issued before** revocation proves the signed statement existed before revocation; it does not prove it was **signed** before revocation.
3. Statements signed in the window between compromise (16:00) and revocation (16:30) are indeterminate: they may be innocent (queued offline) or forged (stolen key). A statement signed before compromise but timestamped after revocation is also indeterminate, because the token proves existence at the later time, not the earlier signing. The design states this rather than hiding it; short-lived keys or witness freshness narrow the window.
4. Statements made after 16:30 without a new key are rejected by policy.

This is why the TSA token matters, and why it is not a complete answer to key compromise.

### Phase 11 — Audit (quarterly)

The auditor replays ClearLog from its checkpoints:

- verifies every consistency proof;
- verifies a sample of inclusion proofs;
- checks that each issuer's **signed** `sequence` is complete. The sequence is inside the statement, so the leaf commits to it: a missing entry leaves a gap in that issuer's sequence, and a verifier can detect the gap even though it cannot know what the missing statement said. A log that drops an entry and renumbers log metadata cannot repair the issuer's signed sequence. A deliberately skipped sequence is indistinguishable from a lost statement, so the auditor also compares against the issuer's published submission record when one exists.

### Who sees what

| Actor | Sees | Does not see |
| --- | --- | --- |
| Dana | Her statement, her receipt, the checkpoint | Sam's private key |
| Sam | Both statements, both receipts, the checkpoint | Dana's private key |
| ClearLog | Signed statements (to verify), hashes | Photo bytes if the payload is off-chain |
| FreeTSA | Hashes only | The statements |
| Witnesses | Checkpoints and consistency proofs | The statements |
| Acme / insurer | The dispute bundle | Private keys |
| Auditor | The log and checkpoints | Off-chain payloads unless granted |

---

## 3. Module map

| Module | Topic | Hours | Milestone |
| --- | --- | --- | --- |
| 0 | Primitives: hashes, signatures, canonical encoding | 8 | M0 |
| 1 | Merkle trees: root, inclusion, consistency | 10 | M1 |
| 2 | COSE and RFC 9942 receipts | 12 | M2 |
| 3 | SCITT and RFC 9943 | 10 | M3 |
| 4 | The log service: Tessera, checkpoints, witnesses | 10 | M3 |
| 5 | Capture, co-sign, offline | 12 | M4–M5 |
| 6 | Threat model and red-team | 10 | M6 |
| 7 | Evaluation and writing | 10 | M6–M7 |

Total: ~82 hours of learning, distributed across the 290 h build; it is not additional.

---

## 4. Module 0 — Primitives

### Concept

- **Hash function.** SHA-256 maps any bytes to 32 bytes. It is collision-resistant (hard to find two inputs with the same output) and preimage-resistant (hard to invert). A one-bit change changes the output completely.
- **Digital signature.** A key pair (private, public). The private key signs; anyone with the public key verifies. It proves possession of the key, not identity; identity comes from the certificate or trust anchor.
- **Canonical encoding.** A signature is over **bytes**, not over an object. If two systems serialize the same object differently (key order, whitespace, integer width), the signatures differ. CBOR has a deterministic encoding profile: shortest integers, sorted map keys, no indefinite lengths, definite lengths.
- **Domain separation.** The same hash function is used for leaves and nodes, so the inputs are prefixed: `0x00` for a leaf, `0x01` for an internal node. Without this, an attacker could present an internal node as a leaf (a second-preimage attack).

### In the slice

- `statement_bytes = deterministic_cbor(statement)`.
- `leaf = SHA256(0x00 || signed_statement_bytes)`.
- The issuer signs `statement_bytes`; the log hashes the **signed statement**, not the raw statement, so the signature itself is committed.

### Exercise

1. Create a JSON statement. Encode it to CBOR twice with different map insertion orders. Compare the bytes. They should be identical under deterministic encoding.
2. Sign the bytes with Ed25519 or ES256. Verify. Flip one byte. Verify again; it must fail.
3. Re-serialize the object before verifying. Explain why verification may fail even though the data is "the same".

### Checkpoint

You can explain why signing an object rather than its canonical bytes is a bug, and why leaf and node hashes use different prefixes.

### Reading

- RFC 8949 §3–4 (CBOR, deterministic encoding).
- RFC 9052 §4 (COSE_Sign1).
- RFC 8032 (Ed25519) or FIPS 186-5 (ECDSA P-256).

---

## 5. Module 1 — Merkle trees

### Concept

A Merkle tree hashes data into leaves, then hashes pairs of hashes into parents, up to one root. Any change to a leaf changes the root. A **proof** lets you recompute the root from one leaf without the whole tree.

RFC 9162 defines the tree hash:

```
MTH({})        = SHA256("")
MTH({d0})      = SHA256(0x00 || d0)
MTH(D[n])      = SHA256(0x01 || MTH(D[0:k]) || MTH(D[k:n]))
                 where k is the largest power of 2 smaller than n
```

The **inclusion proof** is the sibling path from the leaf to the root. The verifier uses the leaf index bits to decide left/right at each level.

Worked example, four leaves:

```
h0 = H(0x00||d0)   h1 = H(0x00||d1)   h2 = H(0x00||d2)   h3 = H(0x00||d3)
h01 = H(0x01||h0||h1)                 h23 = H(0x01||h2||h3)
root = H(0x01||h01||h23)
```

Proof for leaf 2 (index 2, 0-based; binary 10): `[h3, h01]`.

Verify: `h2` from the leaf; `H(0x01||h2||h3) = h23`; `H(0x01||h01||h23) = root`.

The **consistency proof** shows that an older tree is a prefix of a newer one: the old root can be recomputed from the new tree. This is what makes the log append-only across checkpoints.

### In the slice

- The log appends each signed statement as a leaf.
- The receipt carries `[tree_size, leaf_index, inclusion_path]`.
- The verifier recomputes the root, checks it under the receipt signature, and compares it with the checkpoint.

### Exercise

1. Build a tree of 8 leaves. Print the root.
2. Generate and verify an inclusion proof for leaf 5. Then tamper with the leaf, the index, the path, and the tree size, one at a time. Record which check fails.
3. Generate a consistency proof from size 4 to size 8. Verify it. Then tamper with the old root.

### Common bugs

- Missing or wrong `0x00` / `0x01` prefixes.
- Promoting an odd node instead of carrying it correctly.
- Using the index bits in the wrong order.
- Using the tree size at proof time instead of the size at append time.
- Accepting a proof whose path length does not match the tree size.

### Checkpoint

You can compute a four-leaf root by hand and explain why the proof for leaf 2 is `[h3, h01]`.

### Reading

- RFC 9162 §2.1 (tree hash, inclusion, consistency).
- RFC 6962 §2.1 for the original wording.

---

## 6. Module 2 — COSE and RFC 9942 receipts

### Concept

COSE is a CBOR encoding for signatures and encryption. `COSE_Sign1` is a tagged array (tag 18):

```
18([
  protected   : bstr,   ; CBOR map, signed
  unprotected : map,    ; not signed
  payload     : bstr / nil,
  signature   : bstr
])
```

The signature is over the CBOR encoding of:

```
["Signature1", protected, external_aad, payload]
```

The **protected** header is signed; the **unprotected** header is not. Sensitive claims go in protected; bulky proofs can go in unprotected if they are bound by something else.

Common headers: `alg` = 1, `kid` = 4, `cty` = 3, CWT claims = 15 (`iss` = 1, `sub` = 2).

### RFC 9942 in one page

RFC 9942 defines receipts as COSE_Sign1 objects with:

| Label | Name | Meaning |
| --- | --- | --- |
| 394 | `receipts` | Array of receipts attached to a statement |
| 395 | `vds` | Verifiable data structure: `1` = RFC9162_SHA256 |
| 396 | `vdp` | Proofs: `-1` inclusion, `-2` consistency |

An inclusion receipt:

```
18([
  protected:   { 1: alg, 395: 1 },
  unprotected: { 396: { -1: [ <<[tree_size, leaf_index, [path...]]>> ] } },
  payload:     nil,          ; detached; the root is computed
  signature:   ...
])
```

The **payload is the Merkle root** (detached: `nil` in the encoding). The signature covers the root. The proof is in the unprotected header; if an attacker changes it, the computed root will not verify under the receipt signature, so verification fails.

Verification order for inclusion (RFC 9942 §5.2.1):

1. Apply the inclusion proof to the leaf → root.
2. Verify the receipt signature over that root.

Verification order for consistency (§5.3.1) is the reverse: signature first, then proof.

### In the slice

- The log returns one inclusion receipt per signed statement.
- The verifier runs the two-step check and then compares the root with the checkpoint.

### Exercise

1. Encode a `COSE_Sign1` receipt with the CDDL from RFC 9942 Figure 1. Decode it and print the headers.
2. Verify an inclusion receipt. Then tamper with: the proof path, the leaf index, the tree size, the payload (root), and the unprotected header. Record which check fails.
3. Explain why tampering with the unprotected `vdp` does not break the signature but still fails verification.

### Common bugs

- Double `bstr` wrapping: the proof is a `bstr` containing CBOR, not a raw map.
- Wrong tag (18 missing or applied twice).
- `alg` mismatch between the protected header and the key.
- Detached payload handled as `nil` but the root not supplied to the signature check.
- Proof in the protected header (it must be unprotected in this profile).

### Checkpoint

You can explain the difference between the issuer signature and the receipt signature, and why the receipt payload is the root.

### Reading

- RFC 9052 §4 (COSE_Sign1, Sig_structure).
- RFC 9942 §4–5 (receipts, inclusion, consistency).
- RFC 9053 (algorithms).

---

## 7. Module 3 — SCITT and RFC 9943

### Concept

SCITT is the architecture that turns signed statements into transparent statements:

- **Signed Statement** — a statement signed by an issuer (`COSE_Sign1`).
- **Transparency Service (TS)** — verifies the statement against a registration policy, appends it to a verifiable data structure, and issues a **receipt**.
- **Transparent Statement** — the signed statement plus its receipt.
- **Relying Party** — verifies the transparent statement and applies its own policy.
- **Auditor** — replays the log and checks consistency.

The properties the TS must provide: **append-only**, **non-equivocation**, **replayability**. The RFC mandates signature verification, protected-header identifiers (`iss`, `sub`), trust-anchor validation, and transparent registration policies.

### What the RFC leaves open

- The API (SCRAPI is a separate draft).
- The verifiable data structure (RFC 9942 allows several; this project uses RFC 9162).
- The registration policy language.
- The domain profile (PoD is not defined).

### In the slice

- Courier and recipient are issuers.
- The log service is the TS.
- The verifier and auditor are relying parties.
- The registration policy says which issuers and subjects are accepted.

### Exercise

1. Write a registration policy for the slice: accepted issuers, accepted subjects, accepted algorithms, maximum statement size.
2. List the RFC 9943 MUSTs you implement, and the ones you do not. This list becomes your conformance matrix.
3. Explain why the receipt is not a signature on the statement.

### Checkpoint

You can map every actor in the slice to a SCITT role and name the three properties the TS must provide.

### Reading

- RFC 9943 §3–7 (terminology, architecture, signed statements, transparent statements).
- RFC 9943 §9 (security considerations: ordering, accuracy, key management).

---

## 8. Module 4 — The log service

### Concept

A transparency log is an append-only Merkle tree with a **checkpoint**: a signed statement of the tree size and root at a point in time. The checkpoint is what a verifier pins. **Witnesses** cosign checkpoints and gossip; if the log shows different checkpoints to different parties, the witnesses refuse or expose it.

Tessera is a Go library for tile-based logs. It handles the tree, tiles, and proofs; you provide the storage and the checkpoint signing.

### In the slice

- One Tessera log.
- A checkpoint signed by the log key.
- Two witnesses with separate keys, cosigning checkpoints.
- The verifier checks the receipt root against a checkpoint and requires **both** witness signatures (2-of-2).

### Exercise

1. Run Tessera locally. Append 10 statements. Fetch an inclusion proof for entry 7 and verify it.
2. Fetch a consistency proof between size 5 and size 10.
3. Run two witnesses. Show them the same checkpoint; both cosign. Then show them different checkpoints; explain what each does.
4. Tamper with the checkpoint size and observe the verifier's failure.

### Common bugs

- Reading the tree size before an async append completes.
- Signing a checkpoint for a root that does not match the current tree.
- Distributing witness keys in the same place as the log key.
- Accepting a receipt without checking the checkpoint size.

### Checkpoint

You can explain why the receipt alone is not enough without a checkpoint, and what a witness adds.

### Reading

- Tessera documentation and examples.
- C2SP tlog-tiles and checkpoint specifications (verify current versions).
- Sigsum design paper (witness quorum).

---

## 9. Module 5 — Capture, co-sign, offline

### Concept

- **Capture.** A PWA form, a photo (hash it, do not store the image in the log), a signature canvas. The statement is built in the browser.
- **Co-sign.** Two options: two `COSE_Sign1` statements over the same subject, or a multi-signer `COSE_Sign`. The slice uses two statements: simpler, and it matches the RFC 9943 pattern of distinct Issuers sharing a `sub` to form a coherent statement sequence (RFC 9943 §3 and §5).
- **Offline.** The device can sign without a network. A trusted timestamp needs a TSA, which needs a network. The timestamp obtained later proves the statement existed at timestamp time, not at delivery time. This is a design decision, not a bug.

### In the slice

- Courier signs locally; the statement is queued in IndexedDB.
- On reconnect, the statement goes to the TSA and the log.
- The recipient co-signs at handover, online or queued.
- The dispute bundle records both timestamps and the device time, and the verifier knows which is which.

### Exercise

1. Build a form that produces a canonical statement and signs it with a non-extractable WebCrypto key.
2. Queue a statement offline, reload the page, and submit it later. Verify the signature still holds.
3. Obtain an RFC 3161 token after reconnect. Explain in one paragraph what the token proves and what it does not.
4. Tamper with the device time in the statement; show that the signature still verifies, and explain why the TSA time is the stronger claim.

### Common bugs

- Signing `JSON.stringify` output, which is not canonical.
- Hashing the photo after resizing or re-encoding.
- Losing the private key when storage is cleared.
- Duplicate submission after a retry.

### Checkpoint

You can explain the difference between device time, co-signature time, and TSA time, and which one your evaluation relies on.

### Reading

- RFC 3161 §2 (request, response, token, nonce).
- WebCrypto API documentation (non-extractable keys).
- IndexedDB documentation (structured clone, quotas).

---

## 10. Module 6 — Threat model and red-team

### Concept

The system is designed against a specific adversary: an operator who can rewrite, truncate, withhold, or equivocate. The red-team tests whether the design detects each behaviour.

| Attack | Expected detection | Residual |
| --- | --- | --- |
| Log rewrites an entry | Root changes; receipt fails | None if the verifier holds the old receipt |
| Log truncates | Consistency proof fails; checkpoint mismatch | None if a checkpoint is pinned |
| Log withholds an entry | Gap in the issuer-signed sequence; the missing entry has no receipt | Denial of service remains |
| Log equivocates | Witnesses see different checkpoints; gossip exposes | Witness collusion |
| Courier denies signing | Signature verifies against the enrolled key | Key theft claim; needs a trust anchor |
| Stolen device signs | Revocation flags the key after the fact | Statements before revocation remain |
| Replay of an old statement | Nonce and sequence; duplicate detection | None if checked |
| Witness collusion | Two of two witnesses cosign a bad head | Detection fails; the guarantee is only as strong as the quorum |
| TSA compromised | Two TSAs; audit | Backdating window |
| Offline forgery | TSA time after reconnect; co-signature | Device time is untrusted |

### Exercise

1. Run the **ten** attacks in PRP-001 §8.4 (the table above is a summary). For each, record: detected (yes/no), which check caught it, and what remains possible.
2. Demonstrate the split view: two verifiers, two checkpoints, one witness.
3. Write the residual-risk paragraph for the dissertation.

### Checkpoint

You can state, for each attack, which component is responsible and what the residual risk is.

### Reading

- RFC 9943 §9 (security considerations).
- RFC 9162 §2.1.3–2.1.4 (proof verification).
- RFC 3161 §6 (security considerations).

---

## 11. Module 7 — Evaluation and writing

### Concept

The evaluation is the contribution. The **canonical hypotheses are PRP-001 §8.1 (H1–H6)**; this module does not define a second set. It lists the measures used to test them:

| ID | Claim (from PRP-001 §8.1) | Measure |
| --- | --- | --- |
| H1 | Level 3 detects equivocation that level 2 misses; its receipts make the issuer's signed sequence third-party visible, so a verifier can detect a gap | Red-team detection of equivocation and withholding |
| H2 | Level 3 stays within 2× of the latency and cost targets | p50/p99 end-to-end; cost per delivery |
| H3 | A single compromised witness cannot produce an undetected fork; the 2-of-2 quorum is the binding assumption | Collusion analysis |
| H4 | Offline time is the weakest link | Failure rate and ambiguity |
| H5 | Level 5 adds no benefit for this threat model | Level-5 cost and trust comparison |
| H6 | Couriers and recipients complete the flow within 120 s and under 20% task failure | Task time and errors |

Do not add a new hypothesis here. If the evaluation needs one, add it to PRP-001 §8.1 and cite the ID.

Baselines: a signed database (level 1), a hash chain + TSA (level 2), the slice (level 3). If time permits, a cost model for the permissionless paper (level 5).

### Exercise

1. Write the results tables with empty cells before running anything.
2. Define the oracle for each test (what counts as "detected").
3. Write the threats-to-validity section: simulated witnesses, one deployment, no real couriers, synthetic data.

### Checkpoint

You can explain what result would falsify each claim, and what you would report if the result is negative.

### Reading

- PRP-001 §8.
- Any empirical-methods chapter from the department.

---

## 12. The 16-week plan (18 h/week)

| Week | Build | Writing track |
| --- | --- | --- |
| 1 | Module 0; statement + signature | — |
| 2 | Module 1; Merkle tree + inclusion | Problem and scope notes |
| 3 | Module 1; consistency; spike review | Threat model draft |
| 4–5 | Module 2; RFC 9942 receipts | Threat model and requirements |
| 6–7 | Modules 3–4; Tessera + checkpoints | Architecture chapter |
| 8 | Module 4; witnesses and gossip | Architecture chapter |
| 9–10 | Module 5; PWA capture + co-sign | Design decisions chapter |
| 11–12 | Offline, TSA, dispute, revocation | Implementation chapter |
| 13 | Module 6; red-team | Security evaluation |
| 14 | Module 6; conformance and red-team | Security evaluation |
| 15 | Module 7; evaluation | Results chapter |
| 16 | Viva prep and revision | Discussion, conclusion, revision |

At 18 h/week, 16 weeks is 288 h; the 290 h plan fits within the estimate's error. Writing is a parallel track from week 2, not a phase at the end. If the build slips, use the cut ladder in PRP-001 §9.4; if the sustainable pace is below 18 h/week, plan for 18–20 weeks, not a compressed schedule.

---

## 13. Reading order

| Order | Text | Sections | Pages (approx.) |
| --- | --- | --- | --- |
| 1 | RFC 9162 | §2.1 (tree, inclusion, consistency) | 10 |
| 2 | RFC 9942 | §4–5 (receipts, RFC9162_SHA256) | 15 |
| 3 | RFC 9943 | §3–7, §9 (architecture, security) | 25 |
| 4 | RFC 9052 | §4 (COSE_Sign1) | 10 |
| 5 | RFC 8949 | §3–4 (CBOR, deterministic encoding) | 15 |
| 6 | RFC 3161 | §2, §6 (TSA) | 12 |

Do not read the appendices or the IANA sections on the first pass.

---

## 14. Tooling

| Need | Go | Rust | TypeScript | Python |
| --- | --- | --- | --- | --- |
| CBOR | `fxamacker/cbor` | `ciborium` | `cbor-x` | `cbor2` |
| COSE | `veraison/go-cose` | `coset` | `cose-js` (verify maturity) | `pycose` (verify maintenance) |
| Log | Tessera, Trillian | — | — | — |
| Hashing/signing | stdlib `crypto` | `ring`, `ed25519-dalek` | WebCrypto, `@noble/*` | `cryptography` |
| TSA client | `openssl ts` | `openssl ts` | server-side | `openssl ts` |
| Tests | `go test` | `cargo test` | `vitest` | `pytest` |

Use one language for the service and the verifier if possible; two languages doubles the encoding bugs.

---

## 15. Glossary

This is a short subset; **REF-001 is the canonical glossary**.

| Term | Meaning |
| --- | --- |
| Leaf | Hash of one log entry, prefixed `0x00` |
| Node | Hash of two children, prefixed `0x01` |
| Root | Top hash of the Merkle tree at a given size |
| Inclusion proof | Path from a leaf to the root |
| Consistency proof | Proof that an old tree is a prefix of a new one |
| Checkpoint | Signed tree size and root |
| Witness | Independent party that cosigns checkpoints |
| Receipt | Signed proof of inclusion (RFC 9942) |
| Signed Statement | Issuer-signed statement (`COSE_Sign1`) |
| Transparent Statement | Signed statement plus receipt |
| TS | Transparency Service |
| Relying Party | Verifier of transparent statements |
| Equivocation | Showing different histories to different parties |

---

## 16. First-week checklist

- [ ] Read RFC 9162 §2.1 once, slowly.
- [ ] Implement a four-leaf tree; print the root.
- [ ] Implement and verify an inclusion proof for leaf 2.
- [ ] Tamper with the leaf, index, path, and size; record the failures.
- [ ] Encode and decode a CBOR map deterministically.
- [ ] Sign and verify a statement with COSE.
- [ ] Write down, in your own words, the difference between the issuer signature and the receipt signature.
- [ ] Run Tessera; append one entry; fetch one proof.

If you can do these eight things, you are ready for the receipt module. If the Merkle proof still feels mysterious, redo the four-leaf example by hand before moving on.

---

## Appendix A — Expected outcomes for selected exercises

### Module 1 (Merkle tree)

| Tamper | Expected failure |
| --- | --- |
| Change a leaf | Leaf hash changes; recomputed root differs |
| Change the leaf index | Verifier uses the wrong sibling order; root differs |
| Change the tree size | Path length or index range is invalid; reject before hashing |
| Remove a path element | Proof length mismatch; reject |
| Change the old root in a consistency proof | Consistency recomputation fails |

### Module 2 (COSE receipt)

| Tamper | Expected failure |
| --- | --- |
| Change the proof path | Computed root differs; receipt signature does not verify |
| Change the leaf index | Wrong sibling order; computed root differs |
| Change the tree size | Invalid path length or index; computed root differs |
| Change the payload/root | Receipt signature fails |
| Change the unprotected `vdp` | Signature still verifies (it does not cover the header), but the computed root differs from the signed root, so the receipt check fails |

### Module 4 (witnesses)

| Scenario | Expected behaviour |
| --- | --- |
| Both witnesses see the same checkpoint | Both cosign; the quorum is met |
| Witnesses see different checkpoints | Each compares against its last cosigned checkpoint; a witness that sees a fork **refuses to cosign**, retains the old checkpoint, and raises evidence. It does not "pick the longer chain." |
| One witness is offline | The 2-of-2 quorum is not met; the verifier treats the checkpoint as unconfirmed until both return |

### Module 6 (red-team)

| Attack | Expected detection |
| --- | --- |
| Rewrite | Root changes; receipt fails |
| Truncate | Consistency proof fails; checkpoint mismatch |
| Withhold | Gap in the issuer-signed sequence; the missing entry has no receipt |
| Equivocate | Two checkpoints; witnesses/gossip expose the fork |
| Courier denies | Signature verifies; identity binding decides |
| Stolen device | Revocation flags the key; pre-revocation tokens remain valid; the compromise window is indeterminate |
| Replay | Nonce and sequence duplicate detection |
| Clock skew / offline forgery | Device time is untrusted; TSA time after reconnect; co-signature time |
| Witness collusion | 2-of-2 fails if both collude; the quorum is the trust assumption |
| TSA compromise | Backdating window; multiple TSAs and audit |
