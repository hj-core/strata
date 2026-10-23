# ANL-007 — Strongest-Alternative Test for Payment-Linked Delivery

| Field | Value |
| --- | --- |
| Document ID | ANL-007 |
| Title | Strongest-alternative test: what already solves payment-linked delivery evidence |
| Version | 0.1.0 |
| Status | Draft — adversarial comparison; written to find where the proposal loses |
| Last Updated | 2026-10-23 |
| Related documents | PRP-001; PRP-001a; ANL-003; ANL-006 |
| Purpose | Test the proposal against the best existing alternatives, not against paper. State plainly where each alternative wins. |
| Supersedes | The comparison in ANL-003 §2 and PRP-001 Appendix E, which omits ERP matching, EDI acknowledgments, eIDAS trust services and e-CMR |

---

## 1. The rule for this document

The previous proposal died because the comparison baseline was weak and the incumbent was stronger than the proposal admitted. This document is written **to find that outcome again**, if it is there.

Two rules:

1. **The strongest incumbent, not the weakest.** Where an established control already prevents the fraud, it is the baseline.
2. **No weighted scores.** A cell states what an option does. Where it wins, say so.

The scenario under test is the one anchored in ANL-006: **payment-linked B2B delivery, adjudicated after the fact**, where a supplier, a carrier and a payer exchange goods and an insurer or court later decides whose record to believe.

---

## 2. The alternatives

### 2.1 ERP three-way / four-way matching

The dominant control for payment-linked delivery, and the one that most directly threatens the proposal.

| Control | What it compares | What it prevents |
| --- | --- | --- |
| Two-way | Purchase order + invoice | Paying a price you did not agree |
| **Three-way** | Purchase order + **goods receipt** + invoice | **Paying for goods never received** |
| Four-way | Three-way + inspection/quality acceptance | Paying for goods that failed acceptance |

Oracle's own definition: *"Matching is the process of associating an invoice with a purchase order, receipt, or consumption advice. Matching ensures that you pay only for the goods and services that you ordered, received, or consumed."*

The control is bundled in every major ERP, is tested by auditors, and its known weakness is stated in the literature: **segregation of duties.** "Without it a single user can manufacture all three documents and the match proves nothing."

**Verdict: this wins, decisively, on prevention.** A buyer running a three-way match with proper segregation does not pay Leicht's forged invoices. The proposal must not claim to prevent that fraud.

### 2.2 EDI: despatch advice, receiving advice, signed acknowledgments

The B2B messaging layer already standardises exactly these events:

| Message | Function |
| --- | --- |
| `DESADV` | Despatch advice — what was sent |
| `RECADV` | **Receiving advice — what was received** |
| `IFTSTA` | Transport status |
| CONTRL / 997 | Functional acknowledgment of a message |
| EDIINT AS2 signed MDN | **Signed receipt at the transport layer** |

GS1 also publishes a **Digital Signatures** guideline (ratified 2023) supporting independent verification using JWS, XML-DSig and ISO/IEC 20248, explicitly aimed at "independent verification of digitally signed data."

**Verdict: this wins on standardisation and adoption** among large trading partners. A signed `RECADV` is a receiving acknowledgment with an audit trail, and it is already normative. The proposal's receipt is conceptually the same object.

### 2.3 eIDAS qualified signatures and registered delivery

Two instruments matter, and both are legally stronger than anything this project can claim.

| Instrument | Legal effect |
| --- | --- |
| **Qualified electronic signature (QES)** | *"shall have the equivalent legal effect of a handwritten signature"* (eIDAS Art. 25(2)) |
| **Qualified electronic registered delivery service (QERDS)** | Integrity, identified sender and addressee, and **accuracy of the date and time of sending and receipt** |

A QES on a delivery note is, in law, the signed delivery note. A QERDS is, in effect, a legally recognised proof of sending and receipt with trusted time.

**Verdict: this wins on legal recognition, categorically.** The proposal produces evidence, not a qualified trust service, and must never imply otherwise.

### 2.4 e-CMR

The electronic consignment note has an agreed legal framework: electronic signature equivalence to the paper signature, integrity of particulars, and recorded reservations at delivery.

**Verdict: this wins for the legal transport document.** A log can complement it; it cannot replace it.

### 2.5 Digital PoD applications

Commercial apps (Onfleet, Detrack, Track-POD, Scandit and similar) capture photo, signature, timestamp and geolocation at the doorstep, integrated with routing and billing.

**Verdict: this wins on cost, adoption and usability.** It is also what most carriers actually use. It is weaker on non-repudiation because the record sits with the operator.

### 2.6 Third-party control assurance and inspection

