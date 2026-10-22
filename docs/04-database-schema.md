# TECHFIX.NG — Database & Storage Design (Firestore + Cloud Storage)

**Version:** 2.1 (OD-01/OD-02 recorded as owner-resolved 2026-10-22) · **Date:** 2026-10-22 · **Status:** DRAFT — pending owner review and approval.

> Design rule: sensitive document *contents* live in private Cloud Storage; Firestore holds **metadata +
> private storage references + statuses** in restricted collections. General-purpose/public documents never
> carry identity numbers, document URLs, or guarantor contact details.

---

## 1. Collection Map

```
users/{uid}                        — account + role (owner-readable, limited fields)
publicArtisans/{artisanId}         — PUBLIC projection (world-readable; trusted writes only)
artisans/{artisanId}               — private artisan record (owner artisan + admin)
artisans/{artisanId}/identityDocs/{docId}      — ID metadata (owner + admin)
artisans/{artisanId}/evidence/{evidenceId}     — supporting-evidence metadata (owner + admin)
artisans/{artisanId}/guarantor/{gId}           — guarantor info + status (admin + owner-limited*)
artisans/{artisanId}/reviews/{reviewId}        — admin decisions (admin; artisan sees outcome subset*)
artisans/{artisanId}/consents/{consentId}      — consent records (admin; artisan sees own declaration)
applications/{artisanId}           — pipeline aggregate: statuses, timestamps, resubmission (owner-limited + admin)
bookings/{bookingId}               — bookings (customer owner + assigned artisan + admin)
notifications/{uid}/items/{nid}    — per-user notifications (recipient + system)
auditLogs/{logId}                  — append-only (admin-auditors; Functions write)
adminUsers/{uid}                   — admin roster/levels (admins; bootstrap procedure)
```

`*` Owner-limited = the artisan sees only non-sensitive outcome fields (e.g. status + guidance), never
reviewer-internal notes or guarantor contact details beyond what they submitted. Exact field masks are defined
per collection below; contact-disclosure details await OD-11.

---

## 2. Core Documents

### 2.1 `users/{uid}`

| Field | Type | Notes |
|---|---|---|
| `role` | `customer \| artisan \| admin` | Set by trusted code only; mirrored in custom claims |
| `displayName` | string | |
| `email` | string | Auth-managed; lowercased |
| `phone` | string (E.164) | Optional for customers; required for artisans **[PROPOSED]** |
| `emailVerified` | bool | Mirror of Auth state (Function-maintained) |
| `createdAt`, `updatedAt` | timestamp | |

Rules: owner can read own doc; owner can update allowlisted profile fields only; `role` unwritable by owner.

### 2.2 `publicArtisans/{artisanId}` (PUBLIC — world-readable, Function-written only)

| Field | Type | Notes |
|---|---|---|
| `artisanId`, `displayName` | | Public business identity |
| `trade`, `specializations[]` | | |
| `state`, `lga`, `serviceAreas[]` | | Coarse location only (no home address) |
| `bio` | string | Admin-approved text **[PROPOSED]** |
| `portfolioPhotoUrls[]` | signed-or-public URLs | Only admin-approved portfolio images; see §6 |
| `ratingAvg`, `ratingCount` | number | Computed |
| `availabilityNote` | string | e.g. "Mon–Sat, 8am–6pm" |
| `verifiedTradeBadges[]` | string[] | Derived ONLY from actually-checked credentials (BR-6) |
| `isPubliclyVisible` | bool `true` | Present only while publication rule holds; doc deleted otherwise |
| `publishedAt`, `updatedAt` | timestamp | |

**Must never contain:** identity docs/numbers, guarantor data, payment data, review notes, internal statuses,
exact home address, or private contact details beyond the owner-approved disclosure (OD-11).

### 2.3 `artisans/{artisanId}` (private artisan record)

| Field | Type | Notes |
|---|---|---|
| `ownerUid` | string | `== artisanId` for MVP (1:1 account) |
| `profileComplete` | bool | Steps 1–2 done |
| `trade`, `experienceYears`, `bio` (draft) | | Draft until approved |
| `phone`, `state`, `lga`, `address` (private) | | Never projected to public |
| `suspended` | bool | Admin-only |
| `suspensionReason`, `suspendedAt`, `suspendedBy` | | Admin-only |
| `createdAt`, `updatedAt` | timestamp | |

### 2.4 `applications/{artisanId}` (pipeline aggregate — the state machine home)

