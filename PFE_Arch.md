# Production-Minded Architecture and Engineering Principles
## Multi-Tenant AI-Powered Restaurant Reservation Platform

## 0. Fundamental instruction — Read this first

**We should not design a toy restaurant reservation system and then try to make it production-ready. We should design a production-minded reservation system, then implement it progressively through small vertical slices.**

The target is a real, commercially deployable, multi-tenant SaaS platform that could eventually serve independent restaurant businesses.

We must think about production requirements from the beginning: tenant isolation, data integrity, authorization, concurrency, security, reliability, observability, recoverability, maintainability and operational behavior.

However, **production-minded architecture does not mean implementing every production feature immediately.** We must distinguish between:

- **Target architecture:** the system we are designing to grow into.
- **Current implementation scope:** the capabilities we are implementing in the current development phase.
- **Future capabilities:** features that require additional business decisions, infrastructure or integrations.

Build the foundation correctly, then implement it incrementally.

Do not introduce unnecessary complexity, microservices, agents, infrastructure or abstractions without a concrete requirement. Prefer simple, modular and well-separated components that can evolve.

Claude is responsible for helping define requirements, architecture, implementation tasks and precise instructions for Cursor. Cursor implements bounded tasks. The engineering owner must understand, inspect, test and validate every change.

---

# 1. Product vision and system boundaries

## 1.1 Product vision

We are designing a **multi-tenant SaaS reservation and customer-service platform**.

YAISH Intelligence operates the platform. Independent restaurants subscribe to or use the platform, each with its own configuration, operational data, users and reservation workflows.

The platform initially uses restaurant reservation management as its representative business domain.

The long-term platform should support several customer interaction channels:

- AI-powered voice calls.
- Website or web application.
- Staff-facing management interface.
- Potentially other authorized integrations and channels.

Customers should be able to interact naturally with the system, while restaurant staff retain control over operational decisions and exceptions.

## 1.2 Multi-tenant SaaS model

This is a shared platform serving multiple independent restaurant businesses.

For example:

- Restaurant A has its own tables, opening hours, customers, reservations and policies.
- Restaurant B has its own tables, opening hours, customers, reservations and policies.
- Both use the same platform, but neither can access or affect the other's data.

The application may share infrastructure and application services, but tenant data and permissions must be strictly isolated.

The architecture must support:

- A tenant representing a restaurant business or organization.
- Multiple restaurant locations belonging to one tenant, when required.
- Independent configuration for each restaurant location.
- Users belonging to one or more authorized tenants.
- Tenant-specific staff roles and permissions.
- Tenant-scoped reservations, customers, tables, policies and operational records.

**Important:** Multi-tenancy is a security and data-model requirement, not merely a configuration feature.

Every tenant-owned business record must have a clear tenant ownership boundary. Every request must execute within an authenticated and authorized tenant context.

A tenant identifier supplied by an untrusted client or inferred from conversation text must never be sufficient to grant access.

The system must derive and validate tenant context through trusted authentication and authorization mechanisms.

We may begin with one restaurant tenant in synthetic data, but the data model and service boundaries must not assume that only one tenant will ever exist.

## 1.3 Product boundaries

Separate three responsibilities:

1. **YAISH platform:** manages tenants, platform-level users, platform configuration and shared infrastructure.
2. **Restaurant business system:** manages restaurant-specific operations, reservations, tables, policies and staff.
3. **AI interaction layer:** understands customer requests and invokes authorized business operations through controlled tools.

The AI must never become the source of truth for business data.

---

# 2. Architectural principles

These principles are mandatory design constraints.

### 2.1 Backend is the source of truth

The backend and database determine:

- Whether a reservation exists.
- Whether a time is available.
- Whether a booking is valid.
- Whether a customer is authorized to access a reservation.
- Whether a modification or cancellation is permitted.
- Whether an operation succeeded.

The AI may interpret requests, collect missing information and select authorized tools. It cannot independently decide that a business operation succeeded.

### 2.2 Business logic must be independent of the AI

Reservation rules must live in deterministic backend/domain services.

The same reservation service must be usable by:

- The AI agent.
- The staff interface.
- The customer-facing application.
- Future integrations.

A booking must behave identically regardless of which channel initiated it.

### 2.3 Explicit contracts between components

Components communicate through defined interfaces and structured data.

Avoid hidden dependencies, direct database access from unrelated components and implicit assumptions about response formats.

### 2.4 Fail safely

A timeout, interrupted conversation, model failure or lost response must never silently produce an incorrect reservation state.

If an operation's outcome is uncertain, the system must reconcile its actual result before retrying or reporting success.

### 2.5 Security is part of the architecture

Authentication, authorization, tenant isolation, input validation, secret management and auditability must be considered from the beginning.

Do not postpone the design of security boundaries until the application is already built.

### 2.6 Small vertical slices

Each implementation task should deliver a coherent, testable behavior across the necessary layers.

For example, an availability slice may include:

- An API endpoint.
- Request validation.
- A business service.
- A database query.
- Tenant authorization.
- A structured response.
- Tests.

