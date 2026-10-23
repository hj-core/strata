# CNP-002 — Concept Note: Verifiable Employment Records

| Field | Value |
| --- | --- |
| Document ID | CNP-002 |
| Title | Concept note — offline-verifiable employment records with time-relative status |
| Version | 0.1.0 |
| Status | Draft — **on hold**. The strongest-alternative test (`ANL-009`) narrows the claim to one proposition. The decision on whether to proceed is in `DEC-002` §8. |
| Date | 2026-10-23 |
| Module | COMP4299 — system development; dissertation + demo + evaluation |
| Related documents | DEC-002 (decision); ANL-008 (anchors); ANL-004 (demand); PRC-001 (domain search, historical); PRP-001a (handover primitive); RFC 9943, RFC 9942, RFC 9162, RFC 3161 |
| Purpose | State the problem, the demand, the proposed slice, the claim and the honest boundary in one document for review before committing to a build |
| Supersedes | CNP-001 (parcel/adjudicated delivery) as the concept note. CNP-001 is retained as the historical record of the roast. |

---

**Module constraints (2026-10-23).** COMP4299 accepts a **narrow mechanism with a measured boundary**. A **negative headline is not acceptable**: the primary result must be a demonstrated capability, with the boundary stated as a limitation rather than as the finding. A **plausible scenario is sufficient**; no external stakeholder has to be recruited.

These three constraints fix the shape of this note. The claim is narrow (§3), the primary result is positive (§5, §9), the boundary is subordinate (§8), and the scenario is documented and plausible rather than recruited (§4).

---

## 1. The problem

Employment verification runs on a phone call. A landlord, lender, consulate or next employer receives an employment letter, and its only control is to contact the employer and ask whether the letter is genuine.

That control fails in four documented ways:

| Failure | Documented in | Why the call fails |
| --- | --- | --- |
| **Insider-issued** | OCC order, *In re* Amy Lu (2022); Tisha Lee (2026) | The call reaches the insider, who confirms the lie |
| **Applicant-fabricated** | *US v. Osuagwu*; *US v. Smalley*; *R. v. Khatchatourov* | There is no issuance to confirm |
| **Unresolvable issuer** | *Kenyeres* [2020] AATA 1050; *Agbai v Canada* | The employer is foreign, defunct, or does not exist |
| **Cross-border** | *Agbai*; CBP Champlain (2019) | Verification routes through a foreign authority |

The clearest case is the most recent. A US Government Publishing Office contract employee sent fake offer letters on agency letterhead to people seeking to rent apartments. When landlords called the agency to verify, **she answered and confirmed them**. The control was exercised exactly as designed and it failed, because a telephone call cannot distinguish an authorised issuer from an employee with a mailbox.

This is not primarily a signature problem, and it is not a forgery problem in the narrow sense. It is two problems:

1. **Authenticity is not establishable without contacting the issuer**, and the contact channel is the weakness.
2. **Status is not establishable at the decision time.** A letter can be genuine when issued and superseded or revoked later; a verifier who checks "now" cannot tell what was true when the decision was taken.

Full case register: `ANL-008`.

---

## 2. The demand

Employment verification is not a hypothesis. It is a mature, verifier-funded market.

| Measure | Value | Status |
| --- | --- | --- |
| Verifications processed by The Work Number, 2024 | **149 million** | P |
| Average requests per day | ~400,000 | P |
| Contributing employers | 5.06 million | P |
| Entry price per report | from $73.45 | P |
| Who pays | **The verifier** | P |

Two structural facts matter:

- **The verifier pays.** The beneficiary and the payer are the same party. This is the first domain examined where that is true.
- **The market exists because the document is untrusted.** The incumbent's own pitch is that manual verification means "stacks of printouts, pay stubs and games of phone tag."

### 2.1 What this project does not compete for

The Work Number answers "is this person employed, and what do they earn" in seconds, for $73.45, with FCRA-governed reliance. **This project does not compete there and should not try.**

The niche is narrower:

- letters whose **content** matters — role, contract type, sponsorship, terms;
- **cross-border** cases where the employer is not a contributor, or is defunct;
- **consular and licensing** contexts where the verifier has no domestic channel to the issuer;
- **high-volume issuers** — staffing agencies and employers of record — who issue many letters and absorb the cost of every verification call.

