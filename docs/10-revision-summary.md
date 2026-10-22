# TECHFIX.NG — Revision Summary (v2.2): Changes, Conflicts & Remaining Decisions

**Date:** 2026-10-22 · **Status:** DRAFT — pending owner review and approval.
**Addendum (v2.2):** OD-01 resolved (accepted ID types: NIN slip/card, voter's card, driver's licence, international
passport) and OD-02 resolved (manual administrator review for MVP; no automated/biometric/external-provider/government
verification claims). PRD, architecture, schema, security plan, test plan (added T-17/T-18), and backlog (added IDV-6)
aligned; OD-03…OD-12 remain open with no invented policies.
**Addendum (v2.1):** PRD gained §1.4 (Worker.NG functional reference + HopeRise visual inspiration, with guardrails:
no copied branding/assets, no unverified claims, no invented testimonials/records/credentials) and the backlog gained
FND-5 (reference study + original design tokens). No business policies were decided at that time.
**Instruction honored:** documentation updated; **no application code written.**

---

## 1. Summary of Changes (v1 → v2)

This revision incorporates the confirmed requirements update while preserving all previously confirmed rules
(one-time ₦3,000 fee, mandatory admin approval, no booking without authentication).

| # | Change | Documents touched |
|---|--------|-------------------|
| 1 | **Authenticated customer booking** — account + login required; unauthenticated "Book Artisan" → login/register with intent-preserving return; server-validated identity; per-customer booking isolation; history + statuses + validation + error states | PRD §3.2; Architecture §2–3; Schema §2.10; Flows §2; Tests T-01…T-03; Backlog Epics 2, 10 |
| 2 | **Mandatory artisan identity verification** — secure upload interface; proposed allowlist (`jpg/png/pdf`, 5 MB); recommended Nigerian ID options (NIN, voter's card, driver's licence, passport) flagged for approval; minimal PII; separate `identityVerificationStatus`; admin review with pending/verified/rejected/resubmission states; never public; no self-approval; no automation claims | PRD §3.4; Roadmap G-2; Schema §2.5; Flows §3; Security §1–5; Tests T-05/T-06/T-08; Backlog Epic 5 |
| 3 | **Additional artisan verification** — five evidence types; mandatory/optional/trade-specific separation with proposed per-trade matrix; no assumed qualifications; badges only from checked credentials | PRD §3.5; Schema §2.6; Backlog Epic 6 |
| 4 | **Mandatory guarantor requirement** — name/phone/relationship (+ approved extras); separate `guarantorVerificationStatus` + contacted record + admin notes; consent-declaration workflow; independent-confirmation question flagged; restricted access; never public; no liability assumptions; legal-review gates | PRD §3.6; Schema §2.7–2.8; Flows §3–4; Security §6–7; Tests T-07/T-10; Backlog Epic 7 |
| 5 | **11-step artisan registration & approval workflow** with publication rule (§3.8 PRD): payment verified + identity verified + additional approved + guarantor verified + admin approved + not suspended; rejection/resubmission/appeal/refund flagged as owner decisions | PRD §3.3/3.8; Roadmap phases 2–5; Flows §3; Tests T-09…T-14 |
| 6 | **Data model revision** — pipeline aggregate (`applications/*`), restricted subcollections (identityDocs, evidence, guarantor, consents, reviews), public-only projection, bookings ownership, append-only audit logs, status-transition allowlists, private Storage layout with short-lived access | Schema (all); Architecture §3–4 |
| 7 | **Security/privacy/retention update** — 12-item threat model; default-deny rules strategies; file-validation pipeline; admin provisioning; access matrix; interim retention schedule; NDPA-2023 confirmation questions for qualified advice | Security plan (all); Backlog Epic 12 |
| 8 | **MVP roadmap revision** — 9 launch gates (G-1…G-9) incl. authenticated booking, verification, guarantor, separate states, private storage, auth/privacy tests; phased plan 0–7 with decision dependencies | Roadmap (all) |
| 9 | **16 required acceptance tests** with expected results + supporting suites (rules inventory, state machines, abuse, leak scans, a11y, perf) | Test plan (all) |
| 10 | **Owner-decisions register** — 12 items (OD-01…OD-12), no invented answers, interim defaults + blockers recorded | `09-owner-decisions.md` |
| 11 | **Implementation backlog** — 13 epics, ~50 small testable stories with acceptance criteria, dependencies, and a global Definition of Done | `08-implementation-backlog.md` |