Do not build a large collection of disconnected backend files, then frontend files, then agent files and only integrate everything at the end.

---

# 3. Target logical architecture

The target architecture consists of the following logical components.

## 3.1 Client and interaction layer

Responsibilities:

- Customer-facing web interfaces.
- Staff-facing management interface.
- Voice interaction entry point.
- Future authorized external channels.

The client must not directly access the database or bypass backend business rules.

It communicates with backend APIs using authenticated requests.

## 3.2 Voice processing layer

Responsibilities:

- Receive audio.
- Speech-to-text (STT).
- Handle partial and final transcripts.
- Detect silence and interruptions.
- Pass recognized text and conversation events to the conversational layer.
- Convert the final response into audio using text-to-speech (TTS).
- Manage audio-session lifecycle and provider failures.

Voice processing must be separated from reservation business logic.

A voice provider failure must not corrupt an existing reservation.

The voice layer must distinguish between:
- What the customer actually said.
- What the speech recognition system transcribed.
- What the AI interpreted.
- What the backend ultimately accepted.

## 3.3 Conversational orchestration layer

Use a controlled conversational workflow, initially implemented with LangGraph if it remains appropriate.

Responsibilities:

- Understand the customer's intent.
- Maintain conversation context.
- Identify missing information.
- Ask clarification questions.
- Retrieve trusted restaurant information when appropriate.
- Select permitted tools.
- Request confirmation before consequential operations.
- Handle tool errors and uncertain outcomes.
- Escalate to a human when necessary.

The orchestration layer must not contain the authoritative reservation rules.

The agent must not access the production database directly.

It must use explicitly authorized backend tools or APIs.

## 3.4 Business API layer

The backend, initially implemented with FastAPI, exposes controlled business operations.

Responsibilities:

- Authentication and authorization.
- Tenant and restaurant context resolution.
- Input validation.
- Business-rule enforcement.
- Request orchestration.
- Consistent error responses.
- Transaction management.
- Idempotency handling.
- Audit events.
- Integration with persistence and external services.

The API must be usable independently of the AI.

The manual application must be able to perform reservation operations with the LLM path disabled.

## 3.5 Domain and business-service layer

This is the core of the reservation system.

It contains deterministic operations such as:

- Checking availability.
- Creating reservations.
- Retrieving reservations.
- Modifying reservations.
- Cancelling reservations.
- Managing table assignments.
- Enforcing booking policies.
- Managing reservation lifecycle transitions.

Business services should not depend on the LLM, prompts, voice provider or frontend implementation.

They should receive validated, structured inputs and return explicit results.

## 3.6 Persistence layer

Use PostgreSQL as the initial relational database.

It stores authoritative business data, including:

- Tenants.
- Restaurant locations.
- Users and role assignments.
- Restaurant configuration.
- Tables and table availability.
- Customers.
- Reservations.
- Reservation references.
- Reservation lifecycle information.
- Audit records.
- Idempotency records.
- Relevant integration and operational state.

Use migrations to evolve the schema.

Enforce important data relationships and integrity rules in the database wherever possible, not only in application code.

## 3.7 Knowledge retrieval layer

Use RAG only for information that benefits from retrieval, such as:

- Menu descriptions.
- Restaurant policies.
- Services and amenities.
- Frequently asked questions.
- Other restaurant knowledge documents.

Retrieved content is informational, not authoritative for operational state.

For example:

- RAG may explain the cancellation policy.
- The backend determines whether a particular reservation can actually be cancelled.
- RAG may describe opening hours.
- The restaurant's authoritative configuration determines whether a requested time is operationally valid.
- RAG must never determine live table availability.

Retrieved content must be tenant-scoped, access-controlled and treated as untrusted input.

Where appropriate, responses should retain source references so the system can identify where an answer came from.

## 3.8 Integration layer

External services may include:

- STT/TTS providers.
- Telephony providers.
- SMS or email providers.
- Payment providers, if introduced.
- External business systems.
- MCP-connected tools or other authorized integrations.

External integrations must be isolated behind controlled interfaces.

A provider outage must not silently corrupt internal business state.

Each integration should define:
- Authentication.
- Request and response contracts.
- Timeout behavior.
- Retry policy.
- Error mapping.
- Idempotency or duplicate-event handling, where applicable.
- Logging and monitoring.

## 3.9 Observability and operations layer

The platform must provide enough visibility to understand what happened during a customer interaction and a business operation.

Include:

- Structured application logs.
- Metrics.
- Distributed tracing or correlation identifiers.
- Error tracking.
- Health checks.
- Agent/tool execution traces.
- Latency measurements.
- Integration failure visibility.
- Audit history for important business changes.

Do not log unnecessary personal information, authentication secrets or sensitive conversation content.

---

# 4. User roles and permissions

Define permissions explicitly. Do not rely on frontend visibility or AI instructions for access control.

## 4.1 Customer

A customer may:

- Request restaurant information.
- Check availability.
- Request a reservation.
- Retrieve their reservation.
- Modify their reservation, subject to policy.
- Cancel their reservation, subject to policy.
- Request human assistance.
- Provide preferences or special requests.

