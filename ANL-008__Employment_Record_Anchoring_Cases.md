# ANL-008 — Anchoring Cases for Employment-Record Evidence

| Field | Value |
| --- | --- |
| Document ID | ANL-008 |
| Title | Anchoring cases: forged, insider-issued and unverifiable employment records |
| Version | 0.1.0 |
| Status | Draft — evidence annex to DEC-002; fills the anchor gap |
| Last Updated | 2026-10-23 |
| Related documents | DEC-002; ANL-004; ANL-006 (parcel anchors, historical); ANL-007 (to re-cut) |
| Purpose | Establish, from primary and official sources, that the employment-record failure mode is observed and adjudicated, and derive the design requirement from the observed control failure rather than from assumption |
| Supersedes | DEC-002 §4.1, which states that no adjudicated anchors exist. This document is the result of that search. |

---

## 1. Why this document exists

`ANL-004` established that employment verification is a large, paid market, but its fraud evidence was mostly **S** status: trade surveys, vendor blogs, and secondary reports. `DEC-002` §4.1 named this the first analysis task and the gate on the concept note: the parcel case had anchors and no demand, and the employment case had demand and no anchors.

The search found anchors. They are recent, from four jurisdictions, and they are primary or official — court judgments, tribunal decisions, a regulator's enforcement order, and Department of Justice releases.

**The key result is not that forgery happens.** It is *where the control fails*. In every case the control was the same: contact the employer and ask. The failure is that the contact either reaches someone who is complicit, reaches nobody, or reaches no organisation at all.

---

## 2. The cases

### 2.1 United States v. Osuagwu (N.D. Tex., 2018)

**Status:** Jury conviction. **V** (DOJ release).

**Facts.** Over more than a year, the defendant submitted false bank statements, **false employment letters**, false W-2s and false paystubs indicating that purchasers worked for his own company, Inforation Inc., to obtain mortgage loans from Bank of America, J.P. Morgan Chase and Wells Fargo. Convicted on five counts of bank fraud and one of conspiracy.

**What it anchors.** The employment letter as the instrument of mortgage fraud, and the lender as the relying party. The letter was *invented*, and the "employer" was the fraudster.

### 2.2 OCC Order of Prohibition (2022) — *In re* Amy Lu

**Status:** Regulatory enforcement order recording a guilty plea. **V** (OCC official document).

**Facts.** A mortgage consultant and loan officer pleaded guilty to conspiracy to commit bank and wire fraud. Her signed plea agreement stated that she "(b) created false and fictitious supporting documents, including **verification of employment letters**, gift letters, face-to-face interview narratives, and letters of explanation," and falsified borrowers' income and debt-to-income ratios. She earned approximately $990,847 in commissions.

**What it anchors.** The **insider** case. The person producing the verification-of-employment letter was inside the lending institution. Any control of the form "call and ask" reaches the person who created the document.

### 2.3 Kenyeres v Minister for Immigration [2020] AATA 1050

**Status:** Tribunal decision. **V**.

**Facts.** The applicant supplied an employment contract and letter from a purported sponsor. The Tribunal found the documents **bogus on the basis that they were counterfeit: "The Director of the company had not signed them and never made an offer of employment to the applicant."** The signature of the purported sponsor was forged. The visa was cancelled.

**What it anchors.** The employer **denied issuing** — but the denial is of a forged signature, not of a genuine issuance. This distinction matters for the claim; see §5.

### 2.4 Agbai v Canada (MCI) (Federal Court)

**Status:** Federal Court decision on judicial review. **V**. (Neutral citation to be confirmed from the docket.)

**Facts.** The applicant claimed employment at a Nigerian company, supported by an employment letter and reference letter signed by a relative. Immigration, Refugees and Citizenship Canada requested verification through the **Lagos Special Police Fraud Unit**. The response: the applicant did not work there, and **"no company with a name as TIMOTEX Integrated Services Nigeria found around the vicinity. The letter of employment record and reference letter ... is a forgery."**

**What it anchors.** The **unresolvable issuer** and the **cross-border** case. Verification required routing a request through a foreign police unit. That is the status quo at its most expensive, and it is exactly what a locally verifiable issuer signature removes.

### 2.5 R. v. Khatchatourov, 2014 ONCA 464