### Preserved without change
BR-1 one-time ₦3,000 fee · BR-2 mandatory admin approval · BR-3/4 auth-gated booking + public browsing ·
BR-5 no genuineness-by-upload · BR-6 no unchecked certification claims · BR-7 no assumed guarantor liability.

---

## 2. Conflicts & Tensions Identified

| # | Conflict / tension | Where it bites | How this revision handles it (decision needed) |
|---|--------------------|----------------|-----------------------------------------------|
| C-1 | **Fee is paid (step 6) before admin review (steps 8–10)** — so rejected applicants will have paid ₦3,000. This collides with fairness expectations and demands a refund/rejection policy. | PRD FR-ART-2; PAY-5; ADM-5 | Rejection/resubmission/appeal/refund made explicit owner decisions (OD-08/09); interim: disclose pre-payment, record statuses, issue no refunds until policy set. Alternative orderings (review-before-pay) were **not** adopted because the 11-step sequence fixes payment at step 6 — owner may revisit. |
| C-2 | **Guarantor PII vs. need-to-know vs. artisan transparency** — the artisan submits the data but must not retrieve it afterwards; support staff must not see it; verifiers must. | Schema §2.7; Security §6.1 | Restrictive default (verification-role-only + artisan status-only); final masks await OD-11. |
| C-3 | **Consent burden vs. conversion** — independent guarantor confirmation (OTP/callback/declaration) strengthens integrity but adds friction and cost (SMS, ops). | PRD FR-GUA-3 | Built as a policy flag (GUA-4); default off (declaration only) until OD-05/06 resolve. |
| C-4 | **Evidence rigor vs. informal trades** — many artisans lack formal qualifications/licences; uniform mandates would exclude legitimate workers. | PRD §3.5 | Per-trade matrix separates mandatory/optional (OD-03); portfolio + experience mandatory for all **[PROPOSED]**. |
| C-5 | **Review thoroughness vs. launch speed** — manual review of every ID + evidence + guarantor is operationally heavy with no automation in MVP. | Roadmap Phase 5; OD-02 | Accepted for MVP (honest posture, BR-5); resourcing/queues sized in operations planning; automation post-MVP only. |
| C-6 | **Retention for disputes vs. privacy minimization** — rejected-applicant data is both a liability to keep and potentially needed for appeals/fraud prevention. | Security §8 | Quarantine-then-delete schedule with interim durations; final periods OD-10. |
| C-7 | **Disclosure vs. safety** — customers need enough artisan contact info to transact; artisans need protection from exposure. | BOK-6; OD-11 | Exact disclosure fields deferred to OD-11; public profiles carry coarse location only. |

No logical contradictions were found between the new requirements and the preserved business rules — the items above
are policy/sequencing tensions requiring owner judgment, not inconsistencies to correct unilaterally.

---

## 3. Remaining Owner Decisions (see `09-owner-decisions.md`)

**Resolved 2026-10-22:** OD-01 (accepted ID types) · OD-02 (manual administrator review; no
automated/biometric/external-provider/government claims).
**Still open:** OD-03 per-trade evidence mandates · OD-04 guarantor eligibility/extras/agreement ·
OD-05 consent recording + independent confirmation · OD-06 guarantor ID/declaration ·
OD-07 payment provider + extra fees · OD-08 rejection/resubmission/appeal · OD-09 refunds ·
OD-10 retention periods + request SLA · OD-11 access matrix + disclosure rules · OD-12 suspension effects on
existing bookings — plus qualified legal review of consent/declaration/privacy texts and a
Nigerian data-protection confirmation memo (Security §9) before production launch.

## 4. Recommended Next Steps

1. Owner reviews this document set and resolves/defers the remaining OD-03…OD-12 in a Phase 0 workshop (FND-4).
2. Engage qualified Nigerian counsel/DPO advisor on Security §9 questions and legal-review gates (SEC-6/7).
3. Confirm payment provider (OD-07) and stand up sandbox.
4. Approve documents in writing → then, and only then, begin Phase 1 implementation per the backlog.
