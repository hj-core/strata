# BUILD-HANDOVER-005 — Employment Records at One Proposition

| Field | Value |
| --- | --- |
| Prepared | 2026-10-23 |
| For | The next BUILD agent continuing after the employment roast |
| Repo | `/Users/toddhuang/Projects/strata` |
| Branch | `tdd` |
| HEAD | `8246f7f` (verified with `git log`; working tree has untracked documents) |
| Working tree | **Uncommitted.** Seven untracked documents: `PRC-001`, `DEC-002`, `ANL-008`, `ANL-009`, `CNP-002`, `DISCUSSION-SUMMARY-005`, this file. **Do not stage or commit.** |
| Read first | `DISCUSSION-SUMMARY-005`, then `DEC-002`, `ANL-009`, `CNP-002`, `ANL-008` |
| Status | **D8 open: proceed on one proposition, or stop. No build started. Supervisor approval of the direction is still not recorded.** |
| Supersedes | `BUILD-HANDOVER-004` **for the current state**. -004 remains the record of the automotive pivot. |

---

## 1. Status at a glance

| Item | State |
| --- | --- |
| Module | **COMP4299** — confirmed |
| Deliverable | Dissertation + demo + evaluation — confirmed |
| Parcel | **Out** as the primary case (`DISCUSSION-SUMMARY-004`) |
| Automotive | **Out** — traced, dropped (`PRC-001`, `DEC-002` §2) |
| Employment | **Current direction, narrowed** — one proposition (`ANL-009`) |
| Module constraints | Narrow mechanism + measured boundary **accepted**; **negative headline not acceptable**; plausible scenario **sufficient** |
| Claim | A verifier can establish, offline and from the bundle alone, that a record was issued by an authorised issuance role, for that holder, with that content, and had not been superseded or revoked at the decision time |
| Strongest alternatives | **ESSPASS** (legislates the property set) and **Velocity Network** (deploys employment credentials at scale) |
| Surviving cells | Two: offline status at a past decision time; non-equivocation without a chain. The second already failed for the parcel case. |
| Build | **Not started** |
| Decision | **D8 open** |

---

## 2. The job

1. **Do not build yet.** D8 is open and the supervisor has not approved the direction.
2. **Decide D8** — proceed on the one-proposition primitive, or stop and renegotiate the topic. Two cheap checks should precede it (§4).
3. **If proceeding**, run the H2 desk analysis: does a signed status-list snapshot plus a TSA already answer the historical question offline? If yes, the project has no load-bearing log and D8 becomes a stop.
4. **If H2 passes**, re-cut the WBS (`PLN-002` is parcel-shaped), then build the document-variant slice.
5. **Do not run another domain search** without the filter in §6. Five domains have now failed, four for the same structural reason.

---

## 3. What happened since -004

`BUILD-HANDOVER-004` proposed automotive test-evidence and listed six jobs. None of them were executed as written; the direction was tested and dropped instead.

| Step | Result |
| --- | --- |
| Automotive process defined and traced against seven scandals | `PRC-001`: 0 fully flagged, 3 partial, 3 not. Detection carried outside the mechanism in every partial case |
| Automotive dropped | `DEC-002` |
| Employment adopted | `ANL-008`: seven anchors, four at V status |
| RFC 9943 read closely | Four repo assumptions corrected (§5) |
| Employment roasted | `ANL-009`: two fatal findings, one surviving proposition |
| Domain filter derived | §6 below |

**The one-sentence version.** The employment case passed the anchor test and failed the strongest-alternative test, leaving one proposition and one open decision.

---

## 4. D8 — the decision

Three routes, from `ANL-009` §8:

| Route | What it is | Cost | Risk |
| --- | --- | --- | --- |
| **1 — forward-looking documents** | Offer letters, sponsorship commitments, contract terms. Not history, so neither The Work Number nor Velocity covers them. Anchored by Tisha Lee only. | Re-anchor, then build | One anchor is thin; the segment may be too small |
| **2 — the historical-status primitive** | Keep employment as the vehicle; the contribution is offline status at a past decision time, with employment as the domain | Build as planned, narrower claim | The boundary is most of the project; may read as a log feature, not a system |
| **3 — stop** | Report that the primitive has no domain where it is load-bearing | Supervisor conversation | **Blocked as a headline** by the module's "no negative result" constraint |

