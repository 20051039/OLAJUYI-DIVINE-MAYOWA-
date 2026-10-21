# TECHFIX.NG — Implementation Backlog

**Version:** 2.0 (Revised) · **Date:** 2026-10-21 · **Status:** DRAFT — pending owner review and approval.

> **No application code until these documents are reviewed and approved.** When approved, work proceeds top-to-bottom
> by epic dependency. Every story is small, testable, and carries acceptance criteria. Global Definition of Done (DoD)
> is in §9; story-specific criteria add to it, never replace it.

Story ID key: `FND` foundations · `AUTH` auth/routing · `PUB` public directory · `ONB` artisan onboarding ·
`IDV` identity verification · `EVD` additional evidence · `GUA` guarantor · `PAY` payments · `ADM` admin console ·
`BOK` booking · `NTF` notifications · `SEC` security/privacy · `QA` test/launch hardening.

---

## Epic 1 — Foundations (Phase 0)

| ID | Story | Acceptance criteria | Depends |
|----|-------|---------------------|---------|
| FND-1 | Provision dev/staging/prod Firebase projects, Hosting, CI, emulators | All envs deploy from CI; emulator suite runs per PR | — |
| FND-2 | Secret management + provider sandbox accounts | Zero secrets in repo/client; sandbox keys in Secret Manager | FND-1 |
| FND-3 | Structured logging + alerting baseline | Auth/payment/transition logs queryable; webhook-failure alert fires in staging | FND-1 |
| FND-4 | Owner-decision workshop + decision log | OD-01…OD-12 resolved or formally deferred with defaults (see `09-owner-decisions.md`) | — (blocks exits) |

## Epic 2 — Auth, roles & route protection (Phase 1)

| ID | Story | Acceptance criteria | Depends |
|----|-------|---------------------|---------|
| AUTH-1 | Email/password register + login + email verification [PROPOSED] | Register/login/logout; weak-password + duplicate-email errors; verified flag mirrored | FND-1 |
| AUTH-2 | Custom-claims roles (`customer/artisan/admin`) + grant procedure | Claims set only by trusted code; no self-elevation (tested) | AUTH-1 |
| AUTH-3 | Intent-preserving auth gate (`?next=`, allowlisted) | T-01 passes; open-redirect attempts fall back safely | AUTH-1 |
| AUTH-4 | Client route guards per matrix §3.1 + 404-equivalence | Matrix suite green; unauthorized routes 404/403 without oracles | AUTH-2 |
| AUTH-5 | Session-expiry handling + draft preservation | Expired session mid-booking → re-auth → resume (tested) | AUTH-3 |

## Epic 3 — Public directory (Phase 1)

| ID | Story | Acceptance criteria | Depends |
|----|-------|---------------------|---------|
| PUB-1 | `publicArtisans` projection writer (Function) + publication-rule evaluator | Projection exists iff rule holds; regression unpublishes immediately (T-09…T-11) | AUTH-2 |
| PUB-2 | Public serializer allowlist + leak tests | Privacy-leak scan green; serializer unit tests pin the field set | PUB-1 |
| PUB-3 | Listing/search/filter UI + profile page | Search by trade/LGA; profile shows public-only fields; "Book Artisan" CTA gated (T-01) | PUB-1 |

## Epic 4 — Artisan onboarding: account → profile (Phase 2)

| ID | Story | Acceptance criteria | Depends |
|----|-------|---------------------|---------|
| ONB-1 | Artisan registration + role assignment | Artisan account created; cannot access customer-only or admin routes | AUTH-2 |
| ONB-2 | Profile builder (draft, validated, never public) | Draft persists; validation errors inline; nothing reaches public surfaces (T-04) | ONB-1 |
| ONB-3 | 11-step progress tracker | Steps reflect server state; payment ≠ published copy verified (T-09) | ONB-2 |

## Epic 5 — Identity-document submission (Phase 2)

