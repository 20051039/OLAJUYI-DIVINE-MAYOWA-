# TECHFIX.NG — Test Plan & Acceptance Tests

**Version:** 2.0 (Revised) · **Date:** 2026-10-21 · **Status:** DRAFT — pending owner review and approval.

Test layers: **rules-emulator unit tests** (Firestore/Storage) · **Function integration tests** (authz, transitions,
webhooks) · **end-to-end acceptance tests** (the 16 required tests below + supporting suite) · **privacy-leak scans**
(public surfaces) · **abuse tests** (uploads, URL expiry, rate limits). All 16 required tests are launch-gating.

---

## 1. Required Acceptance Tests (T-01 … T-16)

### T-01 — Unauthenticated customer attempts to book an artisan
**Steps:** As a visitor on an approved artisan's public profile, click "Book Artisan".
**Expected result:** ✅ Redirected to login/registration with the booking intent preserved
(e.g. `/login?next=/artisans/{id}/book`); **no booking is created**; after successful authentication the customer
returns to the intended artisan profile/booking flow. `next` values outside the allowlist fall back to home
(no open redirect).

### T-02 — Authenticated customer books an artisan successfully
**Steps:** As a logged-in customer on an approved, non-suspended artisan's profile, complete the booking form with
valid data and submit.
**Expected result:** ✅ Exactly one booking is created with `customerId == auth.uid`, `artisanId` correct,
status `pending`, and a complete audit/status-history entry; customer sees a confirmation; artisan is notified;
server-side validation (not client data) determined the stored values. Double-submit creates no duplicate (idempotency).

### T-03 — Customer attempts to read another customer's booking
**Steps:** As customer B (authenticated), request customer A's booking detail (URL + direct API/rules read).
**Expected result:** ✅ Denied at every layer (UI shows 404-equivalent; rules deny; Function denies); response
reveals **neither the booking's existence nor any of its contents** (no oracle via timing/message differences beyond
negligible noise). Denial is logged.

### T-04 — Artisan attempts to publish their own profile before approval
**Steps:** As an authenticated artisan whose application is incomplete/unapproved, attempt to publish via UI affordances
(if any), direct Firestore writes to `publicArtisans/*` / `isPubliclyVisible`, and crafted Function calls.
**Expected result:** ✅ All attempts fail (no publish affordance exists; direct writes denied by rules; Function
rejects); artisan remains absent from public listings/search; attempts are logged.

### T-05 — Artisan submits an invalid or prohibited file
**Steps:** Attempt uploads of: executable (`.exe`), script renamed to `.pdf`, oversized file (> 5 MB **[PROPOSED]**),
wrong MIME vs extension, empty file, and a valid control file.
**Expected result:** ✅ Every invalid file is rejected with a specific, actionable error (type/size/network) at the
earliest layer, quarantined or never persisted, and never stored in a public location; status stays `not_submitted`
(or prior state unchanged); the valid control file succeeds and enters `pending_review` with hash metadata recorded.

### T-06 — Unauthorized user attempts to access an identity document
**Steps:** As (a) another artisan, (b) a customer, (c) a visitor, (d) support-level admin without verification role —
attempt to read ID file bytes, ID metadata, and ID numbers via app, direct Storage reads, and direct Firestore reads.
**Expected result:** ✅ All denied; direct Storage reads of `private/**` always fail; no signed URL is minted for
unauthorized callers; no ID content/number appears in any response; attempts are audit-logged. (Authorized verifier
access succeeds and is itself logged — covered in T-15.)

### T-07 — Unauthorized user attempts to access guarantor information
**Steps:** As (a) the artisan via retrieval APIs, (b) another artisan, (c) a customer, (d) non-verification admin —
attempt to read the guarantor document/fields.
**Expected result:** ✅ All denied (artisan sees only their pipeline `status` + guidance, not stored guarantor PII);
no guarantor detail appears on any public surface or in any unauthorized response; attempts audit-logged.

### T-08 — Artisan attempts to change their own verification status
**Steps:** As the artisan, attempt direct writes to `identityVerificationStatus`, `additionalVerificationStatus`,
`guarantorVerificationStatus`, `accountApprovalStatus`, `paymentStatus`, and crafted "approve" Function calls.
**Expected result:** ✅ All rejected (rules deny; Function rejects non-admin callers); statuses unchanged;
attempts audit-logged. Only the admin-only decision Function (T-15) can effect these transitions.

### T-09 — Artisan has paid but has incomplete identity verification
**Steps:** Fixture: `paymentStatus=verified`, `identityVerificationStatus ∈ {not_submitted, pending_review}`,
other scopes incomplete. Check public listing/search/profile, booking creation, and artisan dashboard copy.
**Expected result:** ✅ Artisan is **absent** from all public surfaces; booking creation against them fails with a
safe message; dashboard shows "Payment received — verification pending" (or equivalent) with next steps; **payment
alone never publishes**.

### T-10 — Artisan has completed identity verification but has no recorded guarantor approval
**Steps:** Fixture: identity `verified`, `guarantorVerificationStatus ∈ {not_submitted, pending_review,
contacted_pending}` (no `verified`). Check public surfaces, booking creation, dashboard copy.
**Expected result:** ✅ Artisan remains **unpublished** and unbookable; dashboard directs the artisan to complete the
guarantor step (and any consent confirmation the approved policy requires); no guarantor PII is exposed to justify
the state — status + guidance only.

