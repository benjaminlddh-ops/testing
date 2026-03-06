# B2B Hotel Distribution Platform (Booking.com / Expedia style, Hotels-only)

## 1) Scope
Build a **B2B-only hotel platform** for travel agents/corporates with:
- **Supplier API-in integrations** (hotel inventory ingestion).
- **API-out distribution** for your buyers/partners.
- **Multi-currency pricing and settlement**.
- **Flexible markups/rules engine**.
- **Hotel static data normalization + mapping**.
- **Customer profiles with configurable credit limits and controls**.

---

## 2) Core Product Modules

### A. Supplier Integration Hub (API-In)
Handles upstream hotel suppliers (beds, OTAs, channel managers, wholesalers).

**Responsibilities**
- Connector framework per supplier (REST/SOAP/XML/JSON).
- Auth handling (API key, OAuth, signatures).
- Contract/rate plan mapping.
- Search availability, rates, cancellation policy, booking, amend, cancel.
- Error normalization and retry strategy.
- Webhook/event ingestion where supported.

**Design Notes**
- Create a **canonical hotel API schema** internally so each connector maps into one common format.
- Use async queues for resiliency and rate-limit management.
- Keep supplier-specific behavior behind adapter interfaces.

### B. Static Content & Mapping Engine
Unifies hotel master data from multiple suppliers into a single canonical hotel profile.

**Responsibilities**
- Ingest static content (name, address, geo, amenities, images, descriptions).
- Deduplication / matching logic (geo + text + external IDs).
- Maintain **hotel mapping table**: internal_hotel_id ↔ supplier_hotel_id(s).
- Handle room/board/amenity taxonomy mapping.

**Design Notes**
- Store both canonical content and supplier raw content.
- Introduce confidence score for auto-mapping and manual review queues.
- Version static data updates and audit mapping changes.

### C. Pricing, Currency & Markup Engine
Converts supplier net rates into sell rates by currency, customer type, and contract rules.

**Responsibilities**
- FX conversion by source/target currency with timestamped rates.
- Multi-layer markups:
  - Global markup
  - Supplier-level markup
  - Destination/hotel-level markup
  - Customer/segment-level markup
  - Promotional override
- Rounding rules by currency.
- Margin floors/ceilings and exception handling.

**Formula (example)**
`Sell Rate = Round((Net Rate × FX Rate) × (1 + Markup%), Currency Rule)`

**Design Notes**
- Persist priced breakdown: net, fx, markup, taxes, fees, final amount.
- Use priority-based rule engine to resolve conflicts.

### D. B2B Customer & Credit Module
Manages agency/corporate accounts and financial controls.

**Customer Profile Configuration**
- Legal/business details, market, sales owner.
- Allowed currencies.
- Allowed suppliers.
- Markup profile assignment.
- Booking policy controls (max lead time, cancellation window, destinations).

**Credit Management**
- Credit limit per customer.
- Available credit = limit - outstanding exposure.
- Exposure includes booked/not-settled transactions.
- Soft block (warning) and hard block (deny booking) thresholds.
- Optional prepaid wallet mode.

### E. Booking Orchestrator
Coordinates booking lifecycle end-to-end.

**Responsibilities**
- Pre-book validation (credit, pricing freshness, contract checks).
- Book/confirm flow with idempotency keys.
- Amend/cancel with policy-aware penalties.
- Reconciliation states (requested, confirmed, failed, cancelled, refunded).

### F. API-Out Distribution Layer
Public B2B API for your partners/customers.

**Recommended API Set**
- `POST /search/hotels`
- `POST /hotels/{id}/check-rate`
- `POST /bookings`
- `GET /bookings/{id}`
- `POST /bookings/{id}/cancel`
- `GET /static/hotels`
- `GET /finance/credit-status`

**Controls**
- API keys/OAuth2.
- Per-client rate limiting and quotas.
- Signed request/response logs.

---

## 3) Suggested High-Level Architecture

