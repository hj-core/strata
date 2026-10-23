# PRP-001a — SHP-42 Walkthrough with the Four-Statement Chain

| Field | Value |
| --- | --- |
| Document ID | PRP-001a |
| Title | Proposed replacement for PRP-001 §6.2a — dispatch → collect → arrival → receipt |
| Version | 0.1.0 |
| Status | Draft for review — **does not modify PRP-001** |
| Last Updated | 2026-10-23 |
| Related documents | PRP-001 v0.2.0 §6.2, §6.2a, §8.4, Appendix B; ANL-005; TUT-001 Module 6 |
| Purpose | Replace the two-statement parcel demo with a four-statement chain built on a general handover primitive |
| Supersedes | PRP-001 §6.2a **if accepted**. PRP-001 is otherwise unchanged. |

---

## 1. What changes, and why

The current §6.2a has two statements and no origin assertion: the parcel appears at the courier's door, and the consignor is a relying party only. That leaves three gaps:

1. nothing asserts **what left the sender**;
2. there is no **bounded window** between the carrier taking custody and the recipient signing — collect at 09:00 and receipt at 17:00 are equally valid, and anything could happen in between;
3. "no receipt" is a **silent absence**, indistinguishable from loss, refusal, or theft.

This revision fixes all three with four statements and one general rule.

---

## 2. The handover primitive

The profile is not a schema. It is a **two-phase handover**:

```
handover := ( giver statement , taker statement , window )

  taker.prev   = SHA256(canonical COSE_Sign1 bytes of giver statement)
  valid        iff  taker.time − giver.time  ≤  window
```

- The **giver** asserts that the subject is being handed over, at a time and place.
- The **taker** asserts that it accepted the same subject, bound to the exact signed bytes.
- The **window** is policy, not cryptography. It is what turns "no taker statement" from silence into a named outcome.

A handover that fails to close within its window is reported as **`handover_incomplete`** — a positive, named result, not an absent record.

### 2.1 The two variants

| Variant | Handovers | Statements | Issuers |
| --- | --- | --- | --- |
| **Document** | 1: issuer → holder | 2: `issue` → `receipt` | 2 |
| **Parcel** | 2: sender → carrier; carrier → recipient | 4: `dispatch` → `collect`; `arrival` → `receipt` | 3 |

The envelope, receipts, witnesses and TSA are identical. Only the chain depth changes. This is the profile's real content: the same primitive composed to different depths.

---

## 3. The chain

| # | Event | Issuer | Asserts | `prev` | Sequence |
| --- | --- | --- | --- | --- | --- |
| 1 | `dispatch` | Acme (consignor) | Contents left Acme at T₁ | `null` | Acme 1 |
| 2 | `collect` | Dana (carrier) | Carrier took custody at T₂; packaging condition | → dispatch | Dana 1 |
| 3 | `arrival` | Dana (carrier) | Handover offered at the door at T₃ | → collect | Dana 2 |
| 4 | `receipt` | Sam (consignee) | Accepted at T₄; condition | → arrival | Sam 1 |

Each handover is a pair: (**dispatch**, **collect**) for pickup and (**arrival**, **receipt**) for delivery.

`arrival` is signed **at the start of the handover**, before the physical transfer. It asserts that the handover is being offered — not that it was completed. Completion is the taker's signature alone. This distinction is deliberate; see §7.

---

## 4. Cast

| Actor | Party | Key | Role |
| --- | --- | --- | --- |
| Acme Instruments | Consignor | Consignor key (server-side or HSM; low theft risk) | **Issuer:** dispatch |
| Dana | Northline Logistics | Courier device key (mobile; high theft risk) | **Issuer:** collect, arrival |
| Sam | BioLab | Recipient device key | **Issuer:** receipt |
| ClearLog | Independent log operator | Checkpoint key | Transparency Service |
| UniAudit, Aegis Audit | Two witnesses | Witness keys | Cosign checkpoints |
| InsureCo | Insurer | None | Relying party |
| FreeTSA | Timestamp authority | TSA key | Trusted time |
| UniCA | Certification authority | CA root | Binds keys to parties |

Shipment `SHP-42`: one calibrated spectrometer, Acme to BioLab.

Note the key-risk asymmetry: Acme signs from a fixed location; Dana signs twice from a mobile device; Sam signs once at the door. §8 states the consequences.

---

## 5. Phases

### Phase 0 — Enrolment

UniCA issues certificates binding each public key to a party and role: `consignor:acme`, `carrier:northline:courier:dana`, `consignee:biolab:recipient:sam`.

ClearLog publishes its registration policy:

