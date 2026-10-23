# ANL-006 — Anchoring Cases for Adjudicated Handover

| Field | Value |
| --- | --- |
| Document ID | ANL-006 |
| Title | Anchoring cases: documented forgeries of delivery records and what they establish |
| Version | 0.1.0 |
| Status | Draft — evidence annex; anchors the primary scenario |
| Last Updated | 2026-10-23 |
| Related documents | PRP-001; PRP-001a; ANL-003; ANL-004; ANL-005 |
| Purpose | Replace a self-set threat model with documented dispute cases, and derive the scenario from them rather than from assumption |
| Supersedes | The "no public deployment / no documented dispute" framing in DISCUSSION-SUMMARY-002 §6.2 and BUILD-HANDOVER-002 §8, for the dispute class only |

---

## 1. Why this document exists

The supervisor's objection to the previous proposal was that the requirement was **self-set and unanchored**. That objection is answered only by evidence that the threat occurs, not by argument that it could.

This document collects four adjudicated cases in which a delivery, shipment or acceptance record was **forged, altered, backdated, or could not be checked against the issuer's own record**, and one that was adjudicated by an insurer. Each case is a primary legal source.

The purpose is narrow: to establish that the threat model is observed, and to identify which design property each observed failure requires.

---

## 2. The cases

### 2.1 Leicht Transfer & Storage Co. v. Pallet Central Enterprises, Inc.

**Citation:** 2019 WI 61, Wisconsin Supreme Court.
**Status:** Adjudicated. Primary source.

**Facts.** Pallet Central forged delivery tickets and billed Leicht for pallets that were never sold or delivered. The standard practice was that a delivery ticket accompanied the truck driver; on delivery, a **Leicht employee verified the shipment and signed the ticket**; the driver returned a copy to Pallet Central; Pallet Central then invoiced against the signed ticket, and Leicht paid only if the package contained a signed ticket.

Many tickets bore "the apparent signatures of Leicht employees who could not possibly have executed the documents." Pallet Central submitted fraudulent invoice packages worth approximately **$751,000**, of which Leicht paid approximately **$505,000**.

**How it was detected.** Not by verifying any individual document. Leicht noticed that "the number of invoice packages Pallet Central was submitting had dramatically increased," then investigated internally.

**What it anchors.**
- The acceptance signature — the proof-of-delivery artifact — was **forged by the party that supplied the form**.
- The payer held **no independent record of what it had signed**. The only copy was in the fraudster's control.
- Detection depended on a **volume anomaly**, two years after the scheme began.
- The resulting dispute was between the payer and its **insurer** over whether the forged tickets triggered a commercial-crime policy — that is, a third party adjudicating conflicting records.

### 2.2 United States v. Howe

**Citation:** 543 F.3d 128 (3d Cir. 2008).
**Status:** Adjudicated. Primary source.

**Facts.** Howe contracted to supply twenty encryption modules to the US Air Force. He never purchased or delivered them. To obtain payment he submitted to the Defense Finance and Accounting Service an **altered straight bill of lading** falsely indicating delivery to Dover Air Force Base. DFAS paid **$152,850** in reliance on it four days later. He then generated false invoices and **backdated them** to conform to the delivery date he had falsely claimed.

**How it was detected.** The destination base discovered the devices had not arrived, triggering an investigation. Detection came from the **absence at the far end**, not from the document.

**What it anchors.**
- The record was **altered after the fact** and **backdated** to make the false delivery internally consistent.
- Reliance was placed on a document that no independent party could compare against an earlier state.

### 2.3 United States v. Otis Williams

**Citation:** Court of Appeals for the Ninth Circuit, 1996.
**Status:** Adjudicated. Primary source.

**Facts.** Williams, a USPS letter carrier, attempted to deliver food stamp coupons. The recipient was not at home and her friends refused to sign. Williams **forged the recipient's signature on the delivery receipt**, took the coupons, and filed the forged receipt. He was convicted; a postal handwriting expert identified the signature as his.

**How it was detected.** Handwriting analysis, after the loss was reported.