A customer must only access reservations they are authorized to access.

Customer identity verification must be defined independently of the AI conversation.

## 4.2 Restaurant staff

Depending on assigned permissions, staff may:

- View reservations for their authorized location.
- Search reservations.
- Create reservations manually.
- Modify or cancel reservations.
- Confirm pending reservations.
- Assign or change tables.
- Register walk-ins.
- Check customers in.
- Mark no-shows.
- Handle operational exceptions.

Staff actions must be authenticated, authorized and auditable.

## 4.3 Restaurant administrator

May manage authorized restaurant-level configuration, such as:

- Restaurant locations.
- Staff accounts and roles.
- Opening hours.
- Closures and special schedules.
- Tables and seating configuration.
- Reservation policies.
- Approved operational overrides.
- Restaurant information and knowledge content.
- Notification settings.

## 4.4 Platform administrator

May manage platform-level operations, such as:

- Tenant onboarding and suspension.
- Platform-wide configuration.
- Platform operational health.
- Authorized support access.
- Shared integration configuration.

Platform administration must not create unrestricted access to restaurant customer data.

Any exceptional support access to tenant data must be explicitly authorized, limited, logged and reviewable.

## 4.5 AI agent

The agent is not a privileged user.

It receives only the permissions of its authorized integration identity and the specific tools made available to it.

It cannot:
- Bypass customer verification.
- Access another tenant's records.
- Perform staff-only actions using customer permissions.
- Override restaurant policies.
- Write directly to the database.
- Claim a mutation succeeded without backend confirmation.

---

# 5. Actions and business capabilities

The following describes the target product capabilities, not a requirement to implement all of them immediately.

## 5.1 Customer actions

1. Get restaurant information.
2. Check availability.
3. Explore alternative dates or times.
4. Create a reservation request.
5. Confirm a reservation.
6. Retrieve an existing reservation.
7. Modify an existing reservation.
8. Cancel a reservation.
9. Check reservation status.
10. Request human assistance.
11. Join a waiting list, if supported.
12. Provide additional information such as accessibility needs or special requests.

Special requests must not be presented as guaranteed unless the restaurant has explicitly accepted them.

## 5.2 Staff actions

1. View and search reservations.
2. Create reservations manually.
3. Modify and cancel reservations.
4. Confirm pending reservations.
5. Assign and reassign tables.
6. Register walk-ins.
7. Check customers in.
8. Mark no-shows.
9. Manage restaurant schedules and exceptional closures.
10. Manage table inventory and availability.
11. Handle authorized operational overrides.
12. View relevant operational history and statistics.

## 5.3 Platform and restaurant administration

Provide controlled management of tenants, locations, users, permissions, restaurant configuration, integrations and audit history.

## 5.4 AI agent actions

The agent may:

1. Interpret customer intent.
2. Retrieve trusted restaurant information.
3. Collect missing booking information.
4. Check availability through an authoritative tool.
5. Present available options.
6. Ask for explicit confirmation.
7. Submit a booking request through an authorized tool.
8. Retrieve or modify a reservation after authorization.
9. Cancel a reservation after authorization.
10. Communicate verified backend results.
11. Escalate unsupported or sensitive cases.

For consequential operations, the agent must show or repeat the exact details being confirmed.

If any relevant detail changes after confirmation, obtain confirmation again.

---

# 6. Assumptions and configurable business policies

A critical design rule:

**Do not hardcode restaurant-specific business policies as universal platform rules.**

Separate:

1. Platform-level guarantees.
2. Restaurant-specific configuration.
3. Business policies that still require confirmation from Khaled or an actual restaurant client.

The following are proposed requirements for the target system. They are not all confirmed business decisions.

## 6.1 Restaurant and tenant configuration

- Each restaurant business belongs to a tenant.
- A tenant may have one or multiple restaurant locations.
- Each location has its own timezone, operating schedule, table inventory and booking policies.
- Opening hours may vary by weekday, service period and special date.
- Restaurants may have temporary closures, holidays or exceptional schedules.
- Restaurant information must be configurable without changing application code.

## 6.2 Table and seating configuration

- Physical tables have identifiers, capacities and operational status.
- Tables may belong to sections or seating areas.
- Tables may be unavailable because of maintenance, events or operational restrictions.
- Table combinations are allowed only when explicitly configured.
- The system must not assume arbitrary table combinations are possible.
- Reservations may initially be unassigned to a specific table if the restaurant's allocation model supports capacity-based booking.
- If a table is assigned, its capacity and availability must be valid for the complete reservation interval.

## 6.3 Reservation policies

The following should be configurable per restaurant or location where appropriate:

- Minimum advance booking time.
- Maximum booking horizon.
- Allowed booking start-time intervals.
- Reservation duration.
- Minimum and maximum party size.
- Large-group handling.
- Same-day booking rules.
- Walk-in handling.
- Cancellation deadline.
- Modification rules.
- Confirmation requirements.
- No-show and arrival policies.
- Waiting-list behavior.
- Deposits or prepayment requirements, if introduced.