**Status:** Court of Appeal decision. **V**.

**Facts.** A mortgage fraud ring used the identities of recent immigrants and **falsified employment letters, income statements and bank statements** in the names of nominal purchasers to obtain financing. Losses fell on the lenders when the properties went to power of sale.

**What it anchors.** Industrial-scale fabrication of the employment letter as part of a document set. The letter is not an isolated artefact; it is one of several mutually supporting forgeries.

### 2.6 Tisha Lee (D.D.C., plea June 2026)

**Status:** Guilty plea. **P** (secondary report of a US Attorney announcement; primary release to obtain).

**Facts.** A contract employee in the Human Capital section of the US Government Publishing Office, with no authority to issue or verify employment offers, used her government email account to send **false offer letters on agency letterhead** to people seeking to rent apartments in Washington DC — at least ten occasions, at salaries of roughly $85,500 to $207,500. **"When apartment management companies contacted Lee directly to verify the false employment claims, she confirmed them."** She was paid by the beneficiaries.

**What it anchors.** The clearest case in the set. The verification control was exercised and it **worked as designed** — the landlord called the employer and reached the insider, who confirmed the lie. The failure is not that the call was not made; it is that the call cannot distinguish an authorised issuer from an employee with a mailbox.

### 2.7 Supporting: United States v. Smalley (D. Colo., 2020)

**Status:** Jury conviction. **P** (secondary report; DOJ release to obtain).

**Facts.** To obtain a $998,000 loan from Navy Federal Credit Union, the defendant falsified a pay stub and **his employment letter** after the lender requested them. After default, he supplied **two further fraudulent employment letters** in 2018 and 2019 during loss mitigation.

**What it anchors.** Repeat use of the same forged instrument across the life of a loan, including after the fraud was already in default.

### 2.8 Supporting: CBP arrest, Champlain (2019)

**Status:** Arrest and charge — **A, not a finding**.

**Facts.** A Canadian citizen applying for NAFTA work authorisation presented an employment letter; CBP **contacted the employer** and determined the letter "misrepresented material facts about her employment and compensation." Prosecution accepted.

**What it anchors.** The consular/border variant, and the fact that the control depends on the employer being reachable and cooperative at the moment of the check.

---

## 3. The common structure

> **The verifier's only control is to contact the employer, and the contact fails whenever the issuer is complicit, unreachable, non-existent, or foreign.**

| Pattern | Cases | Why the control fails | What the design must supply |
| --- | --- | --- | --- |
| **1. Insider-issued** | Lu, Lee | The call reaches the insider, who confirms | A signature from a key bound to an **authorised issuance role**, not to a mailbox |
| **2. Applicant-fabricated** | Osuagwu, Smalley, Khatchatourov | The letter is invented; there is no issuance to confirm | **Issuer signature**; no key, no valid attestation |
| **3. Unresolvable issuer** | Agbai, Kenyeres, Weinstock | Foreign, defunct, non-existent, or uncooperative | **Offline verification** against a published issuer key; no call required |
| **4. Cross-border** | Agbai, Weinstock | Verification routes through a foreign authority | The same, plus key discovery that works across jurisdictions |

Patterns 3 and 4 are the ones the incumbent does not serve. The Work Number covers pattern 2 for contributing employers, and does not cover patterns 1, 3 or 4.

---

## 4. What the design must supply, and what the log adds

The anchors map cleanly onto a narrower design than `DEC-002` proposed:

| Requirement | Supplied by |
| --- | --- |
| Authenticity and attribution to an authorised issuer | Issuer signature over a canonical record; key bound to a role, not an address |
| Recipient binding | Holder identity in the statement; holder's `receipt` pins the version |
| Offline verification without contacting the employer | Signature verification plus a published key; no phone call |
| Status at a past time | Status list plus log inclusion; verify at the decision time, not "now" |
| Non-equivocation across verifiers | Log checkpoints; witnesses |
| Non-revision after issuance | Log inclusion plus the holder's `receipt` |

The first three are **verifiable-credential properties**. The last three are where the log earns its place, and the third of them is the genuinely hard one.

**Say it this way:** the anchors support an authenticity claim; the log's contribution is *status and versioning*, not authenticity. Claiming that the log is what prevents forged letters would be false, and the cases show why: a forged letter has no valid attestation at all, and a signature is enough to reject it.