**What it anchors.**
- The **courier** forged the **recipient's acceptance** — exactly the adversary in the parcel design.
- Detection required forensic expertise and a complaint. It was not available to the recipient at the time of the event.

### 2.4 Maersk Sealand v. Far East Trading & Others

**Citation:** [2004] EWHC 2929 (Comm), English High Court, Commercial Division.
**Status:** Adjudicated. Primary source.

**Facts.** Six containers of frozen meat were delivered against **forged bills of lading**. The carrier's agents in the Congo accepted the forged documents. The court recorded the decisive point in its own words: the agents

> "were not able to discover that the bills differed from the originals since they **lacked the ability to compare the bills as tendered with the Claimants' electronic record of the original bills**."

**What it anchors.**
- The failure was not a bad signature check. It was the **absence of a channel to compare a presented document against the issuer's authoritative record**.
- The verifier was operationally distant, offline from the issuer, and had no independent copy.

### 2.5 Supporting: United States v. Naser (N.D. Ill., 2026)

**Status:** Indictment — **allegation, not a finding**. Included only as corroboration.

A freight company allegedly submitted false invoice information using fabricated container numbers **and container numbers from earlier deliveries**, to make it appear that additional deliveries had occurred. Alleged loss exceeds **$3 million**. This is a **replay** pattern: reusing identifiers from genuine past events to claim new ones.

### 2.6 Supporting: Little Debbie distributor charges (2026)

**Status:** Criminal charges — **allegation, not a finding**.

A distributor allegedly created fraudulent delivery invoices and **forged employee signatures to verify receipt** of merchandise that never arrived, then passed the invoices into the manufacturer's central billing system.

---

## 3. The common structure

The four primary cases are different in jurisdiction, era and cargo, and they converge on one structure:

> **The authoritative record is held by the party with the incentive to falsify it, and the party relying on it has no independent copy.**

| Case | Who held the record | Who was harmed | What was missing |
| --- | --- | --- | --- |
| Leicht | The supplier (Pallet Central) | The payer (Leicht) | Any record of what Leicht actually accepted |
| Howe | The supplier (Elite) | The payer (DFAS) | Any earlier state to compare against |
| Williams | The courier | The recipient and the postal service | Any acceptance the courier could not produce |
| Maersk | The party presenting the document | The carrier | A channel to the issuer's original record |

In each case the fraud succeeded because the **verifier had nothing independent to check against** — not because a signature was cryptographically weak. Wet signatures were forged; documents were altered and backdated; agents could not reach the original.

That is a materially different problem statement from the one in PRP-001 v0.2.0, and it is the one the evidence supports.

---

## 4. The derived scenario

**Payment-linked delivery, adjudicated after the fact.**

> A supplier ships goods to a payer through a carrier. The payer's acceptance is what releases payment. The supplier controls the paperwork. The payer holds no independent record of what it accepted. When a dispute arises, an insurer, arbitrator or court must decide whose record to believe, and it has no audit rights over either party.

This is the structure of trade credit, procurement, freight claims and commercial insurance. It is common enough to be ordinary and specific enough to be testable.

### 4.1 Why the log is load-bearing here

| Failure observed | Property required | Level |
| --- | --- | --- |
| Supplier forges the payer's acceptance | Acceptance signed with a **key the payer holds** | L1 + key binding |
| Courier forges the recipient's acceptance | Same; the receipt simply cannot be produced | L1 + key binding |
| Payer cannot prove how many acceptances it gave | **Issuer-signed sequence** and inclusion | **L3 profile** |
| Documents backdated to match a false delivery | **Cosigned checkpoint** fixing the tree at time T | **L3 witness** |
| Agent cannot compare a document to the original | **Offline verification against the log**, no call to the issuer | **L3 log** |
| Old identifiers replayed for new claims | `nonce`, `sequence`, `prev` | Profile |

### 4.2 The honest boundary

Digital signatures alone defeat per-document forgery. That is L1, and it is most of what Leicht and Williams needed.

The log's distinctive contribution is the three things L1 cannot supply:

