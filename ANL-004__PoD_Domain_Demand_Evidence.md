# ANL-004 — Domain Demand Evidence and Incumbent Map

| Field | Value |
| --- | --- |
| Document ID | ANL-004 |
| Title | Domain demand evidence and incumbent map for document handover |
| Version | 0.1.0 |
| Status | Draft — evidence annex to PRP-001 v0.2.0; corrects the demand basis |
| Last Updated | 2026-10-22 |
| Related documents | PRP-001 §3, §8, §11; ANL-002; ANL-003; DISCUSSION-SUMMARY-002; BUILD-HANDOVER-002 |
| Purpose | Replace ecosystem evidence ("who runs SCITT?") with market evidence ("who pays for verification today, how much, and what does it cost them?") and map the incumbents the proposal must beat |
| Supersedes | Nothing. Supplements PRP-001 §11 and DISCUSSION-SUMMARY-002 §6.2; both should be corrected against §5 below. |

---

## 1. Why this document exists

The v0.2.0 demand table (DISCUSSION-SUMMARY-002 §6.2, repeated in BUILD-HANDOVER-002 §8) measures the maturity of the **SCITT ecosystem**: Microsoft Signing Transparency in production, DataTrails in preview, a set of drafts, and "no public PoD / university / employment application found."

That is the wrong denominator for gate G2. G2 asks whether a real workflow owner will engage. That question is answered by market evidence — who verifies documents today, what they pay, what breaks — not by counting SCITT adopters. A domain can have intense demand and no SCITT deployment; that is in fact the normal case, and it is the opportunity.

This document therefore does three things:

1. states the demand evidence for the two document cases (credentials and employment) and the parcel case, with sources and status labels (§3);
2. maps the incumbents and assigns each an assurance level from PRP-001 §5.1 (§4);
3. draws the consequences for the proposal, including one finding that contradicts the current text (§5).

**One finding contradicts the current documents.** OpenCerts — a national, production, university-credential deployment — exists, has run since 2018, and anchors a Merkle root on a public chain. "No public university deployment found" is not correct. See §5.1. This is a strongest-alternative result, not a footnote.

All figures below are **company-reported, institution-published, or secondary**, and are labelled. None is a measurement by this project. Prices are public list prices or published fee schedules as of 2026-10-22, not quotes.

---

## 2. Method

- Public sources only: company and institution pages, published fee schedules, standards documents, and one secondary trade source.
- Each claim carries a status: **V** verified against a primary/official source; **P** published by a party with an interest (vendor or institution claim); **S** secondary or unverified.
- No weighted scores. The comparison is qualitative, per the house convention.
- Where sources disagree, the disagreement is stated rather than resolved (§3.4).

---

## 3. Demand evidence

### 3.1 The employment case: verification is a large, paid, intermediated market

The strongest demand evidence in either document case is **employment and income verification**. It is not a hypothesis; it is a mature market with published volumes and prices.

| Measure | Value | Source | Status |
| --- | --- | --- | --- |
| Verifications processed by The Work Number (Equifax), 2024 | **149 million** | Equifax newsroom, 2025-04-29 | P |
| Average requests per day | **~400,000** | Equifax newsroom, 2025-04-29 | P |
| People served outside business hours, 2024 | **52 million** | Equifax newsroom, 2025-04-29 | P |
| Contributing employers | **5.06 million** | theworknumber.com | P |
| Employee records accessible to credentialed verifiers | **839 million+** | theworknumber.com | P |
| Entry price for a report | **from $73.45** | theworknumber.com/pricing | P |
| Who pays | **The verifier**; "no cost to employers or your employees" | ADP/Equifax reference guide; theworknumber.com FAQ | P |

Two structural facts follow, and they matter more than the volume:

1. **The verifier pays.** Employers contribute data at no charge; employees pay nothing; lenders, landlords, and government agencies pay per verification. This is a **demand-side-funded** model, and it inverts the assumption that the institution is the buyer.
2. **The market exists because the document is untrustworthy.** The Work Number's own pitch is that manual verification means "stacks of printouts, pay stubs and games of phone tag," and that verifiers pay "for the speed and convenience ... over traditional methods of manually reaching out to employers." The product is a substitute for trusting a document the applicant supplies.

That second point is precisely the proposal's problem statement, already monetised at scale by an intermediary.

### 3.2 The credential case: verification is also paid, but the price is low