That last point was the stakeholder hypothesis. `ANL-009` found that Velocity Network already targets it — the UK Recruitment & Employment Confederation initiative lets small staffing firms issue and verify employment credentials. The agency route is therefore occupied, and the remaining defensible ground is not a market segment but a **property**: offline status at a past decision time.

ESSPASS occupies the cross-border *social security* case with legislation; it does not cover *employment terms*. That distinction is narrow but real, and §7.3 states it rather than implying the cross-border field is empty.

---

## 3. The proposal

A thin end-to-end slice on the existing handover primitive, at its shallowest composition: the **document variant** (`PRP-001a` §2.1) — `issue` → `receipt`, two statements, two issuers.

| Element | Choice |
| --- | --- |
| Statement | `COSE_Sign1` (RFC 9052), canonical CBOR, CWT `iss`/`sub` (RFC 9597) |
| Chain | `issue` (employer) → `receipt` (employee) |
| Log | RFC 9162 Merkle log via Tessera |
| Receipts | RFC 9942 COSE receipt over an RFC 9162 leaf |
| Time | RFC 3161 token per statement |
| Status | Adopt a status list (W3C Bitstring Status List or IETF Token Status List), **not** a new one |
| Witnesses | Two independent processes cosigning checkpoints (treatment arm) |
| Verifier | Offline CLI/web verifier with named outcomes |
| Disclosure | Salary band selectively disclosable; the landlord sees employment, the lender sees income |

### 3.1 Why the log, and only there

A signed record already rejects a forged letter and already verifies without a call. The log does **not** earn its place on authenticity. It earns it on three things a signature cannot supply:

1. **Status at a past time.** Was this record valid, and this key authorised, at the moment the verifier decided? A status list gives *now*; the log plus inclusion at a checkpoint gives *then*.
2. **Holder-side version pinning.** The employee's `receipt` fixes the exact version issued, so a later revision cannot silently redefine what was held.
3. **Non-equivocation across verifiers.** The employer cannot show one status to a consulate and another to a lender without a fork being detectable.

Of these, (1) is the genuinely hard one and the real technical contribution.

---

## 4. The scenario

**Parties.** Meridian Health (employer, or a staffing agency acting as employer of record), Ana (employee), Ashford Lettings (verifier), ClearLog (independent log operator), two audit firms as witnesses, FreeTSA, UniCA.

**Flow.**

1. Meridian's HR system issues a signed `issue` statement: holder, role, start date, contract type, salary band, status, `prev: null`, `sequence: 1`.
2. Ana's wallet verifies it and signs `receipt` with `prev` over the exact signed bytes. She now holds proof of what was issued.
3. Ashford receives the record, verifies Meridian's signature offline against a published key and checks the role binding, then checks status at the decision time against a checkpoint and status list.
4. Salary is disclosed as a band to Ashford, and in full only if Ana chooses to disclose it.
5. A year later Meridian revises its template and reissues. Ana's `receipt` still pins the original. If Meridian revokes the original, the log shows it existed and was valid at the earlier decision time.

**The manipulations the slice must survive.**

| Attack | Outcome expected |
| --- | --- |
| Letter forged by the applicant | Rejected: no valid attestation exists |
| Letter issued by an insider without authority | Rejected: the key is bound to an issuance role, not a mailbox |
| Superseded version presented as current | Detected: `receipt` pins the version; status list shows supersession |
| Revoked record presented as valid | Detected: status at decision time; `revoked_before_decision` |
| Employer shows different status to two verifiers | Detected: `split_view` against a cosigned checkpoint |
| Salary disclosed to a party that should not see it | Prevented: selective disclosure, not whole-document |

**The honest limit.** If Meridian lies about Ana's salary at issuance, the log records the lie faithfully. The log is not an oracle.

---

## 5. What the work demonstrates

| Test | A: phone call / Work Number | B: signed record | C: signed + status list | **D: logged + witnesses** |
| --- | --- | --- | --- | --- |
| Forged letter | partial (call may fail) | **rejected** | **rejected** | **rejected** |
| Insider-issued letter | **undetected** | rejected if role-bound | rejected if role-bound | rejected if role-bound |
| Superseded version | undetected | undetected | **detected** | **detected** |
| Revoked record, offline, at past time | undetected | undetected | partial (current only) | **detected** |
| Split view | undetected | undetected | undetected | **detected** |
| Salary over-disclosure | undetected | undetected | partial | **controllable** |
| Offline verification, no call | **no** | yes | yes | yes |

