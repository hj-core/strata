# ANL-005 — Macau Payment-Log Scenario: Feasibility Check

| Field | Value |
| --- | --- |
| Document ID | ANL-005 |
| Title | Feasibility check: a mandated university payment log in Macau |
| Version | 0.1.0 |
| Status | Draft — checked against public sources, 2026-10-22 |
| Related documents | ANL-004; PRP-001; DISCUSSION-SUMMARY-002; BUILD-HANDOVER-002 |
| Purpose | Test two facts that decide whether the proposed scenario is real: who owns the procurement mandate, and whether payment data is emitted by a system of record or by a human act |
| Scenario under test | DSEDJ mandates universities to log every contractual payment; government operates the log; two peer departments witness; CCAC verifies |

---

## 1. The two questions

1. **Who owns public procurement rules in Macau?** The scenario assumed DSEDJ.
2. **Does a university's finance system emit a record per payment, or does a clerk decide what to log?**

Both are answerable, and the first answer is not DSEDJ.

---

## 2. Finding 1 — the mandate holder is the Financial Services Bureau (DSF), not DSEDJ

Macau has just replaced its fragmented procurement regime.

| Fact | Detail | Source | Status |
| --- | --- | --- | --- |
| New statute | **Law 10/2025**, Public Procurement Law, approved July 2025, **in force 1 September 2026** | Chambers Public Procurement 2026 (Macau); MdME | P |
| Implementing regulation | **Administrative Regulation 19/2026**, published 13 July 2026 | MdME, 2026-07-22 | P |
| Central portal | **pcp.gov.mo**, "Página Electrónica da Contratação Pública", launched by DSF on 22 July 2026 | gov.mo news, 2026-07-22 | V |
| Operator of the portal | DSF | gov.mo / DSF | V |
| Who must publish | Contracting authorities must publish procurement documents, clarifications, bidder information, evaluation information and **award decisions** on the centralised website managed by DSF | MdME | P |
| Centralised procurement | DSF prepares centralised plans, consolidates needs, runs procedures on behalf of entities; entities then buy from appointed suppliers | MdME | P |
| DSF statutory role | Coordinates and promotes the public procurement regime; organises centralised procurement; **"support, develop and manage the fiscal, financial and procurement information systems for use by the RAEM"** | DSF entity page (RA 6/2026) | V |
| Thresholds | Public tender mandatory at ≥ **MOP 4.5 million** for goods/services; ≥ **MOP 15 million** for works | Chambers 2026 | P |
| Stated principles | Transparency and traceability; competition; efficiency through standardisation and digitalisation; integrity and good governance | MdME | P |
| Digitalisation | All procurement acts and procedures may be conducted electronically | MdME | P |

**Consequences for the scenario:**

1. **DSEDJ is the wrong mandate holder.** Public finance and procurement belong to DSF. Approaching DSEDJ would misdirect G2.
2. **A transparency platform already exists and is seven weeks old.** pcp.gov.mo publishes awards and procurement documents. The "transparency is missing" claim is no longer available.
3. **DSF already owns the procurement information systems.** So the "system of record" question is partly answered at the centre — but see §3.

This is the same structural situation as OpenCerts in the credential case: the incumbent appeared while the proposal was being written. It does not kill the scenario, but it removes the easy version of it.

---

## 3. Finding 2 — payments are distributed, not centralised

The new regime centralises **award publication**. It does not appear to centralise **payment execution**.

| Fact | Detail | Source | Status |
| --- | --- | --- | --- |
| University supplier registration | The University of Macau runs its own **System of Registered Suppliers** (ADMO-PCT), with its own approval workflow and procurement section | UM user guide, May 2026 | V |
| Payment methods | Local suppliers: autopay (AP). Non-local: telegraphic transfer (TT) or bank draft (BD) | UM user guide | V |
| Implication | UM pays suppliers **directly from its own system**; payment data originates in the university, not in a central treasury ledger | Derived from the above | D |
| Disclosure scope | The transparency obligations described in the English-language summaries cover procurement documents, bidders, evaluation and **award** — not payments or contract variations | MdME; Chambers | P |

**What this means.** The publication layer is central; the payment layer is distributed across autonomous entities. That is precisely the shape where a completeness problem can exist: the centre knows what was awarded, not what was actually paid.

**Unverified:** whether AR 19/2026 requires publication or reporting of payment events and contract variations. The English summaries do not mention it. This must be checked against the Chinese/Portuguese text before the scenario is relied on.

---

## 4. The gap that survives