| Measure | Value | Source | Status |
| --- | --- | --- | --- |
| Third-party degree verification fee, one US institution | **$17.95** per verification | Wright State University published fee table | P |
| Third-party enrollment verification fee, same institution | **$2.50** | Wright State University published fee table | P |
| Credentials exchanged via Parchment | **165 million+** | parchment.com | P |
| Parchment higher-education institutions | **6,200+** | Instructure/Parchment | P |
| My eQuals academic records issued | **10 million+** | myequals.org | P |
| My eQuals learner accounts | **3 million+** | myequals.org | P |
| My eQuals coverage | **100% of public universities in Australia and New Zealand; 100+ providers** | myequals.org | P |
| Countries accepting My eQuals records | **135** | myequals.org | P |

See §3.4 on a pricing discrepancy at the National Student Clearinghouse.

### 3.3 The fraud case: the document itself is the attack surface

The demand for verification is driven by document forgery, and the forgery is now cheap.

| Claim | Value | Source | Status |
| --- | --- | --- | --- |
| Housing providers reporting fraud | **93%** | NMHC survey, cited by a trade source | S |
| Reported US real-estate fraud losses, 2025 | **$275 million+** | FBI IC3, cited by a trade source | S |
| Growth in AI-generated document fraud, 2025 | **~500%** | vendor blog | S |
| One diploma-mill scheme's revenue | **$140 million** | US DOJ, SDNY, 2016 | V |
| Estimated worldwide fake-degree sales | **$300 million/year or more** | California Assembly committee report, 2012 | S |
| Fake PhDs bought per year vs earned in the US | **>50,000 vs 40,000–45,000** | same report | S |

The qualitative claim is better supported than the numbers: an employer letter is described in trade guidance as "the easiest document to fabricate — a logo lifted from the company website and a fake HR contact whose number routes to a co-conspirator," and the recommended control is to **look the employer up independently and call the main line**. That control destroys the portability and offline-verifiability that a credential is supposed to provide. It is a phone call to a stranger, and it is the status quo.

The older figures (2012, 2016) are included only to show the problem is long-standing; they should not be presented as current.

### 3.4 The pricing discrepancy, stated rather than resolved

The National Student Clearinghouse's own site describes its education verification services as "at no cost to you," while at least one member institution publishes a **$17.95** third-party degree-verification fee and a **$2.50** enrollment-verification fee. The likely explanation is that the fee falls on the *requesting* third party while being free to the *institution*, but the sources do not say so explicitly.

**Action:** before any cost comparison uses a per-verification price, confirm the payer and the direction of the fee with a named institution or with the Clearinghouse. Do not present $17.95 as "the price of verification" without that confirmation.

---

## 4. Incumbent map

Assurance levels are those of PRP-001 §5.1, used as an analytical device, not a taxonomy. "L3-equivalent" means the non-equivocation property is obtained by a different mechanism.

| Incumbent | Domain | Mechanism | Level | Selective disclosure | Offline verify | Verifier pays |
| --- | --- | --- | --- | --- | --- | --- |
| **Paper signed copies** | Both | Wet ink, independent copies | — (baseline) | n/a | Yes | No |
| **The Work Number (Equifax)** | Employment, income | Centralised database, FCRA-governed, credentialed verifiers | L1 | No — report-level | No | **Yes** |
| **Truework, Argyle, Pinwheel** | Employment, income | Payroll-connect APIs, consumer-permissioned | L1 | Partial (report fields) | No | Yes |
| **National Student Clearinghouse** | Enrollment, degrees | Centralised registry, institution-supplied | L1 | No | No | Yes (see §3.4) |
| **Parchment / Digitary** | Transcripts, diplomas | Exchange network, signed PDFs, learner-held profile | L1 | No | Partial (signed PDF) | Institution / learner |
| **My eQuals** | AU/NZ tertiary credentials | Signed PDFs, issuer portal, learner sharing links | **L1** | No — whole document | Partial (signed PDF) | Institution |
| **OpenCerts / OpenAttestation / TrustVC** | SG academic certificates | Signed document + Merkle root in a document store on Ethereum; DNS-TXT issuer identity; revocation | **L1 + chain anchor** (L3/L5-equivalent non-equivocation) | **Yes** in v3 (`ecdsa-sd-2023`, `bbs-2023`) | Yes | n/a |
| **EBSI** | EU diplomas, micro-credentials | W3C VCs, wallets, permissioned blockchain trust layer | L4-ish | VC-dependent | Wallet-dependent | Public / project |
| **Proposed L3** | Both | RFC 9162 log + 2 witnesses + TSA | L3 | **Not implemented** | Yes | Open |