The bottom row is where the incumbent loses and the proposal wins. The rows above it are where **B and C already win**, and the dissertation must say so rather than credit the log with them.

Supporting measurements: verification latency, bundle size, append throughput, and the number of trust anchors each party must configure. Each labelled as a target or a measurement.

### 5.1 The non-equivocation axis

RFC 9943 requires non-equivocation of the VDS but does not specify a mechanism (§7.1). The split-view row therefore has its own comparison, and it is the one place where the choice of *mechanism*, rather than the choice of level, decides the result.

| Arm | Mechanism | Detection | Cost | Fails when |
| --- | --- | --- | --- | --- |
| **R1** | Auditor replay with read access | After the fact, on a schedule | Read access plus audit labour | Nobody audits, or the fork sits between audits |
| **R2** | Each statement registered with two independent TSs | On comparison of receipts and checkpoints | Two operators, two policies, two receipts | Both operators collude, or the verifier checks only one |
| **R3** | One TS with two cosigning witnesses | Continuously, at checkpoint time | Two cosigners plus gossip | Witnesses are not independent, or the verifier ignores the cosignatures |

R2 is the RFC's own answer (§9.7); R3 is ours. The comparison is the honest form of the witness claim: not "witnesses are required," but "witnesses supply a property the RFC requires at a lower operating cost than the RFC's own alternative."

---

## 6. Contribution

1. **An employment-record profile** with holder-side version pinning, role-bound issuer keys, and field-level salary disclosure. Non-trivial: it defines what "issued", "superseded" and "revoked" mean for a document whose status changes after the holder takes custody, and it requires a registration policy that lets the holder submit the issuer's statement.
2. **Time-relative offline status.** The verifier establishes validity *at the decision time* from the bundle alone. This is the one proposition the project adds to the set RFC 9943 makes checkable (§7.1), and it is the gap `ANL-004` §5.4 identified: status lists solve "now", not "then". Adopting a standard status list rather than inventing one is part of the contribution's honesty.
3. **A measured comparison** of four levels against six documented attacks, including the explicit result that authenticity is a signature property and not a log property, plus a separate comparison of the three non-equivocation mechanisms (§5.1) — the RFC's own two-TS answer against witness cosignatures.
4. **The boundary.** A domain-selection argument (`PRC-001`, `DEC-002`) showing why this case survives tests that eliminated the parcel and automotive directions.

---

## 7. Related work

### 7.1 What RFC 9943 specifies, and what it leaves open

RFC 9943 lists three security guarantees (§9):

1. Statements are identifiable and authenticatable.
2. "Statement provenance and history can be independently and consistently audited."
3. "Issuers can efficiently prove that their Statement is logged by a TS."

History and audit are therefore a first-class guarantee — but they are the RFC's **purpose**, not its normative content. What it specifies is the message format (Signed Statement, Transparent Statement, Receipt; §6–§7) and the registration contract (mandatory checks, transparent policies, reproducible audit; §5.1).

The precision matters. The RFC makes a narrow set of propositions **checkable**, and leaves every proposition about the world unchecked.

| Checkable proposition | Proof | Section |
| --- | --- | --- |
| "Issuer X signed S" | COSE signature | §9 |
| "S is at leaf *n* of a tree of size *m*" | Inclusion proof in the receipt | §7 |
| "Tree of size *m* is a prefix of tree of size *m′*" | Consistency proof, where the VDS supports it | §5.1.3 |
| "The policy that admitted S was in force then" | Policy and trust anchors registered as statements | §5.1.1.1 |
| "Another party sees the same log" | Compare checkpoints | §5.1.3 |

Three different things are called "history," and only the first is checkable.

| Sense | Checkable? | Where the RFC says so |
| --- | --- | --- |
| The **log's** history — statements actually registered | **Yes** | §5.1.3, §4 |
| The **issuer's** history — including what was withheld | **No** | §9.3: issuers "selectively submit some but not all" |
| The **world's** history — what actually happened | **No** | §9.2: "registering a Statement only proves it was produced by an Issuer" |

Four limits follow, and the RFC states all four:

| Limit | Section | Consequence for this project |
| --- | --- | --- |
| Ordering | §9.1 | Log order is not issuance order unless the policy says so; we need a TSA token for issuance time |
| Accuracy | §9.2 | A registered statement may be false; we cannot claim truth |
| Selective submission | §9.3 | The relying party must refuse statements without receipts; completeness is not supplied |
| Revocation | §9.4.2 | "Revocation strategies for compromised keys are out of scope" — **this is our gap** |

**On witnesses.** RFC 9943 does not mention them; the word does not appear. Non-equivocation is a requirement on the **VDS** (§5.1.3), not a role in the protocol, and the RFC leaves the mechanism open. Its own alternatives are an auditor with read access (§4) and **multiple independent TSs** (§9.7: "Running multiple, independent TSs provides different organizations to represent consistent or divergent opinions"). Witness cosignatures are an implementation choice from the CT and Sigsum lineage, and must be described as such — never as an RFC requirement.

**On rollback.** §9.4.2 concedes that a TS whose signing key is compromised "can roll back their Statement Sequence to a point before compromise." Append-only is a property the VDS must support; it is not a guarantee against a compromised operator. Any claim about status at a past time must say how it treats a checkpoint that contradicts an earlier cosigned one.

**On tiers.** RFC 9943 defines no tiers or levels. The useful distinction — a receipt proves inclusion offline, while consistency and non-equivocation require an auditor or a second log — is real, but it is not the RFC's vocabulary, and this document does not call it a tier.

**The contribution, in one sentence.** The RFC fixes a set of checkable propositions. This project adds exactly one: *"this record was valid, and this key was authorised, at the moment the verifier decided."* That proposition is not checkable under RFC 9943, because revocation is out of scope and the log records registration rather than validity.

### 7.2 SCITT's three use cases, and where this one sits

RFC 9943 §2.2 defines exactly three use cases, all software-supply-chain specialisations of the §2.1 problem. The RFC scopes itself to "**single-issuer** signed statement transparency," so multi-party situations are handled by multiple statements about one Subject, not by co-signed statements.

| RFC 9943 §2.2 | Situation | What the consumer needs | What SCITT supplies |
| --- | --- | --- | --- |
| **2.2.1 Security analysis of a product** | Statements about a product accumulate from the producer and third parties as vulnerabilities are found, shielded, patched and replaced | To find statements, attribute them to an authoritative issuer, relate them, and confirm the statement history is complete | Source discovery; provenance and historicity; linking heterogeneous statements; authority checks; a complete *registered* history |
| **2.2.2 Promotion by multiple entities** | A component is re-distributed, bundled, patched or recompiled by others and "takes myriad forms" | To know whether a provider is the originator or an alternative, how the component differs, and the provenance path | Discrimination between original, trustworthy alternative and illegitimate providers; lineage; integrity of transformations |
| **2.2.3 Integrator assembling a product for a vehicle** | An OEM sources integrated software from third parties integrating components from many sources | A component list; retrievable components; "verifiable proofs on build process and build environment with all supplier tiers" | A tiered framework verifying integrity at component and product level; build-integrity annotations |

**Our case is a fourth, and the RFC does not describe it.** Three differences matter:

1. **All three RFC cases are histories of a commodity moving down a chain.** Ours is a record held by a person, presented to an unrelated third party who is not the next party in any chain.
2. **SCITT has no holder role.** The roles are Issuer, Subject, TS and Relying Party. The employee is an Issuer of a `receipt` statement about the same `sub` — which §3 permits ("distinct Issuers ... create a coherent sequence of Signed Statements about the same Artifact") — but the RFC does not describe a holder who both presents the record and can forge it.
3. **In a commodity chain the holder is a custodian; in a credential the holder can be the attacker.** That is why the record must be bound to the holder's identity and why selective disclosure is table stakes rather than a feature.

**One phrase to handle with care.** §2.2.1 says SCITT lets a consumer "confirm that sources provide a complete history of statements related to a given component." Read strictly, that is completeness over **registered** statements, which §9.3 qualifies: issuers may "selectively submit some but not all." It is not completeness over **events**, as `DISCUSSION-SUMMARY-003` established. Quote the RFC's phrase in the related-work section, then state the narrower reading in the same paragraph, or a reviewer will supply the correction.

**Scope note.** RFC 9943 §1 limits itself to software supply chains but states the architecture "is applicable to any other type of supply chain statements." A credential supply chain is an extension, not an application, and the concept note should say so rather than imply SCITT covers it.

