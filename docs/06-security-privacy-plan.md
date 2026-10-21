# TECHFIX.NG — Security, Privacy & Data-Retention Plan

**Version:** 2.0 (Revised) · **Date:** 2026-10-21 · **Status:** DRAFT — pending owner review and approval.

---

## 1. Threat Model (revised for verification & guarantor data)

| # | Threat | Assets at risk | Mitigations (required for launch) | Test |
|---|--------|----------------|-----------------------------------|------|
| T-01 | Unauthorized access to identity documents | ID files, ID numbers | Private paths; deny-all direct reads; short-lived URLs via authz Function; per-access audit | AT-06 |
| T-02 | Unauthorized access to guarantor information | Guarantor PII | Verification-role-only reads; no client access to guarantor docs; audit logging | AT-07 |
| T-03 | Forged/manipulated uploaded documents | Verification integrity | Server MIME sniffing, allowlist, size caps, hash record; manual review checklist; no genuineness claim (BR-5) | AT-05 + review tests |
| T-04 | Malicious file uploads (malware, polyglots, bombs) | Platform + reviewers | Allowlist `jpg/png/pdf` + 5 MB cap **[PROPOSED]**; magic-byte check; no executable handling; reviewer sandbox guidance; AV scan post-MVP **[PROPOSED]** | Upload-abuse suite |
| T-05 | Excessive PII collection | Data subjects | Field minimization (PRD FR-IDV-5/FR-GUA-1); no extra fields without approval; form audits | Schema review gate |
| T-06 | Unauthorized administrator access / rogue admin | All private data | Admin-grant procedure (§6.3); least-privilege levels; fresh-token decisions; full audit trail; periodic access review **[PROPOSED]** | Admin-authz tests |
| T-07 | Verification-status tampering (self-approval, self-publish) | Directory integrity | Function-only transitions; rules deny direct writes; transition allowlist enforced + logged | AT-04, AT-08 |
| T-08 | Leaked document links | ID/evidence files | Short-lived signed URLs (TTL 15 min **[PROPOSED]**); no public URLs for private files; expiry + revocation tests | URL-expiry tests |
| T-09 | Indefinite retention of rejected applicants' data | Data subjects | Retention schedule (§8; durations OD-10); quarantine → auto-delete jobs; deletion certificates in audit log | Retention tests |
| T-10 | Cross-customer booking access | Customer privacy | Rules + Function ownership checks; 404-equivalence (no oracles) | AT-03 |
| T-11 | Payment-state forgery | Revenue integrity | Webhook signature verification; idempotent server-side transitions; no client-writable payment fields | Payment tests |
| T-12 | Session / auth attacks | Accounts | Firebase-managed sessions; allowlisted `next`; rate-limited auth endpoints; email verification **[PROPOSED]** | Auth tests |

---

## 2. Firestore Rules Strategy

- **Default deny.** Every collection starts closed; each granted operation is justified in a rules inventory.
- Ownership pattern: `request.auth.uid == resource.data.<ownerField>` for customer/artisan reads of own data.
- **No direct writes** to: `publicArtisans/*`, all status fields in `applications/*`, `reviews/*` decisions,
  `auditLogs/*`, booking `status`, guarantor docs (artisan client has zero access), admin roster.
- Admin reads/writes gated on verified custom claim `role == "admin"` (+ `adminLevel` where OD-11 tiers apply),
  checked in rules **and** re-checked in Functions.
- Field masks: artisan reads of own application exclude admin-internal subfields (separate outcome-view docs or
  Function-mediated reads — implementation backlog item; the rule is "internal notes never reach the artisan client").
- Emulator-tested: every allow/deny case in `07-test-plan.md` has a rules unit test.

## 3. Cloud Storage Rules Strategy

- `private/**`: **no direct reads by anyone** (not even the owner — reads go through the URL-minting Function
  after authorization, enabling audit + expiry). Writes: owner artisan only, path-bound to `auth.uid`,
  `request.resource.size ≤ 5 MiB` **[PROPOSED]**, `contentType ∈ {image/jpeg, image/png, application/pdf}`.
- `public-approved/**`: public reads; writes Function-only (approved portfolio copies).
- Write-once semantics for under-review files (resubmission = new versioned object; old versions quarantined per §8).
- CORS locked to the app's origins; no wildcard.

## 4. Cloud Functions Authorization

- All callables: `requireAuth` → `requireRole` → input schema validation → state-machine check → write → audit.
- Webhooks: provider signature verification, timestamp tolerance, idempotency keys, replay protection.
- Secrets via Secret Manager; least-privilege service accounts per function group **[PROPOSED]**.
- Structured denial logging (rule/validation failures) feeding anomaly alerts.

## 5. File-Validation Pipeline (defense in depth)

1. Client pre-check (UX): extension + size + type hint.
2. Storage rules: size cap, content-type allowlist, path binding.
3. Function trigger on finalize: magic-byte sniffing, re-measure size, compute SHA-256, quarantine on mismatch,
   record metadata; delete + notify on failure with safe error text.
4. Admin reviewer guidance: view in sandboxed preview; never download to unmanaged devices **[PROPOSED policy]**;
   report suspected forgeries via reason codes (no automated genuineness verdict — BR-5).

