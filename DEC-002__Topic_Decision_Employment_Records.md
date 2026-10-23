# DEC-002 — Topic Decision: Employment-Record Evidence

| Field | Value |
| --- | --- |
| Document ID | DEC-002 |
| Title | Topic decision: leaving automotive test evidence for employment-record evidence |
| Version | 0.1.0 |
| Status | Draft — decision record. The employment direction passed the anchor gate and was narrowed by the strongest-alternative test (`ANL-009`). **D8 is open.** |
| Last Updated | 2026-10-23 |
| Related documents | BUILD-HANDOVER-004; DISCUSSION-SUMMARY-004; PRC-001; ANL-004; ANL-006; ANL-007; PRP-001a |
| Purpose | Record why the automotive direction was dropped, which option replaces it, and what must be true for the replacement to be defensible |
| Supersedes | The automotive recommendation in BUILD-HANDOVER-004 §2 and §15, as a direction. `PRC-001` is retained as the record of the assessment, not as a project plan. |

---

## 1. The decision

**The automotive test-evidence direction is dropped as a solution claim. The project moves to the employment-record case.**

Two things are separated here, because they are often conflated:

1. **The automotive analysis was sound.** `PRC-001` traces seven documented scandals through a six-step process and finds that nothing is fully flagged, three are partially flagged, and three are not flagged. That is a correct result, not a failed one.
2. **The automotive direction cannot support a solution claim on that result.** In all three partial cases the load is carried by an institutional actor outside the mechanism: the authority's production measurement, two relying parties choosing to compare, or a lab controlling whether a test is recorded. The part we would build carries none of the detection. That is the same verdict the roast passed on the parcel case — a cost-and-delay claim, not an access claim — restated with larger numbers.

**Do not write `CNP-002` for automotive.** The evidence does not support it.

---

## 2. Why automotive was dropped

The full trace is in `PRC-001` §9–§10. The summary:

| Case | Verdict | Who actually carries the detection |
| --- | --- | --- |
| Toyota Industries | Partial | Authority COP record (step 6, institutional) |
| Kobe Steel | Partial — best fit | Two customers choosing to compare checkpoints |
| Nissan | Partial (generously) | Credential binding removes the stamping channel only |
| Takata | Partial, conditional | The lab controlling whether a test is created |
| Hino | Not flagged | Whistleblower; third-party committee |
| Daihatsu | Not flagged | Engineering review; committee |
| VW | Not flagged | Independent on-road measurement |

The condition that no case satisfies is the decisive one: **the verifier must be compelled to compare, not merely able to.** Without that, non-equivocation is a property nobody exercises, which is the falsifier the parcel case already failed.

The only structural candidate — mutual recognition between two type-approval authorities, where authority B has no audit rights over authority A's registry — is unanchored. No documented case of a manufacturer submitting divergent test evidence to two authorities was found. Under this project's discipline (anchor first, then design), that is speculation.

---

## 3. What option B is

**The employment record.** An employer's HR function issues a signed statement of employment; the employee holds it; a landlord, lender, consulate, professional regulator or next employer verifies it offline.

This is the **document variant** of the existing handover primitive — `issue` → `receipt`, two statements, two issuers (`PRP-001a` §2.1). It is the shallowest composition of the primitive already specified, which is a build advantage over both the parcel case (two handovers) and automotive (three tiers plus a lab).

### 3.1 The parties

| Party | Role | Statement |
| --- | --- | --- |
| Meridian Health (employer) | Issuer | `issue` — role, start date, contract type, salary band, employment status |
| Ana (employee) | Holder and second issuer | `receipt` — pins the exact version issued |
| Ashford Lettings (verifier) | Relying party | Verifies offline; decides for itself |
| ClearLog | Log operator | Appends, receipts, checkpoints |
| UniAudit, Aegis Audit | Witnesses | Cosign checkpoints (treatment arm) |
| FreeTSA | Time authority | Trusted time |
| UniCA | Certification authority | Binds keys to employers |

### 3.2 The failure mode

> The employee holds a document that the employer can later deny, revise or disown, and the verifier has no way to establish what the employer actually issued, to whom, when, or whether the copy presented is that copy.

This is a **denial and revision** problem, not primarily a forgery problem. Forgery is the incumbent's story and the incumbent has a product for it. Denial is the gap: the issuer is the only party who can prove what it issued, and it is also the party with the incentive not to.

---

## 4. The evidence for B

The demand evidence is the strongest in any domain examined so far, and it is market evidence rather than argument.

| Measure | Value | Status |
| --- | --- | --- |
| Verifications processed by The Work Number, 2024 | **149 million** | P |
| Average requests per day | ~400,000 | P |
| Contributing employers | 5.06 million | P |
| Employee records accessible to credentialed verifiers | 839 million+ | P |
| Entry price per report | from $73.45 | P |
| Who pays | **The verifier**; no cost to employer or employee | P |

Two structural facts matter more than the volume:

1. **The verifier pays.** Lenders, landlords and agencies already buy verification. This is the one domain examined where the beneficiary and the payer are the same party.
2. **The market exists because the document is untrustworthy.** The incumbent's own pitch is that manual verification means "stacks of printouts, pay stubs and games of phone tag." The problem statement is already monetised.