**Two checks before deciding:**

1. **ESSPASS historical status.** Does the proposed regulation or the EUDI Wallet answer "was this valid at a past decision time"? If yes, route 2 collapses.
2. **H2 desk analysis.** Can a signed status-list snapshot plus a TSA, carried in the bundle, survive an honest adversarial review? If yes, the log adds nothing and route 2 is a signature project.

Both are days of work, not weeks. Neither requires a build.

**Recommendation to the next agent:** run both checks, then take D8 to the supervisor with the result. Do not start a build on the assumption that D8 resolves in favour of proceeding.

---

## 5. What is settled — do not re-litigate

**About RFC 9943** (verified against the text):

1. **No tiers.** The phrase "RFC 9943 tiers" in `BUILD-HANDOVER-004` decision 10 is wrong. The RFC defines no levels.
2. **No witnesses.** The word does not appear. Non-equivocation is a VDS requirement (§5.1.3); the mechanisms left open are auditor replay and multiple independent TSs (§9.7). Witnesses are an implementation choice.
3. **Transparent is not public.** §4's "publicly available" is qualified by "anyone with access"; §8 expects authorisation and permits hash-only VDSs.
4. **History is the purpose, not the specification.** Three guarantees in §9; the normative content is the message format and the registration contract.
5. **The RFC authenticates the issuer and the log, not the holder.** `sub` is issuer-defined; there is no proof of possession; client authentication is out of scope (§6.3).
6. **Accuracy is not assured** (§9.2), **selective submission is acknowledged** (§9.3), **revocation is out of scope** (§9.4.2), and a compromised TS may **roll back** (§9.4.2).

**About the case**:

7. **The anchors support authenticity, not denial.** No case shows an employer disowning a genuine issuance. Do not claim "the employer cannot deny."
8. **The market is occupied.** ESSPASS legislates authorised issuance, integrity and non-revocation; Velocity deploys employment-history credentials at scale and pays issuers.
9. **Status and income are not the niche.** The Work Number owns them.
10. **Completeness is not a cryptographic property.** A sequence proves consistency, not completeness (`DISCUSSION-SUMMARY-003`).

**About the module**:

11. Narrow mechanism + measured boundary: **accepted**.
12. Negative headline: **not acceptable**.
13. Plausible scenario: **sufficient**; no stakeholder need be recruited.

---

## 6. The four-question filter

Use this before writing anything about a new domain. If any answer is no, stop.

1. Is there a **mandatory intermediary** the relying party must use and cannot audit?
2. Does the dispute turn on **that intermediary's record**, rather than on facts obtainable elsewhere?
3. Is the intermediary the party with the **incentive to falsify**?
4. Is the log's contribution **more than a signature plus a timestamp**?

| Domain | Passes? | Fails on |
| --- | --- | --- |
| Certificate Transparency | ✓✓✓✓ | — |
| WhatsApp AKD | ✓✓✓✓ | — |
| Parcel | ✗ | 1 |
| Automotive | ✗ | 1, 4 |
| Employment | ✗ | 1, 4 |
| Notarised messaging | ✗ | 1 (courts compel; QERDS exists) |

**The generalisation worth carrying:** the log is load-bearing only where the intermediary is unavoidable. That single sentence explains all five failures and is the strongest analytical asset produced so far.

---

## 7. Open questions and gates