## 6. Access Control & Administration

### 6.1 Who can access sensitive verification records (default — OD-11 confirms)

| Data | Artisan (own) | Customer | Admin (verification role) | Admin (support) | Auditor |
|---|---|---|---|---|---|
| ID files + numbers | Upload + status only | — | ✅ (logged) | — | metadata only |
| Supporting evidence | Upload + status only | — | ✅ (logged) | — | metadata only |
| Guarantor PII | Status + own submitted copy* | — | ✅ (logged) | — | metadata only |
| Reviewer notes | — (guidance text only) | — | ✅ | — | ✅ |
| Bookings | Received only | Own only | Support cases (logged) | ✅ (logged) | metadata only |
| Audit logs | — | — | — | — | ✅ |

`*` The artisan necessarily knows what they typed; the rule is they cannot *retrieve* the stored guarantor doc
after submission except via a fresh authenticated outcome view showing status only. Final shape confirmed in OD-11.

### 6.2 Admin levels **[PROPOSED]** (confirm in OD-11)

`support` (bookings, profiles) · `verifier` (+ verification materials, decisions) · `superadmin` (+ admin grants,
suspension, retention actions). MVP may start with verifier+superadmin only.

### 6.3 Admin provisioning (no self-appointment)

1. First superadmin bootstrapped out-of-band (documented ceremony, two-person integrity **[PROPOSED]**).
2. Further grants via superadmin-only Function; every grant/revoke audit-logged with justification.
3. Quarterly access review **[PROPOSED]**; dormant accounts revoked.

## 7. Privacy Controls

- **Minimization:** collect only FR-IDV-5 / FR-GUA-1 fields; portfolio images screened for incidental PII
  (visible addresses/plates blurred or rejected **[PROPOSED]**).
- **Consent records:** versioned consent artefacts (`consents/*`); guarantor consent declaration mandatory before
  guarantor submission; no guarantor contact outside the approved process.
- **Disclosure discipline:** public serializer allowlist (tested); error-message review (no PII in errors/logs);
  support tooling masks PII by default **[PROPOSED]**.
- **Data-subject rights:** correction/deletion request flow (user flows §2.4), honored subject to retention
  obligations; response SLA **[OWNER DECISION — OD-10]**.
- **No liability language** about guarantors anywhere in product copy or records (FR-GUA-5) until legal review passes.

## 8. Retention & Secure Deletion (schedule durations OD-10; mechanism mandatory)

| Data class | Proposed retention (all [PROPOSED] — OD-10 sets final) | Deletion method |
|---|---|---|
| Approved artisan: ID/evidence/guarantor | Duration of active listing + **[PROPOSED]** 12 months after delisting | Crypto-erasure/overwrite + audit certificate |
| Rejected/withdrawn applicants: ID/evidence/guarantor | **[PROPOSED]** 90 days quarantine (appeal window, OD-08/09) then delete | Same as above |
| Resubmission superseded files | **[PROPOSED]** 30 days then delete | Same |
| Bookings (completed/cancelled) | **[PROPOSED]** 24 months for dispute handling | Anonymize parties, keep aggregates |
| Audit logs | **[PROPOSED]** 36 months, tamper-evident | Restricted purge with dual control |
| Notifications | **[PROPOSED]** 12 months | Routine purge |
| Backups | Aligned to above (oldest restorable backup ≤ longest retention + cycle) | Expire backups accordingly |

Jobs run on schedule; every deletion writes a certificate entry to the audit log. **No sensitive documents are
retained indefinitely.** Until OD-10 is resolved, ship with these proposed values documented as interim.

## 9. Nigerian Data-Protection Requirements (confirmation required)

The following are **flagged for confirmation against current authoritative sources and qualified advice** —
this plan states no legal conclusions:

1. Applicability of the **Nigeria Data Protection Act (NDPA) 2023** and current **NDPC** guidance to TechFix.ng
   as a data controller/processor, including registration/filing and Data Protection Officer expectations.
2. Lawful bases for processing artisan ID data, guarantor third-party data, and customer booking data
   (consent design, contract, legitimate interests).
3. Cross-border transfer implications of the chosen cloud region(s) (data residency expectations).
4. Data-subject rights procedures and response timelines under current NDPC guidance.
5. Breach-notification obligations and timelines.
6. Sector-specific rules (if any) for handling NIN data and guarantor personal data.
7. Record-keeping (ROPA-equivalent) and privacy-notice content requirements.

**Action:** engage qualified Nigerian counsel/DPO advisor before production launch; track as backlog blocker.
The in-product privacy notice, consent wordings, and guarantor declaration text are **legal-review gates** —
no placeholder legal language ships to production.

## 10. Security Test & Review Gates (map to `07-test-plan.md`)

- Rules/storage emulator suites green on every PR; upload-abuse + URL-expiry suites in CI.
- Pre-launch: dependency audit, secrets scan, PII-leak scan of public surfaces, error-message review,
  admin-access review, backup/restore drill, incident-response runbook walkthrough.
- Post-launch **[PROPOSED]**: annual penetration test; quarterly access + retention-compliance review.