| Policy field | Value (default, to calibrate) |
| --- | --- |
| Accepted issuers | `consignor:acme:*`, `carrier:northline:*`, `consignee:biolab:*` |
| Accepted subjects | `shipment:*` |
| Algorithms | Ed25519 or ES256 |
| Max statement size | 4 KB |
| `prev` requirement | `collect`, `arrival`, `receipt` must carry `prev` |
| **`max_pickup_window`** | **24 h** — dispatch → collect |
| **`max_handover_window`** | **30 min** — arrival → receipt |
| `place` precision | Coarse (geohash ≤ 6 chars) or omitted |

Acme configures five anchors: ClearLog's checkpoint key, the two witness keys, FreeTSA's key, and UniCA's root.

The two windows are **design defaults, not measurements.** They are swept in the evaluation.

### Phase 1 — Dispatch (Acme)

Acme builds `S_d` in canonical CBOR:

```
{
  statement_id: "st-001",
  sequence:     1,
  shipment_id:  "SHP-42",
  issuer:       "consignor:acme",
  subject:      "shipment:SHP-42",
  event:        "dispatch",
  time:         "2026-10-23T09:12:00Z",
  place:        { geohash: "wecp..." },
  items:        [ { sku: "SPEC-1", qty: 1, condition: "sealed" } ],
  photos:       [ "sha256:..." ],
  prev:         null,
  nonce:        "..."
}
```

Acme signs `S_d`, obtains an RFC 3161 token over `SHA256(signed_S_d)`, submits it to ClearLog, and receives a receipt. A QR code carrying the statement ID and a pointer to the signed bytes is attached to the parcel.

`S_d` asserts only what Acme is authoritative about: **what left the premises.** It does not assert that the goods work, that the declaration is accurate, or that the carrier received them.

### Phase 2 — Collect (Dana)

Dana scans the QR, fetches `S_d`, verifies Acme's signature offline, and checks that `S_d` is addressed to `SHP-42`. She records packaging condition and signs `S_c` with `prev = SHA256(signed_S_d)`, `sequence = 1`.

```
{
  statement_id: "st-002",
  sequence:     1,
  shipment_id:  "SHP-42",
  issuer:       "carrier:northline:courier:dana",
  subject:      "shipment:SHP-42",
  event:        "collect",
  time:         "2026-10-23T10:40:00Z",
  place:        { geohash: "wecp..." },
  condition:    "sealed, packaging intact",
  prev:         "sha256(signed_S_d)",
  nonce:        "..."
}
```

`collect` asserts custody and **apparent** condition. The carrier does not attest to the contents, which it cannot see.

**If pickup is not co-present** — a locker, a shop counter, a depot — `dispatch` and `collect` are two unilateral assertions rather than a witnessed exchange. That is permitted; the pair still binds, and the window still applies. Do not describe it as a co-signed handover when it is not.

### Phase 3 — Arrival (Dana)

At the address, before the physical handover, Dana signs `S_a` with `prev = SHA256(signed_S_c)`, `sequence = 2`.

```
{
  statement_id: "st-003",
  sequence:     2,
  shipment_id:  "SHP-42",
  issuer:       "carrier:northline:courier:dana",
  subject:      "shipment:SHP-42",
  event:        "arrival",
  time:         "2026-10-23T14:02:10Z",
  place:        { geohash: "wecp..." },
  prev:         "sha256(signed_S_c)",
  nonce:        "..."
}
```

`arrival` asserts that a handover is being offered at T₃. It does **not** assert delivery. Its purposes are to bound the handover window and to give the carrier a voice that cannot overclaim.

If Dana is offline, `S_a` is queued and timestamped on reconnect. The token then proves existence at reconnect time, not at the door — the verifier must apply the offline rule (§6.6 of PRP-001) and say so.

### Phase 4 — Receipt (Sam)

Sam's app verifies the chain offline — `S_d`, `S_c`, `S_a` — and checks:

- Acme's and Dana's signatures;
- each `prev` resolves to the exact signed bytes;
- `S_a.time − S_c.time` and `S_c.time − S_d.time` are inside policy windows;
- the items and condition are acceptable.

Sam then signs `S_r` with `prev = SHA256(signed_S_a)`, `sequence = 1`.

```
{
  statement_id: "st-004",
  sequence:     1,
  shipment_id:  "SHP-42",
  issuer:       "consignee:biolab:recipient:sam",
  subject:      "shipment:SHP-42",
  event:        "receipt",
  time:         "2026-10-23T14:05:30Z",
  condition:    "sealed, packaging intact",
  prev:         "sha256(signed_S_a)",
  nonce:        "..."
}
```

`receipt` is the **only** statement that completes the delivery handover. A missing receipt is never a successful delivery.

### Phase 5 — Trusted time

Each app sends only `SHA256(signed_statement)` to FreeTSA. The returned token's `messageImprint` is that hash and its `genTime` is the TSA clock. Tokens are stored beside the statements, never inside them.

