# DISCUSSION-SUMMARY-003 — PoD Scenario Search and Its Outcome

| Field | Value |
| --- | --- |
| Document ID | DISCUSSION-SUMMARY-003 |
| Title | Discussion summary — the scenario search for the witnessed-log proposal, and why it concluded without one |
| Date | 2026-10-23 |
| Status | Working record — **the scenario search concluded without a defensible scenario**; proposal paused pending G3/G4 |
| Related documents | PRP-001 v0.2.0; PRP-001a; ANL-002 to ANL-007; PLN-002; TUT-001; REVIEW-SESSION-POD |
| Supersedes | `DISCUSSION-SUMMARY-002` as the **current** record. -002 remains the record of the v0.2.0 revision and is not edited. |
| Purpose | Record the scenario search, the eliminations and their evidence, the structural finding, and the remaining options |

**How to read this.** -002 records the discussion that produced PRP-001 v0.2.0. This file records what happened after: a systematic attempt to find a scenario in which the proposal's mechanism is genuinely useful. That attempt did not succeed. The negative result is the content of this document.

---

## 1. Starting point

The session began from PRP-001 v0.2.0: a thin document-handover / PoD slice on a witnessed Merkle transparency log (level 3), with a university/employer primary case and the SHP-42 parcel as the demo.

The supervisor's standing objection, carried over from the previous proposal, was that the requirement was **self-set and unanchored**, no stakeholder imposed it, and the proposed mechanism had no obvious advantage over existing practice. A further explicit bar was added during this session: a **boundary result is acceptable, but it may not be "rarely useful"** — a reasonable scenario must be established in which the mechanism is useful.

The work below was an attempt to meet that bar. It did not.

---

## 2. Method

Two disciplines were applied to every candidate:

1. **Anchoring.** A scenario is only admissible if the threat is *observed*, not asserted. Where possible, anchor in adjudicated primary sources.
2. **Strongest-alternative test.** Compare against the best existing practice, not against paper. State where the incumbent wins.

A third discipline emerged during the search and proved decisive:

3. **The absent-third-party test.** Ask what a verifier who was not present must be convinced of. Distinguish authenticity (signature), time (TSA), denial (signature + time), a conflicting version (log + knowledge of it), and completeness (log + enumeration).

---

## 3. Candidates examined and eliminations

| Domain | Outcome | Why it failed |
| --- | --- | --- |
| **Transcripts / diplomas** | **Failed** | My eQuals covers 100% of public universities in Australia and New Zealand (10M+ records); OpenCerts has run since 2018 anchored on Ethereum with selective disclosure in v3. "No public deployment found" was **wrong** and has been corrected. |
| **Employment letters** | **Failed** | Holder forgery is defeated by a signature; the employer has no incentive to equivocate; the defunct-employer case is served by the holder's copy. |
| **Cross-border mobility** | **Failed** | Each regulator verifies signatures independently; no shared head is needed. |
| **Mortgage / supervisory reporting (Macau)** | **Failed** | The payer (AMCM) already holds the transaction set; the property registry already holds the security interest. |
| **University payment log (Macau)** | **Failed** | UM's statute is a *special regime* relative to the financial regime of autonomous services and funds; its reporting is **annual**, not monthly. The monthly reconciliation anchor does not apply. |
| **Autonomous fund payment log** | **Unproven** | The frame survives and the monthly aggregate anchor exists (RA 6/2006), but the decisive fact — whether payments are system-emitted or clerk-decided — cannot be established without insider access. |
| **Hospital** | **Failed** | The public hospital is an integrated service of the Health Bureau, so DSF issues its payments; private hospitals are paid on volume against reported activity, which is an **oracle** problem, not a completeness problem. |
| **Parcel / PoD** | **Failed after anchoring** | Anchored in four adjudicated cases, but ERP three-way matching, eIDAS QES/QERDS and EDI receiving advice win on prevention, legal recognition, cost and adoption (ANL-007). |
| **Administrative audit trail (A)** | **Not recommended** | Database audit trails and SIEM retention cover the ground; the registry would be adopting a system that weakens its own position in disputes. |
| **Experimental-run provenance (B)** | **Rejected** | 21 CFR Part 11 and commercial CDS already mandate and implement instrument-level audit trails; Registered Reports is a **free** procedural fix for selective reporting, achieving **1.2% adoption** in the subdiscipline that embraced it most. |

Nine domains. Every one failed for the same reason.