The system must distinguish availability from confirmed reservation status.

A positive availability result is not itself a reservation.

## 6.4 Customer identity and contact information

- Define the minimum information required for a booking.
- Support appropriate contact information, such as a phone number and/or email address.
- Normalize phone numbers into a consistent format.
- Define how customers prove ownership of an existing reservation.
- Use a non-guessable reservation reference.
- Define personal-data retention, correction and deletion policies according to applicable requirements.

A name and phone number alone should not automatically be treated as strong identity verification.

## 6.5 Reservation lifecycle

The exact lifecycle must be agreed with the restaurant.

A possible baseline is:

- Pending.
- Confirmed.
- Cancelled.
- Seated.
- Completed.
- No-show.
- Rejected or expired, if pending approval is supported.

Not every restaurant needs every state.

State transitions must be explicit and validated.

Payment status, if payments are later introduced, must remain distinct from reservation status.

---

# 7. Business invariants — rules that must always hold

These are system-level guarantees. Restaurant-specific policies may vary, but these integrity and security principles must not be bypassed.

## 7.1 Reservation integrity

**I1. No conflicting allocation:** A physical table cannot be allocated to overlapping active reservations unless an explicitly authorized seating policy permits the arrangement.

**I2. Capacity:** An assigned table or configured table combination must accommodate the party size.

**I3. Valid schedule:** A reservation must respect the applicable location's operating schedule, service rules and booking policies.

**I4. Valid time:** A reservation cannot be created in the past.

**I5. Valid interval:** Start time must precede end time, and duration must respect the applicable policy.

**I6. Operational availability:** A table marked unavailable cannot be assigned to a new reservation.

**I7. No accidental overbooking:** Capacity must not be exceeded unless a deliberately defined and authorized policy allows it.

**I8. Atomic operations:** A failed booking must not leave partially created or inconsistent reservation data.

**I9. Tenant ownership:** Every reservation, customer relationship, table and restaurant configuration record must belong to the correct tenant and, where relevant, location.

## 7.2 Reservation lifecycle

**I10. Valid state transitions:** Reservation states may change only through explicitly permitted transitions.

**I11. Cancellation:** A cancelled reservation must not continue to occupy active seating capacity.

**I12. Safe modification:** If a modification fails, the original valid reservation must remain unchanged.

**I13. Verified confirmation:** A reservation may be reported as confirmed only when the authoritative backend has persisted that confirmed state.

**I14. Explicit outcome:** Operations must distinguish success, business rejection, technical failure and uncertain outcome.

**I15. No silent state changes:** Important reservation changes must be traceable.

## 7.3 Security and authorization

**I16. Customer ownership:** A customer may retrieve or modify only reservations they are authorized to access.

**I17. Staff authorization:** Staff actions must respect assigned roles, permissions and location scope.

**I18. Tenant isolation:** One tenant must never access or modify another tenant's business records.

**I19. Trusted tenant context:** Tenant identity must be resolved and validated by trusted backend authentication and authorization mechanisms.

**I20. Agent permissions:** The AI agent must not gain permissions beyond its authorized integration identity.

**I21. Sensitive actions:** Retrieval, modification and cancellation require the defined identity-verification and authorization process.

**I22. Authorized overrides:** Operational overrides must require appropriate permissions and produce audit records.

**I23. Safe errors:** Error messages must not reveal private reservation data or confirm the existence of another customer's reservation to an unauthorized requester.

**I24. Secret protection:** Credentials, tokens and sensitive configuration must not be exposed in source code, logs or client responses.

## 7.4 Consistency and reliability

**I25. Idempotency:** Retrying the same logical mutation with the same idempotency key must not create duplicate business effects.

**I26. Concurrency safety:** Simultaneous requests competing for the same capacity must be handled safely.

**I27. Transaction integrity:** Related database changes must be committed or rolled back consistently.

**I28. Safe retries:** A retry must not repeat a consequential operation unknowingly.

**I29. Uncertain outcomes:** If a response is lost after a possible successful commit, the system must reconcile the operation using its idempotency record or operation identifier before attempting another mutation.

**I30. Timezone correctness:** All reservation times must be interpreted using the correct restaurant location timezone. Storage and comparison rules must be consistent.

**I31. External service isolation:** An external provider failure must not silently corrupt internal reservation state.

## 7.5 AI safety and correctness

**I32. Verified results:** The agent must report only results supported by authoritative backend responses.

**I33. No invented facts:** The agent must not invent availability, prices, policies, reservation references or successful operations.

**I34. Confirmation accuracy:** Explicit confirmation must correspond to the exact operation and details submitted.

**I35. No conversational bypass:** Natural-language instructions, prompt injection or retrieved content must not bypass backend authorization or business policies.

**I36. Guarantees versus requests:** The agent must distinguish a customer preference from a confirmed restaurant commitment.

**I37. Controlled escalation:** Unsupported, ambiguous or sensitive cases must be escalated or handled through a defined safe response.

## 7.6 Audit and privacy

**I38. Traceability:** Important actions must have an audit record containing the actor, tenant, operation, time and relevant outcome.

