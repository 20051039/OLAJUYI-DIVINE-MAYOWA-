# TECHFIX.NG — Product Requirements Document (PRD)

**Version:** 2.0 (Revised — Authenticated Booking, Mandatory Verification & Guarantor Requirements)
**Date:** 2026-10-21
**Status:** DRAFT — pending owner review and approval. **No application code is to be written until these documents are reviewed and approved.**
**Related documents:**

- `docs/02-mvp-roadmap.md` — MVP scope, phases, launch gates
- `docs/03-architecture.md` — system, auth, and route-protection architecture
- `docs/04-database-schema.md` — Firestore data model & Cloud Storage layout
- `docs/05-user-flows.md` — customer, artisan, and administrator journeys
- `docs/06-security-privacy-plan.md` — threat model, rules strategy, retention
- `docs/07-test-plan.md` — acceptance tests incl. the 16 required tests
- `docs/08-implementation-backlog.md` — epics, stories, acceptance criteria, DoD
- `docs/09-owner-decisions.md` — unresolved decisions requiring owner input
- `docs/10-revision-summary.md` — summary of changes, conflicts, remaining decisions

> **Convention used in this document set:** anything marked **[OWNER DECISION]** is an unresolved
> business/policy question listed in `docs/09-owner-decisions.md`. Anything marked **[PROPOSED]**
> is a recommendation that stands unless the owner overrides it. Nothing in these documents invents
> answers to owner decisions.

---

## 1. Product Overview

**TechFix.ng** is a Nigerian online directory and booking marketplace that connects customers with
local artisans (e.g., electricians, plumbers, carpenters, tailors, mechanics, painters, tilers).

### 1.1 Product goals

1. Let anyone **browse publicly approved artisan profiles** without creating an account.
2. Let **registered, authenticated customers** book approved artisans and track booking history/status.
3. Onboard artisans through a **verified pipeline**: account → profile → identity verification →
   supporting verification → guarantor → registration fee → administrator review → approval.
4. Ensure **only fully compliant artisans** (verified payment, approved identity and required
   verification, administrator approval, no suspension/restrictions) appear in public listings.
5. Protect sensitive personal data (identity documents, identity numbers, guarantor details) with
   private storage, restricted access, audit trails, and proportionate retention.

### 1.2 Non-goals (MVP)

- Automated identity verification (no verification-service integration is claimed or assumed).
- In-app payments from customers to artisans; customer-to-artisan pricing/escrow.
- Instant messaging beyond booking-linked contact disclosure rules **[OWNER DECISION: what contact
  details a customer sees after booking — see §9]**.
- Native mobile apps (MVP is a responsive web app).
- Multi-country support; Nigeria only.

### 1.3 Previously confirmed business rules (preserved)

| ID | Rule |
|----|------|
| BR-1 | Artisan registration carries a **one-time ₦3,000 registration fee**, verified server-side through the approved payment provider. |
| BR-2 | **Mandatory administrator approval**: no artisan profile is publicly visible without admin approval. |
| BR-3 | **No customer booking without authentication**: customers must register and log in before submitting a booking. |
| BR-4 | Public browsing of approved artisan profiles requires no account. |
| BR-5 | Uploading a document is **not** proof of genuineness; verification requires authorized human review unless an approved verification service is integrated (none is integrated in MVP). |
| BR-6 | No artisan is presented as certified/licensed/verified beyond what has actually been checked. |
| BR-7 | Guarantors are **not** automatically assumed legally liable for an artisan's conduct or debts. |

---

## 2. Users & Roles

| Role | Description |
|------|-------------|
| Visitor (unauthenticated) | Browses/searches public artisan directory. Cannot book. |
| Customer (authenticated) | Books artisans, views own booking history/status. Cannot see or modify another customer's bookings. |
| Artisan (authenticated) | Registers, builds profile, submits verification materials, pays fee, receives bookings once approved. Cannot self-approve or self-publish. |
| Administrator (authorized) | Reviews verification materials, records decisions, approves/rejects/requests information, suspends accounts, manages audit-sensitive actions. |

Role assignment is enforced server-side (auth custom claims + Firestore rules + Cloud Functions).
See `docs/03-architecture.md`.

---

## 3. Functional Requirements

### 3.1 Public directory (visitors + all roles)

- **FR-PUB-1:** Anyone may browse and search **publicly approved** artisan profiles without an account.
- **FR-PUB-2:** Public profiles must contain only approved public fields (see `docs/04-database-schema.md`).
  They must **never** contain identity documents, identity numbers, guarantor details, payment details,
  review notes, or internal statuses.
- **FR-PUB-3:** Public listings/search must exclude any artisan who does not satisfy **all** of:
  verified payment + approved identity and required verification + administrator approval +
  no suspension or other restrictions.