Employment is also the one document case with **no established issuance standard**: in practice, a letterhead and a phone number. Transcripts have My eQuals, Parchment and OpenCerts; employment letters have a template.

### 4.1 The anchor check, which has now been run

This section originally stated that the parcel case had **anchors and no demand**, while option B had **demand and weak anchors**, and that finding anchors was the gate on the concept note.

**The gate is passed.** `ANL-008` records seven documented cases from the US, Canada, Australia and India, including four at **V** status: *United States v. Osuagwu* (jury conviction), the OCC prohibition order in *In re Amy Lu*, *Kenyeres v Minister for Immigration* [2020] AATA 1050, and *Agbai v Canada* (Federal Court).

The common structure is sharper than expected:

> The verifier's only control is to contact the employer, and the contact fails whenever the issuer is complicit, unreachable, non-existent, or foreign.

The clearest case is the most recent: a US Government Publishing Office contract employee issued fake offer letters on agency letterhead and **confirmed them by telephone when landlords called to verify**. The control was exercised and failed, because a call cannot distinguish an authorised issuer from an employee with a mailbox.

**One correction follows.** The anchors support an **authenticity** claim, not a **denial** claim. No case shows an employer disowning a letter it genuinely issued; *Kenyeres* is a forged sponsor signature. §5.3 is rewritten accordingly.

### 4.2 The second gate: the strongest alternative

`ANL-009` then ran the strongest-alternative test, and the employment case did not pass cleanly.

- **Velocity Network** already issues holder-held employment-history credentials at roughly two-million scale across 185 countries, includes staffing agencies, and **pays issuers per credential**.
- **ESSPASS** (proposed regulation COM(2026) 980) requires verifiers to confirm authorised issuance, integrity and non-revocation for cross-border social security documents, and gives verified documents mutual recognition.

Across nine properties, the proposal is alone in two cells: **status at a past decision time, offline**, and **non-equivocation without a chain**. The second is the parcel claim that already failed in `DISCUSSION-SUMMARY-004`. The first is real, narrow, and a primitive rather than a system.

**Consequence.** The direction is not withdrawn, but its claim is cut to one proposition. Whether that is enough to proceed is D8 in §8, and it is the only open decision that matters.

---

## 5. The strongest alternative

The incumbent is **not paper**. It is The Work Number and its peers, and they are strong.

| Alternative | What it does | Where it wins | Where it loses |
| --- | --- | --- | --- |
| **The Work Number (Equifax)** | Centralised payroll-fed database; credentialed verifiers; FCRA-governed | Coverage, speed, price, legal acceptance, 149M/yr | Report-level, not document-level; no selective disclosure; no offline; intermediary holds the record; employee holds nothing |
| **Truework / Argyle / Pinwheel** | Payroll-connect APIs, consumer-permissioned | Modern, consent-based, API-native | Same custody model; coverage limited to connected payrolls |
| **Employer portal + phone call** | The status quo | Free, universal, trusted | Unscalable, slow, destroys portability |
| **W3C VC / EUDI Wallet** | Holder-held, selectively disclosable credentials | Standards, wallets, legal trajectory | No non-equivocation: the issuer can still deny or fork its own status |
| **Signed PDF (My eQuals pattern)** | Issuer-signed document, portal verification | Deployed, understood, legally accepted | No holder-held proof of what was issued; no offline; no witnesses |

### 5.1 Where the proposal loses, stated first

1. **Coverage and price.** A national payroll database answers most employment questions in seconds for $73.45. We do not.
2. **Status and income.** If the question is "is this person employed and what do they earn," The Work Number wins and the proposal should not compete.
3. **Legal acceptance.** A creditor relying on a consumer report has a statutory safe harbour; a self-hosted log has none.
4. **Adoption cost.** HR integration is high and the employer gains least. `ANL-004` §6 already records this inversion.

### 5.2 Where it can win

The niche is **letters whose content matters and for which no registry exists**:

- consular and visa sponsorship letters, where role and terms matter;
- cross-border employment, where the employer is not a contributor and may be defunct;
- professional registration and licensing references;
- disputes where the employee must prove what the employer issued, and the employer denies or revises it.

In those cases the property that matters is not "is she employed" but **"did this employer issue this document, with this content, at this time, and can it later deny it."** No incumbent provides that to the holder.

### 5.3 The claim, and why it is a log claim rather than a VC claim

A verifiable credential alone gives issuer signature, holder control and selective disclosure. It does not give offline status at a past time or non-equivocation across verifiers. The log supplies exactly that, and only that:

> A verifier can establish, offline and without contacting the employer, that a specific employment record was issued by that employer's authorised issuance role, for that holder, with that content, and that it had not been superseded or revoked as at the time of the decision.

Note the order. **Authenticity first** — that is what the anchors support, and a forged letter has no valid attestation at all. **Status and versioning second** — that is where the log earns its place, and it is the part a plain signature cannot supply.