**I39. Audit integrity:** Audit history must not be silently overwritten by ordinary business operations.

**I40. Data minimization:** Personal information must only be collected, stored, processed and logged when justified.

**I41. Recovery:** Recovery procedures must preserve the integrity of confirmed reservations and other authoritative business records.

---

# 8. Reservation operation design

The following behavior should guide the backend API and business services.

## 8.1 Availability

Availability must be calculated by the backend using authoritative data.

It must consider:

- Tenant and restaurant location.
- Requested date and time.
- Party size.
- Reservation duration.
- Opening hours and exceptional closures.
- Active reservations.
- Table assignments or capacity allocation.
- Table combinations, if supported.
- Maintenance blocks and other operational restrictions.
- Restaurant booking policies.

The API should return structured availability results, not a vague boolean alone.

The response should clearly distinguish:
- Available.
- Not available.
- Invalid request.
- Outside booking policy.
- Restaurant closed.
- Technical failure.

Alternative times may be returned when supported.

Availability is time-sensitive. A result from an earlier check is not a guarantee that the same capacity remains available at booking time.

## 8.2 Create reservation

A booking flow should:

1. Validate the request.
2. Establish trusted tenant and location context.
3. Verify customer identity requirements.
4. Validate restaurant policy.
5. Recheck availability at the point of mutation.
6. Allocate capacity or a table safely.
7. Persist the reservation transactionally.
8. Record the operation and relevant audit event.
9. Return the authoritative reservation result.

The system must protect against two customers booking the last available capacity simultaneously.

Application-level availability checks alone are insufficient to guarantee concurrency safety. Use appropriate database transactions, constraints, locking or another proven concurrency-control strategy.

## 8.3 Modify reservation

A modification is a consequential operation.

It must:

1. Verify reservation ownership or staff authorization.
2. Check whether modification is allowed.
3. Validate the requested changes.
4. Check availability for the new details.
5. Apply the change atomically.
6. Preserve the original reservation if the operation fails.
7. Record the change in the audit history.

Do not cancel the original reservation first and then attempt to create a replacement without a safe transactional strategy.

## 8.4 Cancel reservation

Cancellation must:

1. Verify authorization.
2. Check the applicable cancellation policy.
3. Validate the current reservation state.
4. Apply the permitted state transition.
5. Release active capacity appropriately.
6. Record the operation and actor.
7. Return the authoritative result.

Repeated cancellation requests must be handled safely and consistently.

## 8.5 Retrieve reservation

Retrieval must use an appropriate verification mechanism.

A reservation reference is not, by itself, proof of ownership.

Errors must avoid leaking whether another person's reservation exists.

---

# 9. API and data-contract principles

The backend must expose explicit, versionable contracts.

Each operation should define:

- HTTP method and endpoint.
- Authentication requirements.
- Authorization requirements.
- Tenant and location scope.
- Request schema.
- Response schema.
- Validation rules.
- Possible business errors.
- Possible technical errors.
- Idempotency behavior, where applicable.
- Relevant audit behavior.

Use structured error responses with stable error codes.

Do not expose raw database errors, stack traces or internal provider details to customers.

Use appropriate HTTP semantics.

For consequential mutations, support an idempotency mechanism. The idempotency key must identify a logical operation, not simply a customer or a reservation.

The backend must verify that a reused key corresponds to the same intended operation and request payload. Reuse with conflicting content must be rejected safely.

The exact API design should be reviewed before implementation rather than guessed independently by each layer.

---

# 10. Failure cases and recovery behavior

We cannot literally enumerate every possible future failure. Instead, we must establish systematic failure categories and ensure every operation has a defined failure strategy.

## 10.1 Customer and conversation failures

Handle:

- Missing required information.
- Invalid or ambiguous dates and times.
- Timezone misunderstandings.
- Contradictory customer instructions.
- Invalid contact details.
- Party size outside policy.
- Customer changing their mind.
- Customer disconnecting during a booking.
- Repeated requests.
- Unsupported requests.
- Unclear or misunderstood speech.

The agent should ask for clarification rather than guess consequential details.

## 10.2 Availability and seating failures

Handle:

- No availability.
- No suitable table.
- Requested group exceeds capacity.
- Table temporarily unavailable.
- Stale availability results.
- Simultaneous requests for the last available table.
- Invalid table combination.
- Restaurant closure or special event.
- Operational capacity reduction.
- Opening hours changing while reservations already exist.

## 10.3 Reservation lifecycle failures

Handle:

- Reservation not found.
- Failed identity verification.
- Unauthorized modification or cancellation.
- Cancellation outside the allowed window.
- Already-cancelled reservation.
- Invalid lifecycle transition.
- Modification no longer available.
- Customer and staff modifying the same reservation concurrently.
- Pending confirmation expiring.
- Customer arriving early or late.
- No-show handling.

## 10.4 Backend and database failures

Handle:

- Backend unavailable.
- Database unavailable.
- Request timeout.
- Transaction rollback.
- Deadlock.
- Constraint violation.
- Concurrent modification conflict.
- Database migration failure.
- Unexpected internal exception.
- Commit succeeds but response is lost.
- Backup or recovery failure.