- **FR-PUB-4:** Filter/search by trade, location (LGA/state), rating, availability **[PROPOSED]**.

### 3.2 Customer accounts & authenticated booking (NEW — §1 of requirements update)

- **FR-CUST-1:** Customers must create an account and log in before submitting a booking.
- **FR-CUST-2:** An unauthenticated visitor who clicks **"Book Artisan"** is directed to login or
  registration; after successful authentication they are returned to the intended artisan profile or
  booking flow **where practical** (deep-link intent preserved, e.g. `?next=/artisans/{id}/book`).
- **FR-CUST-3:** The server validates the authenticated customer's identity on every booking write
  (auth token → UID; no client-supplied customer ID is trusted).
- **FR-CUST-4:** Every booking is associated with exactly one customer account (`customerId = auth.uid`).
- **FR-CUST-5:** Customers must not be able to read, modify, or cancel another customer's bookings
  (enforced by Firestore security rules **and** server-side checks).
- **FR-CUST-6:** Customers can view their **booking history** with per-booking **status**
  (`pending | confirmed | completed | cancelled | declined` **[PROPOSED]**), timestamps, artisan
  reference, and service details.
- **FR-CUST-7:** Booking form validation (required fields, date/time sanity, service description
  length limits) with clear inline errors; server re-validates everything.
- **FR-CUST-8:** Error states: artisan unavailable/suspended mid-flow, network failure, session expiry
  mid-booking (re-authenticate, preserve draft where practical).
- **FR-CUST-9:** Customers can request correction or deletion of their personal information, subject to
  applicable retention obligations (see `docs/06-security-privacy-plan.md`).

### 3.3 Artisan registration & approval workflow (REVISED — 11-step sequence)

The artisan journey **must** follow this order:

1. Artisan creates an account.
2. Artisan completes their professional profile.
3. Artisan submits valid identity documentation (§3.4).
4. Artisan provides the required additional verification information (§3.5).
5. Artisan provides the required guarantor information and completes the approved consent process (§3.6).
6. Artisan pays the one-time **₦3,000** registration fee.
7. The server verifies the payment through the approved payment provider.
8. The application and verification materials enter the administrator review process.
9. The administrator reviews identity information, supporting evidence, guarantor requirements, and payment status.
10. The administrator approves, rejects, or requests additional information per the approved policy.
11. Only artisans satisfying **all** of §3.8 appear in public listings.

Supporting rules:

- **FR-ART-1:** The workflow must be designed so payment and verification can occur in an appropriate
  order **without allowing an incomplete application to become publicly visible**. Payment completion
  alone must never publish a profile.
- **FR-ART-2:** If an application is **rejected**, the artisan is notified (in-app + email **[PROPOSED]**).
  Whether corrections, resubmission, appeal, or refund are permitted is **[OWNER DECISION]** —
  see `docs/09-owner-decisions.md` (OD-08, OD-09). The system must support recording the decision and
  reason regardless of the final policy.
- **FR-ART-3:** An artisan must not be able to publish their own profile before approval (no client-side
  bypass; publication is a server-side/admin-only transition).
- **FR-ART-4:** Suspended artisans must not be able to accept new bookings; existing bookings follow the
  suspension policy **[OWNER DECISION: OD-12]**.

### 3.4 Mandatory artisan identity verification (NEW)

- **FR-IDV-1:** Every artisan must submit a valid identity document as part of registration. Registration
  cannot proceed past step 3 without a submission that passes file validation.
- **FR-IDV-2:** Provide a **secure identity-document upload interface**: authenticated session only,
  direct-to-private-storage upload with short-lived credentials/URLs, progress + error states, no public URL.
- **FR-IDV-3:** Permitted file types **[PROPOSED, pending file-policy confirmation as part of OD-01]**:
  `image/jpeg`, `image/png`, `application/pdf` only. Maximum file size **[PROPOSED]**: 5 MB per file.
  Validation: MIME sniffing (magic bytes) server-side, extension allowlist, size cap, malware posture per
  `docs/06-security-privacy-plan.md`. Upload error handling: type/size/network/expired-session errors with
  retry guidance; rejected files are never stored in public locations.
- **FR-IDV-4:** Recommended identity-document options for the Nigerian market **[PROPOSED — final accepted
  document types require owner approval (OD-01)]**:
  1. National Identification Number (NIN) slip / NIN card issued by NIMC;
  2. Voter's card (PVC) issued by INEC;
  3. Valid Nigerian driver's licence (FRSC);
  4. Nigerian international passport (NIS).
  The UI presents whatever set the owner approves — no other documents are accepted.
