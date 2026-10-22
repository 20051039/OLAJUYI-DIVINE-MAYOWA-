# TECHFIX.NG — MVP Roadmap

**Version:** 2.1 (OD-01/OD-02 recorded as owner-resolved 2026-10-22)
**Date:** 2026-10-22 · **Status:** DRAFT — pending owner review and approval.
**Companion docs:** PRD (`01-PRD.md`), Architecture (`03-architecture.md`), Backlog (`08-implementation-backlog.md`).

> No application code is to be written until the revised documents are reviewed and approved.
> Phase numbering is logical (dependency order), not calendared — estimates are added after document approval.

---

## 1. MVP Definition (revised)

The MVP is a responsive web app on the approved stack (see `03-architecture.md`) that delivers:

### 1.1 Mandatory before public production launch (launch gates)

| # | Capability | PRD ref |
|---|------------|---------|
| G-1 | **Authenticated customer booking** — register/login required; intent-preserving redirect; server-validated identity; per-customer booking isolation; booking history + statuses + validation + error states | FR-CUST-1…8 |
| G-2 | **Secure artisan identity-document submission** — private upload, allowlisted types, size limits, server validation, error handling | FR-IDV-1…4 |
| G-3 | **Additional artisan verification information** — per-trade evidence capture honoring mandatory/optional separation | FR-ADD-1…2 |
| G-4 | **Guarantor information + approved consent workflow** — collection, consent declaration record, restricted access, independent-confirmation hooks (as decided) | FR-GUA-1…4 |
| G-5 | **Administrative verification & approval** — review queues, decision recording with notes, approve/reject/request-info, suspend | FR-ADM-1…4, FR-IDV-7 |
| G-6 | **Separate states** — `paymentStatus`, `identityVerificationStatus`, `additionalVerificationStatus`, `guarantorVerificationStatus`, `accountApprovalStatus` tracked and enforced independently | §3.8 |
| G-7 | **Secure private document storage** — non-public buckets/paths, short-lived access, restrictive rules | FR-IDV-2/8 |
| G-8 | **Authorization & privacy tests** — all 16 required acceptance tests pass; automated checks that public surfaces leak no sensitive data | `07-test-plan.md` |
| G-9 | **Publication rule enforcement** — public directory never reveals an artisan with incomplete checks (§3.8 of PRD) | FR-PUB-3, §3.8 |

### 1.2 Explicitly out of MVP scope

- Automated identity verification / third-party KYC integration (manual admin review only).
- Customer↔artisan payments, escrow, or in-app chat beyond booking contact-disclosure rules.
- Guarantor legal-liability agreements (blocked on legal review; consent + verification records only).
- Guarantor independent-confirmation channel **unless** the owner selects it (OD-05) — data model supports it either way.
- Native mobile apps; multi-country support.

---

## 2. Phased Delivery Plan

Dependencies flow downward; a phase begins when its entry criteria are met.

### Phase 0 — Foundations & governance (entry: document approval)

- Confirm stack + payment provider shortlist decision path (OD-07); set up environments (dev/staging/prod),
  CI, lint/format, secret management, audit-logging baseline.
- Owner-decision workshop: resolve OD-01…OD-12 or formally defer per §4.
- **Exit:** environments live; decision log current; backlog groomed.

### Phase 1 — Identity, roles & public directory shell

- Firebase Auth (email/password + email verification **[PROPOSED]**), custom-claims roles
  (`customer`, `artisan`, `admin`), session handling.
- Public artisan profile reads from a **public-only projection**; search/filter UI; "Book Artisan" CTA
  wired to auth-gated booking intent (deep link preserved).
- Firestore + Storage rules scaffolding with emulator tests.
- **Exit:** visitors browse seeded approved profiles; unauthenticated "Book Artisan" routes to auth and
  returns to intent (acceptance test T-01).

### Phase 2 — Artisan onboarding: account → profile → identity documents

- Artisan registration, professional-profile builder (draft, never public), validation.
- Secure ID-document upload (FR-IDV-2/3): private paths, allowlist, 5 MB cap **[PROPOSED]**, server
  re-validation via Functions, error states; metadata recorded, status `pending_review`.
- **Exit:** artisan can complete PRD steps 1–3; invalid/prohibited files rejected (T-05); self-publish
  impossible (T-04); unauthorized document access denied (T-06).

### Phase 3 — Additional verification + guarantor workflow