The system must distinguish an operation that definitely failed from one whose result is uncertain.

## 10.5 AI and tool failures

Handle:

- Incorrect intent interpretation.
- Hallucinated information.
- Wrong tool selection.
- Missing or malformed tool arguments.
- Duplicate tool calls.
- Tool timeout.
- Agent losing conversational context.
- Prompt injection.
- Malicious or misleading retrieved content.
- Model provider unavailable.
- Interrupted execution during a mutation.
- AI response generated before the backend operation is complete.

Tool execution must be validated and authorized by the backend independently of the model's reasoning.

## 10.6 Voice failures

Handle:

- No speech detected.
- Background noise.
- Low-confidence transcription.
- Incorrect transcription.
- Silence and long pauses.
- Customer interruption.
- TTS provider failure.
- Audio connection loss.
- Telephony provider outage.
- Partial or out-of-order audio events.
- Human handoff failure.

The system must never interpret an uncertain transcript as sufficient confirmation for a consequential operation without the required verification.

## 10.7 External integration and notification failures

Handle:

- SMS or email provider unavailable.
- Rate limits.
- Invalid provider credentials.
- Duplicate callbacks.
- Delayed callbacks.
- Invalid callback signatures.
- Notification delivery failure.
- Provider reports success but the response is lost.

A notification failure must not automatically mean the underlying reservation failed.

## 10.8 Staff and operational failures

Handle:

- Unauthorized staff action.
- Compromised or disabled account.
- Accidental cancellation.
- Incorrect table assignment.
- Conflicting staff modifications.
- Temporary restaurant closure.
- Operational override.
- Missed check-in.
- Changes to policies that affect existing reservations.

High-impact changes should have appropriate authorization and auditability.

## 10.9 Payment failures — future capability

If deposits or payments are introduced, design them as a separate payment lifecycle.

Consider:

- Declined payments.
- Provider unavailable.
- Duplicate charge attempts.
- Payment succeeds but reservation creation fails.
- Payment callback arrives late.
- Refund failure.
- Pending payment.
- Disputes.

Payment status and reservation status must not be treated as the same thing.

Payments are not assumed to be part of the initial reservation sandbox.

---

# 11. Security architecture

Security must be designed across the whole request path.

## 11.1 Authentication

Define how each actor proves their identity:

- Customers.
- Restaurant staff.
- Restaurant administrators.
- Platform administrators.
- Internal services and AI integrations.

Do not assume that a customer's spoken name or phone number alone provides strong authentication.

## 11.2 Authorization

Apply authorization at the backend service boundary.

Every operation must verify:
- Who is acting.
- Which tenant and location they are acting within.
- Which resource they are trying to access.
- Whether their role permits the operation.
- Whether the current business state permits the operation.

## 11.3 Tenant isolation

Tenant isolation must be enforced at multiple appropriate layers:

- Authentication and request context.
- Authorization checks.
- Service/repository queries.
- Database design and constraints.
- Tests covering cross-tenant access.

Never trust a tenant ID supplied by a customer or generated by the AI without validating it against the authenticated principal.

A cross-tenant access attempt must fail safely.

## 11.4 Input and integration security

Validate all external inputs, including:

- API requests.
- AI tool arguments.
- Retrieved documents.
- External callbacks.
- Voice-transcribed content.

Use secure secret management, appropriate transport security, dependency updates, rate limiting and abuse protection.

## 11.5 Auditability

Important actions must record who performed them, under which tenant, what operation was attempted and its outcome.

Avoid storing unnecessary sensitive conversation content in audit logs.

---

# 12. Reliability, consistency and operational design

## 12.1 Database consistency

Use PostgreSQL transactions for operations that modify related business records.

Important integrity rules should be enforced at the database level when possible.

Choose concurrency controls based on the actual reservation allocation model.

Do not assume that checking availability and then inserting a reservation is safe under concurrent requests.

## 12.2 Idempotency and retry behavior

Every consequential operation needs an explicit retry strategy.

For example, if a booking request times out after the database may have committed it:

- Do not blindly submit a second booking.
- Reconcile using the original idempotency key or operation identifier.
- Return the actual existing result if the operation succeeded.
- Retry only when it is safe to do so.

## 12.3 Background work

Tasks such as notifications, external synchronization and other non-immediate work may eventually require a background job mechanism.

Do not introduce a queue simply because it is common in SaaS architectures. Introduce one when the required asynchronous behavior justifies it.

If a business transaction must reliably trigger an external event, evaluate a transactional outbox or equivalent mechanism so a committed reservation does not lose its associated event.

## 12.4 Backup and recovery

Define:

- Database backup strategy.
- Restore procedure.
- Recovery objectives.
- Migration and rollback strategy.
- Verification that restored data is consistent.

A backup is not sufficient unless restoration is tested.

## 12.5 Availability and graceful degradation

A failure in the AI or voice provider should not make the core reservation service unusable.

The manual application and deterministic backend should remain independently testable and usable.

If an optional integration fails, the system should communicate the limitation and preserve authoritative business state.

