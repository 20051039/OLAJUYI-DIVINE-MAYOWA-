# TECHFIX.NG — System Architecture

**Version:** 2.0 (Revised) · **Date:** 2026-10-21 · **Status:** DRAFT — pending owner review and approval.

---

## 1. Architecture Overview

**Proposed stack** (provider choice OD-07 confirms payments; otherwise this stack stands unless the owner overrides):

```
┌─────────────┐     HTTPS      ┌──────────────────────────────┐
│  Web client │ ◄────────────► │  Firebase Hosting (static)   │
│ (responsive │                │  + CDN                       │
│  SPA)       │                └──────────────────────────────┘
└──────┬──────┘
       │  Firebase SDK (Auth / Firestore / Storage / Functions)
       ▼
┌──────────────────────────────────────────────────────────────┐
│ Firebase project (per environment: dev / staging / prod)     │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐  │
│  │ Firebase   │  │ Firestore  │  │ Cloud Storage          │  │
│  │ Auth       │  │ (rules-    │  │ (private paths + rules │  │
│  │ + custom   │  │ enforced)  │  │  + short-lived URLs)   │  │
│  │ claims     │  └────────────┘  └────────────────────────┘  │
│  └────────────┘  ┌────────────────────────────────────────┐  │
│                  │ Cloud Functions (trusted server logic) │  │
│                  │ • booking create/transition validation │  │
│                  │ • payment webhook verification         │  │
│                  │ • file re-validation & metadata        │  │
│                  │ • status transitions & publication     │  │
│                  │ • admin actions & audit logging        │  │
│                  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
       │ HTTPS webhooks / server API
       ▼
┌──────────────────┐     ┌──────────────────┐
│ Payment provider │     │ Email/SMS (notify│
│ Paystack or      │     │  — [PROPOSED])   │
│ Flutterwave [OD] │     └──────────────────┘
└──────────────────┘
```

**Principles:**

1. **Zero trust in the client.** Every sensitive write/transition is validated by Security Rules **and/or**
   Cloud Functions. Client success callbacks (payments, uploads) are never trusted.
2. **Public data is a projection.** Public artisan profiles are separate, public-only documents maintained by
   trusted code — never the artisan's private record filtered client-side.
3. **Private by default.** Identity docs, guarantor data, review notes live in restricted collections/paths.
4. **State machines are explicit.** All verification/payment/approval transitions are enumerated; illegal
   transitions are rejected server-side (see `04-database-schema.md` §5).
5. **Audit everything sensitive.** Admin decisions, status changes, private-file accesses, consent records —
   append-only audit logs.

---

## 2. Authentication Architecture (revised for mandatory booking auth)

### 2.1 Providers & flows

- Email/password + email verification **[PROPOSED]**; Google sign-in **[PROPOSED, optional]**.
- Roles via **Auth custom claims**: `role ∈ {customer, artisan, admin}` (+ `adminLevel` if tiered access is adopted, OD-11).
- A user may hold **one primary role** for MVP **[PROPOSED]** (a person needing both customer and artisan
  functions uses the artisan account, which includes booking rights — avoids dual-account confusion).
- Sessions: Firebase ID tokens (1 h) with refresh; sensitive actions (review decisions, consent declaration)
  require a **fresh token** (re-auth if older than **[PROPOSED]** 15 min for admin decisions).

### 2.2 Registration & login (customer booking gate)

```
Visitor clicks "Book Artisan" on /artisans/{id}
  → not authenticated?
      → redirect /login?next=/artisans/{id}/book   (intent preserved; allowlisted paths only)
      → login or register → email verify (if required) → return to {next}
  → authenticated? → booking form, customerId bound server-side to auth.uid
```

- `next` parameter is **allowlisted** to same-origin booking/profile paths (open-redirect prevention).
- Booking drafts survive re-authentication where practical (local draft + server validation on submit).
- Session expiry mid-booking → re-authenticate → resume flow; no data loss beyond what validation requires.

### 2.3 Server-side identity validation

- Callable Functions and Firestore rules both resolve identity from the **verified ID token** (`auth.uid`,
  `auth.token.role`). Any `customerId`/`artisanId` supplied by the client that must equal the caller is
  cross-checked and rejected on mismatch.
- Admin-only operations additionally require `auth.token.role == "admin"` (claim set only by the admin-grant
  procedure, §6.3 of the security plan — never self-assignable).

---

## 3. Route Protection Strategy

### 3.1 Client route matrix

