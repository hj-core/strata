# ANL-009 — Strongest-Alternative Test and Roast: Employment Records

| Field | Value |
| --- | --- |
| Document ID | ANL-009 |
| Title | Strongest-alternative test for verifiable employment records — written to find where the proposal loses |
| Version | 0.1.0 |
| Status | Draft — adversarial. Supersedes CNP-002 §7.3 as the incumbent comparison. |
| Last Updated | 2026-10-23 |
| Related documents | CNP-002; DEC-002; ANL-004; ANL-008; ANL-007 (parcel method) |
| Purpose | Test the employment proposal against the strongest deployed and legislated alternatives, not against paper, and state plainly where it wins, ties and loses |
| Supersedes | CNP-002 §7.3. That table omits the two alternatives that matter most. |

---

## 1. The rule

`ANL-007` set the rule for the parcel case and it applies unchanged:

1. **The strongest incumbent, not the weakest.** Where an established system already provides the property, it is the baseline.
2. **No weighted scores.** A cell states what an option does. Where it wins, say so.

`CNP-002` §7.3 compared against The Work Number, payroll APIs, signed PDFs and the generic VC stack. That was not the strongest field. Two alternatives were missing, and either one alone changes the verdict.

---

## 2. The alternatives

### 2.1 The Work Number (Equifax)

Centralised, payroll-fed database; credentialed verifiers; FCRA-governed. 149 million verifications in 2024, 5.06 million contributing employers, from $73.45, paid by the verifier.

**Wins on** coverage, speed, price, legal acceptance and the direction of payment. **Loses on** being report-level rather than document-level, no selective disclosure, no offline verification, and nothing for a document whose *content* matters.

### 2.2 Payroll connectivity — Truework, Argyle, Pinwheel

Consumer-permissioned APIs connected to payroll providers. **Wins on** consent, API-native integration and modern developer experience. **Loses on** the same custody model and on coverage limited to connected payrolls.

### 2.3 Background screeners — HireRight, Checkr, Sterling

The service layer that actually performs employment verification for employers, lenders and landlords. **Wins on** being the buyer's existing workflow: nobody has to change their process. **Loses on** cost and latency, and it is a human-in-the-loop service, not a verifiable record.

### 2.4 Velocity Network — the direct hit

A global consortium (90+ software vendors) operating a trust framework and a "purpose-built decentralized protocol and blockchain network" for career credentials. Publicly claimed: **2 million credentials issued, 100+ registered issuers, credentials claimed in 185 countries**, 47,000 healthcare background screenings in Florida and Texas with an 81% opt-in rate.

It explicitly targets the case `CNP-002` claims:

> "Employers ... issue verifiable digital credentials attesting to their employment history, and instantly verify career data shared by applicants and employees."

And it targets the **staffing-agency issuer** that `CNP-002` §2.1 identified as the stakeholder hypothesis: the UK Recruitment & Employment Confederation launched an industry initiative to let small and mid-sized staffing firms "begin issuing and verifying employment credentials."

It also inverts the cost model that `ANL-004` §6 flagged as the adoption blocker: "The Network is paying you for your contribution every time you issue a credential."

**Wins on** deployment, wallet-held credentials, selective disclosure, staffing-sector adoption, issuer economics, and the existence of a trust framework. **Loses on** depending on a chain for non-equivocation, and — unverified — on offline verification and historical status.

**Status: P.** Vendor and consortium claims, not independently verified in this review.

### 2.5 ESSPASS, DC4EU and the EUDI Wallet — the legislative hit

The European Social Security Pass ran as a pilot from 2021 to 2025 and is now a **proposed regulation** (COM(2026) 980, September 2026). The DC4EU Large Scale Pilot tested digital issuance, wallet storage and cross-border verification: 22 countries, 99 institutions, 16 issuers from 13 countries, 6,097 wallet transactions.

The proposed regulation requires verifiers to be able to confirm exactly three things about a document:

> (a) the document was issued by an authorised institution;
> (b) the data contained in the document have not been tampered with since it was issued;
> (c) the document has not expired or been revoked.

And it gives those checks legal effect:

> "Where a document issued in accordance with this Regulation has been verified, its validity or authenticity shall not be subject to further proof requirements in the host Member State."

**Wins on** legal recognition, mandated verifier capability, mutual recognition across borders, government-funded deployment, and regulated trust anchors (qualified electronic seals, trusted lists). It also covers the cross-border mobility case that `CNP-002` §2.1 named as the unserved niche.

**It is not an employment letter.** ESSPASS covers social security documents — Portable Document A1 and the EHIC — not role, salary or contract terms. But the *verification question* it legislates is identical to the proposal's claim, and the template is now standardised, funded and legally recognised for the adjacent document class.