### T-11 — Artisan has completed all verification requirements but is awaiting administrator approval
**Steps:** Fixture: payment verified + identity verified + additional approved + guarantor verified, but
`accountApprovalStatus=in_review` (final approval pending). Check public surfaces, booking creation, dashboard copy.
**Expected result:** ✅ Artisan remains **unpublished** and unbookable; dashboard shows "Under final review";
publication occurs only after the final admin approval (T-12).

### T-12 — Administrator approves a fully compliant artisan
**Steps:** As a verification-authorized admin with a fresh session, approve the T-11 fixture via the decision Function
(final scope), with mandatory reason/notes.
**Expected result:** ✅ `accountApprovalStatus=approved`, `isPubliclyVisible=true`, public projection created with
**public-only fields**; artisan appears in listings/search; bookings can be created; artisan notified; decision +
publication + notification all audit-logged with actor, timestamp, and reason.

### T-13 — Rejected artisan attempts to appear in public search results
**Steps:** Fixture: rejected artisan (any scope) attempts profile access via search, listing filters, guessed
deep-links, and crafted API queries.
**Expected result:** ✅ Artisan appears in **no** public result set; direct profile deep-link returns the safe
"unavailable" state with no private leakage; rejection notification was sent; any resubmission/appeal affordances
match exactly the owner-approved policy (OD-08/09) — nothing more.

### T-14 — Suspended artisan attempts to accept a new booking
**Steps:** Fixture: previously approved artisan is suspended (reason recorded). Attempt: artisan-side "accept" actions
on new booking requests; customer-side new booking creation; visibility checks.
**Expected result:** ✅ Suspended artisan is immediately absent from public surfaces; new booking creation fails
safely; no new "accept" action succeeds; existing bookings behave per the approved suspension policy (OD-12);
suspension/unsuspension are audit-logged.

### T-15 — Authorized administrator reviews a document and records a decision
**Steps:** As a verification-authorized admin: open review queue, view a private document via the short-lived viewer,
record an approve and (on a second fixture) a reject/request-info decision with mandatory notes.
**Expected result:** ✅ Private material renders only in the authorized, expiring viewer; each file access is
audit-logged; decisions lacking mandatory reason/notes are rejected; valid decisions update exactly the scoped
status, write a review record + audit entry, recompute publication, and notify the artisan with guidance text only
(no internal notes). Stale-session decision attempts require re-authentication.

### T-16 — Applicant requests correction or deletion of personal information
**Steps:** As an artisan applicant (and separately a customer), submit a correction request and a deletion request;
admin processes each per the retention schedule.
**Expected result:** ✅ Requests are acknowledged and tracked; corrections apply to the stated fields (audit-logged);
deletions remove in-scope data (files + metadata) subject to retention obligations, with a deletion certificate in
the audit log; where retention lawfully requires keeping data, the outcome notice states the basis and the scheduled
deletion date. No silent drops: every request reaches a communicated outcome.

---

## 2. Supporting Suites (required, non-exhaustive)

| Suite | Coverage | Gate |
|---|---|---|
| Route-protection matrix | Every row of `03-architecture.md` §3.1, all four roles | Launch |
| Rules-emulator allow/deny inventory | Each granted operation + representative denials | Every PR |
| Booking state machine | All legal transitions + rejection of illegal ones | Launch |
| Payment verification | Forged/tampered/replayed webhooks rejected; idempotent retries; client callbacks untrusted | Launch |
| Upload abuse | Polyglots, bombs, MIME spoofing, path traversal, overwrite attempts | Launch |
| Signed-URL expiry | Access before/after TTL; revoked sessions; unauthorized mint attempts | Launch |
| Privacy-leak scan | Crawl public pages/APIs; assert zero matches for ID numbers, guarantor phones/names, internal statuses, storage paths | Every PR + pre-launch |
| Error-message review | Auth/validation/404 messages reveal no PII or existence oracles | Pre-launch |
| Notification suite | Every transition in PRD §3.10 fires exactly-once to the right parties | Launch |
| Retention/deletion jobs | Quarantine → delete → certificate; backup-expiry alignment | Launch (with interim OD-10 values if undecided) |
| Accessibility | WCAG 2.1 AA on public + booking + onboarding flows | Pre-launch |
| Performance smoke | PRD NFR-3 budgets on staging | Pre-launch |

## 3. Test Data & Environments

- Seeded fixtures per state combination in the publication rule (all 2⁵-relevant combos covered by T-09…T-12 + unit tests).
- Synthetic PII only in dev/staging; production verification tests use redacted, consenting pilot accounts.
- Payment tests run against provider sandbox (OD-07); no live-money tests pre-launch.

## 4. Entry / Exit Criteria

- **Entry (per phase):** fixtures + rules tests written before the feature (groomed in `08-implementation-backlog.md`).
- **Exit (launch):** all 16 tests pass on staging + production-parity check; privacy-leak scan clean; no unresolved
  launch-gating owner decision without documented deferral (see `09-owner-decisions.md`).