| Route | Visitor | Customer | Artisan | Admin |
|---|---|---|---|---|
| `/`, `/artisans`, `/artisans/{id}` (public) | ✅ | ✅ | ✅ | ✅ |
| `/login`, `/register` | ✅ | redirect home | redirect home | redirect home |
| `/artisans/{id}/book` | ➡️ login (`?next=…`) | ✅ | ✅ (own booking rights) | ✅ view-only |
| `/bookings`, `/bookings/{bid}` | ➡️ login | ✅ own only | ✅ received only | ✅ (support, audited) |
| `/artisan/onboarding/*` | ➡️ login | ➡️ role home | ✅ own only | ✅ (support, audited) |
| `/artisan/dashboard` | ➡️ login | ➡️ role home | ✅ own only | ✅ (support, audited) |
| `/admin/*` | ➡️ login then 403 | 403 | 403 | ✅ least-privilege |

Client guards are **UX only** — every route's data is independently protected by Firestore/Storage rules + Functions.

### 3.2 API / data-access enforcement layers

| Operation | Layer 1 (rules) | Layer 2 (Functions) |
|---|---|---|
| Create booking | rules: authed, `customerId == auth.uid`, artisan publishable (via public doc) | Function validates artisan visibility snapshot, slot/status, writes canonical record |
| Read booking | rules: owner customer OR assigned artisan OR admin | — |
| Artisan profile edit (draft) | rules: owner artisan, approval-pending fields only | — |
| Publish/unpublish artisan | rules: deny all direct writes | Function-only, evaluates publication rule |
| Status transitions (verify/approve/…) | rules: deny all direct writes | Admin-only Function with audit log |
| Upload ID/supporting doc | Storage rules: owner artisan, path/type/size | Function re-validates bytes, writes metadata |
| Read private file | Storage rules: deny all direct reads | Signed/short-lived URL minted by Function after authz |
| Payment webhook | — | Provider-signature-verified Function, idempotent |
| Admin decision | rules: deny all direct writes | Admin-only Function, fresh-token + audit |

---

## 4. Component Design

### 4.1 Public profile projection

- Canonical artisan record (private) → Cloud Function trigger maintains `publicArtisans/{id}` **only while**
  the publication rule holds; deletes/omits it the moment any condition regresses.
- Public search/listing queries **only** `publicArtisans` — incomplete artisans are unreachable by construction,
  not by filter discipline.

### 4.2 Booking service (Function-backed)

- `createBooking({artisanId, serviceDetails, preferredTime, …})`: verifies caller auth, artisan `isPubliclyVisible`
  at submit time, input validation; creates booking with `customerId = auth.uid`, status `pending`.
- Transitions (`confirm/decline/cancel/complete`) validated against the booking state machine; suspended-artisan
  block enforced here (T-14) as well as in rules.

### 4.3 Verification pipeline (Function-backed)

- `submitIdentityDocument`, `submitEvidence`, `submitGuarantor` (metadata writes; files via resumable private upload).
- `recordReviewDecision({applicationId, scope: identity|additional|guarantor|final, decision, notes})`:
  admin-only, fresh token, mandatory notes, audit-logged; drives status fields and publication recompute.
- `requestResubmission` / `recordAppeal` / `recordRefund` exist as recording endpoints; their **policies** are
  owner decisions (OD-08/09) — endpoints enforce whatever policy flags are configured.

### 4.4 Payment verification

- Provider webhook → signature verification → idempotent `paymentStatus` update (`pending → verified/failed`) →
  publication recompute → artisan notification. No client-writable payment fields exist.

### 4.5 Notifications

- Trigger-based (status transitions, booking events) → in-app notification docs + email/SMS fan-out **[PROPOSED]**.
- Guarantor outreach, if any, only through the owner-approved process (PRD FR-GUA-3).

---

## 5. Environments & Operations

- Separate Firebase projects: `techfix-dev`, `techfix-staging`, `techfix-prod` **[PROPOSED]**.
- Emulator Suite for rules + Functions tests in CI; staging mirrors prod config with test provider keys.
- Secrets in Secret Manager; no secrets in client bundle or repo.
- Structured logging (JSON) with correlation IDs; alerts on webhook failures, rule-denial spikes, decision anomalies.
- Backups: scheduled Firestore exports; Storage versioning/lifecycle per retention policy (OD-10).

---

## 6. Key Architectural Decisions (ADRs, summarized)

| ID | Decision | Rationale |
|----|----------|-----------|
| ADR-1 | Public profiles are a separate projection, not filtered private docs | Makes leaks structurally impossible; satisfies FR-PUB-2/3 |
| ADR-2 | All status transitions via Functions; direct writes denied | Prevents self-approval, self-publish, forged payment states |
| ADR-3 | Private files never directly readable; short-lived URLs via Function | Prevents leaked-link abuse; enables per-access audit |
| ADR-4 | Server-side MIME sniffing + allowlist + size cap on uploads | Defense in depth against malicious uploads |
| ADR-5 | Manual admin review for MVP; no KYC-vendor claim | Honest posture per BR-5; provider decision deferred (OD-02) |
| ADR-6 | Payment verification webhook-only, idempotent | Prevents forged payment states |
| ADR-7 | Intent-preserving auth redirect with allowlisted `next` | Satisfies FR-CUST-2 without open redirects |