---

## 4. The PoD anchor, and why it still failed

Four adjudicated cases were found and used to replace the self-set threat model (ANL-006):

| Case | What happened | Property anchored |
| --- | --- | --- |
| *Leicht Transfer & Storage v. Pallet Central*, 2019 WI 61 | Supplier forged delivery tickets; ~$751k invoiced, ~$505k paid; detected by an invoice-**volume** anomaly | Completeness of the payer's acceptance set |
| *US v. Howe*, 543 F.3d 128 (3d Cir. 2008) | Altered bill of lading for undelivered goods; invoices **backdated** | Anti-backdating via cosigned checkpoint |
| *US v. Otis Williams*, 9th Cir. 1996 | Letter carrier forged the recipient's delivery receipt | Recipient-held key; the receipt cannot be produced by the courier |
| *Maersk Sealand* [2004] EWHC 2929 (Comm) | Delivery against forged bills of lading; agents "lacked the ability to compare the bills as tendered with the Claimants' electronic record" | Offline verification against the issuer's record |

The common structure: **the authoritative record is held by the party with the incentive to falsify it, and the relying party has no independent copy.** That is a real, documented threat, and it is what the proposal had been missing.

It was still not enough, because the strongest-alternative test (ANL-007) showed the incumbents are stronger:

| Axis | Winner |
| --- | --- |
| Prevention | **ERP three-way / four-way matching** |
| Legal recognition | **eIDAS QES** (Art. 25(2)) and **QERDS**; **e-CMR** |
| Bilateral receipt | **EDI `RECADV` + signed MDN** |
| Cost | **Bundled ERP; cents per QES** |
| Adoption | **All of the above** |

The only defensible residual was: *an adjudicator with no audit rights, needing transaction-level evidence that a set of acceptances is complete and that neither party showed a different history to the other.* That is real, but narrow — and no verifier with standing was found to demand it.

The proposed reframe was an **attestation layer over existing controls** rather than a replacement for them. It was not adopted because the user judged, on the evidence, that the parcel case would not persuade the supervisor.

---

## 5. The two internal-stakeholder candidates

Both were proposed to escape the stakeholder problem by moving inside the university.

**A — tamper-evident administrative audit trail** (e.g. grade amendments; registry issues, student holds, appeals committee verifies). Structurally sound but not recommended: database audit logs and SIEM retention cover it, and the registry would be adopting a system that makes its own amendments more contestable.

**B — experimental-run provenance.** The strongest technical fit of anything examined: completeness of the run set is the disputed property, and the record can be machine-emitted. It was roasted against the incumbents and **rejected**:

- 21 CFR §11.10(e) already requires secure, computer-generated, time-stamped audit trails that do not obscure prior records;
- commercial CDS already keep an **Injection Audit Trail** per injection, an **Instrument Audit Trail** per instrument, hash verification of raw files, and immutable version numbers;
- ELNs already provide tamper-proof audit trails and Part 11 e-signatures;
- Registered Reports already address selective reporting **procedurally and for free**, with strong evidence of effect (96% significant results in traditional publications vs 44% in Registered Reports) — and still reach only **1.2%** of articles in the subdiscipline that adopted them most, because the incentive structure does not reward it;
- the dominant drivers of irreproducibility are publication bias and analysis flexibility, which a run log does not address.

If a free solution achieves 1.2%, an expensive technical one achieves zero.

---

## 6. The structural finding

> **The property the log uniquely provides is either already supplied by an incumbent, or no party with authority is asking for it.**

Three supporting observations, each established during the search:

1. **Completeness is not a cryptographic property.** No mechanism can prove that an event which never entered the system did not happen. The log makes a completeness claim *checkable*; the institution (mandate, machine emission, independent count) makes it true.
2. **A sequence number does not guarantee completeness.** It guarantees ordering and internal contiguity. A researcher who logs three of ten runs produces a perfect sequence 1, 2, 3.
3. **A mandate and witness independence pull in opposite directions.** A mandate requires a sovereign or a contract; independence requires the witnesses to be beyond that sovereign's reach. The band where both hold is narrow — Certificate Transparency, enforced consortia, browser-grade market power — and is not reachable by a one-term project.

The public evidence agrees: RFC 9943 is four months old, the ecosystem is supply-heavy and demand-light, and the one production user is scoped to Microsoft's own services.

---

## 7. What survives

