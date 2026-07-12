# FleetFlow — Product Requirements Document (PRD)

**Document ID:** FF-PRD-001  
**Product:** FleetFlow On-Demand Logistics & Courier Dispatch System  
**Version:** 1.0.0  
**Status:** Active  
**Last Updated:** 2026-07-11  
**Owner:** Principal Product Architecture  
**Board:** [FleetFlow Production Kanban](https://github.com/users/aldoprogrammer/projects/2)

---

## 1. Executive Summary & Business Scale

### 1.1 Product Vision

FleetFlow is an enterprise-grade on-demand logistics platform that connects **Merchants** (B2B shippers) with **Driver Partners** (courier fleet) through automated dispatch, real-time order lifecycle tracking, and auditable financial ledger operations. The platform is engineered as a **multi-repo, multi-tenant** ecosystem capable of scaling from a single regional operations hub in **Bengkulu** to nationwide deployments with isolated warehouse capacity, regional governance, and per-merchant API key boundaries.

FleetFlow's commercial model centers on **transactional dispatch fees**: merchants prepay wallet balances; upon successful driver assignment, the platform debits the merchant, credits the driver partner at **90% of order price**, and retains a **10% platform fee** as revenue.

### 1.2 Strategic Expansion Trajectory

| Stage | Geography | Tenancy Model | Primary Objective |
|-------|-----------|---------------|-------------------|
| **Pilot** | Bengkulu City + surrounding districts | Single warehouse hub, 3 merchant tenants, 5 driver partners | Prove matching SLA, ledger integrity, and B2B API adoption |
| **Regional** | Sumatra western corridor (Bengkulu → Palembang → Jambi) | Multi-warehouse with regional manager oversight | Enforce warehouse capacity limits, regional reporting, fleet load balancing |
| **National** | Java + outer islands hub-and-spoke | Full multi-tenant isolation per region and merchant | Sub-3s dispatch queue execution at scale, analytics dashboards, Flutter background telemetry |

### 1.3 Business Objectives

1. **Dispatch reliability:** ≥ 95% of `PENDING` orders reach `ASSIGNED` or `CANCELLED` within 10 seconds when an eligible driver exists within the 10 km Haversine radius.
2. **Financial integrity:** Zero orphan ledger entries; every `ASSIGNED` order produces exactly one merchant `DEBIT` and one driver `CREDIT` inside a single Prisma transaction block.
3. **Tenant isolation:** Merchant API keys, wallet balances, and order visibility are strictly scoped; cross-tenant data leakage is a P0 security incident.
4. **Operational visibility:** Regional Managers and Fleet Operators can observe fleet telemetry, order queues, and warehouse utilization without merchant PII exposure beyond policy.

### 1.4 System Boundaries

**In scope (current release train):**

- NestJS REST API (`fleetflow-api`) with `x-api-key` B2B auth and JWT RBAC for dashboard users
- PostgreSQL system of record via Prisma ORM
- Redis-backed BullMQ `dispatch-queue` for asynchronous driver matching
- Next.js operations portal (`fleetflow-web`) for merchant order creation and live tracker
- Flutter driver client (`fleetflow-app`) for partner login and job acceptance flows
- Dockerized local infrastructure (`fleetflow-infra`) for PostgreSQL 15 and Redis 7

**Out of scope (documented for later phases):**

- Multi-region active-active database failover
- gRPC matching microservice extraction (matching currently runs in-process via `matching.processor.ts`)
- Native payment gateway settlement (Stripe integration stubbed for future billing)

---

## 2. User Persona Journeys

### 2.1 Merchant Admin (`MERCHANT_ADMIN`)

**Profile:** Operations staff at a B2B shipper (e.g., Acme Commerce Jakarta). Manages dispatch intake, monitors order status, and maintains prepaid wallet balance.

**Authentication:** JWT login (`POST /v1/auth/login`) with role `MERCHANT_ADMIN`, scoped to `merchantId`. Alternative: machine-to-machine via `x-api-key` header for ERP integrations.

**Primary journey — Create dispatch order:**

1. Merchant Admin opens the Next.js portal at `/orders/create` or calls `POST /v1/orders` with API key.
2. System validates payload: `vehicleTypeRequired` (BIKE | CAR | TRUCK), pickup/delivery addresses, latitude/longitude bounds.
3. `PricingService` computes order price from vehicle base fare + per-km Haversine distance between pickup and delivery coordinates.
4. System verifies `merchant.balance >= price`. If insufficient, returns `400` with explicit balance message via `GlobalExceptionFilter`.
5. Prisma transaction creates order in `DRAFT`, appends `OrderTimeline` entry, transitions to `PENDING`, enqueues BullMQ `dispatch-order` job.
6. Merchant Admin is redirected to `/orders/{id}` tracker; UI polls `GET /v1/orders/:id` every 3 seconds until terminal state.

**Acceptance criteria:**

- [ ] Merchant can only read orders where `order.merchantId === user.merchantId`
- [ ] API key path grants `orders:create` and `orders:read:own` only
- [ ] Balance is not debited until `ASSIGNED` state (debit occurs in matching processor, not at create time)

---

### 2.2 Driver Partner (`DRIVER_PARTNER`)

**Profile:** Independent courier attached to a `Vehicle` (BIKE, CAR, or TRUCK). Operates within a regional hub, publishes live coordinates, accepts assigned jobs via Flutter app.

**Authentication:** JWT login with role `DRIVER_PARTNER`, scoped to `driverId`.

**Primary journey — Receive and fulfill assignment:**

1. Driver Partner sets status to `AVAILABLE` (seed/default for active drivers in Jakarta pilot coordinates).
2. BullMQ `MatchingProcessor` queries drivers where `status = AVAILABLE` and `vehicle.type = order.vehicleTypeRequired`.
3. Haversine distance from driver `(currentLat, currentLng)` to order pickup `(pickupLat, pickupLng)` is computed in-memory.
4. Closest driver within **10 km** is selected. Prisma `$transaction` atomically:
   - Sets order `ASSIGNED`, writes timeline note
   - Sets driver `ON_TRIP`
   - Debits merchant balance by `order.price`
   - Credits driver ledger at `order.price * 0.90`
   - Records platform fee implicitly as `price * 0.10`
5. Driver Partner views assigned order via `GET /v1/orders/:id` (scoped to `assignedDriverId`).
6. Flutter app displays job offer screen; partner accepts job, updates status through future `PICKED_UP` → `DELIVERED` endpoints.

**Failure journey — No driver in radius:**

1. Processor sets order `CANCELLED`
2. Timeline entry documents failure reason: no available driver within 10 km matching window
3. Merchant balance remains unchanged (no debit)

**Acceptance criteria:**

- [ ] Driver can only read orders where `assignedDriverId === user.driverId`
- [ ] Driver status returns to `AVAILABLE` after delivery completion (Phase 2)
- [ ] Flutter integration test validates login → job offer → accept flow

---

### 2.3 Regional Manager (`REGIONAL_MANAGER`)

**Profile:** Oversees multiple warehouse hubs within a geographic region (e.g., Sumatra West). Requires cross-merchant visibility for SLA reporting, not wallet mutation rights.

**Authentication:** JWT with role `REGIONAL_MANAGER`.

**Permissions:** `orders:read:all`, `orders:cancel`, `drivers:manage`, `fleet:manage`, `ledger:read`, `merchants:manage`.

**Primary journey — Regional SLA review:**

1. Regional Manager logs into operations dashboard (Phase 3 analytics view).
2. Queries all orders in region filtered by `status`, `createdAt`, `vehicleTypeRequired`.
3. Reviews cancellation rate for `MATCHING` failures (no driver in 10 km).
4. Initiates driver onboarding or warehouse capacity adjustment via `drivers:manage` and `merchants:manage` permissions.
5. Exports ledger read-only report for merchant wallet activity in region.

**Acceptance criteria:**

- [ ] Regional Manager cannot create orders on behalf of merchants without explicit delegation (Phase 2)
- [ ] Cross-tenant order reads are permitted; wallet mutations are not
- [ ] Audit log captures regional manager actions (Phase 2)

---

### 2.4 Head of Warehouse (`HEAD_OF_WAREHOUSE`)

**Profile:** Manages physical warehouse capacity, inbound parcel staging, and outbound dispatch queues for a single hub. Enforces **multi-tenant warehouse limits** (max concurrent `MATCHING` + `ASSIGNED` orders per hub).

**Authentication:** JWT with role `HEAD_OF_WAREHOUSE`.

**Permissions:** `orders:read:all`, `orders:cancel`, `fleet:manage`, `ledger:read`.

**Primary journey — Warehouse capacity enforcement:**

1. Head of Warehouse monitors active order count against hub `capacityLimit` (Phase 2 schema extension).
2. When hub approaches capacity threshold (e.g., 85%), system throttles new `PENDING` enqueue or routes to alternate hub.
3. Head of Warehouse cancels pre-pickup orders (`orders:cancel`) for operational exceptions (damaged parcel, merchant recall).
4. Reviews fleet utilization: ratio of `AVAILABLE` vs `ON_TRIP` drivers attached to hub.

**Warehouse limit rules (target specification):**

| Constraint | Default (Bengkulu pilot) | Enforcement Point |
|------------|--------------------------|-------------------|
| Max concurrent `ASSIGNED` orders per hub | 50 | Order create pre-check (Phase 2) |
| Max `MATCHING` queue depth per hub | 20 | BullMQ enqueue guard (Phase 2) |
| Max vehicle type mismatch retries | 0 (hard filter) | Matching processor |

**Acceptance criteria:**

- [ ] Warehouse limit breach returns `429` with retry-after header (Phase 2)
- [ ] Cancellation writes `CANCELLED` timeline with operator note and `cancelledBy` user ID

---

### 2.5 Fleet Operator (`FLEET_OPERATOR`)

**Profile:** Real-time dispatch coordinator. Manages driver availability, intervenes in failed matching loops, and monitors BullMQ worker health.

**Authentication:** JWT with role `FLEET_OPERATOR`.

**Permissions:** `orders:read:all`, `drivers:manage`, `fleet:manage`.

**Primary journey — Dispatch loop intervention:**

1. Fleet Operator opens fleet telemetry view showing driver coordinates, `DriverStatus`, and vehicle type.
2. Observes orders stuck in `MATCHING` beyond SLA threshold (default: 5 seconds).
3. Manually overrides driver assignment (Phase 2) or requeues dispatch job with `attempts: 3` exponential backoff (current: 2s base delay).
4. Marks drivers `OFFLINE` or `AVAILABLE` based on shift schedules.
5. Verifies BullMQ `dispatch-queue` consumer is processing: checks Redis connectivity via `GET /v1/health/ready`.

**Acceptance criteria:**

- [ ] Fleet Operator can read all orders regardless of merchant
- [ ] Driver status mutations are logged with timestamp and operator ID (Phase 2)
- [ ] Failed BullMQ jobs surface in ops dashboard after 3 retry exhaustion

---

## 3. Core Matching Logic Specifications

### 3.1 Order State Machine

```
DRAFT → PENDING → MATCHING → ASSIGNED → PICKED_UP → DELIVERED
                              ↘ CANCELLED (no driver within 10 km)
                              ↘ CANCELLED (warehouse ops / merchant cancel)
```

| State | Trigger | Persistent Fields Updated |
|-------|---------|---------------------------|
| `DRAFT` | `POST /v1/orders` initial create | `price`, `vehicleTypeRequired`, coordinates |
| `PENDING` | Same request, pre-enqueue | `OrderTimeline` PENDING entry |
| `MATCHING` | BullMQ worker start | `OrderTimeline` MATCHING entry |
| `ASSIGNED` | Successful match + ledger tx | `assignedDriverId`, `matchDistanceKm`, driver `ON_TRIP` |
| `CANCELLED` | No eligible driver | Timeline failure note, no balance change |

### 3.2 Haversine Radius Filtering Algorithm

**Input:** Order `O` with pickup coordinate `(pickupLat, pickupLng)` and `vehicleTypeRequired`.  
**Driver set:** All `Driver` rows where `status = AVAILABLE` and `vehicle.type = O.vehicleTypeRequired`.

**Algorithm (executed in `geo-matching.service.ts`):**

1. For each candidate driver `D`, compute Haversine distance in kilometers between `(D.currentLat, D.currentLng)` and `(O.pickupLat, O.pickupLng)` using Earth radius `R = 6371 km`.
2. Filter candidates where `distanceKm <= 10.0` (strict 10 km radius, inclusive boundary).
3. Sort ascending by `distanceKm`.
4. Select index 0 (closest driver). Persist `matchDistanceKm` rounded to 3 decimal places.
5. If filtered set is empty, transition order to `CANCELLED`.

**Formula:**

```
a = sin²(Δφ/2) + cos(φ1) · cos(φ2) · sin²(Δλ/2)
distanceKm = 2R · asin(√a)
```

Where `φ` is latitude and `λ` is longitude in radians.

### 3.3 BullMQ Dispatch Worker Flow

**Queue:** `dispatch-queue`  
**Job name:** `dispatch-order`  
**Processor:** `MatchingProcessor` (`@Processor('dispatch-queue')`)

**Execution steps:**

1. Load order by `orderId` from job payload with `merchant` relation.
2. Update order status to `MATCHING`; insert `OrderTimeline` row.
3. Query available drivers with matching vehicle type.
4. Run Haversine filter and selection.
5. **On match — atomic Prisma `$transaction` block:**
   ```text
   BEGIN
     UPDATE orders SET status='ASSIGNED', assignedDriverId, matchDistanceKm
     INSERT order_timelines (ASSIGNED, note with driver name)
     UPDATE drivers SET status='ON_TRIP'
     UPDATE merchants SET balance = balance - order.price
     INSERT transactions (merchantId, DEBIT, amount=order.price)
     INSERT transactions (driverId, CREDIT, amount=order.price * 0.90)
   COMMIT
   ```
6. **On no match — atomic update:**
   ```text
   UPDATE orders SET status='CANCELLED'
   INSERT order_timelines (CANCELLED, failure notice)
   ```

**Retry policy:** `attempts: 3`, exponential backoff `delay: 2000ms`, `removeOnComplete: true`.

### 3.4 Pricing Engine (Pre-Match)

Executed synchronously at order creation before enqueue:

| Vehicle | Base Fare (IDR) | Per-km Rate (IDR) |
|---------|-----------------|-------------------|
| BIKE | 15,000 | 2,500 |
| CAR | 35,000 | 4,500 |
| TRUCK | 90,000 | 9,000 |

`price = baseFare + ceil(haversineKm) × perKmRate`

Balance check uses `merchant.balance >= price` at create time. Debit occurs only at assignment.

---

## 4. Performance & Security Metrics

### 4.1 Performance Targets

| Metric | Target | Measurement Point |
|--------|--------|-------------------|
| Order create API latency (p95) | < 500 ms | `POST /v1/orders` excluding network |
| BullMQ job pickup latency (p95) | < 1,000 ms | Enqueue timestamp → worker start |
| Full matching cycle (p95) | < 3,000 ms | Worker start → `ASSIGNED` or `CANCELLED` commit |
| Order tracker poll interval | 3,000 ms | `OrderTracker.tsx` `setInterval` |
| PostgreSQL query (driver availability) | < 100 ms | Indexed on `drivers.status`, `(currentLat, currentLng)` |
| Redis BullMQ throughput | ≥ 100 jobs/sec | Load test Phase 2 |

### 4.2 Security Requirements

| Control | Implementation | Verification |
|---------|----------------|--------------|
| B2B API key isolation | `ApiKeyGuard` / `HybridAuthGuard` loads `Merchant` by `x-api-key`; orders scoped to `merchantId` | Integration test: cross-merchant read returns `403` |
| JWT RBAC | Six roles with permission matrix in `@fleetflow/shared` | `rbac.integration.spec.ts` |
| Error envelope consistency | `GlobalExceptionFilter` returns `{ success, statusCode, message, timestamp }` | API contract tests |
| Balance manipulation prevention | Ledger mutations only inside Prisma `$transaction` with matching processor | Unit + integration coverage |
| Credential storage | `User.passwordHash` via bcryptjs; seed password rotated in production | Security review checklist |
| CORS restriction | `localhost:3001` allowlist for web portal | `main.ts` configuration audit |

### 4.3 API Key Isolation Rules

1. Each `Merchant` row owns a unique `apiKey` (seed example: `ff_live_merchant_acme_7f3c9a2e`).
2. API key authentication grants only `orders:create` and `orders:read:own`.
3. API keys must never be logged, committed to git, or returned in `GET` responses.
4. Key rotation procedure (Phase 2): generate new key, dual-key grace period, revoke old key.

### 4.4 Safe Balance Validation Rules

1. **Pre-create check:** `merchant.balance >= calculatedPrice` — reject with `400` if false.
2. **Pre-debit check (inside transaction):** Re-read merchant balance; abort if insufficient (race condition guard, Phase 2 hardening).
3. **Immutability:** `Transaction` rows are append-only; no UPDATE or DELETE on ledger table.
4. **Double-entry invariant:** For every `ASSIGNED` order, exactly one merchant `DEBIT` and one driver `CREDIT` must exist with `creditAmount = price × 0.90`.

### 4.5 Non-Functional Requirements Summary

| Category | Requirement |
|----------|-------------|
| Availability (local dev) | Docker Compose healthchecks for Postgres and Redis |
| Observability | Swagger at `/docs`, `OrderTimeline` audit trail per order |
| Testability | Playwright E2E, NestJS integration, Flutter widget + integration suites |
| Portability | Next.js native host; API + data stores dockerized |
| Data residency | Indonesia region preference for production (Phase 3) |

---

## 5. Appendix — Entity Reference

| Model | Purpose |
|-------|---------|
| `Merchant` | B2B tenant, wallet balance, API key |
| `Driver` | Courier partner, live coordinates, vehicle 1:1 |
| `Vehicle` | Plate, type, capacityKg |
| `Order` | Dispatch request, pricing, assignment |
| `OrderTimeline` | Immutable status audit log |
| `Transaction` | Ledger DEBIT/CREDIT entries |
| `User` | RBAC dashboard account linked to merchant or driver |

---

**Approval chain:** Product Architecture → Backend Guild → Mobile Guild → QA Sign-off  
**Next review date:** 2026-08-11
