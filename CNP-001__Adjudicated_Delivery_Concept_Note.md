# CNP-001 — Concept Note: Evidence for Adjudicated Delivery

| Field | Value |
| --- | --- |
| Document ID | CNP-001 |
| Title | Concept note — third-party verifiable evidence for payment-linked delivery disputes |
| Version | 0.1.0 |
| Status | Draft for supervisor review |
| Date | 2026-10-23 |
| Module | COMP4299 — system development; dissertation + demo + evaluation |
| Related documents | PRP-001 v0.2.0 (baseline); PRP-001a (four-statement chain); ANL-006 (anchoring cases); ANL-007 (strongest-alternative test) |
| Purpose | State the problem, the unmet demand, the proposed solution, the demonstration, and the honest boundary in one page for review before rewriting the proposal |

---

## 1. The problem

Delivery disputes are adjudicated by a party that was **not present**: an insurer, an arbitrator, or a court. It receives incompatible records from a supplier, a carrier and a payer, and it has no right of access to any of their systems. Today it resolves the conflict by investigation, forensic expertise, and bargaining power.

Four adjudicated cases show what this costs:

- *Leicht v. Pallet Central* (Wis. 2019): a supplier forged receipt signatures and billed ~$505,000 for pallets never delivered. Detected two years later, by noticing that the **number** of invoices had risen.
- *US v. Howe* (3d Cir. 2008): an altered bill of lading and **backdated** invoices extracted $152,850 for undelivered goods.
- *US v. Williams* (9th Cir. 1996): a letter carrier forged the recipient's delivery receipt. Detected by handwriting analysis.
- *Maersk Sealand* (2004): delivery against forged bills of lading; the court recorded that the agents "lacked the ability to compare the bills as tendered with the Claimants' electronic record of the original bills."

The common structure: **the authoritative record is held by the party with the incentive to falsify it, and the relying party has no independent copy.**

---

## 2. The unmet demand

An adjudicator with no audit rights cannot currently establish three things:

| Requirement | Why it matters | Who provides it today |
| --- | --- | --- |
| That a set of acceptances is **complete** | Leicht's fraud was a count anomaly | Nobody, to an outsider |
| That neither party showed a **different history** to the other | Split-view fraud | Nobody |
| **Offline**, without access to either party's systems | The insurer is not a party | Nobody |

Existing controls each solve part of the problem, but all are verifiable only by the parties to them, or by an auditor with a right of access:

| Control | Solves | Verifiable by |
| --- | --- | --- |
| ERP three-way matching | Payment without receipt | The buyer only |
| eIDAS QES | Signature authenticity | Anyone, but for one document |
| eIDAS QERDS | Send and receipt with time | The parties |
| EDI `RECADV` + signed MDN | Bilateral acknowledgment | The two trading partners |
| e-CMR | The legal transport document | The parties and authorities |
| SOC 2 / surveyor | Assurance over controls, or one loss | Annual/sample, or post hoc |

**The demand is real, narrow, and unmet: complete, non-equivocable delivery evidence that a non-party can verify offline.**

---

## 3. The proposed solution

A thin end-to-end slice on a witnessed Merkle transparency log.

| Element | Choice |
| --- | --- |
| Statement | `COSE_Sign1` (RFC 9052/9053), canonical CBOR |
| Chain | `dispatch → collect → arrival → receipt`; the **handover primitive** = giver statement + taker statement + bounded window |
| Log | RFC 9162 Merkle log via Tessera |
| Receipts | **RFC 9942 COSE receipt over an RFC 9162 leaf — implemented here** |
| Time | RFC 3161 token over each signed statement |
| Witnesses | Two independent processes, 2-of-2, cosigning checkpoints |
| Verifier | Offline CLI/web verifier; named failure outcomes |
| Output | A dispute bundle that verifies with no access to the operator |

---

## 4. The scenario

SHP-42: Acme Instruments ships a calibrated spectrometer to BioLab through Northline Logistics. InsureCo adjudicates a damage claim.

- Acme signs `dispatch` — what left the premises.
- Dana (Northline) signs `collect` — custody and apparent condition.
- Dana signs `arrival` — a handover is offered at 14:02.
- Sam (BioLab) signs `receipt` at 14:05.

InsureCo verifies the chain offline and applies policy. The scenario is concrete, plausible, and anchored in the four cases above. No external stakeholder is required; the adjudicator is a role, not an organisation that must be recruited.

---

## 5. What the work will demonstrate

The result is designed to lean positive: the mechanism detects behaviour that existing controls do not detect **at the adjudication boundary**.