| Asset | Status |
| --- | --- |
| **RFC 9162 receipt encoder/verifier** | The one concrete technical gap: `scitt-verifier` supports CCF receipts only. Real, testable, needs no stakeholder, already budgeted at 25 h. |
| **The handover primitive** (giver + taker + bounded window; PRP-001a) | Non-trivial profile semantics; answers the "profile reduces to a schema" kill criterion. Reusable as a chapter or component. |
| **The anchoring method** | Anchor in adjudicated cases; run the strongest-alternative test; label every claim. Transferable to any topic. |
| **The negative analysis** | Nine eliminations with evidence. A legitimate analysis chapter or short paper on when witnessed logs are not worth their cost. |
| **The evidence base** | ANL-004 to ANL-007: demand data, Macau finance law, four adjudicated cases, and the incumbent matrix. |

---

## 8. Current state and options

**State.** No build has started. No branch or commit activity. PRP-001 v0.2.0 is unchanged and its review remains formally pending, though the neutral outcome of that review has arguably been settled by this search.

| Option | Description | Condition |
| --- | --- | --- |
| **T1 — technical contribution** | RFC 9162 receipt verifier + conformance tests + the boundary analysis | Module must accept an artifact-plus-evaluation project |
| **T2 — boundary study** | The same build, but the headline finding is the applicability boundary | Supervisor must accept a negative headline result |
| **C — change topic** | Problem-first topic with an internal stakeholder | Module requires a named stakeholder and an obvious advantage |

**The decisive unknown remains G3/G4.** If a named stakeholder is required, T1 and T2 fail the rubric and C is the only path. That question was put to the supervisor and is not yet answered.

---

## 9. Open questions

1. **G3/G4** — does the module require a named stakeholder, or is a working artifact plus evaluation sufficient?
2. **G1** — is a witnessed Merkle log acceptable, or is a consensus ledger expected?
3. **Negative result** — is a boundary result acceptable as the headline finding?
4. **PRP-001 disposition** — mark v0.2.0 superseded, or retain it as the record of the revision?
5. **REVIEW-SESSION-POD** — re-scope or close; a review of a proposal whose scenario has been withdrawn is of limited value.
6. **BUILD-HANDOVER-002** — now stale; supersede with a handover reflecting this document.

---

## 10. Documents produced during this search

| Document | Content |
| --- | --- |
| `ANL-004__PoD_Domain_Demand_Evidence.md` | Demand evidence; incumbent map; the OpenCerts correction |
| `ANL-005__Macau_Payment_Log_Feasibility_Check.md` | Macau procurement law, DSF's role, payment distribution, the monthly anchor, the classification check |
| `ANL-006__Adjudication_Anchoring_Cases.md` | Four adjudicated delivery-forgery cases and the derived scenario |
| `ANL-007__PoD_Strongest_Alternative_Test.md` | Incumbent matrix: ERP matching, eIDAS, EDI, e-CMR; the falsification conditions |
| `PRP-001a__SHP-42_Four_Statement_Walkthrough.md` | Four-statement chain and the handover primitive; proposed replacement for PRP-001 §6.2a |

---

## 11. One-paragraph summary

The proposal was tested against nine candidate domains under two disciplines: anchor the threat in observed cases, and compare against the strongest existing practice. Every domain failed for the same reason — the property a witnessed log uniquely provides is either already supplied by an incumbent (ERP three-way matching, eIDAS qualified signatures and registered delivery, EDI receiving advice, e-CMR, Part 11 audit trails, commercial CDS, Registered Reports) or no party with standing is asking for it. The parcel case was successfully anchored in four adjudicated cases of forged delivery records, which replaced the self-set threat model, but it still lost the strongest-alternative test on prevention, legal recognition, cost and adoption. The two internal-stakeholder alternatives — an administrative audit trail and experimental-run provenance — were assessed and rejected, the second decisively, because Part 11 and CDS already implement instrument-level audit trails and because a free procedural fix for selective reporting achieves only 1.2% adoption. Three technical facts were established along the way: completeness is not a cryptographic property, a sequence number does not guarantee completeness, and a mandate and witness independence pull in opposite directions. What survives is one concrete technical gap — there is no standard offline verifier for RFC 9162 receipts — together with a well-specified handover primitive, a reusable method, and a negative analysis worth writing up. The decisive open question is no longer technical: it is whether the module requires a named stakeholder, which determines whether the receipt-verifier project is viable or whether the topic must change.
