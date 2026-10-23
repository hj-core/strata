# ANL-003 — Handover Status-Quo Comparison: Paper, Portal, e-CMR/eBL, and the Proposed Log

| Field | Value |
| --- | --- |
| Document ID | ANL-003 |
| Title | Proof of delivery and document handover: status-quo comparison and when to use which |
| Version | 0.1.0 |
| Status | Draft — comparison annex to PRP-001 |
| Last Updated | 2026-10-22 |
| Related documents | PRP-001 §3.3, §7.2, Appendix E; ANL-002; eIDAS; e-CMR Additional Protocol; MLETR |
| Purpose | Answer "why do we need this?" against existing practice, without a strawman |

---

## 1. The question

> What does the proposed witnessed transparency log add over the existing ways of proving a handover — a parcel delivery or a document transfer — and when is it actually worth using?

The primary motivation is **document handover in universities and employers** — transcripts, diplomas, employment letters, and contracts — where the institution is both the issuer and the record keeper. The parcel delivery is the worked demo instance. The same protocol serves both.

The honest baseline is **two signed copies**, one held by the sender and one by the recipient (for a parcel, the courier and the recipient; for a document, the registry or HR function and the student or employee). It is cheap, legally familiar, and sufficient for many handovers. The proposal is justified only where that model fails: when the copies can diverge, a record can be withheld, a platform can equivocate, or a third party must verify without trusting the holder.

## 2. The comparison

Qualitative contrast; no weighted score. "Portal" means a carrier, university, or HR platform holding the record; "e-CMR/eBL" means the legal electronic consignment note or bill of lading.

| Criterion | Paper signed copies | Scanned PDF / portal | e-CMR / eBL | Proposed L3 |
| --- | --- | --- | --- | --- |
| **Who holds the evidence** | Each party | Platform operator | Platform or registry | Both parties, plus the log and witnesses |
| **Tamper after signing** | Forensic detection; not cheap or immediate | Platform-dependent | Platform-dependent | Cryptographic; immediate |
| **Withholding** | Hard to detect | Operator can omit a record | Operator can omit a record | Receipt plus issuer-signed sequence |
| **Equivocation** | Copies can diverge; no shared head | Operator can show different records | Operator or registry can show different records | Witnesses detect a split view |
| **Trusted time** | Handwritten; disputed | Server clock | Server clock or TSA | RFC 3161 token |
| **Third-party offline verification** | Manual inspection | Needs the platform | Needs the platform or registry | Bundle plus five anchors |
| **Legal recognition** | Established | Varies | Strong under e-CMR / MLETR | Evidence; not a qualified trust service |
| **Cost / complexity** | Low cost, high retrieval; low complexity | Low cost; low complexity | Medium cost; medium complexity | Medium cost; high complexity |
| **Best for** | Low-value, local, cooperative | Routine digital records on a trusted platform | The legal consignment or transferable document | High-value, disputed, multi-party, audited handovers |

## 3. When to use which

- **Paper signed copies** — low-value, local handovers; both parties cooperative; the main need is a familiar legal record; no third-party audit.
- **Scanned PDF or portal** — routine digital records where the platform is trusted, disputes are rare, and the operator's audit log is acceptable evidence.
- **e-CMR / eBL** — when the legal document itself is the object: the consignment note or the transferable bill. Use the legal instrument; the transparency log can complement it, not replace it.
- **Proposed L3** — when the record holder may rewrite, withhold, or equivocate; the handover is high-value or dispute-prone; the parties are separate organisations; a third party (insurer, auditor, court, employer) must verify offline; or the volume makes manual retrieval and forensic comparison expensive.

The proposed system is not a universal replacement. It is a complement for the cases where the two-copy model's lack of a shared head matters.

## 4. What it adds, and what it does not

**Adds:**

- a shared, witnessed head that both parties can compare;
- an inclusion receipt for each statement;
- a `prev` link from the receipt statement to the exact signed handover statement;
- withholding detection through the issuer-signed `sequence`;
- trusted time from an RFC 3161 authority;
- offline verification by a third party, without the operator;
- an audit path that does not depend on the platform's honesty.

**Does not add:**

