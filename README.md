# TechFix.ng — Planning Documents

Product requirements, MVP roadmap, architecture, data design, user flows, security plan,
test plan, backlog, and owner-decision register for **TechFix.ng**, a Nigerian artisan
directory and booking marketplace.

> **Status: DRAFT v2.2 (2026-10-22)** — pending owner review and approval. v2.2 records owner approval of OD-01 (accepted ID types) and OD-02 (manual administrator review); OD-03…OD-12 remain open.
> **No application code is to be written until these documents are reviewed and approved.**

## Document index

| Doc | Contents |
|-----|----------|
| `docs/01-PRD.md` | Product Requirements Document — goals, roles, functional requirements (authenticated booking, ID verification, additional evidence, guarantor, 11-step workflow, payments, publication rule), NFRs |
| `docs/02-mvp-roadmap.md` | MVP definition, 9 launch gates, phased plan (0–7), decision dependencies |
| `docs/03-architecture.md` | System, authentication, and route-protection architecture; ADRs |
| `docs/04-database-schema.md` | Firestore collections, status state machines, Cloud Storage layout, validation rules |
| `docs/05-user-flows.md` | Visitor, customer, artisan (11-step), and administrator journeys |
| `docs/06-security-privacy-plan.md` | Threat model, rules strategies, access control, retention schedule, NDPA confirmation questions |
| `docs/07-test-plan.md` | 16 required acceptance tests with expected results + supporting suites |
| `docs/08-implementation-backlog.md` | 13 epics / ~50 stories with acceptance criteria, dependencies, Definition of Done |
| `docs/09-owner-decisions.md` | Owner decisions: OD-01 + OD-02 resolved 2026-10-22; OD-03…OD-12 open — no invented answers |
| `docs/10-revision-summary.md` | Summary of changes, conflicts identified, remaining decisions, next steps |

## Preserved business rules

- One-time **₦3,000** artisan registration fee (server-verified).
- **Mandatory administrator approval** before any public visibility.
- **No customer booking without authentication** (public browsing needs no account).
- Upload ≠ genuineness · no unchecked certification claims · no assumed guarantor liability.