---

## 5. What the anchors do not establish

1. **They do not show an employer denying a letter it genuinely issued.** *Kenyeres* is a forged signature attributed to a sponsor; the employer's denial is of a forgery, not of its own act. The "the employer cannot later deny what it issued" narrative in `DEC-002` §5.3 is **not anchored by these cases**, and the claim must be rewritten. What is anchored is: *the verifier cannot establish authenticity without contacting the issuer, and the contact fails.*
2. **They do not establish prevalence.** Seven cases across four jurisdictions show the pattern; they do not measure how often it occurs. `ANL-004` §3.3's market figures are the prevalence proxy and remain **S** status.
3. **They do not establish that verifiers would accept a self-hosted log.** No case turns on the admissibility of a transparency-log record.
4. **Two are charges, not findings.** 2.8 is an arrest, and 2.6 is a plea reported second-hand. Label them accordingly.

---

## 6. Evidence register

| # | Source | Claim used | Status |
| --- | --- | --- | --- |
| 1 | *United States v. Osuagwu*, N.D. Tex. (2018), DOJ release | False employment letters and pay documents used to obtain mortgages; jury conviction | **V** |
| 2 | OCC Order of Prohibition (2022), *In re* Amy Lu | Loan officer created false verification-of-employment letters; guilty plea; ~$990k commissions | **V** |
| 3 | *Kenyeres v Minister for Immigration* [2020] AATA 1050 | Sponsor's signature forged; employer never made an offer; documents counterfeit; visa cancelled | **V** |
| 4 | *Agbai v Canada (MCI)*, FC | Employment and reference letters forged; verification required a foreign police unit; company not found | **V** |
| 5 | *R. v. Khatchatourov*, 2014 ONCA 464 | Falsified employment letters in a mortgage fraud ring | **V** |
| 6 | Tisha Lee, D.D.C. (2026), US Attorney announcement via secondary report | Fake agency offer letters; landlord verification calls reached the insider, who confirmed them; guilty plea | **P** |
| 7 | *United States v. Smalley*, D. Colo. (2020), secondary report | Falsified employment letter to a credit union; further forged letters during loss mitigation; conviction | **P** |
| 8 | CBP, Champlain (2019) | Falsified employment letter for NAFTA work authorisation; employer contacted and denied | **A** (charge) |

**Status key.** V = primary or official source; P = published by an interested party or reported second-hand; A = allegation or charge, not a finding.

---

## 7. Consequences for the project

1. **The anchor gate is passed.** Four **V**-status adjudicated or official sources, comparable in number and quality to the parcel case's four.
2. **The claim changes emphasis.** Authenticity and offline verification are what the evidence supports; non-revision and status-at-a-past-time are what the log adds. Write it in that order.
3. **The strongest alternative is now precise.** The Work Number answers pattern 2 for contributing employers at $73.45 and 149M verifications a year. It does not answer patterns 1, 3 or 4, and it does not give the holder a record.
4. **The evaluation has a real comparison.** Replay patterns 1–4 against: the phone call (status quo), a signed letter (L1), a signed letter plus status list, and a logged letter with witnesses. The measure is which pattern each arm flags and what the verifier must trust.
5. **The demo writes itself.** Tisha Lee is the demonstration: a landlord calls, the insider confirms, and the call is indistinguishable from a genuine verification. Then the same scenario with an issuer-signed record the insider cannot produce.

---

## 8. One-paragraph summary

Seven documented cases from the US, Canada, Australia and India show a single failure structure: the verifier's only control over an employment record is to contact the employer, and that contact fails when the issuer is complicit (a loan officer creating verification letters, a government contract employee confirming fake offers to landlords by phone), when the applicant invented the employer, or when the employer is foreign, defunct or non-existent. The anchors support an authenticity claim, not a denial claim: the cases are forgeries and insider issuances, not employers disowning their own acts. The design requirement is therefore an issuer signature bound to an authorised role, verifiable offline without a call, with the log supplying what a signature cannot — status at a past time, version pinning by the holder, and non-equivocation across verifiers. The anchor gap named in DEC-002 is closed; the remaining gap is prevalence, and that is what ANL-004's market evidence is for.
