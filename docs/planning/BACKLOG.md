# FleetFlow Product Backlog

**Document ID:** FF-BACKLOG-001  
**Sprint cadence:** 2-week iterations  
**Last groomed:** 2026-07-11  
**Board:** [FleetFlow Production Kanban](https://github.com/users/aldoprogrammer/projects/2)

---

## Priority Legend

| Label | Meaning |
|-------|---------|
| `P0` | Release blocker — security, ledger integrity, or dispatch failure |
| `P1` | Core feature required for current phase exit criteria |
| `P2` | Important enhancement, not phase-gating |
| `P3` | Nice-to-have, backlog parking lot |

---

## Epic 1: Multi-Tenant Role Access Controls

**Goal:** Enforce enterprise RBAC across API, web portal, and Flutter client with warehouse-scoped visibility and regional reporting for operations leadership.

**Business value:** Enables Bengkulu pilot → regional expansion without cross-tenant data leakage. Supports Merchant Admin self-service while giving Regional Managers and Fleet Operators governed oversight.

### 1.1 RBAC Foundation (Shipped — v1.0.0)

- [x] `P0` Define `UserRole` enum: `SUPERADMIN`, `REGIONAL_MANAGER`, `HEAD_OF_WAREHOUSE`, `FLEET_OPERATOR`, `MERCHANT_ADMIN`, `DRIVER_PARTNER`
- [x] `P0` Implement permission matrix in `@fleetflow/shared` (`PERMISSIONS`, `ROLE_PERMISSIONS`)
- [x] `P0` Add `User` Prisma model with `merchantId` / `driverId` scoping links
- [x] `P0` JWT login endpoint `POST /v1/auth/login` with role + password validation
- [x] `P0` `HybridAuthGuard`: support `x-api-key` OR Bearer JWT on order endpoints
- [x] `P0` `PermissionsGuard` + `RequirePermissions` / `RequireAnyPermission` decorators
- [x] `P0` Seed six RBAC users with password `FleetFlow!2026`
- [x] `P1` Integration tests for order access scoping (`rbac.integration.spec.ts`)

### 1.2 Warehouse Manager CRUD (Phase 2)

- [ ] `P1` Add `Warehouse` model: `id`, `regionCode`, `name`, `capacityLimit`, `currentLoad`, `hubLat`, `hubLng`
- [ ] `P1` Link `Driver` → `warehouseId` for hub assignment
- [ ] `P1` `HEAD_OF_WAREHOUSE` endpoints: `GET /v1/warehouses/:id/utilization`, `PATCH /v1/warehouses/:id/capacity`
- [ ] `P1` Enforce warehouse concurrent order cap before BullMQ enqueue
- [ ] `P1` CRUD UI in Next.js `/admin/warehouses` with role guard
- [ ] `P2` Warehouse limit breach response: `429 Too Many Requests` with `Retry-After` header
- [ ] `P2` Audit log table: `AuditEvent` with `actorId`, `action`, `entityType`, `entityId`, `metadata`

### 1.3 Regional Reports (Phase 2)

- [ ] `P1` `GET /v1/reports/regional/orders` — aggregated counts by `status`, `vehicleTypeRequired`, date range
- [ ] `P1` `GET /v1/reports/regional/cancellations` — `MATCHING` failure rate by hub
- [ ] `P1` `GET /v1/reports/regional/ledger` — read-only merchant wallet summary (no PII export)
- [ ] `P2` CSV export for Regional Manager dashboard
- [ ] `P2` Scheduled report email digest (weekly SLA summary)

### 1.4 Fleet Operator Driver Telemetry (Phase 2–3)

- [ ] `P1` `GET /v1/fleet/drivers` — paginated list with `status`, `currentLat`, `currentLng`, `vehicle.type`
- [ ] `P1` `PATCH /v1/fleet/drivers/:id/status` — transition `AVAILABLE` | `ON_TRIP` | `OFFLINE`
- [ ] `P1` Real-time map component in Next.js showing driver pins (Mapbox / Google Maps)
- [ ] `P2` WebSocket or SSE channel `fleet:positions` for sub-5s telemetry refresh
- [ ] `P2` Geofence alert when driver exits assigned hub boundary
- [ ] `P3` Historical route playback for completed deliveries

---

## Epic 2: Geolocation Dispatch Loop

**Goal:** Reliable automated driver matching using Haversine distance, BullMQ orchestration, and strict order state transitions from intake through assignment or cancellation.

**Business value:** Core product differentiator — sub-3-second matching SLA with transparent timeline audit for merchants and operators.

### 2.1 Matching Engine Core (Shipped — v1.0.0)

- [x] `P0` Haversine distance function with Earth radius 6371 km
- [x] `P0` `findClosestDriverWithinRadius` — 10 km strict filter, ascending sort
- [x] `P0` `MatchingProcessor` on `dispatch-queue` with `@nestjs/bullmq`
- [x] `P0` State transitions: `DRAFT` → `PENDING` → `MATCHING` → `ASSIGNED` | `CANCELLED`
- [x] `P0` `OrderTimeline` entries at every transition with human-readable notes
- [x] `P0` Vehicle type hard filter: `vehicle.type === order.vehicleTypeRequired`
- [x] `P1` Unit tests for geo-matching and pricing (`geo-matching.service.spec.ts`, `pricing.service.spec.ts`)

### 2.2 BullMQ Process Hardening (Phase 2)

- [ ] `P0` Idempotent job processing: guard against double-assignment if job retried
- [ ] `P1` Dead-letter queue for jobs exhausted after 3 attempts
- [ ] `P1` `GET /v1/ops/queues/dispatch` — queue depth, active jobs, failed count
- [ ] `P1` Metrics: job duration histogram, matching success rate counter
- [ ] `P2` Priority lanes: EXPRESS orders jump queue head
- [ ] `P2` Staggered retry per vehicle type when first pass finds zero candidates

### 2.3 Worker Failure Retries (Phase 2)

- [x] `P1` Exponential backoff: `attempts: 3`, `delay: 2000ms` (baseline)
- [ ] `P1` Distinguish transient failures (DB connection) vs permanent (invalid order ID)
- [ ] `P1` Alert Fleet Operator when job lands in failed state after max retries
- [ ] `P2` Manual requeue endpoint: `POST /v1/orders/:id/requeue` (`FLEET_OPERATOR` only)
- [ ] `P2` Circuit breaker: pause enqueue if Redis unavailable > 30s

### 2.4 Post-Assignment Lifecycle (Phase 2)

- [x] `P1` `POST /v1/orders/:id/pickup` — driver marks `PICKED_UP`
- [x] `P1` `POST /v1/orders/:id/deliver` — driver marks `DELIVERED`, driver returns `AVAILABLE`
- [x] `P1` Flutter: wire active trip actions to live API
- [x] `P2` Proof-of-delivery / departure photo upload (Cloudinary + local fallback)
- [x] `P2` Ops manual completion override with audit reason
- [ ] `P2` Customer notification webhook on `DELIVERED`

### 2.5 Bengkulu → Regional Coordinate Indexing (Phase 2)

- [ ] `P1` Composite index `(status, vehicleTypeRequired)` on `drivers` join `vehicles`
- [ ] `P1` PostGIS extension evaluation for production geo queries
- [ ] `P2` Pre-computed hub bounding boxes to reduce candidate set before Haversine
- [ ] `P3` Python matching microservice extraction behind HTTP adapter

---

## Epic 3: Financial Ledger & Wallets

**Goal:** Double-entry style transaction bookkeeping with merchant prepaid wallets, driver partner payouts, and platform fee retention — all inside atomic Prisma transactions.

**Business value:** Financial trust for B2B merchants; auditable revenue capture for FleetFlow operations.

### 3.1 Ledger Foundation (Shipped — v1.0.0)

- [x] `P0` `Transaction` model: `DEBIT` | `CREDIT`, optional `merchantId` / `driverId`
- [x] `P0` Merchant balance field on `Merchant` model
- [x] `P0` Pre-create balance validation in `OrdersService.createOrder`
- [x] `P0` Assignment-time debit + driver credit (90%) in matching processor transaction block
- [x] `P0` Seed ledger entries for three merchants (low balance + sufficient balance scenarios)
- [x] `P1` API key merchant `ff_live_merchant_startup_1b4d8e6f` — insufficient balance test case

### 3.2 Double-Entry Bookkeeping Hardening (Phase 2)

- [ ] `P0` Re-read merchant balance inside assignment transaction (optimistic concurrency / `SELECT FOR UPDATE`)
- [ ] `P1` `platformFee` column on `Order` — explicit `price * 0.10` persistence
- [ ] `P1` `Transaction` linkage: `orderId` foreign key on every order-related ledger row
- [ ] `P1` Ledger reconciliation endpoint: `GET /v1/ledger/reconcile?date=YYYY-MM-DD`
- [ ] `P2` Immutable ledger: DB trigger preventing UPDATE/DELETE on `transactions`
- [ ] `P2` Merchant wallet top-up via Stripe PaymentIntent (stub → live)

### 3.3 Driver Partner Payouts (Phase 2–3)

- [ ] `P1` `GET /v1/drivers/me/wallet` — accumulated credits, pending payout
- [ ] `P1` Payout request flow: driver initiates → ops approval → bank transfer record
- [ ] `P2` Weekly automated payout batch for partners above minimum threshold
- [ ] `P2` Tax withholding flag per driver registration profile
- [ ] `P3` Instant payout premium tier (fee surcharge)

### 3.4 Multi-Tenant Wallet Isolation (Phase 2)

- [ ] `P0` Verify API key cannot read another merchant's transactions
- [ ] `P1` `SUPERADMIN` ledger dashboard with tenant filter
- [ ] `P1` Regional Manager read-only wallet summaries aggregated by hub
- [ ] `P2` Low-balance alert webhook for merchants below configurable threshold
- [ ] `P2` Credit line support for enterprise merchants (`ff_live_merchant_enterprise_9c2a5d1b` tier)

---

## Epic 4: Quality Assurance Automation

**Goal:** Continuous verification across unit, integration, E2E, smoke, and mobile widget layers — mapped directly to PRD acceptance criteria.

**Business value:** Prevents regression in matching logic, RBAC scoping, and merchant order flows during rapid multi-repo development.

### 4.1 API Testing (Shipped — v1.0.0)

- [x] `P0` Jest unit config (`test/jest-unit.json`)
- [x] `P0` Geo-matching unit tests (Haversine accuracy, radius filter, closest selection)
- [x] `P0` Pricing service unit tests (vehicle tier ordering)
- [x] `P0` Health integration tests via Supertest (`health.integration.spec.ts`)
- [x] `P0` RBAC integration tests (`rbac.integration.spec.ts`)
- [x] `P1` Live stack smoke spec (`stack-smoke.e2e.spec.ts`) with `QA_RUN_LIVE_STACK` gate
- [ ] `P1` Auth login integration test with seeded `MERCHANT_ADMIN` user
- [ ] `P1` Order create integration test with mocked Prisma + BullMQ
- [ ] `P2` Matching processor integration test with testcontainers Postgres + Redis

### 4.2 Playwright E2E — Web Portal (Shipped — v1.0.0)

- [x] `P0` `playwright.config.ts` with Next.js `webServer` auto-start on `:3001`
- [x] `P0` `e2e/portal.spec.ts` — home page and create form field visibility
- [x] `P0` `e2e/orders.spec.ts` — full create → tracker flow with mocked API
- [x] `P1` Jest unit test for `extractApiErrorMessage` (`orders.test.ts`)
- [ ] `P1` Playwright test against live API (optional `QA_RUN_LIVE_STACK` mode)
- [ ] `P1` Visual regression baseline for order tracker step UI
- [ ] `P2` Accessibility audit (`axe-playwright`) on create order form
- [ ] `P2` RBAC login flow E2E for `MERCHANT_ADMIN` JWT path

### 4.3 Flutter Reactive Connection Widget Tests (Shipped baseline — v1.0.0)

- [x] `P0` Widget test: login validation error on empty password (`login_page_test.dart`)
- [x] `P0` Widget test: login navigation on valid credentials (`login_page_test.dart`)
- [x] `P0` Widget test: job offer accept banner (`job_offer_page_test.dart`)
- [x] `P0` Integration test: full driver login → accept job (`integration_test/app_test.dart`)
- [ ] `P1` Widget test: loading spinner state on `login_button` during submit
- [ ] `P1` Integration test with mock `dio` against `POST /v1/auth/login`
- [ ] `P2` Golden tests for `LoginPage` and `JobOfferPage` themes
- [ ] `P2` Background location ping widget test with `geolocator` mock

### 4.4 Infrastructure & Smoke QA (Shipped — v1.0.0)

- [x] `P0` `fleetflow-infra/qa/smoke-stack.mjs` — API liveness, readiness, web portal HTTP 200
- [x] `P0` Root scripts: `pnpm test:qa`, `pnpm test:qa:web`, `pnpm test:qa:flutter`, `pnpm test:smoke`
- [x] `P1` `QA_TESTING.md` full pyramid documentation
- [ ] `P1` GitHub Actions workflow: `test:qa` on every PR
- [ ] `P1` GitHub Actions workflow: `test:smoke` on merge to `main` with docker compose
- [ ] `P2` Prisma sync automation in CI (`pnpm prisma:sync --seed`)
- [ ] `P2` Test coverage reporting to PR comments

### 4.5 Verification Matrix Maintenance (Ongoing)

- [ ] `P1` Link each PRD acceptance criterion to a test file in `QA_TESTING.md`
- [ ] `P1` Quarterly backlog grooming: close shipped items, re-prioritize `P0` security tasks
- [ ] `P2` Chaos test: kill Redis mid-matching, verify order does not double-debit

---

## Backlog Statistics

| Epic | Shipped | Remaining | P0 Open |
|------|---------|-----------|---------|
| Epic 1: RBAC | 8 | 17 | 0 |
| Epic 2: Dispatch | 7 | 18 | 1 |
| Epic 3: Ledger | 6 | 14 | 1 |
| Epic 4: QA | 16 | 14 | 0 |
| **Total** | **37** | **63** | **2** |

---

**Grooming notes (2026-07-11):** Focus next sprint on Epic 2.2 idempotent matching and Epic 3.2 balance re-read inside transaction — both are P0 ledger integrity hardening items before Bengkulu production pilot.
