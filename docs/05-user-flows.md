# TECHFIX.NG — User Flows & Customer/Artisan Journeys

**Version:** 2.0 (Revised) · **Date:** 2026-10-21 · **Status:** DRAFT — pending owner review and approval.

---

## 1. Visitor — Browse Public Directory (no account required)

```
1. Visitor opens TechFix.ng → sees search + trade categories + featured approved artisans
2. Visitor searches/filters (trade, state/LGA) → results from public projection ONLY
3. Visitor opens artisan profile → public fields only (bio, trade, service areas,
   approved portfolio, ratings, availability, verified-trade badges)
4. Visitor clicks "Book Artisan"
     → Authenticated? → booking flow (§2.2)
     → Not authenticated? → /login?next=/artisans/{id}/book (§2.1)
```

Error/edge states: artisan unpublished mid-view ("no longer available" page, no private leakage);
search with no results (helpful empty state); offline behavior (cached public pages where feasible).

---

## 2. Customer Journeys (authentication mandatory for booking)

### 2.1 Register / login with intent return

```
1. Unauthenticated visitor hits auth gate (Book click or /bookings deep link)
2. System routes to /login?next={allowlisted intent} with "Log in to continue booking {artisan}"
3. Customer logs in OR registers (name, email, phone [PROPOSED optional], password)
4. Email verification if required [PROPOSED] → then automatic return to {intent}
5. Customer lands back on intended artisan profile / booking form; draft preserved
```

Validation & errors: duplicate email, weak password, unverified-email restriction, expired session,
invalid/expired `next` (fall back to home — never open-redirect).

### 2.2 Submit a booking (authenticated)

```
1. Customer on artisan profile → "Book Artisan" → booking form
2. Customer enters: service description, preferred date/time, location note,
   contact phone (prefilled, editable)
3. Client validates (required fields, date not in past, lengths) → submit
4. SERVER validates: auth token → customerId; artisan publicly visible NOW;
   re-validates all fields; creates booking (status: pending)
5. Confirmation screen + notification to customer and artisan
```

Error states: artisan unpublished/suspended mid-flow (clear message, suggest alternatives — no private data);
slot/status conflict; network failure (safe retry with idempotency key — no duplicate bookings);
session expiry (re-auth, draft preserved).

### 2.3 Booking history & status

```
1. Customer opens /bookings → list of OWN bookings (newest first), each with
   status chip, artisan name/trade, date, reference
2. Customer opens booking detail → full thread: status history, artisan contact
   per disclosure policy (OD-11), actions allowed by state machine
   (pending → cancel; confirmed → cancel per policy; completed → rate [PROPOSED])
3. Customer attempts another customer's booking URL → 404-equivalent (no existence oracle)
```

### 2.4 Correction / deletion request

```
1. Customer (or artisan applicant) opens Privacy → "Request correction or deletion"
2. Submits scoped request (which data, why) → tracked ticket [PROPOSED: in-app + email ack]
3. Admin processes per retention obligations (OD-10): corrects, deletes where allowed,
   or refuses with lawful basis + explanation
4. Requester notified of outcome
```

---

## 3. Artisan Journey (11-step registration & approval)

