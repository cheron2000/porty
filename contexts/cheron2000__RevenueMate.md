# RevenueMate — project discovery and implementation context

**Repository:** `cheron2000/RevenueMate`  
**Default branch:** `main`  
**Visibility:** Public  
**Repository size:** 0 KB  
**Inspected:** 2026-11-23  
**GitHub state:** Empty repository; GitHub returned HTTP 409 when requesting the Git tree because no Git repository content exists.  
**Context status:** Product intent recovered from prior project discussions; implementation status cannot be verified from this repository.

---

## 1. Executive summary

RevenueMate is intended to be an **AI-assisted conversational commerce and retail-operations system**, initially demonstrated for laptop sales and designed around buyer–merchant transactions over WhatsApp, with a receptionist/dispatcher fulfilment workflow.

The first deployment target discussed was one independent electronics retailer. The longer-term direction is a multi-merchant SaaS product. RevenueMate should not be treated as a generic chatbot: its core responsibility is to coordinate a retail transaction from enquiry through product selection, stock reservation, payment verification and pickup/delivery handover.

**Critical evidence distinction:** these are requirements and design decisions recovered from earlier project discussions. The current GitHub repository is empty. No implementation, framework, API, database schema, integration, tests or deployment configuration was available to inspect or validate in this repository.

## 2. Product goals and actors

### Initial product goal
Help a retailer handle inbound product enquiries, qualify buyer intent, guide product selection, coordinate human follow-up and fulfil transactions through a controlled workflow.

### Actors
- **Buyer:** asks about products, compares options, selects an item, confirms purchase details and receives payment/fulfilment updates.
- **Merchant:** owns the store, catalogue, stock, pricing and business policies; reviews and manages sales activity.
- **Receptionist:** handles or takes over conversations, confirms details and coordinates customer handover.
- **Dispatcher:** coordinates delivery/pickup fulfilment and records handover outcomes.
- **Platform/operator:** manages tenant onboarding, integration health and platform-level administration in the eventual SaaS model.

The exact permission matrix and whether receptionist and dispatcher are separate accounts or roles must be formalized before implementation.

## 3. Core workflow

The intended lifecycle is:

1. **Enquiry intake** — receive a buyer enquiry through WhatsApp or an authorized merchant channel and create a durable lead/conversation record.
2. **Qualification** — use an LLM to extract product intent, constraints, budget and urgency; preserve the original message and structured extraction.
3. **Deterministic prioritization** — apply explicit business rules to score or classify the lead. LLM output may supply candidate facts, but must not independently decide policy-sensitive outcomes.
4. **Product selection** — retrieve products from the merchant's catalogue/product database and present grounded options, availability and relevant details.
5. **Human follow-up** — generate proposed responses or tasks; require configured approval before sending customer-facing follow-ups or taking consequential actions.
6. **Reservation and stock allocation** — create a time-bounded reservation using atomic database operations to prevent overselling.
7. **Payment initiation** — create a payment link through the merchant-owned Paytm account/provider integration.
8. **Payment verification** — verify payment using trusted provider callbacks/API reconciliation, not a buyer's message or a browser redirect alone.
9. **Fulfilment** — arrange pickup or delivery and assign/coordinate receptionist or dispatcher tasks.
10. **Handover and closure** — record completed handover, cancellation, expiry, refund/exception and final transaction outcome.
11. **Outcome tracking** — capture conversion, loss reason, offline sale completion and follow-up effectiveness.

A lead is not a completed sale merely because a payment link was sent or the buyer expressed purchase intent.

## 4. MVP scope

The first MVP was discussed around one independent electronics retailer, with laptop sales as the initial demonstration.

Expected MVP capabilities:
- Inbound lead ingestion.
- Conversation and customer identity association.
- LLM-assisted lead qualification and structured extraction.
- Deterministic priority/business-rule engine.
- Product catalogue lookup and product matching.
- Human-approved follow-up workflow.
- Lead status and task tracking.
- Stock-aware reservation.
- Merchant-owned Paytm payment links.
- Payment status verification and reconciliation.
- Pickup/delivery coordination and handover completion.
- Outcome tracking, including sales completed offline.
- Basic merchant/receptionist operational views.