**The pattern.** Every deployed incumbent is L1 plus an operator, except the two blockchain-backed programmes (OpenCerts, EBSI), which get append-only integrity from a chain rather than from witnesses. **No deployed system in this list uses independent witnesses.** That is the actual white space, and it is narrower than "document handover" — it is "non-equivocation without a chain."

### 4.1 What the incumbents already do that the proposal does not

| Property | Incumbent | Proposed L3 | Consequence |
| --- | --- | --- | --- |
| Selective disclosure | OpenCerts v3, EBSI | Not implemented | The proposal is behind on the privacy property it names as essential |
| National/regional coverage | My eQuals 100% of AU/NZ public universities | None | Adoption is the barrier, not the technology |
| Established legal acceptance | My eQuals "legally valid"; eIDAS-aligned signatures | Evidence only | Legal position is weaker |
| Revocation | OpenCerts, VC status lists | Minimal signed status list (C3) | Standards already exist (see §5.4) |

---

## 5. What this does to the proposal

### 5.1 The "no public deployment" claim is false for the credential case

`DISCUSSION-SUMMARY-002` §6.2 and `BUILD-HANDOVER-002` §8 state that no public PoD, university, or employment application was found. For **employment** that may still hold. For **university credentials** it does not:

- **OpenCerts** is a Singapore Smart Nation programme, live since 2018, listing **17 institutions** on its verifier portal in 2026 (all five polytechnics, ITE, NIE, NUS, NTU, SMU, SIT, SUSS, SUTD, NAFA, LASALLE, SEAB, SSG).
- It seals each certificate, stores a **Merkle root in a document store contract on Ethereum**, binds issuer identity through **DNS-TXT**, and supports **revocation**.
- Version 3 migrates to **TrustVC** (W3C VC Data Model 2.0) with **selective disclosure** (`ecdsa-sd-2023`, `bbs-2023`) and `did:web` issuers; institutions must migrate by **31 March 2027**.

Consequences, in order of severity:

1. **The strongest-alternative test fails for transcripts.** A national deployment already delivers integrity, issuer binding, revocation, and selective disclosure, with a chain supplying the append-only property. The proposal cannot claim this use case as unserved.
2. **The gap is the chain, not the credential.** OpenCerts buys non-equivocation from Ethereum. The defensible question becomes: *can independent witnesses deliver the same non-equivocation without a public chain, and at what cost?* That is a sharper thesis than "is a log better than paper," and it is already latent in PRP-001 Appendix E (L3 vs L5).
3. **The demand-table correction is mandatory before G1/G2.** Presenting "no deployment found" to a supervisor who knows OpenCerts is a credibility failure at the first meeting.
4. **Selective disclosure moves from optional to table stakes.** The incumbent has it. A slice that logs a single document hash is behind the state of the art in the very domain it claims.

### 5.2 The transcript case and the employment case are not equally defensible

| | Transcript / diploma | Employment letter / proof of employment |
| --- | --- | --- |
| Incumbent | My eQuals, Parchment, NSC, OpenCerts | The Work Number, payroll APIs, phone calls |
| Incumbent level | L1 and chain-anchored | L1, centralised, verifier-paid |
| Issuance standard | Established (signed PDF, VC) | **None** — a letterhead and a phone number |
| Forgery control today | Platform + registry | Call the main line and hope |
| Verifier reachability | Via institution or platform | Direct — the verifier is the buyer |
| Cross-border pain | Moderate | **High** (mobile workers, defunct employers) |
| Privacy need | Grades | **Salary, employment status** |
| Regulatory hazard | FERPA/GDPR | **FCRA** (US consumer reporting) |

**The employment case is stronger on every axis that matters to this project**: no established issuance standard, a documented forgery problem, a verifier who already pays, a clear three-party model (employer issues, employee holds, landlord/lender/next employer verifies), and a privacy requirement that makes selective disclosure load-bearing rather than decorative.

The transcript case is stronger on **institutional legitimacy** (it is visibly a university problem) and **demo legibility**. It is weaker on novelty, because the incumbents are mature.

### 5.3 Recommendation: narrow the primary case to the employment document

Concretely:

- **Primary case:** an employer's HR function issues a proof-of-employment / employment-letter statement to an employee; a landlord, lender, visa authority, or next employer verifies it offline.
- **Second issuer:** the employee, pinning the version issued (the version-pinning role agreed for the document case). This is the receipt statement's only job, and here it has real work to do: the employee holds proof that *this* letter was issued, so a later correction or revocation cannot retroactively redefine it.
- **Demo:** SHP-42 parcel, unchanged. The mapping is cleaner than for transcripts, because the carrier/recipient pair already models issuer/holder.
- **Transcripts and diplomas:** related work and a comparison arm, not the primary case. Cite My eQuals and OpenCerts explicitly.
- **Scope guard:** the log is evidence of **issuance**, not a consumer report. The verifier performs its own decisioning. No scoring, no eligibility output, no FCRA-covered consumer report is produced. State this in §3.5 and §12.

If the user prefers to keep transcripts primary for institutional reasons, the honest consequence is that the contribution must be positioned **against OpenCerts and My eQuals**, and the novelty claim shrinks to "witnesses instead of a chain" plus "EU-compatible, no public ledger." That is still defensible; it is just a different, narrower paper.

### 5.4 Offline revocation is partly commoditised

C3 ("offline revocation") is listed as a contribution. Two standards already define the status-list mechanism:

- **W3C Bitstring Status List v1.0**, W3C Recommendation, 2025-05-15 — bitstring status lists for verifiable credentials, 131,072 entries in 16 KB, GZIP-compressible.
- **IETF Token Status List** (`draft-ietf-oauth-status-list`, at `-21` as of this review) — the JOSE/COSE equivalent for JWT, SD-JWT VC, CWT, and ISO mdoc.

Do not build a status list from scratch; adopt one. What remains genuinely hard — and therefore what C3 should claim — is **status at a past time, verified offline**: was this credential unrevoked at the TSA time, using only evidence in the bundle? That is the same indeterminacy problem PRP-001 §6.2a Phase 10 already identifies, and it is a real gap. The contribution should be stated as *time-relative offline status*, not *revocation*.

---

## 6. The who-pays matrix

This is the adoption-cost table PRP-001 §9 and PLN-002 do not yet have. It separates who bears which cost, because the infrastructure cost and the adoption cost fall on different parties.

| Cost | Employer / registry | Employee / student | Verifier (landlord, lender, employer) | Log operator | Witnesses | TSA / CA |
| --- | --- | --- | --- | --- | --- | --- |
| Issuance integration (SIS/HR) | **High** | None | None | None | None | None |
| Identity onboarding / credentials | Medium | Low | Medium | None | None | Medium |
| Per-issuance marginal cost | Low | None | None | Low | Low | Low |
| Per-verification price | None in the incumbent model | None | **Paid today** | None | None | None |
| Liability / compliance (FCRA, GDPR, eIDAS) | Medium | None | **High** | Low | Low | Medium |
| Witness operation and audit | None | None | None | None | **High** | None |
| Change management / training | **High** | Low | Medium | Low | Low | None |

**Reading.** The incumbent market charges the verifier and gives the issuer a free service. That is why adoption works: the party with the budget (lender, landlord, screener) pays, and the party that must change its workflow (HR) pays nothing. **A proposal that asks HR to run integration work and asks the verifier to configure five trust anchors has inverted the incumbent's incentive.** Any adoption story must say who absorbs the integration cost, and the honest answer is that it is the issuer — which is exactly the party that gains least in the current model.

This is the strongest argument for the **verification-vendor-as-stakeholder** route: a vendor already pays per verification, already absorbs integration cost, and already sells to the verifier. It is also the strongest argument against approaching a university registry first, which would be asked to pay for a capability its own portal and the national platform already provide.

---

## 7. Implications for the gates

| Gate | Effect of this evidence |
| --- | --- |
| **G1** | Strengthened, but reframed. OpenCerts is a working blockchain credential system; the supervisor's "blockchain" requirement is satisfiable by an existing product. The project's question becomes "witnesses instead of a chain," which is a *better* G1 conversation than terminology. |
| **G2** | Redirected. The reachable stakeholder is a **verification vendor or screening firm**, not a registrar. See §8. Split G2 into G2a (validation) and G2b (commitment). |
| **G3** | Unchanged. |
| **G4** | Unchanged, but the comparison arm should now be OpenCerts/My eQuals, not paper. |
| **G6** | Unchanged, but the FCRA scoping guard in §5.3 is a new constraint. |

**PRP-001 §7.2 and §11 must be revised** to cite My eQuals, OpenCerts/TrustVC, EBSI, The Work Number, and the status-list standards. As written, the related-work table omits the closest deployed systems, which is the same failure mode as the previous proposal.

---

## 8. G2a stakeholder brief (one page, extractable)

**Purpose.** Test demand without asking anyone to commit to a build.

**Who to approach, in order of reachability:**

