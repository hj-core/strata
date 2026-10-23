# DISCUSSION-SUMMARY-005 — Automotive to Employment, the RFC 9943 Pass, and the Roast

| Field | Value |
| --- | --- |
| Document ID | DISCUSSION-SUMMARY-005 |
| Title | Discussion summary — dropping automotive, the employment pivot, the RFC 9943 analysis, and the strongest-alternative roast |
| Date | 2026-10-23 |
| Status | Working record — current state. **D8 (proceed or stop) is open.** |
| Related documents | DEC-002; CNP-002; ANL-008; ANL-009; PRC-001; BUILD-HANDOVER-004; BUILD-HANDOVER-005 |
| Supersedes | DISCUSSION-SUMMARY-004 **for the current state**. -004 remains the record of the automotive pivot; it is not edited. |
| Purpose | Record what this session decided, what it discovered about RFC 9943, why the employment case was narrowed, and the method that should govern the next choice |

**How to read this.** -004 recorded the decision to leave the parcel case and proposed automotive test-evidence. This file records what happened after: the automotive direction was tested and dropped, an employment direction was adopted, the standard was read closely for the first time, and the employment direction was then roasted and narrowed to a single proposition. The session ends with an open decision, not a build.

---

## 1. What this session did

| Step | Outcome |
| --- | --- |
| Traced the automotive case through a six-step process | `PRC-001`: 0 of 7 scandals fully flagged, 3 partial, 3 not; detection carried outside the mechanism |
| Dropped automotive | `DEC-002` §1–§2 |
| Established the employment case's demand and anchors | `ANL-004` (demand), `ANL-008` (seven anchors, four at V) |
| Read RFC 9943 properly | §7.1 of `CNP-002`; corrections to four previously assumed facts |
| Roasted the employment case against the strongest field | `ANL-009`: two fatal findings, one surviving proposition |
| Derived a reusable domain filter | §6 below |

---

## 2. The automotive decision

The process (`PRC-001`) had six steps — declare, attest, test, register, audit, measure production — and thirteen named outcomes. Traced against seven documented scandals:

| Case | Verdict |
| --- | --- |
| Toyota Industries | Partial — calibration contradiction visible, carried by the authority's COP record |
| Kobe Steel | Partial — best fit; enumeration and cross-customer equivocation |
| Nissan | Partial — removes the stamping channel only |
| Takata | Partial, conditional on the lab controlling event creation |
| Hino | Not flagged |
| Daihatsu | Not flagged |
| VW | Not flagged |

**Nothing fully caught.** The decisive condition no case satisfies: **the verifier must be compelled to compare, not merely able to.** In every partial case, the load was carried by an institution outside the mechanism — production measurement, two customers choosing to compare, a lab in the path.

The automotive direction was therefore dropped, not because the analysis failed but because it succeeded. `PRC-001` is retained as the domain-selection record.

---

## 3. The employment pivot

`ANL-008` found seven documented cases, four at primary or official status:

| Case | What it shows | Status |
| --- | --- | --- |
| *US v. Osuagwu* (N.D. Tex. 2018) | False employment letters used to obtain mortgages | V |
| OCC order, *In re* Amy Lu (2022) | Loan officer created verification-of-employment letters | V |
| *Kenyeres* [2020] AATA 1050 | Sponsor's signature forged; employer never made an offer | V |
| *Agbai v Canada* (FC) | Letters forged; verification routed through a foreign police unit | V |
| *R. v. Khatchatourov* 2014 ONCA 464 | Falsified employment letters in a mortgage-fraud ring | V |
| Tisha Lee (D.D.C. 2026) | Agency insider issued fake offer letters and **confirmed them by phone** when landlords called | P |
| *US v. Smalley* (D. Colo. 2020) | Falsified employment letter; two more during loss mitigation | P |

**The common structure:** the verifier's only control is to contact the employer, and the contact fails when the issuer is complicit, unreachable, non-existent or foreign.