| # | Question | Why it matters |
| --- | --- | --- |
| G1 | **Supervisor approval of the employment direction** | Still not recorded; D8 depends on it |
| G2 | Module constraints | **Answered** 2026-10-23; confirm in writing |
| G3 | **D8 — proceed or stop** | The only open decision that matters |
| G4 | **H2** — does a status-list snapshot + TSA answer the historical question offline? | Decides whether the log is load-bearing |
| G5 | **ESSPASS historical status** — does the regulation or wallet answer past-time validity? | Could collapse the surviving proposition |
| G6 | **D1** — confirm the primary case is the letter of terms, not status/income | Status/income is owned by incumbents |
| G7 | WBS re-cut | `PLN-002` is parcel-shaped and does not carry over |
| G8 | Anchor status labels | Done in `ANL-008`; keep the V/P/A discipline |
| G9 | Prevalence | The anchors show the pattern, not its frequency |

---

## 8. Document map and read order

**Current**

1. `DISCUSSION-SUMMARY-005` — what this session decided and discovered.
2. `DEC-002` — why automotive was dropped, why employment, what must be true (D8).
3. `ANL-009` — the roast; the two fatal findings; the two surviving cells.
4. `CNP-002` — the concept note, on hold, with the module constraints and the RFC analysis.
5. `ANL-008` — seven anchors and the four failure patterns.
6. `PRC-001` — the automotive process and the seven-case trace; **historical**, retained as domain-selection evidence.

**Historical (do not edit)**

- `CNP-001` — the parcel concept note (roast object).
- `BUILD-HANDOVER-004` — the automotive pivot.
- `BUILD-HANDOVER-003` — the parcel roast commission.
- `BUILD-HANDOVER-002`, `BUILD-HANDOVER.md` — earlier states.
- `DISCUSSION-SUMMARY-003`, `-004` — the parcel search and the automotive pivot.
- `PRP-001` v0.2.0, `PRP-001a`, `ANL-004`, `ANL-005`, `ANL-006`, `ANL-007` — parcel-era method and evidence.

**Standards and libraries**

- RFC 9943 (SCITT architecture), RFC 9942 (COSE receipts), RFC 9162 (CT v2 Merkle log), RFC 3161 (TSA), RFC 9052 (COSE), RFC 9597 (CWT claims).
- W3C Bitstring Status List; IETF Token Status List.
- Tessera; `scitt-cose`; `transparency-dev/witness`; Meta's `akd` (reference for key transparency).

---

## 9. Git and document state

- Branch `tdd`; HEAD `8246f7f`; working tree has seven untracked documents.
- The user commits the documents. **Do not stage or commit.**
- `BUILD-HANDOVER-004`'s field values are stale: it recorded `/home/hj-core/project`, HEAD `cdb5a93`, and an uncommitted tree. The actual state is above.

---

## 10. Cautions

- Do not call it a blockchain; call it a witnessed append-only log.
- Do not claim the RFC requires witnesses, defines tiers, or makes the log public.
- Do not claim completeness, prevention, or legal non-repudiation.
- Do not present a boundary as the dissertation's headline.
- Do not claim novelty for authorised issuance, integrity or revocation — ESSPASS legislates all three.
- Do not claim "no one is doing this" — Velocity and ESSPASS are.
- Do not treat the module constraints as settled without written confirmation.
- Do not start a sixth domain search without the filter in §6.
- Do not build before D8 and G1 are answered.

---

## 11. One-paragraph summary

The automotive direction was traced against seven scandals and dropped: nothing was fully caught, and in every partial case the detection was carried by an institution outside the mechanism. The employment direction replaced it with seven documented anchors, four at primary or official status, all sharing one structure — the verifier's only control is to call the employer, and the call fails on insiders, absent employers and foreign issuers. A close reading of RFC 9943 corrected four repo assumptions: there are no tiers, witnesses are not in the RFC, transparency is not publicity, and history is the purpose rather than the specification; the protocol authenticates the issuer and the log but not the holder. The strongest-alternative roast then found that ESSPASS legislates the three properties the proposal claimed and Velocity deploys employment credentials at scale, leaving the proposal alone in two cells: offline status at a past decision time, and non-equivocation without a chain. The second is the parcel claim that already failed. The project now rests on one proposition, and the next agent's job is not to build it but to decide — with two cheap checks and the supervisor — whether one proposition is enough. The four-question filter derived in this session is the cheapest way to test the next idea before writing about it.