| ID | Story | Acceptance criteria | Depends |
|----|-------|---------------------|---------|
| IDV-1 | Upload UI (doc-type selector per OD-01, requirements, progress, errors) | Copy matches approved doc list; errors actionable | ONB-2 |
| IDV-2 | Storage rules: private path, allowlist, 5 MB cap [PROPOSED], write-once | Emulator tests: valid writes pass; all T-05 cases denied | FND-1 |
| IDV-3 | Finalize Function: magic-byte sniff, hash, metadata, quarantine-on-fail | Spoofed/oversize files quarantined; metadata recorded; status → `pending_review` | IDV-2 |
| IDV-4 | Minimal ID fields capture (FR-IDV-5) + validation | Only approved fields stored; numbers never in public docs | IDV-1 |
| IDV-5 | Resubmission versioning | New version created; old quarantined; count tracked (policy OD-08) | IDV-3 |

## Epic 6 — Additional verification evidence (Phase 3)

| ID | Story | Acceptance criteria | Depends |
|----|-------|---------------------|---------|
| EVD-1 | Evidence capture UI (portfolio, experience, qualifications, licences, references) | All five types submittable with guidance | ONB-2 |
| EVD-2 | Per-trade mandatory/optional enforcement (OD-03 matrix) | Correct required/optional behavior per trade (tested incl. no-licence trades) | EVD-1 |
| EVD-3 | Credential-checked flags → public badges pipeline | Badges appear only from `credentialChecked=true` (BR-6 test) | PUB-1, EVD-2 |
| EVD-4 | Admin-requested-evidence loop | Admin can request extras; artisan notified; resubmit path works | ADM-2 |

## Epic 7 — Guarantor workflow (Phase 3)

| ID | Story | Acceptance criteria | Depends |
|----|-------|---------------------|---------|
| GUA-1 | Guarantor capture form (name, phone, relationship + OD-04 extras) | Validation; no liability language anywhere (copy test) | ONB-2 |
| GUA-2 | Consent-declaration record (versioned wording, timestamp) | Submission blocked until declaration recorded; record in `consents/*` | GUA-1 |
| GUA-3 | Guarantor doc rules: verification-role-only reads, artisan status-only | T-07 passes; emulator inventory updated | GUA-1 |
| GUA-4 | Independent-confirmation hooks behind policy flag (OD-05/06) | Flag off → skipped; flag on → required channel works (or deferred build if decided post-MVP) | GUA-2 |
| GUA-5 | Guarantor contact record fields + admin procedure checklist | Contact/outcome recordable only via approved process (T-15 scope) | ADM-2 |

## Epic 8 — Payments (Phase 4)

| ID | Story | Acceptance criteria | Depends |
|----|-------|---------------------|---------|
| PAY-1 | Provider integration interface + chosen provider (OD-07) | Sandbox charge succeeds end-to-end | FND-2 |
| PAY-2 | Webhook verification Function (signature, idempotency, replay) | Forged/replayed webhooks rejected; retries safe | PAY-1 |
| PAY-3 | `paymentStatus` transitions + receipt UX (verified ≠ published) | T-09 passes; receipt shows; no publish | PAY-2 |
| PAY-4 | Failure/retry UX + `failed` handling | Failed payments recoverable; no stuck `pending` without alert | PAY-2 |
| PAY-5 | Refund-status recording hooks (policy OD-09) | Statuses recordable; behavior matches policy flags | PAY-3 |

## Epic 9 — Admin review & approval console (Phase 5)

| ID | Story | Acceptance criteria | Depends |
|----|-------|---------------------|---------|
| ADM-1 | Review queues + filters (per-scope pending, resubmissions, flagged) | Queues reflect aggregate statuses; pagination | IDV-3, EVD-2, GUA-3 |
| ADM-2 | Decision Function (admin-only, fresh token, mandatory notes, audit) | T-15 passes; unauthorized/stale/incomplete attempts rejected + logged | AUTH-2 |
| ADM-3 | Short-lived private-file viewer + per-access audit | T-06 passes for unauthorized; authorized views logged; TTL enforced | IDV-3 |
| ADM-4 | Final-approval gating (all scopes verified + payment verified) | T-11 blocks; T-12 publishes (T-12…T-13 pass) | ADM-2, PUB-1 |
| ADM-5 | Reject / request-info flows + artisan notifications | T-13 passes; guidance-only copy (no internal notes leak) | ADM-2 |
| ADM-6 | Suspend / unsuspend + immediate unpublish + booking block | T-14 passes; OD-12 edge copy per policy | ADM-2 |
| ADM-7 | Admin provisioning + levels (OD-11) + access-review report | No self-grant; grants/revokes logged | AUTH-2 |