**A correction recorded here.** The initial framing said the distinctive value was that "the employer cannot later deny or silently revise." The anchors do not support that: none shows an employer disowning a genuine issuance. They support an **authenticity** claim. The claim was rewritten in that order.

---

## 4. RFC 9943: what it does and does not do

Four facts were established from the text, and each corrected an assumption in the earlier documents.

1. **History and audit are the purpose, not the specification.** §9 names three guarantees, the second being that "provenance and history can be independently and consistently audited." But the normative content is the message format and the registration contract. Audit is a property requirement on the VDS, not a specified procedure.
2. **"Transparent" does not mean "public."** §4 says "publicly available," then qualifies it with "anyone with access." §8 expects encryption and authorisation, and permits hash-only VDSs. For low-entropy employment payloads, a public log needs a salted commitment.
3. **Witnesses are not in the RFC.** The word does not appear. Non-equivocation is a VDS requirement (§5.1.3); the mechanisms left open are auditor replay (§4) and **multiple independent TSs** (§9.7). Witness cosignatures are an implementation choice from the CT/Sigsum lineage.
4. **There are no tiers.** The repo's earlier phrase "RFC 9943 tiers" was wrong. The useful distinction — a receipt proves inclusion offline; consistency and non-equivocation need an auditor or a second log — is real but is not the RFC's vocabulary.

Four limits are stated by the RFC itself: ordering is not issuance order (§9.1), statements may be false (§9.2), issuers may submit selectively (§9.3), and revocation strategy is out of scope (§9.4.2). A compromised TS may also **roll back** its sequence (§9.4.2).

**Authentication.** The RFC authenticates the **issuer** (signature, identity binding, certificate path, trust anchors) and the **log** (receipt signature). It does **not** authenticate the **holder**: `sub` is issuer-defined and there is no proof of possession. Client authentication to the TS is explicitly out of scope (§6.3). The authorised-role problem can be addressed within the RFC by scoping `iss` to a role (`issuer:org:hr-issuance`) and enforcing that namespace in the registration policy.

**The one-sentence framing.** The RFC makes a narrow set of propositions *checkable*; it does not make history *complete* or *true*.

---

## 5. The strongest-alternative roast

`ANL-009` compared the proposal against the strongest deployed and legislated alternatives. Two were missing from `CNP-002` §7.3.

- **Velocity Network**: 90+ vendors, ~2 million credentials, 100+ issuers, 185 countries, employment-history credentials, staffing-sector adoption, and it **pays issuers** per credential.
- **ESSPASS**: proposed regulation COM(2026) 980 requires verifiers to confirm (a) authorised issuance, (b) integrity, (c) non-revocation, and gives verified documents mutual recognition.

Across nine properties, the proposal is alone in exactly **two cells**: status at a past decision time offline, and non-equivocation without a chain. The second is the parcel claim that already failed. The first is a real but narrow primitive.

**Findings:** R1 (ESSPASS legislates the property set) and R2 (Velocity owns the deployment) are fatal to novelty and the adoption story as originally framed. R3 records the process failure: the comparison was against the wrong field. R4: four of five primary anchors concern existing employment, which the incumbents already serve.

**Do not resurrect** "no one is doing this," "unserved market," or "the employer cannot deny issuance." None survives the evidence.

---

## 6. The filter that should govern the next choice

Four questions. If any answer is no, stop before writing anything.

1. Is there a **mandatory intermediary** the relying party must use and cannot audit?
2. Does the dispute turn on **that intermediary's record**, rather than on facts obtainable elsewhere?
3. Is the intermediary the party with the **incentive to falsify**?
4. Is the log's contribution **more than a signature plus a timestamp**?