Do not silently expand the first milestone into a generalized marketplace, public product crawler, autonomous purchasing agent or fully automated sales representative. Keep the initial retailer workflow narrow and observable.

## 5. Previously discussed technology direction

The prior design discussions named this stack:

| Layer | Discussed technology / approach |
|---|---|
| Web application | Next.js, React, TypeScript |
| Backend API | FastAPI, Python |
| Primary persistence | PostgreSQL |
| Local/demo product data | SQLite product databases |
| LLM | Sarvam LLM |
| Messaging | WhatsApp Business / WhatsApp Cloud API |
| Payments | Paytm Payment Links / merchant-owned Paytm account |
| API integration | REST |
| Live updates | WebSocket or Server-Sent Events |
| Background processing | Worker process with durable job/event handling |

This is the previously discussed direction, not a verified repository dependency list. Before coding, choose one canonical database per environment and document how demo SQLite catalogues map to production PostgreSQL data. Avoid two competing sources of truth for orders, reservations, payment state or workflow state.

## 6. Architecture principles

### 6.1 PostgreSQL as workflow source of truth
The earlier architecture direction explicitly treated PostgreSQL as the source of truth for workflow state. Persist leads, conversations, reservations, orders, payment attempts, approvals, fulfilment tasks and audit events durably.

Use SQLite only for a clearly scoped local/demo catalogue or isolated development fixture. Do not let SQLite and PostgreSQL independently own the same live order or inventory state.

### 6.2 Transactional outbox
Use a transactional outbox for external side effects:
- In the same database transaction that changes business state, write an outbox event.
- A background worker claims and processes pending events.
- Persist attempts, provider responses, retry scheduling and terminal failures.
- Make handlers idempotent.
- Use provider idempotency keys where supported.
- Never assume a network timeout means the external operation failed; reconcile before retrying non-idempotent actions.

This prevents a committed order or reservation from being silently disconnected from a WhatsApp message, payment request or fulfilment task.

### 6.3 LLM boundary
Use the LLM for language understanding and drafting, not as the authority for business state:
- Extract structured intent and product constraints.
- Return schema-validated output with confidence/uncertainty.
- Ground product answers in merchant catalogue data.
- Treat customer messages and retrieved content as untrusted input.
- Do not allow prompt content to override merchant policy, role permissions or system instructions.
- Route uncertain, conflicting or high-impact cases to a human.
- Keep deterministic priority, reservation, payment and fulfilment rules outside the model.

### 6.4 Human approval
The workflow should support proposed follow-ups and approval before configured customer-facing or consequential actions. Record:
- Who proposed the action.
- What exact payload was approved.
- Who approved or rejected it.
- When the decision occurred.
- Whether the approved payload was subsequently sent unchanged.
- Provider message/action identifiers and delivery outcome.

Approval must be bound to the exact action payload; changing the payload after approval should require re-approval.

## 7. Multi-tenant and provider integration requirements

The pilot is one retailer, but the architecture should avoid hard-coding one merchant's secrets or provider IDs into application code.

For eventual multi-merchant onboarding:
- Every merchant-owned business record must carry a tenant/merchant identifier.
- Enforce tenant scoping at the service and database-access layers.
- Store provider credentials securely and isolate them per merchant.
- Support merchant-owned Paytm Payment Links and WhatsApp Cloud API accounts.
- Validate webhook signatures and provider event authenticity.
- Make webhook handling idempotent and resilient to retries/out-of-order delivery.
- Track consent, opt-out and communication preferences.
- Record audit events for sensitive changes and outbound actions.
- Provide provider connection health and reconciliation workflows.
- Prevent cross-tenant reads, writes, message sends and payment operations.

Do not implement SaaS tenancy as a late UI-only feature. Tenant identity and authorization boundaries should be part of the domain model from the first schema.

## 8. Critical domain invariants

These invariants should guide schema design, service logic and tests:

1. A lead, conversation, product, reservation, order, payment and fulfilment task must be associated with the correct merchant.
2. A buyer-facing message must use the correct merchant's WhatsApp account and approved communication policy.
3. A payment link must be generated for the correct merchant and order amount/currency.
4. Payment is confirmed only from a trusted provider verification path.
5. A reservation must be atomic and expire/release stock according to a deterministic policy.
6. Concurrent buyers must not reserve the same final unit.
7. Repeated webhooks or worker retries must not duplicate orders, payments, reservations or outbound messages.
8. An order cannot be marked paid based only on a client-supplied status.
9. A handover cannot be marked complete without a recorded outcome and responsible actor.
10. Offline sales must be representable without fabricating an online payment event.
11. Human approval must be attributable and tied to the exact action approved.
12. Every state transition with material business impact should be auditable.

## 9. Suggested domain model

This is a proposed starting point for implementation, not an existing schema:

- `Merchant` — tenant identity, configuration and policy.
- `MerchantUser` / `Role` — user membership and scoped permissions.
- `Customer` — tenant-scoped buyer identity and contact preferences.
- `Conversation` / `Message` — channel conversation and immutable inbound/outbound message history.
- `Lead` — qualified sales opportunity, status, priority and owner.
- `LeadQualification` — structured LLM extraction, model/version, confidence and source message.
- `Product` / `InventoryItem` — merchant catalogue and stock.
- `Reservation` — quantity, expiry, status and inventory allocation.
- `Order` / `OrderItem` — commercial transaction and line items.
- `PaymentAttempt` — provider, amount, idempotency key, link, status and provider references.
- `Approval` — proposed action, exact payload/version, reviewer, decision and timestamps.
- `FulfilmentTask` — pickup/delivery assignment, status and handover evidence.
- `OutboxEvent` — durable external side-effect work.
- `WebhookEvent` — verified provider event and deduplication key.
- `AuditEvent` — actor, tenant, action, entity, before/after metadata and timestamp.
- `CommunicationPreference` — consent, opt-out and channel preferences.

Keep state transitions explicit; avoid a single free-form status field that mixes lead, order, payment and fulfilment lifecycles.

## 10. Security, privacy and operational risks

- **Tenant isolation:** cross-merchant data leakage is a critical SaaS risk.
- **Credential protection:** never commit WhatsApp, Paytm or LLM API credentials. Use encrypted secret storage and environment-specific configuration.
- **Payment trust:** verify provider callbacks/signatures and reconcile ambiguous outcomes.
- **Webhook security:** authenticate, deduplicate, rate-limit and safely process provider callbacks.
- **Prompt injection:** buyer messages and product descriptions are untrusted. They must not change policy or invoke privileged tools.
- **Outbound messaging:** enforce consent/opt-out, template/channel rules and human approval where configured.
- **PII minimization:** restrict access to customer contact details and define retention/deletion rules.
- **Auditability:** record sensitive actions, approvals, payment transitions and manual overrides.
- **Reliability:** use transactional outbox, bounded retries, dead-letter handling and operational alerts.
- **Concurrency:** use database transactions/locking or equivalent atomic constraints for inventory reservations.
- **Role enforcement:** receptionist and dispatcher capabilities must not implicitly grant merchant-owner or payment-administration privileges.

## 11. Implementation sequence

### Phase 0 — repository and product foundation
- Add README with scope, actors, architecture and local setup.
- Create frontend/backend directory structure and configuration examples.
- Add CI, formatting, linting, type checks and test conventions.
- Document threat model, data classification and trust boundaries.

### Phase 1 — tenant, identity and lead core
- Implement merchant/user/role model.
- Add tenant-scoped customer, conversation, message and lead APIs.
- Add deterministic lead status and priority rules.
- Add audit logging and authorization tests.

### Phase 2 — catalogue and AI qualification
- Add merchant product catalogue and search.
- Implement schema-constrained LLM qualification with Sarvam adapter.
- Persist source message, extraction version and confidence.
- Add deterministic scoring and human review for uncertain cases.