- **FR-IDV-5:** Collect **only** the personal information necessary for the approved verification process:
  document type, document reference/number (stored privately, never on public profiles), full name as on
  document, expiry where applicable, and the file itself. No additional fields without approval.
- **FR-IDV-6:** Record the document's verification status **separately** from payment and account-approval
  status: `identityVerificationStatus ∈ {not_submitted, pending_review, verified, rejected, resubmission_required}`.
- **FR-IDV-7:** Authorized administrators review submissions and record a verification decision with notes
  and timestamp (audit-logged).
- **FR-IDV-8:** Identity documents and identity numbers are **never** exposed on public artisan profiles,
  in public APIs, or to other artisans/customers.
- **FR-IDV-9:** Artisans cannot approve their own documents (status transitions to `verified`/`rejected`
  are admin-only, server-enforced).
- **FR-IDV-10:** The product must not claim automated identity verification exists unless an actual,
  approved verification service is integrated. MVP assumes **manual admin review only**.
  Whether verification stays manual or uses an external provider is **[OWNER DECISION: OD-02]**.

### 3.5 Additional artisan verification (NEW)

- **FR-ADD-1:** Artisans must provide additional information/evidence supporting verification and
  professional credibility. The system evaluates/supports these evidence types:
  - (a) Trade or professional qualifications, where applicable;
  - (b) Relevant licences or certifications, where applicable;
  - (c) Work portfolio photographs;
  - (d) Work experience and references;
  - (e) Additional evidence requested by an authorized administrator.
- **FR-ADD-2:** The product **separates mandatory requirements from optional or trade-specific evidence**
  (matrix in §3.5.1). It must not assume every artisan holds a formal qualification or licence.
- **FR-ADD-3:** An artisan must not be presented as professionally certified unless the relevant
  credential has actually been checked (profile badges derive only from verified records).
- **FR-ADD-4:** Final per-trade mandatory/optional policy is **[OWNER DECISION: OD-03]**.

#### 3.5.1 Proposed evidence matrix (all cells [PROPOSED], final policy OD-03)

| Evidence | Electrician | Plumber | Carpenter | Tailor | Mechanic | Painter | Tiler | Other trades |
|---|---|---|---|---|---|---|---|---|
| Portfolio photos (min 3) | M | M | M | M | M | M | M | M |
| Work experience statement | M | M | M | M | M | M | M | M |
| Trade qualification (where exists) | T | T | O | O | T | O | O | T/O |
| Licence/certification (where exists) | T | T | O | O | O | O | O | T/O |
| References (≥1 contactable) | M | M | M | O | M | O | O | [DECIDE] |
| Admin-requested extras | C | C | C | C | C | C | C | C |

`M` = mandatory for all · `T` = mandatory only where that trade has a recognized credential ·
`O` = optional · `C` = conditional on admin request. The admin can always request additional evidence
(type (e) above) for any trade.

### 3.6 Mandatory guarantor requirement (NEW)

- **FR-GUA-1:** Every artisan must provide a guarantor as part of registration/verification. The workflow
  collects, as **proposed requirements pending final guarantor-policy confirmation**:
  full name, contact telephone number, relationship to the artisan, and any additional information approved
  as necessary (e.g. home address — **[OWNER DECISION: OD-04]**).
- **FR-GUA-2:** The system records **guarantor verification status** separately:
  `guarantorVerificationStatus ∈ {not_submitted, pending_review, contacted_pending, verified, rejected, resubmission_required}` **[PROPOSED]**,
  plus whether the guarantor has been contacted/verified, by whom, when, and admin review notes.
- **FR-GUA-3:** **Consent workflow (approved process required before any contact):**
  1. The artisan confirms they have obtained the guarantor's informed consent before submitting the
     guarantor's personal information (in-app declaration + timestamp recorded).
  2. Whether the guarantor must **independently confirm** consent/identity (e.g. OTP to guarantor's phone,
     signed declaration upload) is **[OWNER DECISION: OD-05, OD-06]** — the data model supports both.
  3. The platform must not contact a guarantor or send them personal information without following the
     approved process once the owner confirms it.
- **FR-GUA-4:** Guarantor information is restricted to authorized personnel who need it for verification
  (role-based access + audit logging). It is **never** displayed on public artisan profiles.
- **FR-GUA-5:** The system must not represent a guarantor as legally liable. Any guarantor agreement,
  liability terms, or legal declarations require **qualified legal review before implementation**
  (tracked as OD-04/OD-06 blockers; MVP ships with consent + verification records only, no liability language).
- **FR-GUA-6:** Guarantor eligibility criteria (e.g. age, relationship exclusions, one-guarantor-per-artisan
  limits) are **[OWNER DECISION: OD-04]**. Data model supports storing criteria outcomes once defined.