| Domain | Passes? | Fails on |
| --- | --- | --- |
| Certificate Transparency | ✓✓✓✓ | — |
| WhatsApp AKD | ✓✓✓✓ | — |
| Parcel | ✗ | 1 — the insurer can investigate |
| Automotive | ✗ | 1, 4 |
| Employment | ✗ | 1, 4 |
| Notarised messaging | ✗ | 1 — courts compel; QERDS exists |

The filter explains all four failed domains in one sentence: **the log is load-bearing only where the intermediary is unavoidable.**

---

## 7. The messaging idea, recorded because it will recur

Treating each message as a logged statement fails the same two tests: courts have compulsory process, and QERDS/registered email already proves sending and receipt with legal effect. It also reverses a deliberate security property — secure messengers are deniable by construction — and the scale is four orders of magnitude beyond the 4,000 appends/day assumption. The one version worth re-examining is **platform-as-counterparty** (gig, tenancy, employment), where the record holder is the party in the dispute; it still needs a mandate to overcome adoption resistance.

---

## 8. Current state and the open decision

**Module constraints, answered this session:**

1. COMP4299 accepts a **narrow mechanism with a measured boundary**.
2. A **negative headline is not acceptable**; the primary result must be a demonstrated capability.
3. A **plausible scenario is sufficient**; no external stakeholder need be recruited.

**The consequence.** The boundary study is no longer available as a headline, and both earlier domains are ruled out because neither can produce a positive capability claim. Employment is the only direction left, and after `ANL-009` it is one proposition:

> A verifier can establish, offline and from the bundle alone, that a record was issued by an authorised issuance role, for that holder, with that content, and that it had not been superseded or revoked at the decision time.

**D8 is open.** Proceed on that proposition, or stop. The module accepts a narrow mechanism, but the boundary is now most of the project, and that tension is the decision.

---

## 9. What to keep

- **Anchor first, then design.** `ANL-008` changed the claim; the roast changed it again. Documents written before the anchors were wrong.
- **Strongest incumbent, not the weakest.** `ANL-009` found two alternatives that `CNP-002` had omitted; both were one search away.
- **Read the standard.** Four assumptions about RFC 9943 were wrong, and all four were in the repo.
- **State the boundary in the same paragraph as the claim.** Otherwise a reviewer supplies the correction.
- **Label status.** V / P / A, and planning estimates are not measurements.

---

## 10. Open questions

1. **D8** — proceed on one proposition, or stop.
2. **D1** — the primary case is now implicitly the *letter of terms*; confirm it, because status and income are owned by incumbents.
3. **H2** — does a signed status-list snapshot plus a TSA already answer the historical question offline? This is the decisive empirical test.
4. **ESSPASS historical status** — does the regulation or the EUDI Wallet answer status at a past time? If yes, the surviving proposition collapses.
5. **Supervisor approval** — the direction still needs a yes, and the module constraints should be confirmed in writing.
6. **WBS** — `PLN-002` is parcel-shaped and has not been re-cut.

---

## 11. One-paragraph summary

The automotive direction was dropped after a seven-case trace showed nothing fully caught and detection carried by institutions outside the mechanism. The employment direction replaced it, anchored in seven documented cases, four at primary or official status, all sharing one structure: the verifier's only control is to call the employer, and the call fails on insiders, absent employers and foreign issuers. A close reading of RFC 9943 corrected four assumptions — history is the purpose not the specification, transparency is not publicity, witnesses are not in the RFC, and there are no tiers — and established that the protocol authenticates the issuer and the log but not the holder. The strongest-alternative roast then found that ESSPASS legislates the property set and Velocity deploys employment credentials at scale, leaving the proposal alone in two cells: offline status at a past decision time, and non-equivocation without a chain. The second is the parcel claim that already failed. The session ends with a single open question: whether one proposition is enough to build, given a module that accepts a narrow mechanism but not a negative headline. The filter derived here — mandatory intermediary, dispute on its record, incentive to falsify, more than a signature — is the cheapest way to test the next idea before writing about it.
