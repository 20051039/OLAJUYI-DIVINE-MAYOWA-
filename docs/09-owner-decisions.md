# TECHFIX.NG — Owner Decisions Required

**Version:** 2.0 · **Date:** 2026-10-21 · **Status:** OPEN — none resolved.

> Concise list of unresolved decisions. This document (and the whole set) **invents no answers**.
> Each item records: the question, why it matters, the interim default the plan ships with (if any),
> and what it blocks. Resolve in the Phase 0 workshop or formally defer with a recorded default.

| ID | Decision | Why it matters / blocked work | Interim default (if deferred) |
|----|----------|-------------------------------|------------------------------|
| OD-01 | **Which identity documents are acceptable?** (Recommendation: NIN slip/card, voter's card, driver's licence, international passport — §3.4 PRD.) | Blocks IDV-1 UI/validation copy; Phase 2 exit. | Ship with the 4 recommended types clearly labeled "pending final approval"; do not accept others. |
| OD-02 | **Manual verification or external verification provider?** | Determines review operating model; any vendor integration is post-MVP. | **Manual admin review only** for MVP; make no automation claims. |
| OD-03 | **Which supporting documents are mandatory for each artisan trade?** (Proposed matrix: PRD §3.5.1.) | Blocks EVD-2 enforcement; Phase 3 exit. | Ship with the proposed matrix labeled interim; admin can always request extras. |
| OD-04 | **Guarantor policy:** eligibility criteria (age, relationship exclusions, limits), any extra fields beyond name/phone/relationship, and any agreement/declaration text (legal review required). | Blocks GUA-1/GUA-5 final copy; consent wording gate. | Collect name + phone + relationship only; no extra fields; no liability language; no guarantor contact beyond admin-recorded process. |
| OD-05 | **How is guarantor consent recorded — and must the guarantor independently confirm?** (e.g. OTP, callback, signed declaration.) | Blocks GUA-2/GUA-4 final UX; Phase 3 exit. | Artisan declaration only (versioned + timestamped); independent-confirmation hooks built behind a flag, off by default. |
| OD-06 | **Must guarantors provide identification or sign a declaration?** (Legal review required for any declaration.) | Blocks guarantor file handling + legal gates. | No guarantor ID/declaration collected; storage path reserved. |
| OD-07 | **Payment provider choice** (shortlist: Paystack / Flutterwave) **and: are there any verification fees beyond the ₦3,000 registration fee?** | Blocks PAY-1 integration; fee schedule copy. | Build against provider interface; assume **no** extra fees; sandbox until chosen. |
| OD-08 | **What happens on rejection — corrections, resubmission, appeal?** (Windows, limits, process.) | Blocks ADM-5 copy + resubmission policy flags; Phase 5 exit. | `resubmission_required` loop allowed with count tracking; no formal appeal channel; copy states policy is interim. |
| OD-09 | **Are refunds permitted, and on what terms?** (Eligibility, window, method — note: fee is paid *before* admin review.) | Blocks PAY-5 behavior + finance handling. | Record `refunded` status capability; issue no refunds until policy set; disclose clearly pre-payment. |
| OD-10 | **Retention periods** for identity documents, guarantor info, bookings, audit logs — plus data-subject request SLA. (Proposed schedule: security plan §8.) | Blocks SEC-4 final durations; SEC-5 SLA copy. | Ship with proposed interim values, documented as interim. |
| OD-11 | **Who can access sensitive verification records?** (Confirm access matrix + admin levels + customer/artisan contact-disclosure rules.) | Blocks ADM-7, BOK-6, GUA-3 final masks. | Most restrictive option: verification-role-only; minimal disclosure. |
| OD-12 | **What happens to existing bookings when an artisan is suspended?** (Complete, cancel, reassign?) | Blocks ADM-6/BOK-5 edge-case copy. | New bookings blocked immediately (mandatory); existing bookings frozen pending decision, with parties notified. |

## Decision log (fill as resolved)

| ID | Decision | Decided by / date | Effective in |
|----|----------|-------------------|--------------|
| — | *None yet* | — | — |

**Rule:** no production launch with an OD item neither resolved nor formally deferred with a recorded interim default
and owner sign-off (PRD acceptance criterion 5; launch-gate audit QA-5).