| Test | L1 signed records | L2 hash chain + TSA | **L3 witnessed log** |
| --- | --- | --- | --- |
| Forged acceptance presented by the supplier | Rejected (wrong key) | Rejected | **Rejected + no receipt exists** |
| Acceptance omitted from the claimed set | Undetected | Undetected | **Sequence gap + count mismatch** |
| Record backdated to match a false delivery | Undetected | Undetected with a known head | **Checkpoint mismatch** |
| Old identifiers replayed for a new claim | Undetected | Undetected | **Rejected by `prev`/nonce/sequence** |
| Different histories shown to payer and insurer | Undetected | Undetected | **Fork against a cosigned checkpoint** |
| Offline verification by a non-party | Signature only | Signature + token | **Signature + inclusion + witnesses + time** |

Supporting measurements: verification latency, proof size, append throughput, and marginal infrastructure cost. Each is labelled as a target or a measurement.

---

## 6. Contribution

1. **The handover profile.** A giver/taker/window primitive with a defined state machine and named failure outcomes (`handover_incomplete`, `window_exceeded`). Non-trivial semantics; answers the "profile reduces to a schema" objection.
2. **The RFC 9162 receipt path.** `scitt-verifier` supports CCF receipts only. There is no standard offline verifier for RFC 9162 leaves. This is a concrete implementation gap, and the slice fills it.
3. **The demonstration.** A measured comparison of L1, L2 and L3 against five documented fraud patterns, showing which level detects which, and where none of them can.

---

## 7. Honest boundary and non-claims

- It does **not prevent** forged delivery records. A signature prevents per-document forgery; the log prevents denial, backdating, selective presentation, and makes the count provable.
- It is **not** a qualified trust service and carries **no legal recognition**. QES, QERDS and e-CMR are legally stronger, and the dissertation says so.
- It does **not** beat ERP three-way matching on prevention, or EDI on bilateral acknowledgment. It supplies what those cannot: verification by a non-party with no access.
- It does **not** prove physical truth, contents, or that anyone inspected the goods.
- The anchoring cases establish that the pattern is real and adjudicated; they do **not** establish its prevalence.

---

## 8. Evaluation design

| Hypothesis | Falsified if |
| --- | --- |
| H1 — L3 detects all four documented attacks; L1 and L2 each miss at least two | Any attack is undetected at L3, or L2 detects them all |
| H2 — The RFC 9162 receipt path encodes and verifies correctly | A receipt fails against a Tessera log or a test vector |
| H3 — Offline verification of a four-statement bundle completes in under 1 s | p99 exceeds 2 s |
| H4 — Proof size stays under 4 KB at 1M leaves | Measured size exceeds 4 KB |
| H5 — Marginal infrastructure cost is below $0.01 per handover | Measured cost exceeds $0.02 |

Workload: 1,000 handovers/day over 30 simulated days, 10% offline, with the four attack patterns injected at known points. Baselines: L1 (signed records) and L2 (hash chain + TSA). No weighted score.

---

## 9. Risks

| Risk | Mitigation |
| --- | --- |
| The RFC 9162 receipt wrapper is harder than 25 h | Two-week spike; fall back to a Tessera proof checked against a signed checkpoint (PLN-002 Tier 3) |
| Witnesses are simulated on one host | Separate processes and keys; independence stated as simulated |
| The adjudicator is a role, not an organisation | Stated explicitly; the scenario is anchored, not recruited |
| Scope creep into ERP integration | Out of scope; the log exports assertions, it does not read the ERP |
| Overclaiming legal effect | Non-claims section; comparison table retained in the dissertation |

---

## 10. One-paragraph summary

Delivery disputes are adjudicated by parties that were not present and have no right of access to the records they must judge; four adjudicated cases show forged acceptances, altered and backdated shipment documents, and verifiers unable to compare a presented document with the issuer's original. Existing controls — ERP three-way matching, qualified signatures, registered delivery, EDI acknowledgments, e-CMR — each solve part of the problem but are verifiable only by parties with access. This project builds a thin profile and verification path on a witnessed Merkle log that exports those controls' results as a complete, non-equivocable, offline-verifiable record, and demonstrates against the four documented patterns that it detects what L1 and L2 do not. The contribution is the handover profile, the RFC 9162 receipt path — for which no standard offline verifier currently exists — and a measured demonstration at the adjudication boundary. The work does not prevent fraud, carries no legal recognition, and does not displace the controls it exports; it supplies the one property they cannot.