1. **Completeness** — the payer can prove it issued *n* acceptances, not *n + 20*. Leicht's fraud was a count anomaly; a signed sequence makes the count a proof rather than a suspicion.
2. **Anti-backdating** — Howe's fraud depended on making an altered document look contemporaneous. A cosigned checkpoint fixes what existed at a time.
3. **Third-party verifiability** — the insurer or court can check the payer's record without trusting the payer's database. In Leicht, the payer's own record would have been self-serving; the log's would not.

Say it that way. Do not claim the log prevents forgery; it prevents **denial, backdating, and selective presentation**, and it makes the count provable.

---

## 5. What this changes in the proposal

| Item | Before | After |
| --- | --- | --- |
| Threat model | "A record keeper may equivocate or withhold" (asserted) | Observed: forged acceptance, altered and backdated shipment record, unverifiable presented document |
| Primary case | Document handover (transcripts) | **Payment-linked delivery under third-party adjudication** |
| Verifier | Recipient, then government | **Insurer, arbitrator or court** |
| Mandate | None identified | **Condition of cover or of trade terms** — the insurer or payer can impose it |
| Advantage | Argued | **Demonstrated by replaying four documented patterns** |
| Boundary | "Rarely useful" | "Useful where the record holder is the party with the incentive to falsify it and the verifier has no independent copy" |

---

## 6. Evidence register

| # | Source | Claim used | Status |
| --- | --- | --- | --- |
| 1 | *Leicht Transfer & Storage Co. v. Pallet Central Enterprises, Inc.*, 2019 WI 61 | Forged delivery tickets; ~$751k invoiced, ~$505k paid; detected by invoice-volume anomaly; insurer coverage dispute | **V** |
| 2 | *United States v. Howe*, 543 F.3d 128 (3d Cir. 2008) | Altered straight bill of lading; $152,850 paid; invoices backdated to match false delivery | **V** |
| 3 | *United States v. Otis Williams*, 9th Cir. 1996 | Letter carrier forged recipient's delivery receipt; handwriting evidence | **V** |
| 4 | *Maersk Sealand v. Far East Trading & Others* [2004] EWHC 2929 (Comm) | Delivery against forged bills of lading; agents unable to compare against the carrier's electronic record | **V** |
| 5 | *United States v. Naser*, N.D. Ill. (2026) | Alleged replay of container numbers from earlier deliveries; >$3M | **A** (indictment) |
| 6 | FreightWaves, 2026-08-08 | Alleged forged delivery invoices and employee signatures; Little Debbie distributor | **A** (charges) |

**Status key:** V = adjudicated primary source; A = allegation or charge, not a finding.

---

## 7. Caveats

- Cases 5 and 6 are allegations and must not be described as established facts. They corroborate a pattern; they do not prove it.
- The four primary cases are US and English commercial disputes. They establish that the *pattern* occurs, not its frequency. No prevalence claim is made.
- A log records assertions. If the payer's staff genuinely sign for goods that did not arrive, the log faithfully records a false acceptance; the oracle problem is untouched. In Leicht the signatures were forged, not given, which is why the design helps there.
- The cases do not establish demand, willingness to pay, or adoption. They establish that the failure mode is real and adjudicated.
- The insurance-coverage question in *Leicht* turned on policy wording, not on record integrity. It is cited for the underlying facts and the adjudication structure, not for its holding.

---

## 8. One-paragraph summary

Four adjudicated cases — *Leicht* (Wisconsin, 2019), *Howe* (3d Cir., 2008), *Williams* (9th Cir., 1996) and *Maersk Sealand* (2004) — show the same failure: the authoritative delivery record is held by the party with the incentive to falsify it, and the party relying on it has no independent copy. Signatures were forged, shipment records altered and backdated, and presented documents could not be compared against the issuer's original. That structure — payment-linked delivery under third-party adjudication — is the scenario this project should test. Digital signatures defeat per-document forgery; the witnessed log adds provable completeness, anti-backdating, and third-party verifiability of the payer's own record. The threat model is now observed rather than asserted, which is what the previous proposal lacked.
