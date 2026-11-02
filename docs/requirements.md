# Restaurant Voice Agent Sandbox: Requirements (v0.2 draft)

Scope: Preparation phase only (19 Oct 2026 - 14 Jan 2027). Synthetic data only.

Labels used: [OFFICIAL] stated in a project document or confirmed by the
supervisor. [DECISION] agreed by the engineering owner. [RECOMMENDATION]
engineering suggestion. [ASSUMPTION] sandbox choice, to confirm with the
supervisor. [OPEN QUESTION] needs an answer. [MISSING INPUT] not available.

## 1. Context
- [OFFICIAL] (FYP request, internship agreement) The formal PFE is "Design and
  Development of an AI-Powered Voice Agent Platform for Automated Customer
  Service and Business Process Integration", planned 15 Jan - 15 May 2027.
- [OFFICIAL] (FYP request) A representative use case such as restaurant
  information and reservations validates the architecture. The restaurant is
  not the whole project.
- [OFFICIAL] (FYP request) AI reasoning is separated from deterministic
  business logic. Backend services control actions, validations and state
  changes.
- [OFFICIAL] (Roadmap) Preparation builds a restaurant sandbox: a reliable
  reservation service, then a tool-using agent, then a voice interface.
  Stack: Python/FastAPI, PostgreSQL, LangGraph, React/TypeScript, Docker
  Compose, RAG, an MCP read tool, speech-to-text and text-to-speech.
- [OFFICIAL] (Roadmap, Phase 1) Clarify who uses the system, which actions
  they can perform and which rules must always hold. Define opening hours,
  reservation slots, party-size limits, cancellation behavior and the
  handling of incomplete requests.
- [MISSING INPUT] The "first assignment" mentioned in the roadmap is not
  available. Nothing in this document is derived from it.

## 2. Confirmed project decisions
- D1: The private repo is the main development repo (branches, commits and
  reviewed PRs, no daily limit).
- D2: The supervisor's repo is a separate delivery destination, at most one
  coordinated delivery per day.
- D3: One restaurant for the initial scenario, with a tenant-aware direction.
  Multi-tenancy is a recommendation to validate, not an official requirement.
- D4: Customer identification and reservation lookup are separate from
  trusted tenant authentication and authorization.
- D5: Open company questions are documented. A safe labelled assumption lets
  preparation continue.
- D6: The first assignment is a missing input and is not invented.
- D7: Work is not marked complete without evidence.

## 3. Roles
- Customer: a person using the restaurant service (phone, later chat or web).
- Staff: authenticated restaurant employee.
- Agent: the AI system acting for a customer, using only approved tools.
- [OPEN QUESTION] Platform administrator: out of scope for now.

## 4. Trusted context and identity
Three separate things:
1. Trusted context (tenant and role). It comes from authentication, never
   from request fields or conversation text.
2. Customer contact details (name, phone). Data stored on a reservation.
   They are not credentials.
3. Reservation lookup. By reservation code, always inside the trusted
   tenant context.

- [RECOMMENDATION] Every service function receives a trusted context object.
  In the sandbox it is a fixed server-side setting until authentication is
  built (the roadmap prepares authentication boundaries in Phase 1 and
  completes permission denial in Phase 2).
- [RECOMMENDATION] A "tenants" table with one seeded tenant, and a tenant_id
  on tables and reservations. No tenant management, no per-tenant settings,
  no schema-per-tenant, no row-level security, no multiple locations.

| Operation | Requires |
|---|---|
| Restaurant info, availability | Tenant context |
| Create reservation | Tenant context, contact details, explicit confirmation |
| Customer view / modify / cancel | Tenant context, reservation code, phone match, confirmation, generic errors, attempt limits |
| Staff operations | Authenticated staff role |

- [ASSUMPTION] The "code + phone match" is a weak verification step for the
  sandbox. It is NOT full authorization and is not proof of tenant context.
- [OPEN QUESTION] How are callers verified in a real deployment?

## 5. Actions
1. Ask restaurant information (hours, menu, policies). Read-only.
2. Check availability (date, time, party size).
3. Create a reservation (after confirmation).
4. View a reservation.
5. Modify a reservation.
6. Cancel a reservation.

## 6. Sandbox business assumptions
- A1: Open every day 12:00-22:00. Last booking slot 21:00.
- A2: Slots every 30 minutes.
- A3: A reservation lasts 90 minutes.
- A4: Party size 1 to 8. More than 8 is escalated to staff.
- A5: 10 tables: four seats-2, four seats-4, two seats-6.
- A6: Bookings at least 1 hour ahead and at most 30 days ahead.
- A7: Cancellation allowed until 2 hours before the reservation.
- A8: Restaurant time zone Africa/Tunis (the roadmap uses it for scheduling).
- A9: A modification follows the same rules as a new booking and is atomic.
- A10: One seeded tenant ("Demo Restaurant"), one location.

## 7. Incomplete requests
To book: date, time, party size, name, phone. If anything is missing the
system asks. It never guesses.

## 8. Business invariants (must always hold)
- I1: A table is never booked for two overlapping time ranges.
- I2: Party size never exceeds table capacity and is between 1 and 8.
- I3: Reservations are inside opening hours and on the slot grid.
- I4: A reservation is never created or moved into the past.
- I5: Status is "confirmed" or "cancelled". Cancelled is final.
- I6: Every operation is authorized from trusted context (and, for
  customers, the verification step in section 4).
- I7: Sending the same create request twice creates one reservation.
- I8: Only the backend writes to the database. The agent has no direct
  access.
- I9: Every read and write is scoped to the tenant from the trusted context.

## 9. Failure cases to handle
- No availability (propose nearby times).
- Invalid input (bad date, party size 0 or 20, bad phone format).
- Wrong code or phone: one generic message that reveals nothing.
- Cancelling an already cancelled reservation.
- Too late to cancel.
- Duplicate request.
- Two customers booking the last table at the same moment.
- Backend or database unavailable: clear error, no partial save.

## 10. Open questions for the supervisor
- Q1: Is multi-tenancy wanted, and do tenants need several locations?
- Q2: How are real callers verified (caller ID, other)?
- Q3: How do staff authenticate?
- Q4: Which model, speech and telephony providers, and what scale?
- Q5: Will the work be remote or on site? Who is the school liaison?
- Q6: When is the weekly review meeting?
- Q7: Retention rules for customer phone numbers (personal data)?
- Q8: Is the "first assignment" available?

## 11. Out of scope for now
Payments, deposits, multiple locations, real customer data, SIP and
telephony, tenant management screens, administrator features.

## 12. Planned first slice (not started)
[RECOMMENDATION] Read-only availability: GET /availability with date, time and
party_size, using PostgreSQL tables and the tenant context. Acceptance:
free slot returns tables; full slot returns none plus nearby times; invalid
input returns a clear 4xx; automated tests pass. This starts only after this
document is reviewed.