- proof that the goods arrived or the document is genuine or complete, or that anyone inspected them;
- proof that the device was uncompromised;
- legal non-repudiation or a qualified trust service;
- confidentiality of the payload (the log stores hashes; the payload is stored separately);
- identity proofing (that is the CA's job);
- protection against collusion between sender and recipient;
- a replacement for e-CMR, eBL, or the legal document itself;
- universal superiority over paper.

## 5. The claim under test

> In handovers where the record holder may rewrite, withhold, or equivocate, a witnessed Merkle log with RFC 9942 receipts and RFC 3161 time gives a third party offline-verifiable evidence that the two-copy model does not, at a cost acceptable for high-value or dispute-prone handovers.

This is conditional and falsifiable. It is falsified if a paper or portal policy detects every red-team case, if cost or latency exceeds the targets, or if the chosen domain has no disputes — in which case the threat model is contrived and the project should stop (PRP-001 §8.6).

## 6. Pros of the proposed method against each baseline

The table in §2 states the criteria; this section states the pairwise advantages, and the counter-advantages, so the comparison is not a strawman.

| Baseline | What the proposed method does better | What the baseline still does better |
| --- | --- | --- |
| **Paper signed copies** | One shared, witnessed head; tamper-evident after signing; withholding detection; equivocation detection; trusted time; offline third-party verification; audit at scale; photo and file hashes bound to the statement | Simpler and cheaper for low volume; no keys or infrastructure; legally familiar; human-readable; works without power or network; physical forensic evidence |
| **Scanned PDF / portal** | No need to trust the operator for integrity; receipts and witnesses; offline verification; shared head; trusted time; vendor independence; data minimisation; independent audit | Integrated UX, onboarding, notifications, support; existing workflows; payload storage; possible compliance certifications; lower operational effort |
| **e-CMR / eBL** | Independent transparency and non-equivocation around the document; offline third-party verification; audit; can bind the document hash into the log | Legal recognition and regulatory acceptance; handles the legal document and transferability; industry adoption; established liability framework |
| **Signed PDF by email** | Inclusion, time, and non-equivocation; the email server cannot silently alter or withhold a statement without detection; a third party can verify | Simplicity; wide understanding; no infrastructure; legally familiar |

**Net.** The proposed method's distinctive advantages are not better signatures, cheaper paper, or a nicer interface. They are: a shared head, independent verification, withholding and equivocation detection, trusted time, and audit at scale. If those properties are not needed, the baseline is the better choice.

**Do not claim:** physical or documentary truth, legal non-repudiation, a replacement for e-CMR or eBL, lower cost for low-value handovers, or greater simplicity. The proposed method is heavier machinery; it earns its place only in the cases in §3.

## 7. The operator as record keeper

This is the sharpest test: the party whose honesty is in question also holds the record. In the primary case, that party is the university registry or the employer's HR function; in the parcel case, it is the carrier. Under a trusted operator, the four approaches differ mainly in convenience and legal form; under an operator-record-keeper, the differences become security properties.

| Under "operator = record keeper" | Paper signed copies | Scanned PDF / portal | e-CMR / eBL | Proposed L3 |
| --- | --- | --- | --- | --- |
| **Authoritative record** | Each physical copy | The operator's platform | The operator's platform or registry | The operator's log, plus the recipient's receipt and external witnesses |
| **Can the operator alter it?** | Its own copy; not the recipient's without physical access | Yes | Yes, unless the registry or legal framework is independent | It cannot alter a signed statement; a fork needs both witnesses |
| **Can it withhold it?** | Its copy; the recipient's copy remains | Yes | Yes | Yes; the receipt and issuer sequence expose it, if the recipient holds them |
| **Can it equivocate?** | Yes, by showing different copies | Yes, different tenant views | Yes, unless an independent registry prevents it | Not with 2-of-2 independent witnesses; a stale head is still possible |
| **Trusted time** | Handwritten; disputed | Server clock | Platform clock or TSA | External TSA |
| **Third-party verification** | Manual or forensic | Needs the platform | Needs the platform or registry | Offline bundle plus five anchors |
| **Residual risk** | Divergence, forgery, loss | Total operator trust | Legal recourse, but technical trust in the operator | Witness, CA, and TSA independence; availability; stale head |

**Key condition.** The proposed L3 survives a malicious operator-record-keeper only if the witnesses, the TSA, the CA, and the verifier are outside the operator's control. If the operator runs the log and both witnesses, the quorum is cosmetic: the same actor cosigns its own head. The minimum separation is:

- operator / log operator — one role;
- witness A and witness B — external, separate organisations;
- TSA — external;
- CA — external, or explicitly accepted by the relying party as its identity root;
- verifier — run by the relying party, not the operator.

Anything less collapses toward the portal.

**Paper is stronger here than it looks.** The recipient's copy is independent of the operator's platform, so the operator cannot alter it without physical access. For a simple condition or version dispute, paper may be enough. It cannot, however, prove trusted time, inclusion, or a shared history, or let a third party verify at scale.

**Portal and e-CMR.** An operator-run portal is the worst case: the operator is the sole record holder, and the recipient's PDF export is a copy, not a receipt. An operator-run e-CMR platform may give the recipient legal recourse through the document framework, but the technical evidence remains operator-controlled; an independent registry or notarization changes that, the operator's own platform does not.

## 8. Caveats

- The comparison is qualitative; costs and latencies are planning estimates, not measurements.
- Legal recognition depends on jurisdiction and on the trust service used; verify the current eIDAS, e-CMR, and MLETR texts before citing them.
- The proposed system produces evidence, not a qualified trust service.
- Paper is the baseline, not a strawman: for many handovers it is the right choice.
- e-CMR and eBL solve the legal-document problem; the log addresses the transparency problem around it.