**Status: V** for the proposed regulation text; **P** for pilot outcomes.

### 2.6 The generic VC stack — W3C VC, SD-JWT, mdoc, status lists

Holder-held credentials with selective disclosure (SD-JWT), offline proximity verification (ISO 18013-5 mdoc), and standardised status lists (W3C Bitstring Status List; IETF Token Status List). `ANL-004` §5.4 already established that these are commoditised and should be adopted rather than rebuilt.

**Wins on** standards, wallet support, selective disclosure and offline verification. **Loses on** status at a past time and on non-equivocation.

### 2.7 Signed PDF — the My eQuals pattern

Issuer-signed document, portal-based verification, deployed nationally. **Wins on** simplicity and legal acceptance. **Loses on** holder-side proof, offline status and revocation history.

### 2.8 The phone call

The status quo, and still the control in every anchor in `ANL-008`. **Wins on** being free, universal and already trusted. **Loses on** insiders, absent employers and foreign issuers — which is the whole point.

---

## 3. The comparison

No weighted score. A cell says what the option provides.

| Property | Phone call | Signed PDF | Work Number | Payroll API | **Velocity** | **ESSPASS / EUDI** | VC + status list | **Proposed** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Issuer-authenticated | ✓ (human) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Holder-held | ✗ | ✓ (file) | ✗ | partial | ✓ | ✓ | ✓ | ✓ |
| Selective disclosure | ✗ | ✗ | ✗ | partial | ✓ | ✓ | ✓ | ✓ |
| Offline verification | ✗ | ✓ | ✗ | ✗ | unverified | partial (mdoc) | ✓ | ✓ |
| Status at verification time | ✓ | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Status at a past decision time** | ✗ | ✗ | partial | partial | unverified | ✗ (real-time) | ✗ | **✓** |
| **Non-equivocation without a chain** | ✗ | ✗ | ✗ | ✗ | ✗ (chain) | ✗ (ledger) | ✗ | **✓ (witnesses)** |
| No regulated framework needed | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| Deployed | ✓ | ✓ | ✓ | ✓ | ✓ | pilot → production | ✓ | ✗ |

**The honest reading of the bottom rows.** The proposal is alone in exactly **two** cells: *status at a past decision time* and *non-equivocation without a chain*. In every other cell it ties with, or loses to, something already deployed or legislated.

One of those two cells is the parcel claim that failed in `DISCUSSION-SUMMARY-004`: non-equivocation without custody, which nobody exercises. That leaves **one** cell of genuine white space.

---

## 4. Where the incumbents win, stated plainly

1. **Legal recognition.** ESSPASS gives verified documents mutual recognition across member states. The proposal has none, and cannot get any.
2. **Deployment.** Two million credentials in 185 countries; 149 million verifications a year; a funded EU regulation. The proposal has a prototype, at best.
3. **Trust anchoring.** The EU trusts qualified electronic seals and trusted lists — regulated, auditable, legally defined. Witness cosignatures are a research construct by comparison, and are weaker, not stronger, in a regulated market.
4. **Adoption economics.** Velocity pays issuers per credential. The proposal asks the employer to integrate at its own cost — the exact inversion `ANL-004` §6 identified as the blocker. The incumbent has already solved the problem the proposal has not.
5. **Coverage of the anchors.** Four of the five primary anchors in `ANL-008` concern *existing or past employment* — Osuagwu, Smalley, Kenyeres, Agbai. That is precisely what The Work Number and Velocity already verify. Only Tisha Lee concerns an *offer*, which is forward-looking and outside both.

---

## 5. The residual gap

Stated as narrowly as the evidence allows:

> **A verifier holding a bundle, offline, with no prior relationship to the issuer or the log, can establish what the record's status was at a past decision time, and can detect that the issuer showed a different status to another verifier — without trusting a chain or a regulator.**

That is real. It is also very small. And it is a *technical* gap, not a market one: no verifier has asked for it, no regulation requires it, and the two deployed systems that come closest satisfy their users without it.

---

## 6. The falsifiers

The claim fails if any of these holds, and three of them now look likely:

1. **ESSPASS or the EUDI Wallet already answers the historical-status question** through retained status lists or ledger state. *(Needs checking; the regulation is silent on past-time status.)*
2. **Velocity already supports offline verification and historical status**, making the residual empty.
3. **The forward-looking offer/sponsorship case is too thin to anchor a dissertation**, leaving only existing employment — which is served.
4. **No verifier ever contests a status-at-a-past-time question**, making the remaining cell unexercised — the parcel falsifier again.
5. **A regulated trust framework is required for acceptance**, making a self-hosted log inadmissible in the only market that matters.