The testable properties are: issuer attribution to an authorised role; offline verification without a call; status at a past time; holder-side version pinning; selective disclosure; and non-equivocation across verifiers.

**What it does not prove.** That the employer told the truth about the salary or the role; that the employee is still employed at verification time; that the verifier's decision is lawful; that the record is a consumer report. Those are separate, and the anchors do not support claiming them.

---

## 6. Risks

| Risk | Severity | Mitigation |
| --- | --- | --- |
| **The niche is too small to matter.** Letters of terms may be a thin slice of a market the incumbent already serves. | **High** | Establish the cross-border and denial segments with numbers before committing; if they are small, say so and re-scope to the boundary study |
| **EUDI Wallet subsumes the credential.** Holder-held, selectively disclosable employment credentials are on the EU roadmap. | **High** | Position against it explicitly; the residual is non-equivocation, which wallets do not supply |
| **No adjudicated anchors.** The parcel case had four; this has market statistics. | **High** | First task: find adjudicated employment-document fraud and denial cases. Gate the concept note on it |
| **FCRA scope creep.** Reliance on employment/income data is regulated. | Medium | The log evidences issuance only; no scoring, no eligibility, no consumer report. State it in the concept note and the dissertation |
| **Employer bears integration cost and gains least.** | Medium | Position the verifier or a verification vendor as the stakeholder, not the employer |
| **Demo legibility.** An employment letter is less vivid than a parcel or an engine. | Low | The denial scenario is the demo: issue, revise, deny, verify offline |

---

## 7. What is retained

| Asset | Disposition |
| --- | --- |
| `PRC-001` | **Retained as analysis.** The seven-case matrix is the reason for this decision and belongs in the dissertation as the domain-selection argument |
| Handover primitive (`PRP-001a`) | **Retained and simplified.** B uses the document variant, depth 1 |
| RFC 9162 / RFC 9942 receipt path | **Retained.** Still the technical contribution, now exercised on a simpler chain |
| Witness and checkpoint design | **Retained.** Non-equivocation is now the primary property, not a treatment arm |
| `CNP-001`, parcel scenario | **Historical.** Not edited |
| `ANL-006` | **Re-cut.** Parcel cases become method; employment anchors to be added |
| `ANL-007` | **Re-cut.** The Work Number, payroll APIs and EUDI replace ERP matching and QERDS as the strongest alternatives |
| `PLN-002` | **Re-cut.** The WBS is parcel-shaped and does not carry over |

The automotive work is not wasted: it is the evidence that the domain was searched properly, and it is a stronger domain-selection chapter than the negative result it replaces.

---

## 8. Open questions

| # | Question | Gates |
| --- | --- | --- |
| D1 | Is the primary case the **letter of terms** (content matters) or **status/income** (incumbent's territory)? | The claim |
| D2 | ~~Are there adjudicated employment-document fraud or denial cases?~~ **Answered** — seven cases, four at **V** (`ANL-008`). Authenticity anchored; denial not anchored. | Closed |
| D3 | How large is the cross-border / non-contributing-employer segment? | Viability |
| D4 | ~~Does the module accept a verifier-facing or employee-facing artifact rather than a vivid demo?~~ **Answered** — the module accepts a narrow mechanism with a measured boundary; a negative headline is not acceptable. | Closed |
| D5 | ~~Who is the stakeholder?~~ **Answered** — a plausible scenario is sufficient; no concrete external stakeholder must be recruited. | Closed |
| D6 | Formal supervisor approval of the employment direction. The module constraints are settled; the direction itself still needs a yes. | G1 |
| D7 | Is the EUDI Wallet a baseline or a competitor? | Positioning |
| D8 | **Does the project proceed on one proposition** — offline status at a past decision time — or stop? The module accepts a narrow mechanism with a measured boundary, but after `ANL-009` the boundary is most of the project. | The build |

**D1 is the one to settle before any writing.** If the answer is status/income, the project competes with a 149-million-verification incumbent and loses. If the answer is the letter of terms, the incumbent does not reach it, and the claim is defensible.

---

## 9. One-paragraph summary

The automotive test-evidence direction is dropped, not because the analysis failed but because it succeeded: traced against seven documented scandals, the process flags nothing fully, three partially and three not at all, and in every partial case the detection is carried by an institution outside the mechanism. The project moves to the employment record — the one domain examined with a large, paid, verifier-funded market and no established issuance standard for the document itself. The case is the document variant of the handover primitive: employer issues, employee holds, verifier checks offline. The anchor gap closed with seven documented cases, four at primary or official status, showing a single failure structure: the verifier's only control is to contact the employer, and the contact fails when the issuer is complicit, unreachable, non-existent or foreign. The strongest-alternative test then narrowed the claim: ESSPASS legislates authorised issuance, integrity and non-revocation for social security documents, and Velocity deploys holder-held employment credentials at scale. What remains is one proposition — a verifier can establish offline, from the bundle alone, that a record was valid at a past decision time — plus non-equivocation without a chain, which is the parcel claim that already failed. The decision now is whether one proposition is enough to build. That is D8, and it is the only question in this document that still matters.