| Layer | Centralised? | Public? | Where corruption hides |
| --- | --- | --- | --- |
| Procurement procedure | Yes — pcp.gov.mo | Yes | Less exposed now |
| Award decision | Yes — pcp.gov.mo | Yes | Less exposed now |
| **Contract execution / variations** | **No** | **No** | **Cost overruns, additional works** |
| **Payments** | **No** | **No** | **Split payments, duplicate invoices, ghost suppliers** |

The surviving gap is **payment-level completeness across autonomous entities**. That is defensible, and it is narrower than "procurement transparency," which the government has just claimed.

---

## 5. Corrections to the scenario

| Element | As proposed | Corrected |
| --- | --- | --- |
| Mandate holder | DSEDJ | **DSF** (education funding condition can supplement) |
| Log operator | "the government" | **DSF** — it already owns the procurement systems |
| Issuers | Universities | Public entities; public universities first. **Private universities need a subsidy or accreditation condition** |
| Verifier | CCAC | **Commission of Audit** for financial completeness; **CCAC** for corruption. Consider both |
| Witnesses | Two peer departments | One peer department + **Commission of Audit** (statutory weight); do not let the verifier also witness |
| Log visibility | Private, for privacy | **Two-tier**: awards public (as the law requires) + payment-level log restricted to audit bodies |
| Privacy rationale | Government operation secures privacy | Privacy comes from hash-only payloads, a restricted verifier set, and a lawful basis — not from ownership. Government operation **concentrates** metadata |

**The privacy tension.** The new law's direction is *publicity*. A fully private log would be less transparent than the regime it sits inside. The coherent design is two-tier: public award data as required, plus a restricted payment-level log for the audit bodies. That preserves the law's transparency objective and still protects supplier and payment detail.

---

## 6. Risks

| Risk | Severity | Note |
| --- | --- | --- |
| **Redundancy** — the government is already building this area | High | pcp.gov.mo is seven weeks old; the regime is mid-implementation. The project must sit *above* it, not beside it |
| **Applicability** — are public universities "public services" under Law 10/2025? | High | Unverified. UM is an autonomous public legal person; the scope needs a legal reading |
| **Perimeter migration** | High | Universities have foundations, affiliated companies and research institutes. Payments move outside the legal perimeter and the log cannot see them |
| **Emission is discretionary** | High | If a clerk decides what to log, the design is theatre. **Unresolved** — see §7 |
| **Oracle** | Medium | Kickbacks and over-invoicing are invisible; the log proves attribution and completeness, not truth |
| **Witness independence** | Medium | All roles sit under one sovereign; the claim must be anti-backdating and intra-government separation, not "the state cannot rewrite" |
| **University reporting is annual, not monthly** | High — **confirmed** | UM's statute is a "regime especial em relação ao regime financeiro dos serviços e fundos autónomos", and the tutela approves its budget, accounts and report annually. The monthly anchor in §7.3 does not apply to it. Use an autonomous fund as the reference instance (§7.5). |

---

## 7. The test result — the gap is real

**Question:** does DSF already hold transaction-level payment data, making the log redundant?

**Answer: no — not for entities with administrative or financial autonomy.** The centre holds the **envelope and the aggregate**, not the individual payment.

### 7.1 Evidence

The financial regime is **Regulation RA 6/2006** (Regime de administração financeira pública), as republished in full by Order 426/2009. Its Article 1 applies to **the whole public administrative sector, explicitly including services and bodies with administrative or financial autonomy**.

| Rule | Provision | Consequence |
| --- | --- | --- |
| Payment authorisation and issuance | "A autorização e a emissão dos meios de pagamento competem, **consoante o regime de autonomia**, à DSF ou aos órgãos e dirigentes dos serviços e organismos" | Split by regime: DSF issues for integrated services; **the entity itself issues for autonomous bodies** |
| Monthly reporting | Bodies with administrative autonomy **must send their accounts to DSF monthly**, on template maps defined by DSF (Art. 40; restated in RA 28/2009) | DSF receives **aggregates on a fixed template**, not transactions |
| Template content | "Mapa comparativo das despesas orçamentadas, discriminadas de acordo com a **classificação** definida pelo regime de contabilidade pública" | Reporting is by **economic classification**, not by payee or contract |
| Funding mechanism | Funds for administrative-autonomy bodies are released **by advance through treasury operations**; the body then disburses | The Treasury sees the **advance**, not each payment |
| Accounting basis | Cash accounting for integrated and administrative-autonomy bodies; accrual for financial-autonomy bodies (AMCM, Postal Savings, etc.) | No transaction journal reaches the centre |
| General accounts | The **Conta Geral** reports by economic and organic classification, with separate management accounts per autonomous body | Published and audited data is **aggregate** |
| Audit recipient | DSF prepares the Conta Geral and submits it to the **Comissariado da Auditoria** | The Commission of Audit is the **existing statutory recipient** — not a hypothetical verifier |