- **API Gateway** (auth, throttling, routing)
- **Supplier Connector Services** (one adapter per supplier)
- **Search & Cache Service** (fast aggregated shopping)
- **Static Mapping Service** (hotel/room/amenity normalization)
- **Pricing Engine Service** (FX + markups + rounding)
- **Customer & Credit Service** (limits, policies, entitlements)
- **Booking Service** (orchestration and state machine)
- **Finance/Reconciliation Service**
- **Admin Portal** (operations, mapping review, pricing rules, credit controls)
- **Data Stores**
  - Transactional DB (bookings, customers, finance)
  - Search index (hotel content/search)
  - Cache (availability/pricing short TTL)
  - Message queue (async retries/events)

---

## 4) Data Model (Minimum Entities)

- `suppliers`
- `supplier_hotels`
- `hotels` (canonical)
- `hotel_mappings` (hotel_id, supplier_id, supplier_hotel_id, confidence, status)
- `customers`
- `customer_settings` (currencies, policies, supplier access)
- `credit_accounts` (limit, utilized, available, block_mode)
- `markup_rules` (scope, priority, value, valid_from/to)
- `fx_rates` (base, quote, rate, timestamp)
- `search_logs`
- `bookings`
- `booking_items`
- `booking_financials` (net, fx, markup, sell, tax, fees)
- `invoices` / `settlements`

---

## 5) Critical Workflows

### Search Flow
1. API-out receives search request.
2. Customer entitlement + currency checks.
3. Fan-out to relevant supplier connectors.
4. Normalize responses.
5. Apply FX + markup.
6. Return sorted/filtered results.

### Booking Flow
1. Revalidate rate and policy.
2. Run credit check.
3. Place booking with supplier.
4. Persist booking + financial snapshot.
5. Update credit exposure.
6. Return confirmation.

### Cancellation Flow
1. Retrieve booking and policy.
2. Cancel with supplier.
3. Calculate penalty/refund.
4. Update credit exposure and settlement records.

---

## 6) MVP Roadmap (Recommended)

### Phase 1 (8–12 weeks)
- 2 supplier integrations.
- Canonical hotel mapping for key markets.
- Basic search + booking + cancel API-out.
- Single markup layer + FX conversion.
- Customer onboarding + fixed credit limits.
- Admin screens for mappings, markups, and credit status.

### Phase 2
- Advanced rule engine for markups.
- Multi-level users/roles per customer.
- Reconciliation automation and invoicing.
- Performance optimization with smarter cache strategy.

### Phase 3
- Dynamic promotions, preferred hotel logic.
- ML-based hotel deduplication/mapping.
- Advanced fraud/risk controls and anomaly alerts.

---

## 7) Non-Functional Requirements
- High availability (99.9%+ target).
- Observability: structured logs, tracing, supplier-level SLA dashboards.
- Security: encryption at rest/in transit, audit trails, PII controls, least privilege.
- Idempotency for booking/cancel endpoints.
- Scalability for bursty search traffic.

---

## 8) Practical Tech Stack (Example)
- Backend: Node.js/TypeScript or Java/Kotlin or Go (microservices).
- DB: PostgreSQL for transactions.
- Cache: Redis.
- Queue: RabbitMQ/Kafka/SQS.
- Search: OpenSearch/Elasticsearch for hotel content lookup.
- API docs: OpenAPI/Swagger.
- Infra: Kubernetes + CI/CD + observability stack.

---

## 9) Risks & Mitigations
- **Supplier inconsistency** → strict adapter contracts + contract tests.
- **Mapping quality issues** → human review workflow + confidence thresholds.
- **Credit overrun** → real-time exposure updates + hard-stop booking policies.
- **FX volatility** → rate timestamping and configurable refresh intervals.

---

## 10) Immediate Next Steps
1. Finalize canonical schemas (hotel content, rate, booking).
2. Choose first two suppliers and get sandbox credentials.
3. Define markup rule precedence and default customer policy matrix.
4. Build MVP admin controls for mapping + credit + markup.
5. Launch pilot with 3–5 B2B customers.