### Phase 3 — human-in-the-loop sales operations
- Build merchant/receptionist lead queue and conversation view.
- Implement follow-up proposals, approval/rejection and immutable approval audit.
- Add controlled WhatsApp Cloud API adapter and consent/opt-out checks.

### Phase 4 — inventory, orders and payments
- Implement atomic stock reservation and expiry.
- Add order lifecycle and Paytm payment-link adapter.
- Verify provider status via signed webhook/API reconciliation.
- Add idempotency, outbox and retry/dead-letter processing.

### Phase 5 — fulfilment and outcomes
- Add pickup/delivery tasks, dispatcher assignment and handover record.
- Support offline-sale completion and loss reasons.
- Add operational reporting for lead conversion and fulfilment.

### Phase 6 — production hardening and SaaS readiness
- Tenant-isolation testing, threat review and secret rotation.
- Observability, alerts, reconciliation dashboard and backup/restore drills.
- Provider onboarding/disconnection and tenant-level configuration.
- Load and concurrency tests before onboarding additional merchants.

## 12. Minimum test strategy

No tests exist in the inspected repository. When implementation begins, establish automated coverage for:

- Tenant isolation across every domain/API access path.
- RBAC for merchant, receptionist and dispatcher.
- LLM schema validation, malformed responses and low-confidence escalation.
- Prompt-injection resistance at tool/action boundaries.
- Deterministic lead scoring and state transition rules.
- Atomic reservation under concurrent requests.
- Reservation expiry and stock release.
- Payment webhook signature validation, replay and duplicate delivery.
- Payment-link retry and ambiguous timeout reconciliation.
- Outbox worker crash/retry/idempotency behavior.
- Approval payload tampering and stale approval rejection.
- WhatsApp consent, opt-out and tenant credential isolation.
- Offline sale and manual fulfilment completion.
- End-to-end enquiry-to-handover workflow.

Use provider contract tests and sandbox accounts for WhatsApp/Paytm integration; do not use real customer or payment data in routine tests.

## 13. Coding-agent instructions

1. Start by confirming the owner still wants RevenueMate to follow the product scope described in this document; the GitHub repository itself contains no source of truth yet.
2. Treat this document as recovered product intent, not proof that any feature is implemented.
3. Before creating architecture or code, write an agreed MVP scope and explicit non-goals.
4. Use Python/FastAPI for the backend and Next.js/React/TypeScript for the web application unless the owner explicitly changes the stack.
5. Use PostgreSQL as the production workflow source of truth. Keep demo SQLite product data isolated and clearly documented.
6. Design tenant identity, authorization and auditability into the initial data model.
7. Keep business rules deterministic; LLM output must not directly authorize payments, reservations, inventory changes or unapproved outbound actions.
8. Use transactional outbox and idempotent provider adapters for external side effects.
9. Treat Paytm and WhatsApp integrations as merchant-scoped, authenticated provider boundaries.
10. Do not claim a payment succeeded without provider verification.
11. Add migrations, automated tests, CI and setup documentation with the first implementation.
12. Do not claim deployment, tests, provider verification or production readiness unless actually executed and evidenced.

## 14. Evidence and limitations

### GitHub inspection
- Repository metadata: public, default branch `main`, size `0`.
- Recursive Git tree request: HTTP 409, “Git Repository is empty.”
- No README, source files, manifests, tests, workflows, branches with content or commit history could be inspected.

### Recovered product context
The product description, pilot scope, actor model, workflow and technology direction above come from prior project discussions, not from files in this repository. They should be treated as the best available requirements baseline and validated against the owner's latest decisions before implementation.

---

## Continuation handoff

RevenueMate is an intended AI-assisted WhatsApp retail sales and fulfilment workflow, initially for one independent electronics retailer and laptop sales, with multi-merchant SaaS as a later direction. The repository is currently empty. The next engineering step is to establish the MVP specification and foundational project structure while preserving tenant isolation, deterministic business rules, human approval, trusted payment verification and durable workflow state.