### 7.3 The incumbent comparison

**Superseded by `ANL-009`.** The table below is the short form. The full matrix, the two alternatives that were missing, and the roast are in that document.

| Alternative | What it does | Where it wins | Where it loses |
| --- | --- | --- | --- |
| **The Work Number** | Centralised payroll-fed database; credentialed verifiers; FCRA-governed | Coverage, speed, price, legal acceptance | Report-level; no selective disclosure; no offline; nothing for letters of terms |
| **Velocity Network** | Consortium trust framework and blockchain for holder-held career credentials; ~2M credentials, 100+ issuers, 185 countries | Deployment, wallet-held credentials, selective disclosure, staffing-sector adoption, **pays issuers** | Non-equivocation depends on a chain; offline and historical status unverified |
| **ESSPASS / EUDI Wallet** | Proposed EU regulation plus a completed large-scale pilot for cross-border social security documents | **Legislated** authorised issuance, integrity and non-revocation; mutual recognition; regulated trust anchors | Covers social security documents, not employment terms; real-time, not historical |
| **Truework / Argyle / Pinwheel** | Payroll-connect APIs, consumer-permissioned | Consent-based, API-native | Same custody model; connected payrolls only |
| **Background screeners** | The service layer that performs verification today | Already in the buyer's workflow | Cost, latency; not a verifiable record |
| **Signed PDF (My eQuals pattern)** | Issuer-signed document, portal verification | Simple, deployed, legally understood | No holder-side proof; no offline status |
| **W3C VC / SD-JWT / mdoc + status lists** | Holder-held credentials, selective disclosure, offline proximity, standard status | Standards, wallets, offline verification | Status is "now"; no historical status; no non-equivocation |
| **Phone call to the employer** | The status quo in every anchor | Free, universal | Fails on insiders, absent employers, foreign issuers |

**Two findings change the claim.**

- **R1 — ESSPASS legislates the property set.** The proposed regulation (COM(2026) 980) requires verifiers to confirm that a document was issued by an authorised institution, has not been tampered with, and has not expired or been revoked — and gives verified documents mutual recognition. That is this proposal's claim, in law, for the adjacent document class.
- **R2 — Velocity owns the deployment.** A consortium of 90+ vendors already issues holder-held employment-history credentials at scale, includes staffing agencies, and pays issuers per credential — inverting the cost problem `ANL-004` §6 identified.

**What remains.** Across nine properties, the proposal is alone in exactly two: **status at a past decision time, offline**, and **non-equivocation without a chain**. The second is the parcel claim that failed in `DISCUSSION-SUMMARY-004`. The first is the basis of §9's positive result, and it is a primitive, not a system.

### 7.4 Why not a registry

If the verifier-operated registry is run by a party the verifier already trusts, it is simpler and better, and this project is over-engineered. The log is defensible only where the registry operator is the *issuer* — the party with the incentive to withhold or equivocate — and the verifier has no audit rights over it. That is the condition the evaluation must test, not assume.

---

## 8. Honest boundary and non-claims

- It does **not** prove the employer told the truth about the salary, role or dates. It proves what the employer asserted, to whom, and when.
- It does **not** prove the employee is still employed at verification time; it proves status *as at the decision time* and, separately, *now* where a current status is available.
- It does **not** prevent forgery. It makes authenticity establishable without contacting the issuer, which is a different and narrower claim.
- It is **not** a consumer report and produces no score or eligibility output. The verifier performs its own decisioning. FCRA scoping is explicit.
- It is **not** a qualified trust service and carries **no** legal recognition. QES and QERDS are legally stronger, and the dissertation says so.
- It is **not** a blockchain; it is an append-only log with receipts and optional witnesses.
- The anchors establish that the failure mode is real and documented. They do **not** establish its prevalence; `ANL-004`'s market figures are the prevalence proxy and remain **P/S** status.

---

## 9. Evaluation design

**The primary result is positive, and it is narrower than the earlier drafts of this note claimed.** The mechanism establishes, offline and from the bundle alone, that a record was issued by an authorised issuance role, for that holder, with that content, and that it had not been superseded or revoked at the decision time — without contacting the issuer. **No baseline achieves all four of those properties offline, from the bundle alone, at a past decision time.** ESSPASS achieves three of them at verification time through real-time checks, with legal recognition; Velocity achieves authenticity and holder control at scale. Neither answers the historical question without a live lookup. The hypotheses below measure that capability and its cost. H2 is not a pass/fail gate on the project; it establishes how much of the detection the log carries, so that the boundary in §8 can be stated precisely.