---

# 13. Observability and evaluation

We must be able to explain what happened during a complete customer interaction.

Use correlation identifiers to connect:

- Voice session.
- Conversation.
- Agent execution.
- Tool call.
- Backend request.
- Database operation.
- External provider request.

Monitor at least:

- API latency and error rates.
- Database errors and query performance.
- Reservation success and rejection rates.
- Concurrency conflicts.
- Idempotency conflicts and duplicate attempts.
- Agent tool selection and execution outcomes.
- STT/TTS latency and failures.
- External integration failures.
- Human escalation frequency.

For the AI system, create evaluation cases covering:

- Normal booking.
- Missing information.
- Ambiguous dates.
- No availability.
- Changed booking details.
- Unauthorized requests.
- Misleading retrieved content.
- Backend timeout.
- Duplicate tool execution.
- Successful mutation with lost response.
- Voice misunderstanding and interruption.

Measure latency, correctness, robustness and usability.

Do not evaluate the AI only by whether its final response sounds natural. Evaluate whether it chose the correct tool, supplied valid arguments, respected authorization and accurately reported the backend outcome.

---

# 14. Data model direction

The exact schema must be designed and reviewed before implementation, but the target domain should account for the following entities.

## Platform and tenancy

- Tenant.
- Restaurant location.
- User.
- Tenant membership.
- Role and permission assignment.

## Restaurant configuration

- Restaurant profile.
- Operating hours.
- Exceptional closure or schedule.
- Reservation policy.
- Service period, if needed.
- Restaurant knowledge document.

## Seating and capacity

- Table.
- Table section.
- Table availability block.
- Table combination configuration, if supported.
- Table assignment.

## Customer and reservation

- Customer.
- Reservation.
- Reservation reference.
- Reservation status history.
- Reservation allocation.
- Special request, if supported.

## Security and operations

- Audit event.
- Idempotency record.
- Integration configuration.
- External operation or notification record, when needed.

Important modeling rules:

- Tenant ownership must be explicit.
- Location ownership must be explicit where relevant.
- Use stable internal identifiers.
- Do not use customer phone numbers or reservation codes as database primary keys.
- Define uniqueness and foreign-key constraints deliberately.
- Avoid storing derived availability as the authoritative truth.
- Keep conversation state separate from reservation records.
- Keep payment records separate if payments are introduced.

Do not create every table immediately. Design the relationships and introduce schema elements when the corresponding vertical slice requires them.

---

# 15. Technology direction

Use the technologies already selected for the preparation sandbox unless a concrete engineering reason justifies changing them.

Initial direction:

- **Backend:** Python + FastAPI.
- **Database:** PostgreSQL.
- **Frontend:** React + TypeScript.
- **Agent orchestration:** LangGraph.
- **Knowledge retrieval:** a suitable vector store and/or PostgreSQL-based retrieval, selected according to actual requirements.
- **Containerization and local environment:** Docker + Docker Compose.
- **External tool integration:** controlled APIs and MCP where justified.
- **Voice:** selected STT/TTS and, later, telephony providers.

Technology choices must follow requirements and measurable trade-offs.

Do not assume that the initial local development stack is automatically the final production deployment architecture.

Do not introduce microservices, Kubernetes, distributed event infrastructure or multiple agents without evidence that the system needs them.

A modular monolith is a reasonable initial architecture: one deployable backend with clearly separated domain, API, persistence, integration and orchestration boundaries. It can evolve if scale, deployment independence or team ownership later justify service separation.

---

# 16. Implementation strategy — progressive vertical slices

The architecture above is the target. Implementation must remain incremental.

Every slice should have:

1. A clear business outcome.
2. Explicit questions and assumptions.
3. Acceptance criteria.
4. A reviewed design.
5. Defined API and data contracts.
6. Relevant authorization requirements.
7. Positive and negative tests.
8. Failure behavior.
9. Reproduction instructions.
10. A reviewable pull request.

## Slice 0 — Requirements and domain foundation

Before coding:

- Define actors and permissions.
- Define tenant and location concepts.
- Define the reservation lifecycle.
- Identify configurable restaurant policies.
- Identify unresolved business decisions.
- Define the initial domain model.
- Define the first API contract.
- Document invariants and failure cases.

Do not pretend unconfirmed policies are established facts.

## Slice 1 — Read-only availability

Implement a small end-to-end availability operation.

Include:

- Tenant/location context.
- Request validation.
- Restaurant schedule.
- Table/capacity data.
- Availability business service.
- PostgreSQL persistence.
- API response contract.
- Authorization boundary.
- Tests for available, unavailable, invalid and cross-tenant cases.

Connect one read-only agent tool only after the backend contract is clear.

## Slice 2 — Reservation creation

Implement:

- Required customer information.
- Booking policy validation.
- Availability revalidation.
- Safe capacity allocation.
- Transactional persistence.
- Idempotency.
- Duplicate request handling.
- Explicit success and failure outcomes.
- Tests for concurrent requests.

Add agent confirmation only after the backend operation is reliable.

## Slice 3 — Retrieval, modification and cancellation