| Mechanism | What it gives |
| --- | --- |
| SOC 2 / ISAE 3402 | Annual, sample-based attestation that controls are designed and operating |
| Surveyor / inspection company | Independent, expert, post-hoc assessment of a specific loss |
| Audit right | Contractual permission to inspect the counterparty's systems |

**Verdict: these win on breadth and credibility.** They are the established way an outsider gains confidence in another party's records.

### 2.7 Blockchain trade platforms

CargoX, WaveBL and Bolero provide electronic bills of lading with platform-level integrity and, in some cases, legal recognition through trade-document frameworks.

**Verdict: these win for transferable documents of title.** They are heavy, and they do not address general payment-linked delivery.

---

## 3. The comparison

No weighted score. A cell says what the option provides.

| Option | Forged acceptance | Payment without receipt | **Provable completeness** | Anti-backdating | **Third-party offline verification** | **Non-equivocation** | Legal recognition | Cost | Adoption |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Paper signed note | ✗ | ✗ | ✗ | ✗ | manual | ✗ | ✓ | Low | Universal |
| Digital PoD app | ✓ | partial | ✗ | ✓ | ✗ (operator holds it) | ✗ | Contractual | Low | High |
| **ERP three-way match** | ✓ | **✓** | ✓ **internal** | ✓ | ✗ **internal only** | ✗ | Internal control | Bundled | **High** |
| EDI `RECADV` + signed MDN | ✓ | partial | ✗ | partial | ✗ (bilateral) | ✗ | Contractual | Medium | High (large firms) |
| **eIDAS QES** | ✓ | ✗ | ✗ | ✓ | ✓ | ✗ | **✓ equivalent to handwritten** | Per signature | Medium |
| **eIDAS QERDS** | ✓ | partial | ✗ | ✓ | ✓ | ✗ | **✓ registered delivery** | Per message | Medium |
| e-CMR | ✓ | n/a | ✗ | ✓ | ✓ | ✗ | **✓** | Platform | Medium |
| SOC 2 / ISAE 3402 | n/a | n/a | sample | n/a | ✓ (report) | ✗ | Assurance | High | Medium |
| Surveyor | n/a | n/a | sample | n/a | ✓ | ✗ | Expert evidence | High | High |
| Blockchain eBL | ✓ | n/a | partial | ✓ | platform | partial | Varies | High | Low |
| **Proposed L3** | ✓ | ✓ (policy) | **✓** | **✓** | **✓** | **✓** | **Evidence only** | Fixed + governance | **None** |

**The honest reading of the bottom row:** the only column where the proposal is alone is the combination of *provable completeness*, *third-party offline verification* and *non-equivocation*. Every individual cell in that combination is available elsewhere. The combination is not.

---

## 4. Where the incumbents win, stated plainly

1. **Prevention.** ERP three-way match stops payment for goods never received. The proposal does not, and should not claim to.
2. **Legal recognition.** QES, QERDS and e-CMR have statutory effect. The proposal has none.
3. **Cost.** Three-way match is bundled; digital PoD is cheap; a QES costs cents. The proposal needs a log, two witnesses, a TSA, a CA and governance.
4. **Adoption.** Every option above is deployed. The proposal has no deployment.
5. **Bilateral receipt.** EDI signed MDNs and QERDS already do point-to-point acknowledgment with time.

If the supervisor asks "why not just use three-way matching and a qualified signature?", the answer cannot be "because a log is better." It has to be narrower, and it is.

---

## 5. The residual gap

The incumbents share one limitation: **they are only verifiable by the parties to them, or by an auditor with rights of access.**

- Three-way match is internal to the buyer. The insurer cannot run it.
- EDI acknowledgments are bilateral. The insurer is not a party.
- A QES proves a signature. It does not prove that a *set* of acceptances is complete.
- A QERDS proves sending and receipt of a message. It does not prove that no other message was withheld.
- SOC 2 attests controls annually, by sample. It does not evidence a specific transaction.
- A surveyor assesses a specific loss, expensively, after the fact.

The gap is precise:

> **An adjudicator with no audit rights over either party, needing transaction-level evidence that a set of acceptances is complete and that neither party has shown a different history to the other.**

No incumbent provides that. That is the proposal's claim, and it is narrow.

---

## 6. The reframe that makes this defensible

The proposal should stop presenting itself as an alternative to three-way matching. It should present itself as an **attestation layer over controls that already exist**.

```
ERP three-way match  ──▶  "this invoice matched a goods receipt"
        │
        ▼
   signed statement  ──▶  witnessed log  ──▶  inclusion receipt
        │
        ▼
adjudicator verifies offline, without audit rights
```

Under this framing:

- the **control** stays where it is — three-way match, EDI, QES;
- the log does not re-implement it;
- the log **exports the result** as a record that a third party can verify and neither party can rewrite, backdate or selectively present.