- Evidence capture per §3.5 matrix (portfolio, experience, qualifications/licences where applicable,
  references, admin-requested extras); mandatory/optional separation enforced per trade.
- Guarantor capture (FR-GUA-1), consent-declaration record (FR-GUA-3), restricted access (FR-GUA-4);
  independent-confirmation hooks built behind a policy flag (OD-05/OD-06).
- **Exit:** artisan can complete PRD steps 4–5; guarantor data isolated (T-07); paid-but-incomplete and
  verified-but-no-guarantor states behave per T-09/T-10.

### Phase 4 — Payments (fee verification)

- Provider integration (OD-07), server-side webhook/API verification, `paymentStatus` transitions,
  idempotent handlers, failure/retry UX, refund-status recording hooks (policy OD-09).
- **Exit:** ₦3,000 fee payable and server-verified end-to-end in staging; client callbacks untrusted by test.

### Phase 5 — Admin review & approval console

- Queues, least-privilege private-material viewers (short-lived URLs, audit-logged), decision recording
  (approve/reject/request-info with mandatory notes), suspension, notifications.
- Publication-rule engine (`isPubliclyVisible` computed by trusted code; instant unpublish on regression).
- **Exit:** full 11-step journey green in staging (T-11, T-12, T-15); rejected/suspended artisans absent
  from public results (T-13, T-14).

### Phase 6 — Customer booking (authenticated, end-to-end)

- Booking request/confirm/decline/cancel/complete flows, history + statuses, validation + error states,
  contact-disclosure rules (OD-11), notifications.
- **Exit:** T-01/T-02/T-03 pass; session-expiry and artisan-suspended-mid-flow handled.

### Phase 7 — Hardening, privacy & launch readiness

- Threat-model mitigations (`06-security-privacy-plan.md`), rules fuzzing, file-upload abuse tests,
  PII-leak scans, retention/deletion jobs (policy OD-10), correction/deletion request flow (T-16),
  performance/a11y pass, runbooks, launch-gate audit (§1.1).
- **Exit:** all launch gates G-1…G-9 evidenced; owner sign-off; production launch.

---

## 3. Milestone / Gate Mapping

| Gate | Satisfied by phases | Primary evidence |
|------|---------------------|------------------|
| G-1 Authenticated booking | 1, 6 | T-01, T-02, T-03 + route-protection matrix |
| G-2 ID-document submission | 2 | T-05, T-06 + upload-validation tests |
| G-3 Additional verification | 3 | Per-trade evidence tests |
| G-4 Guarantor + consent | 3 | T-07, T-10 + consent-record tests |
| G-5 Admin verification/approval | 5 | T-12, T-15 + decision-audit tests |
| G-6 Separate states | 2, 3, 4, 5 | T-09, T-10, T-11 + transition tests |
| G-7 Private document storage | 2, 3, 7 | Storage-rules tests, URL-expiry tests |
| G-8 Auth & privacy tests | 7 (continuous from 1) | Full `07-test-plan.md` suite green |
| G-9 Publication rule | 5, 7 | T-04, T-09, T-10, T-11, T-13, T-14 |

---

## 4. Decision Dependencies (what blocks what)

- **OD-01 (accepted ID documents) — RESOLVED 2026-10-22:** the 4 approved types unblock Phase 2 UI copy/validation lists.
- **OD-03 (per-trade evidence policy)** blocks Phase 3 mandatory/optional enforcement.
- **OD-04/05/06 (guarantor policy, consent recording, independent confirmation)** block Phase 3 consent UX finalization;
  consent-declaration + restricted storage can proceed on the proposed baseline.
- **OD-07 (payment provider; extra fees)** blocks Phase 4 integration choice (scaffold with a provider interface meanwhile).
- **OD-08/09 (rejection/resubmission/appeal/refund)** block Phase 5 notification copy and refund handling; decision
  recording + resubmission-required state proceed regardless.
- **OD-10 (retention periods)** and **OD-11 (who sees what contact data)** block Phase 7 retention jobs and Phase 6
  disclosure rules; restrictive defaults ship until decided.
- **OD-02 (manual vs automated verification) — RESOLVED 2026-10-22:** **manual administrator review** for MVP; no automated/biometric/external-provider/government claims; any verification-service integration is post-MVP and needs a new decision.
- **OD-12 (suspension effects on existing bookings)** blocks Phase 5/6 edge-case copy; new-booking block is mandatory regardless.

Unresolved decisions at launch require explicit documented deferral with restrictive defaults — see `09-owner-decisions.md`.