Implement:

- Reservation ownership verification.
- Staff permissions.
- Reservation lifecycle transitions.
- Safe modification.
- Cancellation policy.
- Idempotency and retry behavior.
- Audit records.
- Tests for unauthorized and invalid state transitions.

## Slice 4 — Manual staff application

Build the React/TypeScript interface on the same backend services.

Include loading, empty, validation, success and error states.

The application must work with the LLM path disabled.

## Slice 5 — Agent orchestration and controlled tools

Add:

- LangGraph workflow.
- Structured conversation state.
- Missing-information collection.
- Tool argument validation.
- Explicit confirmation.
- Controlled mutations.
- Tool failure handling.
- Duplicate execution protection.
- Human escalation.
- Persistent checkpoints and recovery where required.

The agent must use the same authoritative business service as the manual application.

## Slice 6 — Knowledge retrieval and evaluation

Add tenant-scoped restaurant knowledge retrieval.

Test retrieval quality, source grounding, misleading content and prompt injection.

Keep operational data, such as live availability and reservation status, connected to backend tools.

## Slice 7 — Voice prototype

Connect:

Audio → STT → conversational workflow → authorized business tool → response → TTS.

Start with controlled audio interaction, interruption handling and measurable latency.

Keep full-duplex conversation and production telephony as separate capabilities to develop and evaluate.

## Slice 8 — Production hardening and deployment

Based on actual product and client requirements, address:

- Production authentication and authorization.
- Tenant isolation verification.
- Secret management.
- Rate limiting and abuse protection.
- Monitoring and alerting.
- Backups and tested restoration.
- Deployment automation.
- Capacity and performance testing.
- Security testing.
- Provider failure and recovery.
- Operational documentation.
- Client onboarding and tenant configuration.

A slice is complete only when its behavior is demonstrated and its relevant failure cases are tested, not merely when its code has been written.

---

# 17. Engineering workflow and responsibilities

Follow this workflow for every bounded change:

**Issue → questions → assumptions and acceptance criteria → design → branch → implementation → tests/evaluations → PR → review → corrections → merge.**

### Claude

Claude should:

- Help analyze requirements.
- Identify ambiguities and missing decisions.
- Propose and review architecture.
- Define bounded vertical slices.
- Write precise implementation instructions for Cursor.
- Identify relevant tests and edge cases.
- Review implementation results against acceptance criteria.
- Avoid inventing business policies.
- Avoid asking Cursor to implement unrelated future capabilities prematurely.

### Cursor

Cursor should:

- Implement only the assigned bounded task.
- Respect existing architecture and contracts.
- Avoid unnecessary refactoring.
- Add or update tests.
- Report files changed and design decisions.
- Report actual test and validation results.
- Identify assumptions or blockers instead of silently guessing.

### Engineering owner

The engineering owner remains responsible for:

- Understanding the design.
- Reviewing generated code.
- Running and inspecting tests.
- Validating business behavior.
- Checking security and data boundaries.
- Understanding failures and trade-offs.
- Making changes independently when needed.
- Reviewing and approving pull requests.

AI-generated code is not automatically correct because it compiles or passes a limited test suite.

---

# 18. Decisions that must be confirmed, not guessed

The architecture must distinguish design guarantees from business decisions that require Khaled or a real restaurant client's input.

Examples:

1. Does one tenant represent one restaurant brand, one legal business or another organizational boundary?
2. Can one tenant own multiple restaurant locations?
3. Are reservations assigned to physical tables immediately or allocated by capacity first?
4. Are table combinations supported?
5. What is the exact reservation lifecycle?
6. Which booking policies are configurable?
7. What customer identity-verification method is required?
8. Are bookings automatically confirmed or sometimes pending staff approval?
9. What are the rules for large groups, deposits and cancellations?
10. Which notification channels are required?
11. Which voice and telephony providers will be used?
12. What are the expected client scale, availability targets and data-retention requirements?
13. Which privacy and regulatory requirements apply to the actual deployment locations?

Do not silently turn an example into a permanent product requirement.

For every unresolved question, document:
- The question.
- Why it matters.
- A proposed default, if appropriate.
- The consequences of choosing differently.
- Who must approve the decision.

---

# 19. Final engineering standard

Before considering any feature complete, ask:

- Does it respect tenant isolation?
- Does it enforce authorization on the backend?
- Does it preserve database integrity?
- Does it handle concurrent requests safely?
- Is its behavior deterministic where business rules are involved?
- Are retries and duplicate requests safe?
- Does it handle uncertain outcomes?
- Are errors explicit and safe?
- Can we trace what happened?
- Are the important positive and negative cases tested?
- Can another channel use the same business service?
- Can we explain and maintain the implementation?

The goal is not to build the largest architecture or implement every enterprise feature immediately.

The goal is to build a **coherent, secure, reliable and extensible SaaS foundation** whose business behavior is trustworthy, whose AI interactions are controlled, and whose capabilities can grow without repeatedly redesigning the system.

**Our guiding principle: production-minded by design, incremental by implementation, and evidence-driven at every step.**