## Epic 10 — Customer booking (Phase 6)

| ID | Story | Acceptance criteria | Depends |
|----|-------|---------------------|---------|
| BOK-1 | Booking form + client validation + error states | Invalid inputs blocked inline with clear messages | PUB-3, AUTH-3 |
| BOK-2 | `createBooking` Function (auth, visibility snapshot, validation, idempotency) | T-02 passes; duplicates impossible; unpublished target rejected | BOK-1, PUB-1 |
| BOK-3 | Booking rules: owner/artisan/admin reads; Function-only status | T-03 passes; emulator inventory updated | BOK-2 |
| BOK-4 | Booking history + detail + status timeline | Own bookings listed with statuses; others 404 (T-03) | BOK-2 |
| BOK-5 | Transitions (confirm/decline/cancel/complete) + suspension block | State-machine suite + T-14 pass | BOK-2, ADM-6 |
| BOK-6 | Contact-disclosure rules (OD-11) | Exactly the approved fields visible per party (tested) | BOK-4 |

## Epic 11 — Notifications (Phases 5–6)

| ID | Story | Acceptance criteria | Depends |
|----|-------|---------------------|---------|
| NTF-1 | In-app notification center + triggers (verification + booking events) | Each PRD §3.10 event fires exactly once to correct parties | ADM-2, BOK-2 |
| NTF-2 | Email fan-out [PROPOSED] (+ SMS [PROPOSED] if approved) | Delivery in staging; no PII beyond approved content | NTF-1 |

## Epic 12 — Security, privacy & retention (continuous; exit Phase 7)

| ID | Story | Acceptance criteria | Depends |
|----|-------|---------------------|---------|
| SEC-1 | Rules/storage inventory + emulator suites in CI | Green on every PR; coverage of allow/deny matrix | FND-1 |
| SEC-2 | Upload-abuse + signed-URL-expiry suites | Threats T-04/T-08 mitigations evidenced | IDV-3, ADM-3 |
| SEC-3 | Privacy-leak scan automation (public crawl) | Zero sensitive matches; runs per PR + pre-launch | PUB-2 |
| SEC-4 | Retention jobs (quarantine → delete → certificate) per §8 schedule | Jobs run on fixtures; interim OD-10 values documented | IDV-3, GUA-3 |
| SEC-5 | Correction/deletion request flow (submit → track → outcome) | T-16 passes | ADM-2 |
| SEC-6 | Legal-review gates: privacy notice, consent wordings, guarantor declaration | Signed-off texts only; blockers tracked to OD-04/05/06/10 | — |
| SEC-7 | Nigerian data-protection confirmation memo (counsel/DPO) | §9 questions answered by qualified advice before launch | — |

## Epic 13 — Launch hardening (Phase 7)

| ID | Story | Acceptance criteria | Depends |
|----|-------|---------------------|---------|
| QA-1 | Full 16-test acceptance run on staging + prod-parity check | All pass; evidence archived | All above |
| QA-2 | Dependency/secret/PII audits + error-message review | Zero criticals; review signed | SEC-1…3 |
| QA-3 | A11y (AA) + performance budgets (NFR-3) | Public/booking/onboarding pass budgets | PUB-3, BOK-1, ONB-2 |
| QA-4 | Runbooks (incidents, webhooks, retention, restore drill) | Drill completed; on-call paths defined | FND-3 |
| QA-5 | Launch-gate audit (G-1…G-9) + owner sign-off | Gates evidenced; sign-off recorded | QA-1 |

---

## 9. Global Definition of Done (every story)

1. Acceptance criteria demonstrated on a review environment.
2. Rules/Function authorization tests added and green (no story weakens the allow/deny inventory).
3. Privacy check: no new PII collected, stored, or displayed beyond the approved schema; public serializer unaffected or re-tested.
4. Audit logging added for sensitive actions the story introduces.
5. Docs updated (this backlog checked off; affected PRD/architecture/schema sections revised).
6. No unresolved **[OWNER DECISION]** dependency without a recorded interim default + owner acknowledgment.