---

## 7. The roast's findings

| # | Finding | Severity |
| --- | --- | --- |
| **R1** | The three properties the proposal claims — authorised issuer, integrity, not revoked — are **exactly** the three the ESSPASS regulation requires verifiers to confirm, with legal effect, for the adjacent document class | **Fatal to novelty** |
| **R2** | A deployed credential network already issues employment-history credentials at scale, to staffing agencies, and pays issuers to do it | **Fatal to the adoption story** |
| **R3** | `CNP-002` §7.3 omitted both of the above; the comparison was against the wrong field | **High — process failure** |
| **R4** | Four of five primary anchors concern existing employment, which the incumbents already serve | **High** |
| **R5** | "Offline" is not a differentiator: signed PDFs are offline, and mdoc is offline by design | **High** |
| **R6** | The surviving cells are historical status and non-equivocation-without-a-chain; the second is the parcel claim that already failed | **High** |
| **R7** | The EU's trust model is regulated; witnesses are weaker there, not stronger | **Medium** |
| **R8** | The proposal still has no answer to "who pays for issuance," which Velocity has already answered by paying | **Medium** |

---

## 8. What would have to change

Three routes, and none is comfortable.

1. **Narrow to forward-looking documents.** Offer letters, sponsorship commitments and contract terms are not history, so neither The Work Number nor Velocity covers them. Tisha Lee anchors the offer case. But one anchor is thin, and the segment may be too small.
2. **Narrow to historical status as the contribution**, with employment merely the vehicle, and evaluate against ESSPASS's real-time model and Velocity's ledger state. This keeps a positive result — "incumbents cannot answer this offline" — but it is a primitive, not a system.
3. **Stop.** If R1 and R2 hold after checking, the employment case is served, and the honest conclusion is that the domain search has now failed four times for the same structural reason: the witnessed log's distinctive property is either already supplied or unexercised.

Route 3 is not available as a *dissertation headline* under the module constraints, but it is available as a decision.

---

## 9. Evidence register

| # | Source | Claim used | Status |
| --- | --- | --- | --- |
| 1 | European Commission, proposal COM(2026) 980, Sept 2026 | ESSPASS regulation; verifiers must confirm authorised issuance, integrity, and non-revocation; verified documents not subject to further proof in host state | **V** |
| 2 | European Commission, ESSPASS pages and Q&A (2026) | Pilot 2021–2025; 12 countries in two consortia; wallet-based verification | **V** |
| 3 | DC4EU D6.2, D6.3.2 and final report (2025–2026) | 22 countries, 99 institutions, 16 issuers, 6,097 transactions; PuB-EAA model; eIDAS/QEAA trust | **P** |
| 4 | Velocity Network Foundation, 2025–2026 | 2M credentials, 100+ issuers, 185 countries, 90+ vendors; employment-history credentials; issuer remuneration; UK REC staffing initiative | **P** |
| 5 | Equifax / theworknumber.com (2025) | 149M verifications, 5.06M employers, from $73.45, verifier pays | **P** |
| 6 | `ANL-008` | The seven employment-document anchors | **V/P/A** as labelled |
| 7 | `ANL-004` §5.4 | Status lists are standardised; time-relative offline status is the residual | **V** |

---

## 10. Caveats

- Velocity's and DC4EU's figures are self-reported. Their *coverage of letters of terms* is unverified; they demonstrably cover employment history and social security status.
- ESSPASS is a **proposal**, not law, at the time of writing. Its timetable could slip. Its existence still defeats a "no one is doing this" claim.
- The comparison is written from the verifier's perspective. An employee-centric comparison would weight holder control and portability more heavily, where the proposal is competitive but not alone.
- No weighted score is used, and nothing here measures cost or latency.

---

## 11. One-paragraph summary

The employment proposal was compared against the wrong field. The strongest alternatives are not The Work Number and paper; they are Velocity Network, which already issues holder-held employment-history credentials at two-million scale across 185 countries and pays issuers to do it, and the European Social Security Pass, whose proposed regulation requires verifiers to confirm exactly the three properties the proposal claims — authorised issuance, integrity, and non-revocation — and gives verified documents mutual recognition across member states. Against those, the proposal is alone in two cells: status at a past decision time, and non-equivocation without a chain. The second is the parcel claim that already failed. The first is a real but narrow technical gap that no verifier has asked for and no regulation requires. The roast's conclusion is that the employment case, as framed, is served: the mechanism is standardised, the market is occupied, and the adoption economics are inverted against us. What survives is a primitive, not a system, and the honest next step is to decide whether a primitive is enough — or whether four failed domains are telling us something about the mechanism itself.