```
Step  1. CREATE ACCOUNT ── register as artisan; verify email [PROPOSED]
Step  2. COMPLETE PROFILE ── trade, experience, bio draft, phone, state/LGA,
         service areas (saved as private DRAFT — never public)
Step  3. SUBMIT IDENTITY DOCUMENT ── choose doc type (owner-approved list, OD-01),
         upload file (allowlist + 5 MB [PROPOSED]), enter minimal fields;
         → identityVerificationStatus = pending_review
Step  4. ADDITIONAL VERIFICATION ── portfolio photos (≥3 [PROPOSED]),
         experience statement, qualifications/licences where applicable,
         references; mandatory vs optional enforced per trade matrix (OD-03);
         → additionalVerificationStatus = pending_review
Step  5. GUARANTOR ── full name, phone, relationship (+ approved extras, OD-04);
         artisan records informed-consent declaration; independent guarantor
         confirmation only if OD-05 requires it;
         → guarantorVerificationStatus = pending_review
Step  6. PAY FEE ── pay one-time ₦3,000 via approved provider (OD-07)
Step  7. SERVER VERIFIES PAYMENT ── webhook/API confirmation;
         → paymentStatus = verified (artisan sees receipt — NOT publication)
Step  8. ENTERS REVIEW ── application queued; artisan sees "Under review" tracker
Step  9. ADMIN REVIEWS ── identity, evidence, guarantor, payment (§4)
Step 10. DECISION ── approved | rejected (notified; resubmission/appeal/refund
         per OD-08/09) | resubmission-required (artisan corrects & resubmits)
Step 11. PUBLISHED ── ONLY when payment verified + identity verified +
         additional approved + guarantor verified + admin approved + not suspended
         → profile appears in public listings; artisan can receive bookings
```

Journey rules:

- Progress tracker shows the 11 steps with per-step state; steps unlock in order **[PROPOSED]** (steps 3–5
  may be completed in any order once step 2 is done, but all are required before payment — restriction
  keeps the pipeline auditable; relax only with owner approval).
- **Payment never publishes.** A paid artisan with incomplete verification sees "Payment received —
  verification pending" and remains invisible publicly (acceptance tests T-09/T-10/T-11).
- Rejected artisans see the decision, shareable guidance, and whatever recourse the approved policy allows
  (OD-08/09) — never reviewer-internal notes.
- Suspended artisans: removed from public results immediately; cannot accept new bookings; existing bookings
  handled per OD-12.

---

## 4. Administrator Journeys

### 4.1 Verification review (per scope: identity → additional → guarantor → final)

```
1. Admin logs in (admin role; fresh token for decisions [PROPOSED 15 min])
2. Admin opens Review Queue → filters (pending identity / additional / guarantor /
   final approval / resubmissions / flagged)
3. Admin opens application → sees pipeline aggregate + private materials via
   short-lived, audit-logged viewers (each access logged)
4. Admin checks materials against policy checklists (per-scope criteria; no automated
   genuineness claim — manual review only for MVP)
5. Admin records decision (scope + approve/reject/request-info + MANDATORY reason
   + notes + optional guidance for artisan) → Function validates authority,
   writes status, audit log, notification, publication recompute
6. Final approval only enabled when payment verified + all three verification scopes approved
```

### 4.2 Guarantor handling (under approved process only)

```
1. Admin opens guarantor section (restricted to verification role; access logged)
2. If policy requires contact (OD-04/05): admin follows the approved script/channel,
   records contacted/when/outcome — no personal data sent to the guarantor beyond
   what the approved process allows
3. Admin records guarantor scope decision
```

No guarantor contact or data disclosure occurs without the owner-approved process (PRD FR-GUA-3).

### 4.3 Suspension

```
1. Admin opens artisan → Suspend (reason MANDATORY) → publication recompute removes
   profile immediately; new bookings blocked; parties to existing bookings notified
2. Unsuspend reverses, re-evaluating the full publication rule (re-review if policy requires)
```

### 4.4 Correction / deletion requests

Admin processes data-subject requests per §2.4 within the owner-approved SLA **[OWNER DECISION — part of OD-10]**.

---

## 5. Cross-Cutting UX Rules

- Every auth gate explains *why* login is needed and where the user will return afterward.
- Every status shown to artisans/customers uses plain language ("Under review", "Needs attention")
  with next-step guidance — never internal enum names or reviewer notes.
- Public pages and error messages must not leak existence or details of private records
  (uniform 404s, generic auth errors, no UID/enumeration oracles).
- All file uploads show type/size requirements up front, progress, and specific, actionable errors.
