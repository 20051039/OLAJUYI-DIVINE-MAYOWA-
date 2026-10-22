# TECHFIX.NG — Owner Decisions Required

**Version:** 2.1 · **Date:** 2026-10-22 · **Status:** PARTIALLY RESOLVED — OD-01 and OD-02 resolved 2026-10-22; OD-03…OD-12 open.
**Changelog (2.1):** Recorded owner approval of OD-01 (accepted ID document types) and OD-02 (manual administrator
review for MVP). No other decisions resolved; no new business policies invented.

> Decision register. This document (and the whole set) **invents no answers** — resolved items record only what the
> owner approved, with date and source; open items keep their interim defaults.
> Each item records: the question, why it matters, and the approved policy or interim default.
> Resolve remaining items in the Phase 0 workshop or formally defer with a recorded default.

| ID | Status | Decision | Why it matters / blocked work | Approved policy or interim default |
|----|--------|----------|-------------------------------|-----------------------------------|
| OD-01 | ✅ RESOLVED 2026-10-22 | **Which identity documents are acceptable?** | Unblocked IDV-1 UI/validation copy; Phase 2 doc-type exit criterion met. | **Approved:** NIN slip/card, voter's card, driver's licence, international passport. No other documents accepted. (Upload file-type/size limits remain [PROPOSED] engineering constraints, not decided policy.) |
| OD-02 | ✅ RESOLVED 2026-10-22 | **Manual verification or external verification provider?** | Sets the review operating model; manual-review implementation (IDV-6, ADM-2, T-15/T-17/T-18) unblocked. | **Approved policy:** the MVP uses **manual administrator review** of artisan identity documents. **No automated, biometric, external-provider, or official government identity-verification claims may be made.** Manual review must never be presented as proof of government-database validity or biometric identity. Any future verification-service integration is post-MVP and requires a new owner decision. |
| OD-03 | OPEN | **Which supporting documents are mandatory for each artisan trade?** (Proposed matrix: PRD §3.5.1.) | Blocks EVD-2 enforcement; Phase 3 exit. | Ship with the proposed matrix labeled interim; admin can always request extras. |
| OD-04 | OPEN | **Guarantor policy:** eligibility criteria (age, relationship exclusions, limits), any extra fields beyond name/phone/relationship, and any agreement/declaration text (legal review required). | Blocks GUA-1/GUA-5 final copy; consent wording gate. | Collect name + phone + relationship only; no extra fields; no liability language; no guarantor contact beyond admin-recorded process. |
| OD-05 | OPEN | **How is guarantor consent recorded — and must the guarantor independently confirm?** (e.g. OTP, callback, signed declaration.) | Blocks GUA-2/GUA-4 final UX; Phase 3 exit. | Artisan declaration only (versioned + timestamped); independent-confirmation hooks built behind a flag, off by default. |
| OD-06 | OPEN | **Must guarantors provide identification or sign a declaration?** (Legal review required for any declaration.) | Blocks guarantor file handling + legal gates. | No guarantor ID/declaration collected; storage path reserved. |
| OD-07 | OPEN | **Payment provider choice** (shortlist: Paystack / Flutterwave) **and: are there any verification fees beyond the ₦3,000 registration fee?** | Blocks PAY-1 integration; fee schedule copy. | Build against provider interface; assume **no** extra fees; sandbox until chosen. |
| OD-08 | OPEN | **What happens on rejection — corrections, resubmission, appeal?** (Windows, limits, process.) | Blocks ADM-5 copy + resubmission policy flags; Phase 5 exit. | `resubmission_required` loop allowed with count tracking; no formal appeal channel; copy states policy is interim. |
| OD-09 | OPEN | **Are refunds permitted, and on what terms?** (Eligibility, window, method — note: fee is paid *before* admin review.) | Blocks PAY-5 behavior + finance handling. | Record `refunded` status capability; issue no refunds until policy set; disclose clearly pre-payment. |
| OD-10 | OPEN | **Retention periods** for identity documents, guarantor info, bookings, audit logs — plus data-subject request SLA. (Proposed schedule: security plan §8.) | Blocks SEC-4 final durations; SEC-5 SLA copy. | Ship with proposed interim values, documented as interim. |
| OD-11 | OPEN | **Who can access sensitive verification records?** (Confirm access matrix + admin levels + customer/artisan contact-disclosure rules.) | Blocks ADM-7, BOK-6, GUA-3 final masks. | Most restrictive option: verification-role-only; minimal disclosure. |
| OD-12 | OPEN | **What happens to existing bookings when an artisan is suspended?** (Complete, cancel, reassign?) | Blocks ADM-6/BOK-5 edge-case copy. | New bookings blocked immediately (mandatory); existing bookings frozen pending decision, with parties notified. |

## Resolved-decision implementation notes

- **OD-01:** build the doc-type selector, validation, and metadata enum around exactly the 4 approved types
  (PRD FR-IDV-4; schema §2.5; backlog IDV-1). Accepting any other document type requires a new owner decision.
- **OD-02:** build the identity scope as a human review queue with authorized server-side decision actions and
  reviewer/timestamp/outcome/reason audit records (arch §4.3; schema §2.9/§4; security §5/§7; backlog IDV-6, ADM-2).
  No verification vendor SDK, biometric check, or government-database lookup is in scope; no product copy, badge,
  notification, or record may imply one (wording audited by T-18). Identity approval stays separate from payment,
  supporting evidence, guarantor, overall admin approval, and suspension states, and never publishes alone.

## Decision log

| ID | Decision | Decided by / date | Effective in |
|----|----------|-------------------|--------------|
| OD-01 | Accepted ID types: NIN slip/card, voter's card, driver's licence, international passport — no others | Owner / 2026-10-22 | PRD FR-IDV-4; schema `documentType`; backlog IDV-1 |
| OD-02 | MVP uses manual administrator review; no automated/biometric/external-provider/government verification claims; manual review never presented as proof of government-database validity or biometric identity | Owner / 2026-10-22 | PRD FR-IDV-7/FR-IDV-10; arch §4.3 + ADR-5; schema §2.9/§4; security §5/§7; tests T-15/T-17/T-18; backlog IDV-6 |

**Rule:** no production launch with an OD item neither resolved nor formally deferred with a recorded interim default
and owner sign-off (PRD acceptance criterion 5; launch-gate audit QA-5).
