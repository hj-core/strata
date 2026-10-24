# DEC-003a — Section 5.1 Comparison Handout

> *Companion to `DEC-003` and `CNP-004` v0.6.4. One-page view of the comparison for the supervisor meeting.*

## The five arms

| Arm | Design | Role |
| --- | --- | --- |
| **A** | A signed certificate and a telephone call | Current practice; a generic description, not a verified account of Macau practice |
| **B** | A signed DCC under X.509 with an RFC 3161 token | The strongest per-document baseline; the CA binds the key, the signature gives non-repudiation |
| **C** | A central registry with a certificate hash, key directory, subject index and signed TSA-timestamped receipt; no Merkle log | The strongest non-log baseline |
| **D1** | C plus a Merkle log, signed checkpoints and a subject index, without witnesses | The ablation that isolates the witness layer |
| **D2** | D1 plus 2-of-3 witnesses, with at least 2 of the 3 assumed honest and each checking a consistency proof before cosigning | **The proposal** |

## The matrix

Legend: ✓ yes · ◐ partial · ✗ no · † conditional on the operator not equivocating, or on the party having cached a checkpoint before the event being proved.

| Question | A | B | C | D1 (ablation) | D2 |
| --- | --- | --- | --- | --- | --- |
| Certificate authentic and unaltered | ◐ | ✓ | ✓ | ✓ | ✓ |
| Key belongs to the named laboratory | ✗ | ✓ | ✓ | ✓ | ✓ |
| The laboratory's key signed this certificate | ◐ | ✓ | ✓ | ✓ | ✓ |
| Signing time independent of the laboratory | ✗ | ✓ | ✓ | ✓ | ✓ |
| The laboratory cannot deny issuance | ✗ | ✓ | ✓ | ✓ | ✓ |
| **Commitment: offline proof the certificate entered the archive within 1 cadence** | ✗ | ✗ | ◐† | ◐† | **✓** |
| Enumerate the certificates for an asset | ✗ | ✗ | ◐† | ◐† | ✓ |
| Show a second certificate for the same asset-period | ✗ | ✗ | ◐† | ◐† | ✓ |
| **Provable absence: O(log N) proof a scheduled entry is absent** | ✗ | ✗ | ✗ | ◐† | **✓** |
| **Tamper evidence: detect deletion or alteration of a logged entry** | ✗ | ✗ | ◐† | ◐† | **✓** |
| **Backfill detection: a period empty at its deadline holds a leaf at query time** | ✗ | ✗ | ✗ | ◐† | **✓** |
| Offline verification without trusting operator history | ✗ | ✓ (sig only) | ✗ | ✗ | ✓ |
| **Non-equivocation: prevent an un-cosigned fork; detect a cosigned fork** | ✗ | ✗ | ✗ | ✗ | **✓** |
| Survives deletion of the registry page | ✗ | ✓ (file only) | ✓ (receipt only) | ✓ | ✓ |
| The work was performed | ✗ | ✗ | ✗ | ✗ | ✗ |
| All certificates disclosed (completeness) | ✗ | ✗ | ✗ | ✗ | ✗ |
| Legal recognition | ◐ | ◐ | ✗ | ✗ | ✗ |

## How to read it

**A, B and C are existing or non-log practice.** D1 is not a baseline: it is C plus the log, checkpoints and subject index, without witnesses. D2 is D1 plus the witness layer. D2 is the proposal. The distinction matters because D1 shares the proposal's data structures; it is the ablation, not a competitor.

**C is the strongest non-log baseline.** It can commit with a signed, TSA-timestamped receipt and can enumerate what it holds, but its enumeration is an unproven database response: it cannot prove that the returned list is complete. It cannot prove absence, it cannot detect deletion of an entry no holder has a receipt for, and it cannot prove the state at a past deadline. Its commitment mark is ◐† because a receipt asserts receipt, not inclusion in a persistent log.

**D1 adds the Merkle structures.** It introduces inclusion, non-inclusion, consistency and deadline proofs. They are against an operator-signed checkpoint. Without a checkpoint cached before the event, a post-incident investigator cannot tell a fork from a consistent history: the operator can hand over a different branch, and the investigator must either trust the operator or run the O(N) `--audit`. Its commitment, enumeration, second-certificate, absence, tamper and backfill rows are therefore ◐†.

**D2 adds the witness layer.** At least 2 of the 3 witnesses must be honest, each checks a consistency proof before cosigning, and at least 2 sign the audit attestation after verifying the projection. A divergent checkpoint cannot obtain 2 cosignatures. A verifier holding a single checkpoint can rely on it. A cosigned fork is detected by comparison. D2 is the only arm with unconditional proofs on all five archive rows — commitment, absence, tamper evidence, backfill and non-equivocation — against an untrusted operator.

## Trust assumptions

| Arm | What the party must believe |
| --- | --- |
| **A** | The person who answers the telephone |
| **B** | The CA bound the key, the key was not compromised before revocation, and the TSA is honest |
| **C** | The operator's account of its own contents; the operator does not omit an entry; the TSA is honest |
| **D1** | The operator does not equivocate, or the party cached the relevant checkpoint before the event; the index attestation is operator-signed; the TSA is honest |
| **D2** | At least 2 of the 3 witnesses are honest and each checks a consistency proof before cosigning; the party holds the relevant checkpoint or snapshot; the TSA is honest; the 2-of-3 audit attestation is checked for the query checkpoint and each deadline checkpoint; a cosigned fork is detected by comparison, not by inspecting a single checkpoint |

## What the matrix does not show

- No arm establishes that the work was performed, that the archive contains every certificate issued, that the individual signer was authorised, or that the register has legal recognition.
- The marks compare proof strength and trust, not implementation cost.
- The quantitative comparison — detection, latency, proof size and trust-anchor count — is the evaluation in `CNP-004` C7, not this table.

*Source: `CNP-004` v0.6.4 §5.1. The full notes and limits are in §4.5 and §5.1 of that document.*