That is additive, not competitive, and it is honest about what is being replaced (nothing) and what is being added (third-party verifiability of the control's output).

---

## 7. What would falsify the claim

The proposal's claim fails if any of these holds:

1. An insurer, arbitrator or court will accept a party's ERP audit trail as sufficient evidence without independent verification — making the log unnecessary.
2. A QERDS plus a signed `RECADV` already satisfies the adjudicator's need in practice.
3. The cost of the log exceeds the cost of surveying or auditing the claims it would cover.
4. The adjudicator requires a qualified trust service, making a self-hosted log inadmissible in practice.
5. The completeness property is never contested — i.e. no adjudication turns on "how many acceptances existed."

Each of these is testable, and the evaluation should test them rather than assume the answer.

---

## 8. What the comparison changes in the proposal

| Item | Before | After |
| --- | --- | --- |
| Positioning | A PoD system | An **attestation layer** over existing delivery controls |
| Claim | Better than existing approaches | **Additive**: exports control output as third-party-verifiable evidence |
| Baseline | Paper, portal | **ERP three-way match + QES + EDI** |
| Legal claim | Implied evidence | Explicitly **not** a qualified trust service, **not** equivalent to QES or e-CMR |
| Cost claim | <$0.01/handover | Marginal infrastructure only; **total cost is not competitive** with bundled controls |
| Contribution | The system | The **profile, the attestation layer, and the boundary** |

---

## 9. Evidence register

| # | Source | Claim used | Status |
| --- | --- | --- | --- |
| 1 | Oracle Fusion Cloud Financials, *Matching Invoice Lines* | Matching associates invoice with PO, receipt or consumption advice; ensures payment only for what was ordered and received | **V** (vendor documentation) |
| 2 | Oracle, *Match Approval Level Options* | Two-way, three-way, four-way definitions | **V** |
| 3 | GEP, *Mastering Three-Way Matching* | "An invoice with no corroborating PO and receipt simply cannot pass"; three-way match is a structural barrier to invoice fraud | **P** (vendor guide) |
| 4 | ERP Research glossary | Four-way match adds inspection/quality acceptance; segregation of duties is what gives the control its strength | **P** |
| 5 | SAPinsider, *Three-Way Invoice Control Assessment* | Three-way match can be bypassed if poorly implemented; tolerance keys; SoD conflicts | **P** |
| 6 | GS1 EANCOM, transport and delivery messages | `DESADV`, `RECADV`, `IFTSTA` exist as standard messages | **V** |
| 7 | GS1, *Digital Signatures Technical Implementation Guideline* | Ratified 2023; independent verification via JWS, XML-DSig, ISO/IEC 20248 | **V** |
| 8 | GS1 EDIINT AS1/AS2 guidelines | Signed MDNs provide a signed acknowledgment of receipt at transport level | **V** |
| 9 | eIDAS Regulation 910/2014, Art. 25(2) | QES has equivalent legal effect to a handwritten signature | **V** |
| 10 | eIDAS, QERDS definition (Art. 3(37), Art. 44) | Integrity, identified sender and addressee, accuracy of date and time of sending and receipt | **V** |
| 11 | IRU, *Standard Contract for e-CMR*, Arts. 1–4 | Electronic signature equivalent to paper; reservations recorded electronically | **V** |
| 12 | ANL-006 cases | The fraud patterns the comparison is tested against | **V/A** as labelled |

---

## 10. Caveats

- SOC 2 and ISAE 3402 are characterised at a general level and were not independently verified in this review. Their inclusion is to represent the third-party-assurance family, not to make a specific claim about either standard.
- Cost figures are not quoted because no reliable public price exists for the log's fixed components (two independent witnesses, TSA, CA, governance). The comparison therefore says "fixed + governance" rather than inventing a number.
- Adoption estimates are qualitative. No deployment of the proposed combination was found.
- The comparison is written from the buyer's and adjudicator's perspective. A carrier's perspective would weight different cells.

---

## 11. One-paragraph summary

The strongest alternatives to payment-linked delivery evidence are not paper. They are ERP three-way matching, which prevents payment for goods never received; EDI receiving advice with signed acknowledgments; qualified electronic signatures, which are legally equivalent to handwritten signatures; qualified registered delivery, which proves sending and receipt with trusted time; and e-CMR for the legal transport document. Each of these beats the proposal on prevention, legal recognition, cost or adoption. What none provides is a transaction-level record that an adjudicator with no audit rights can verify offline, that proves a set of acceptances is complete, and that prevents either party showing a different history to the other. That is the only defensible claim, and it is narrow. The proposal should therefore present itself as an attestation layer over existing controls rather than a replacement for them — and should state plainly, in the dissertation, the five conditions under which its claim fails.