### 7.2 What this means

For a university — autonomous, with its own supplier register and its own payment instruments (ANL-005 §3) — the transaction-level payment record lives **in the university**, not in DSF. DSF can verify that total spending on goods and services matches the budget line. It cannot verify **to whom** the payments went, in what order, or whether a contract was split.

That is the gap, and it is precisely the completeness gap the abstraction predicted: **the centre has the aggregate set, not the transaction set.**

### 7.3 A stronger design falls out of this

The existing monthly statutory return is not just a justification — it is an **anchor**.

Because the university must report an audited monthly aggregate independently of the log, the verifier can reconcile the two:

```
sum(logged contractual payments for period P)  ==  reported aggregate for P
```

A mismatch is evidence in either direction:

- **log sum < reported aggregate** → payments are missing from the log;
- **log sum > reported aggregate** → the return understates.

This fixes the weakest point in the earlier design: the issuer-signed sequence is self-reported, so a corrupt official can maintain a fake continuous counter. **The statutory aggregate is reported through a separate channel and audited by a separate body, so it cannot be silently reconciled away.** The sequence catches gaps inside the log; the aggregate catches gaps between the log and the accounts.

That mutual check is a genuine contribution, and it is specific to this institutional setting.

**Design note:** the aggregate is by economic classification and includes salaries and transfers, so the reconciliation must be scoped to the contractual-payment subset. That is a specification task, not a flaw.

### 7.4 What is still unresolved

**Emission.** The test answers whether the centre holds the data; it does not answer whether a university's payment system can emit a signed statement per payment, or whether a clerk chooses what to log. That remains the one fact requiring an institutional contact.

**Legal classification.** Whether a public university falls under administrative autonomy, financial autonomy, or the special-organism regime under RA 6/2006 determines the exact reporting duty and must be confirmed against the consolidated text, not inferred. It has now been checked — see §7.5.

### 7.5 Classification check — the university is the wrong reference instance

| Entity | Legal nature | Reporting | Anchor quality |
| --- | --- | --- | --- |
| **University of Macau** | Public legal person with academic, disciplinary, administrative, financial and patrimonial autonomy (Lei 1/2006; Order 14/2006, Arts. 1 and 8) | Its statute is expressly a **"regime especial em relação ao regime financeiro dos serviços e fundos autónomos"**; the tutela approves its budget, accounts and annual report | **Annual** — coarse |
| **Public HEIs generally** | Public legal persons with scientific, pedagogical, administrative and financial autonomy (Lei 10/2017 Art. 6(1)) | Art. 9(2) expressly contemplates public HEIs that are **special organisms** under Lei 15/2017 Art. 4(2) | Varies by institution |
| **Education Fund (FES)** | Public legal person with **administrative, financial and patrimonial autonomy** (Lei 10/2017 Art. 33; RA 17/2022, replacing RA 16/2018) | Ordinary autonomous-fund regime under RA 6/2006 → **monthly** accounts to DSF | **Monthly** — usable |

**Consequence.** UM's financial regime is special and its reporting is annual. The monthly reconciliation in §7.3 does **not** apply to it. The university remains a valid issuer in principle, but it is a poor **reference instance**.

The clean instance is an **autonomous fund**: unambiguously inside RA 6/2006, monthly reporting, disburses to third parties, audited. The Education Fund is the clearest legal fit. The **Science and Technology Development Fund** may be a better fit on volume, since research funding produces many more discrete awards than institutional block grants — that has not been checked.

**Still open:** whether UM is formally an "organismo especial" under Lei 15/2017 Art. 4(2) is not conclusively established. What *is* established is that it is not an ordinary administrative-autonomy body.

**Design consequence.** With a fund as issuer, the reconciliation window is monthly and the anchor is strong. With a university as issuer, the window is annual and the anchor is weak — enough to catch a systematic omission over a year, not enough to catch a single split contract. That is a material difference in what the system can claim.

---

## 8. Verdict

**The scenario is viable but must be restated.** Its original form — DSEDJ mandates, government logs, CCAC verifies — is misdirected on the mandate holder and pre-empted on transparency.

The restated version:

> DSF, which already operates the central procurement systems, requires autonomous public entities to emit a signed statement for **every payment and contract variation** into a witnessed log. Awards remain public as the law requires; payments remain restricted to the audit bodies. The Commission of Audit verifies completeness for a period offline, without the entity's cooperation, and cannot be shown a backdated record.

That is a coherent, specific, timely project — and it is *more* defensible than the original because the gap it addresses is the one the new law leaves open.