1. A **verification vendor or screening firm** (employment/income verification, background screening) — pays per verification, already bears integration cost, sells to the verifier.
2. A **payroll provider or HR platform** — controls issuance for many employers; can issue at the source.
3. A **landlord, letting agent, or lender** — the relying party that pays and currently makes the phone call.
4. A **university registry or national credential body** — only if the transcript case is retained; expect the answer "we already have My eQuals/Parchment."

**Questions (ten, in this order):**

1. How do you verify an employment letter or proof of income today, step by step?
2. What proportion of documents you receive are forged or altered, and how do you find out?
3. What does one verification cost you, all-in, including staff time?
4. How long does it take, and how often does it fail or time out?
5. Have you had a case where the employer denied issuing a document you hold?
6. Have you had a case where two versions of the same document existed?
7. Do you need to verify offline, or is connectivity always available?
8. Which fields do you actually need — and which do you receive that you would rather not hold?
9. What would make you refuse to accept a self-hosted, non-QTSP log as evidence?
10. Who else would have to agree before you changed this process?

**What would count as validation:** a named person who confirms (a) the problem in questions 2, 5, or 6 occurs, (b) they pay for verification today, and (c) they would take a follow-up call. Nothing more.

**What would count as falsification:** "we call the employer and it works"; "forgery is not a material problem"; "we would only accept a qualified trust service"; or "our platform already covers this."

**Do not ask** for a pilot, a letter of support, or data access at this stage. Those are G2b.

---

## 9. Evidence register

| # | Source | Claim used | Status |
| --- | --- | --- | --- |
| 1 | Equifax newsroom, 2025-04-29 | 149M verifications 2024; ~400k/day; 52M after-hours | P |
| 2 | theworknumber.com/pricing, 2026 | From $73.45; verifiers pay; no cost to employers/employees | P |
| 3 | ADP/Equifax Employment Verification Reference Guide, 2026 | Verifier pays transactional fee; no cost to employer/employee | P |
| 4 | theworknumber.com | 5.06M employers; 839M+ records | P |
| 5 | Wright State University fee table | $17.95 degree verify; $2.50 enrollment verify | P |
| 6 | studentclearinghouse.org | "at no cost to you" — conflicts with #5 | P |
| 7 | parchment.com / instructure.com | 165M+ credentials; 6.2K+ institutions; 13M+ learners | P |
| 8 | myequals.org | 10M+ records; 3M+ accounts; 100% AU/NZ public universities; 135 countries | P |
| 9 | opencerts.io and docs.opencerts.io | 17 institutions; Merkle root in document store; DNS-TXT; revocation; v3/TrustVC migration by 2027-03-31; `ecdsa-sd-2023`/`bbs-2023` | V |
| 10 | GovTech Singapore, 2018-08-27 | OpenCerts launched on Ethereum; tamper-resistant certificates | V |
| 11 | Open-Attestation GitHub | Merkle root published to a document store contract; signed root | V |
| 12 | ebsi.eu; ec.europa.eu EBSI pages | VC framework; diploma use case; 350+ organisations, 30+ countries; 2021 multi-university pilot | V |
| 13 | W3C Bitstring Status List v1.0, REC 2025-05-15 | Status list mechanism; 131,072 entries / 16 KB | V |
| 14 | draft-ietf-oauth-status-list-21 | Token Status List for JOSE/COSE | V |
| 15 | US DOJ SDNY, 2016-12-21 | $140M diploma-mill scheme | V |
| 16 | California Assembly committee report, 2012 | Fake-degree sales estimates | S |
| 17 | Trade sources citing NMHC / FBI IC3 / vendor data | 93% fraud incidence; $275M losses; ~500% AI fraud growth | S |
| 18 | Trade guidance on rental fraud | Employer letter "easiest document to fabricate"; call main line | S |

---

## 10. Caveats

- Every volume and price is **company- or institution-reported**, not audited, and not measured by this project.
- The market evidence establishes that **verification is paid for**; it does not establish that a witnessed log is the right substitute. That remains the hypothesis under test.
- OpenCerts' assurance depends on Ethereum's properties. Whether that is "L5-equivalent" for this purpose is an analytical judgement, not a measurement; state it as such.
- The fraud figures in §3.3 are the weakest evidence in this document, are mostly secondary, and should not be quoted as current without re-verification.
- The FCRA scoping guard in §5.3 is a **legal hypothesis**, not legal advice; confirm with the module's ethics/legal guidance before framing the employment case.
- Absence of evidence is not evidence of absence: other national deployments may exist that this review did not find.