| Field | Type | Notes |
|---|---|---|
| `paymentStatus` | `unpaid\|pending\|verified\|failed\|refunded` | Function-only (webhook) |
| `paymentRef`, `paidAt`, `amountKobo` (=300000) | | Receipt metadata, no card data |
| `identityVerificationStatus` | `not_submitted\|pending_review\|verified\|rejected\|resubmission_required` | Function-only (admin decision) |
| `additionalVerificationStatus` | `not_submitted\|pending_review\|approved\|rejected\|resubmission_required` | Function-only |
| `guarantorVerificationStatus` | `not_submitted\|pending_review\|contacted_pending\|verified\|rejected\|resubmission_required` | Function-only |
| `accountApprovalStatus` | `draft\|in_review\|approved\|rejected\|resubmission_required\|suspended` | Function-only |
| `isPubliclyVisible` | bool | Computed by trusted code per publication rule |
| `currentStep` | 1–11 | UX progress pointer (derived, not authoritative) |
| `rejectionReason`, `rejectionAt`, `rejectedBy` | | Set on reject |
| `resubmissionCount`, `lastResubmittedAt` | | Resubmission tracking (policy OD-08) |
| `appealStatus`, `refundStatus` | enums + metadata | Recording hooks (policy OD-08/09) |
| `submittedAt`, `decidedAt`, `updatedAt` | timestamp | |

Rules: artisan reads own aggregate; **no direct status writes** by artisan; admin reads all; writes Function-only.

### 2.5 `artisans/{artisanId}/identityDocs/{docId}` (restricted metadata)

| Field | Type | Notes |
|---|---|---|
| `documentType` | enum — OD-01 approved list (resolved 2026-10-22) | `nin_slip\|voters_card\|drivers_licence\|passport` — exactly these four; no others accepted |
| `storagePath` | string | Private path, e.g. `private/artisans/{id}/identity/{docId}.pdf` — **never a public URL** |
| `fileName`, `mimeType`, `sizeBytes`, `sha256` | | Integrity + validation record |
| `idNumberLast4` | string | Only last-4 in Firestore **[PROPOSED]**; full number captured only where the approved manual-review workflow demonstrably requires it — still never public |
| `fullNumber` | string (restricted) | Only if required by approved process; admin + auditor access, never returned to client except reviewer view |
| `nameOnDocument`, `expiryDate` | | Minimal necessary fields |
| `status` | mirrors identity scope | `pending_review\|verified\|rejected\|resubmission_required` |
| `uploadedAt`, `reviewedAt`, `reviewedBy`, `reviewNotes` (admin-only subfield) | | |

### 2.6 `artisans/{artisanId}/evidence/{evidenceId}` (supporting verification)

| Field | Type | Notes |
|---|---|---|
| `evidenceType` | `qualification\|licence\|portfolio\|experience\|reference\|admin_requested` | |
| `trade` | string | For per-trade policy evaluation (OD-03) |
| `storagePath` (+ `mimeType`, `sizeBytes`, `sha256` if file) | | Private path; portfolio images approved for public get copied/approved separately |
| `textContent` / `referenceContact` | | Experience statements; reference contacts (restricted like guarantor data) |
| `credentialChecked` | bool | True only after actual check (BR-6); drives `verifiedTradeBadges[]` |
| `status`, `reviewedAt`, `reviewedBy`, `reviewNotes(admin)` | | |

### 2.7 `artisans/{artisanId}/guarantor/{gId}` (highly restricted)

| Field | Type | Notes |
|---|---|---|
| `fullName` | string | **[PROPOSED required]** pending OD-04 |
| `phone` | string (E.164) | **[PROPOSED required]** — never public, never to artisan support staff beyond need-to-know |
| `relationship` | string | **[PROPOSED required]** |
| `extraInfo` | map | Only owner-approved additional fields (OD-04) |
| `contacted` | bool | Whether guarantor has been contacted (per approved process only) |
| `contactedAt`, `contactedBy`, `contactChannel`, `contactOutcome` | | Contact record |
| `status` | mirrors guarantor scope | `pending_review\|contacted_pending\|verified\|rejected\|resubmission_required` |
| `reviewNotes(admin)`, `reviewedAt`, `reviewedBy` | | |

Rules: **admin (verification role) + auditors only** for reads; artisan sees only `status` + guidance text via the
aggregate/outcome view, not this document. No liability language stored (FR-GUA-5).

### 2.8 `artisans/{artisanId}/consents/{consentId}`

| Field | Type | Notes |
|---|---|---|
| `type` | `guarantor_consent_declaration \| guarantor_independent_confirmation \| data_processing` | |
| `declaredBy` (artisan uid), `declarationText` (versioned), `declaredAt` | | Artisan's informed-consent declaration |
| `confirmationChannel`, `confirmedAt`, `confirmationRef` | | Independent confirmation, if OD-05 requires it |
| `policyVersion` | string | Pins which consent wording was used |

### 2.9 `artisans/{artisanId}/reviews/{reviewId}`

| Field | Type | Notes |
|---|---|---|
| `scope` | `identity\|additional\|guarantor\|final\|suspension` | |
| `decision` | `approve\|reject\|request_info\|suspend\|unsuspend` | |
| `reasonCode`, `notes` (admin-only), `guidanceForArtisan` (shareable) | | |
| `decidedBy`, `decidedAt` | | Reviewer + timestamp; mirrored to `auditLogs` |