### 3.7 Payments (preserved + clarified)

- **FR-PAY-1:** One-time **₦3,000** artisan registration fee, payable in-app via the approved provider.
- **FR-PAY-2:** Payment provider **[OWNER DECISION: OD-07]** — Paystack and Flutterwave are shortlisted
  **[PROPOSED]**; the server verifies every payment via provider webhook/API before recording
  `paymentStatus = verified`. Client-side success callbacks are never trusted.
- **FR-PAY-3:** `paymentStatus ∈ {unpaid, pending, verified, failed, refunded}` is recorded **separately**
  from identity, guarantor, and account-approval states.
- **FR-PAY-4:** Whether any verification fees exist beyond the ₦3,000 fee is **[OWNER DECISION: OD-07]**.
  The MVP assumes none.
- **FR-PAY-5:** Refund handling (eligibility, window, method) is **[OWNER DECISION: OD-09]**; the data model
  and admin console support recording refunds once policy is set.

### 3.8 Publication rule (single source of truth)

An artisan appears in public listings/search **if and only if all** of the following hold:

```
paymentStatus == "verified"
AND identityVerificationStatus == "verified"
AND additionalVerificationStatus == "approved"        // per-trade required evidence satisfied
AND guarantorVerificationStatus == "verified"
AND accountApprovalStatus == "approved"               // admin final approval
AND suspended == false
AND profileComplete == true
```

Publication is computed/enforced server-side (`isPubliclyVisible` maintained by trusted code only).
Any transition that breaks the conjunction must remove the artisan from public results immediately.

### 3.9 Administration

- **FR-ADM-1:** Admin queue of applications with filters by status (pending identity review, pending guarantor
  review, pending final approval, resubmission, rejected, suspended).
- **FR-ADM-2:** Admins can view private materials (least-privilege, audit-logged), record decisions with
  mandatory reason codes/notes, and trigger artisan notifications.
- **FR-ADM-3:** Server-side authorization on all review operations; admins cannot be self-appointed
  (bootstrap/grant procedure in `docs/06-security-privacy-plan.md`).
- **FR-ADM-4:** Suspension/unsuspension with reason; suspended artisans leave public results and cannot
  accept new bookings.

### 3.10 Notifications (supporting)

- **FR-NOT-1:** Status-change notifications to artisans (received, under review, decision, resubmission
  requested, approved, rejected, suspended) — in-app required; email/SMS **[PROPOSED]** channels.
- **FR-NOT-2:** Booking notifications to customer and artisan (new booking, confirmed, declined, cancelled,
  completed, reminders **[PROPOSED]**).
- **FR-NOT-3:** Guarantor contact (if any) follows only the owner-approved process (FR-GUA-3).

---

## 4. Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-1 | Security: OWASP ASVS L2 posture for auth/session/access-control; all sensitive operations server-authorized. |
| NFR-2 | Privacy: PII minimization, private storage, restrictive rules, audit trails, proportionate retention (see `docs/06-security-privacy-plan.md`). |
| NFR-3 | Performance **[PROPOSED]**: public listing p95 < 2 s on 3G; booking submit p95 < 3 s. |
| NFR-4 | Availability **[PROPOSED]**: 99.5% monthly excluding planned maintenance; status page post-MVP. |
| NFR-5 | Accessibility: WCAG 2.1 AA for public + booking flows. |
| NFR-6 | Observability: structured logs for auth, booking, payment verification, status transitions; alerting on payment-webhook and rule-denial anomalies. |

---

## 5. Acceptance Criteria (PRD level)

1. All §3 requirements trace to backlog items in `docs/08-implementation-backlog.md`.
2. All 16 required acceptance tests in `docs/07-test-plan.md` pass with stated expected results.
3. No public page/API response contains identity documents, identity numbers, guarantor details, or internal statuses (verified by automated privacy tests).
4. The publication rule (§3.8) holds under every tested state combination.
5. All **[OWNER DECISION]** items are resolved or explicitly deferred with owner sign-off before production launch, except those the launch-gate table marks deferrable.

---

## 6. Open Issues & Assumptions

- All policy questions are consolidated in `docs/09-owner-decisions.md` — this PRD invents none of the answers.
- Stack assumption **[PROPOSED]**: Firebase (Auth, Firestore, Cloud Storage, Cloud Functions, Hosting) +
  Paystack/Flutterwave — see `docs/03-architecture.md`. Owner confirms provider choice (OD-07).
- Nigerian data-protection obligations (e.g. NDPA 2023 / NDPC guidance) must be confirmed against current
  authoritative sources and qualified advice — see `docs/06-security-privacy-plan.md` §9.
- Guarantor liability/consent legal wording requires qualified legal review before implementation (FR-GUA-5).