**The feasibility test has now been run and passed** for the general frame, and the classification check has changed the reference instance. DSF does not hold transaction-level payment data for autonomous bodies; it holds monthly statutory aggregates (§7). But UM reports annually under a special regime, so the university is the wrong reference instance (§7.5). **The clean instance is an autonomous fund**, preferably one with high disbursement volume.

One thing remains open, and it is decisive: whether payment records can be **emitted** by the system of record rather than chosen by a person. That needs an institutional contact, and it is now the only fact standing between the frame and a defensible primary case.

It is no longer a proof-of-delivery project. It is a **regulatory completeness** project with a measurable baseline: what the Commission of Audit can establish from aggregates today, versus from a transaction log reconciled against the same aggregates.

---

## 9. Evidence register

| # | Source | Claim | Status |
| --- | --- | --- | --- |
| 1 | gov.mo news, 2026-07-22 | DSF launched pcp.gov.mo for procurement publication and consultation | V |
| 2 | DSF entity page, RA 6/2026 | DSF coordinates centralised procurement and develops/manages RAEM procurement information systems | V |
| 3 | gov.mo PS-1361 | DSF runs centralised tenders for daily consumables; supplier registration with DSF | V |
| 4 | MdME, 2026-07-22 | AR 19/2026; centralised procurement; disclosure on DSF-managed website; digitalisation; four principles | P |
| 5 | Chambers Public Procurement 2026 (Macau) | Law 10/2025 in force 2026-09-01; MOP4.5M/MOP15M thresholds; five procedures | P |
| 6 | UM System of Registered Suppliers user guide, May 2026 | UM operates its own supplier registration and payment methods (AP/TT/BD) | V |
| 7 | MdME / Chambers summaries | Disclosure obligations cover award stage; payment stage not mentioned | P |
| 8 | RA 6/2006, republished by Order 426/2009 | Applies to the whole public administrative sector including administrative and financial autonomy; payment authorisation and instrument issuance split by autonomy regime | V |
| 9 | RA 6/2006 Art. 40; RA 28/2009 Art. 40(4) | Administrative-autonomy bodies send monthly accounts to DSF on templates defined by DSF | V |
| 10 | Order 388/2006 | Defines the payment instruments for public expenditure and treasury fund movements | V |
| 11 | Order 1/2007 | Instructions for processing expenditure on acquisition of goods and services | V |
| 12 | Order 80/2010 | Procedure for releasing budget credits, recording expenditure, and returning surplus for administrative-autonomy bodies | V |
| 13 | Conta Geral / Relatório sobre a execução do orçamento (DSF) | Reports by economic and organic classification, with separate management accounts per autonomous body; DSF submits the Conta Geral to the Comissariado da Auditoria | V |
| 14 | Lei 1/2006; Order 14/2006 (UM Statutes) | UM is a public legal person with academic, disciplinary, administrative, financial and patrimonial autonomy; statute is a special regime relative to autonomous services and funds; tutela approves annual budget, accounts and report | V |
| 15 | Lei 10/2017 Arts. 6, 9, 33 | Public HEIs are public legal persons with administrative and financial autonomy; Art. 9(2) contemplates HEIs that are special organisms under Lei 15/2017 Art. 4(2); the education autonomous fund is a public legal person with administrative, financial and patrimonial autonomy | V |
| 16 | RA 17/2022 (Education Fund), replacing RA 16/2018 | The FES is a public legal person with administrative, financial and patrimonial autonomy, operating alongside GAES; it grants financing, institutional support, equipment support and research project support | V |

---

## 10. Caveats

- The English-language summaries of Law 10/2025 and AR 19/2026 are secondary. The Chinese and Portuguese texts are authoritative and have not been read in full here.
- The financial-regime findings in §7 come from the **Portuguese** consolidated texts (RA 6/2006 as republished by Order 426/2009, RA 28/2009, and the associated orders). They should be checked against the current consolidated version before being cited in the dissertation.
- Whether a public university is an administrative-autonomy body, a financial-autonomy body, or a special organism under RA 6/2006 is **not confirmed**. It determines the exact monthly reporting duty and therefore the reconciliation design in §7.3. This is a legal question requiring the consolidated text and, ideally, a practitioner.
- The claim that DSF does not receive transaction-level data is an inference from the reporting rules, not a direct statement that no such data exists. DSF may collect additional data informally, through audit, or through the new procurement systems. Confirm before relying on it.
- The monthly accounting templates are by economic classification and include salaries and transfers; the reconciliation in §7.3 must be scoped to the contractual-payment subset.
- The regime is mid-implementation: Law 10/2025 took effect on 2026-09-01 and AR 19/2026 seven weeks earlier. Rules and systems may change during the project.