Four statements, four tokens. `S_a`'s token proves the arrival assertion existed at the TSA time; it does not prove Dana was at the address.

### Phase 6 — Registration and receipts

The app submits all four statements. ClearLog checks signatures and certificate chains, the protected-header `iss`/`sub` against the body, issuer authority for the subject, algorithm, size, `prev` presence, and **sequence continuity per issuer**.

It computes leaves `L = SHA256(0x00 || signed_statement)`, appends four leaves, and returns an RFC 9942 inclusion receipt per statement. The app now holds four transparent statements.

Acme's dispatch may be registered before the parcel is collected; ordering in the log follows submission, not custody.

### Phase 7 — Checkpoint and witnesses

ClearLog signs `CP = { origin: "clearlog", size: n, root }` and sends a consistency proof to both witnesses. Each verifies against the last checkpoint it cosigned and returns a cosignature. The witnesses gossip the cosigned checkpoint.

The witness function here is **anti-backdating**: a cosigned checkpoint fixes what was in the tree at time T. Neither Acme nor Dana can insert a statement into a past tree state to match a fabricated invoice.

### Phase 8 — Verification at handover (Sam)

Ten checks, named on failure:

1. Acme's signature over `S_d`, using Acme's certificate.
2. Dana's signatures over `S_c` and `S_a`, using Dana's certificate.
3. Sam's own signature over `S_r`.
4. `prev` chain: `collect → dispatch`, `arrival → collect`, `receipt → arrival`, each resolving to the exact signed bytes.
5. Sequence continuity: Acme `1`; Dana `1, 2`; Sam `1`.
6. Inclusion: each of the four leaves, with its path, against the checkpoint root at size `n`.
7. ClearLog's receipt signature over `root`.
8. `root` equals the checkpoint root; the checkpoint carries two valid witness signatures.
9. TSA tokens match the statement hashes and fall inside the applicable windows.
10. Policy: issuers, subjects, and **window compliance** — `collect − dispatch ≤ max_pickup_window`; `receipt − arrival ≤ max_handover_window`.

Named outcomes beyond the ten failures:

| Outcome | Meaning |
| --- | --- |
| `dispatch_not_collected` | Dispatch exists; no collect inside `max_pickup_window` |
| `announced_handover_incomplete` | Arrival exists; no receipt inside `max_handover_window` |
| `window_exceeded` | Both statements exist but the interval exceeds policy |

A failure names the check that failed, not "invalid."

### Phase 9 — Verification by the consignor

Acme's portal fetches the four transparent statements, runs the same checks, and applies its policy: both handovers closed; issuers as expected; windows met; condition fields agree or the discrepancy is flagged.

Acme trusts the witnesses and the cryptography, not ClearLog's honesty. It needs ClearLog only for availability.

### Phase 10 — Dispute

The insurer receives a bundle: four signed statements, four receipts, four TSA tokens, the checkpoint with two witness signatures, and the CA certificates. It verifies offline, with no access to ClearLog.

The bundle establishes who asserted what, when, in what order, and that nothing changed since signing. It does not establish who damaged the pallet.

**The `announced_handover_incomplete` outcome is evidence, not a verdict.** It narrows the question from "did it arrive?" to "the carrier announced a handover and no receipt followed," which is a better question to ask a courier.

### Phase 11 — Revocation

If Dana's phone is stolen at 16:00 and UniCA revokes at 16:30, the verifier checks revocation **at the TSA time**. Statements signed in the compromise window are indeterminate: they may be queued-legitimate or forged. `arrival` and `receipt` are the most exposed, because they are signed in the field.

Short-lived courier keys and witness freshness narrow the window. The design states the residual rather than hiding it.

### Phase 12 — Audit

The auditor replays ClearLog from its checkpoints, verifies consistency and a sample of inclusion proofs, and checks each issuer's **signed** sequence. A missing entry leaves a gap in that issuer's sequence.

Dana's sequence `1, 2` is the completeness check for the carrier's own events: a carrier that omits a collect or arrival leaves a visible gap in its own counter. The gap does not reveal what the missing statement said.

---

## 6. The state machine

```
                 ┌──────────────┐
   T1  dispatch  │              │
  ──────────────▶│  handover 1  │──── collect within 24 h ──▶ ok
                 │              │──── window expires ────────▶ dispatch_not_collected
                 └──────────────┘
                        │
                        ▼
                 ┌──────────────┐
   T3  arrival   │              │
  ──────────────▶│  handover 2  │──── receipt within 30 min ──▶ ok
                 │              │──── window expires ─────────▶ announced_handover_incomplete
                 └──────────────┘
```

A valid delivery requires **both** handovers closed. The verifier reports the first branch that fails.

---

## 7. Why `arrival` is signed before the transfer

Three reasons, and the third is the one that matters for device theft.