| Hypothesis | Falsified if |
| --- | --- |
| **H1** — A role-bound issuer key rejects an insider-issued letter that a mailbox-based scheme accepts | Role binding adds nothing over a signature with an email-verified issuer |
| **H2** — Time-relative offline status is established from the bundle alone | A plain status list plus a TSA does it at lower cost — **the decisive test** |
| **H3** — Offline verification completes in under 1 s at p99 | p99 exceeds 2 s |
| **H4** — Salary band is disclosed to a landlord and full salary only on the holder's consent, under one issuer signature | Selective disclosure requires re-issuance or breaks the signature |
| **H5** — Against a TS that shows different histories to two verifiers, witness cosignatures (R3) detect the fork at lower operating cost than registering each statement with two independent TSs (R2) | R2 detects at equal or lower cost, or R3's detection depends on a verifier that never checks the cosignatures — **report as untested, not confirmed** |

Workload: 1,000 issuances/day over 30 simulated days, 10% offline verification, six attacks injected at known points. Levels A–D as in §5, with the split-view attack run against mechanisms R1–R3 as in §5.1. No weighted score.

**H2 measures the log's marginal contribution, not the project's viability.** If an unanchored status-list snapshot proves sufficient in the evaluation, the result narrows to the signed-record plus anchored-bundle claim; that is reported in §8 as a limitation and in the dissertation as a finding about where the log is and is not load-bearing. It is not the headline.

---

## 10. Risks

| Risk | Mitigation |
| --- | --- |
| **R1 — ESSPASS legislates the property set** (`ANL-009`) | Position the claim on offline historical status, which the regulation does not cover; state the overlap explicitly in the dissertation and never claim novelty for authorised issuance, integrity or revocation |
| **R2 — Velocity owns employment-history credentials** (`ANL-009`) | Narrow to the document class it does not cover (letters of terms, forward-looking offers) or to the historical-status primitive; do not compete on coverage |
| **H2 weak** — a status list plus a TSA covers part of the gap | The result narrows to the anchored-snapshot claim; the log's marginal contribution is measured and reported in §8 as a limitation, not as the headline |
| The niche is too small to matter | Size the cross-border and high-volume-issuer segments before committing; report the number, not an adjective |
| EUDI Wallet subsumes the credential | Position explicitly; the residual is time-relative status and non-equivocation, not the credential format |
| Employer bears integration cost and gains least | Target high-volume issuers and verifier-side stakeholders, not single employers |
| FCRA scope creep | Issue-only evidence; no scoring, no eligibility; state it in §8 and §12 of the dissertation |
| Demo legibility | The Tisha Lee scenario is the demo: the call that confirms a lie, then the same record the insider cannot produce |

---

## 11. One-paragraph summary

Employment verification runs on a phone call, and the call fails in four documented ways: the insider answers, the employer never existed, the employer is foreign or defunct, or the check routes through a foreign authority. Seven cases, four at primary or official status, establish the failure mode; a market of 149 million verifications a year establishes that the verifier pays. This project builds the document variant of the handover primitive — employer issues, employee holds a version-pinned receipt, verifier checks offline — on an RFC 9162 log with RFC 9942 receipts and an adopted status list. The primary result is a capability no baseline provides **offline and at a past decision time**: a verifier establishes, from the bundle alone and without contacting the issuer, that a record was issued by an authorised role, for that holder, with that content, and that it had not been superseded or revoked at the decision time. The comparison is honest about the field: ESSPASS legislates authorised issuance, integrity and revocation for social security documents, and Velocity deploys employment-history credentials at scale; neither answers the historical question offline. Across nine properties the proposal is alone in two cells, and one of them is the parcel claim that already failed. Authenticity is a signature property and the dissertation says so; the log earns its place on time-relative status, holder-side version pinning and non-equivocation, and H2 measures exactly how much of each it carries. Where the log is not load-bearing, that is reported as a limitation. This case is a fourth SCITT use case, not one of RFC 9943's three: it is a credential held by a person who can also forge it, not a commodity moving down a chain.