Identity-scope reviews are **manual administrator decisions** (OD-02 resolved 2026-10-22). Every review record
captures reviewer (`decidedBy`), timestamp (`decidedAt`), outcome (`decision`), and reason (`reasonCode` + `notes`),
mirrored to append-only `auditLogs/*`. Review records must never assert automated, biometric, external-provider, or
government-database validation.

### 2.10 `bookings/{bookingId}`

| Field | Type | Notes |
|---|---|---|
| `customerId` | string | `== auth.uid` of creator; immutable; indexed |
| `artisanId` | string | Must reference a publicly visible artisan at creation |
| `status` | `pending\|confirmed\|completed\|cancelled\|declined` **[PROPOSED]** | Function-validated transitions |
| `serviceDescription`, `preferredDate`, `locationNote` | | Customer input, validated |
| `artisanSnapshot` | `{displayName, trade, phone(disclosure per OD-11)}` | Point-in-time copy |
| `statusHistory[]` | `{from,to,at,by}` | Append-only via Function |
| `createdAt`, `updatedAt` | timestamp | |

Rules: read iff `auth.uid == customerId` OR assigned artisan OR admin; create iff authed + `customerId == auth.uid` +
artisan publishable; status writes Function-only.

### 2.11 `notifications/{uid}/items/{nid}`

`{type, title, body, link, read, createdAt}` — recipient-read (+ system write). Verification/booking events.

### 2.12 `auditLogs/{logId}` (append-only)

`{actorUid, actorRole, action, targetPath, before, after, reason, at, requestId}` — Function-written; no client writes;
admin-auditor reads. Covers: decisions, transitions, private-file URL minting, consent records, admin grants.

### 2.13 `adminUsers/{uid}`

`{level, grantedBy, grantedAt, revokedAt}` — controls verification-role membership; bootstrap procedure in security plan.

---

## 3. Status Ownership & Transition Authority

| State | Writer | Reader |
|---|---|---|
| `paymentStatus` | Payment-webhook Function only | artisan (own), admin |
| `identity/additional/guarantor/account` statuses | Review-decision Function (admin auth) only | artisan sees own aggregate; sub-doc internals admin-only |
| `isPubliclyVisible` / `publicArtisans/*` | Publication Function only | public read |
| Booking `status` | Booking Function only | parties + admin |
| Private file bytes | artisan upload (write-once path), Function validation | short-lived URL via Function after authz |

---

## 4. Permitted Status Transitions (enforced server-side)

```
payment:      unpaid → pending → verified | failed ; verified → refunded (policy OD-09)
identity:     not_submitted → pending_review → verified | rejected | resubmission_required
              resubmission_required → pending_review (on resubmit, if OD-08 allows)
additional:   (same shape as identity) → approved instead of verified
guarantor:    not_submitted → pending_review → contacted_pending → verified | rejected | resubmission_required
account:      draft → in_review → approved | rejected | resubmission_required ; approved ⇄ suspended
booking:      pending → confirmed | declined | cancelled ; confirmed → completed | cancelled
```

Illegal transitions (e.g. artisan self-verify, publish without approval, skip contacted-state where policy requires
it) are rejected and logged. Identity-scope transitions are effected only by the authorized manual-review decision
Function (OD-02 resolved 2026-10-22; verification role, fresh token); applicant-invoked or direct-write transitions
are rejected and logged. Identity approval alone never publishes — the publication gate (PRD §3.8) requires payment
verified + additional approved + guarantor verified + final admin approval + not suspended.

---

## 5. Cloud Storage Layout

```
gs://<bucket>/
  private/
    artisans/{artisanId}/
      identity/{docId}                    — ID files (owner-write-once, no direct read)
      evidence/{evidenceId}               — supporting files
      guarantor/{gId}/declaration         — guarantor declaration file, if OD-06 requires
  public-approved/
    artisans/{artisanId}/portfolio/{file} — ONLY admin-approved portfolio copies (public read)
```

- Upload constraints (enforced in Storage rules + re-checked by Function): allowlist
  `image/jpeg, image/png, application/pdf` **[PROPOSED]**; ≤ 5 MB **[PROPOSED]**; path must match
  `auth.uid`; write-once (no overwrite of under-review files — resubmission creates a new version).
- Reads: `private/**` denies all direct reads; access only via Function-minted short-lived signed URLs
  (TTL **[PROPOSED]** 15 min), each mint audit-logged.
- Retention lifecycle (durations OD-10): rejected-applicant files quarantined then auto-deleted; see security plan §8.

---

## 6. Validation Rules Summary

- All writes: type checks, length caps (bio ≤ 500 chars **[PROPOSED]**, notes ≤ 2000 **[PROPOSED]**), enum membership.
- Phone numbers E.164; emails lowercased; timestamps server-set (`serverTimestamp()` / Function time).
- No sensitive field may appear in `publicArtisans/*` — enforced by a dedicated rules unit test + a Function-side
  allowlist serializer (defense in depth).
- Idempotency keys on payment webhooks, booking creates (client-generated UUID), and decision records.