1. **It bounds the handover window.** Collect at 09:00 and receipt at 17:00 leaves an eight-hour hole. Arrival at 14:02 and receipt at 14:05 leaves three minutes.
2. **It is signed while the carrier still has the parcel.** The carrier's assertion is not contingent on the recipient's cooperation after the goods have changed hands.
3. **A forged arrival cannot close the chain.** It asserts that a handover was offered, not that it completed. Even with Dana's device, an attacker can mint arrivals but not receipts, and an arrival without a receipt is evidence of a *failed* handover, not a successful one.

This is why the courier's field signature is acceptable here but a field **completion** signature would not be. Same physical exposure, far lower consequence.

**What `arrival` does not prove:** that Dana was at the address. Location is self-reported and can be spoofed. It is an assertion, not a fact.

---

## 8. Key risk by role

| Role | Location | Theft risk | Control |
| --- | --- | --- | --- |
| Acme (consignor) | Office, warehouse | Low | Server-side or HSM signing; organisational key |
| Dana (carrier) | Street, vehicle, doorstep | **High** | Device-bound short-lived key; biometric/PIN; fast revocation; **receipt required as backstop** |
| Sam (recipient) | Home, office | Medium | Device-bound key; duress consideration |

The courier key is the weakest link and will sometimes be compromised. The controls bound the window; they do not eliminate it.

---

## 9. Privacy

The `arrival` statement is the **highest-linkage record in the profile**: it places a named courier at a named recipient's address at a precise time. That is more sensitive than the pickup location and more sensitive than the receipt, which need not carry a place at all.

Requirements:

- `place` precision is policy-limited (geohash ≤ 6 characters) or omitted for `arrival`;
- the log stores hashes only; the payload is off-chain and encrypted;
- the arrival event is named in the linkage analysis (C5) as the principal re-identification risk;
- an arrival statement that never closes is retained under the same retention rule as any other statement, not longer.

---

## 10. Effect on the red-team suite

The canonical ten cases in PRP-001 §8.4 are unchanged. This revision changes which cases the parcel arm can detect:

| Case | Effect of the four-statement chain |
| --- | --- |
| 3 — operator withholds a record | Sequence gaps now exist for **three** issuers, not two |
| 5 — sender denies signing | Acme now signs; denial is possible and evidenced |
| 6 — stolen device signs a false statement | Detectable in outcome: a forged `arrival` cannot close a handover |
| 7 — replay of an old statement | `prev` binds each receipt to exact bytes; replay fails the chain |
| 8 — clock skew / offline forgery | `arrival` is the most exposed; window checks surface the skew |

**New attack, existing case.** A courier could batch several `arrival` statements at the start of a shift, making the timestamps meaningless. This is a *policy discipline* attack, not a cryptographic one, and it belongs under case 6 as a sub-case. Detection: clustered arrival times, or arrivals inconsistent with the recorded route. Do not add an eleventh case; the list is canonical.

---

## 11. Open questions

1. **Pickup topology.** Is Acme → Northline co-present, or a locker/depot drop? This decides whether handover 1 is a witnessed exchange or two unilateral assertions. It does not change the protocol.
2. **Window values.** 24 h and 30 min are defaults, not measurements. They must be calibrated against real delivery practice and swept in the evaluation.
3. **Failure statements.** Should the carrier be able to sign `attempted`/`refused`/`damaged`, or is the named outcome sufficient? This slice assumes the latter; a failure statement would take the chain to five.
4. **Multi-leg.** A carrier-to-carrier transfer needs an extra handover pair. Out of scope, but the primitive composes.
5. **Consignor key custody.** Server-side signing protects the key but weakens the "device signed" claim. The decision log must state which is used and why.

---

## 12. Non-claims (unchanged)

- It does not prove the parcel arrived intact, that the contents were as declared, or that anyone inspected them.
- It does not prove Dana was at the address; `arrival` is an assertion.
- It does not prove Sam read anything.
- It is evidence, not a qualified trust service; not a blockchain.
- The 24 h and 30 min windows are design defaults, not measurements.

---

## 13. One-paragraph summary

The parcel demo becomes four statements over two handovers: **dispatch** (Acme) → **collect** (Dana) → **arrival** (Dana) → **receipt** (Sam), each linked by `prev` to the exact signed bytes of its predecessor. The profile's unit is not a schema but a **handover primitive** — a giver assertion and a taker assertion within a policy window — composed once for documents and twice for parcels. `arrival` is signed at the start of the handover, which bounds the handover window, gives the carrier a voice, and ensures that a stolen courier device can mint arrivals but never close a chain. The verifier reports named outcomes rather than silence: `dispatch_not_collected`, `announced_handover_incomplete`, `window_exceeded`. The two windows are design defaults, not measurements. The system still proves assertions, inclusion, order, time and non-equivocation — never the physical event.
