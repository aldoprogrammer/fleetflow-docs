# FleetFlow Product Roadmap

**Document ID:** FF-ROADMAP-001  
**Horizon:** 2026 Q3 – 2027 Q2  
**Last Updated:** 2026-07-11  
**Board:** [FleetFlow Production Kanban](https://github.com/users/aldoprogrammer/projects/2)

---

## Roadmap Overview

```
2026 Q3          2026 Q4          2027 Q1          2027 Q2
│ Phase 1        │ Phase 2        │ Phase 2 cont.  │ Phase 3        │
│ MVP Local Core │ Multi-Warehouse│ Regional Scale │ Analytics +    │
│ Bengkulu Pilot │ Scalability   │ Sumatra West   │ Cross-Platform │
└────────────────┴────────────────┴────────────────┴────────────────┘
```

| Phase | Theme | Exit Criteria |
|-------|-------|---------------|
| **Phase 1** | MVP Local Core | Bengkulu hub live with 3 merchants, 5 drivers, full dispatch loop, RBAC, QA pyramid |
| **Phase 2** | Multi-Warehouse Scalability | 3+ hubs, warehouse limits, fleet telemetry, ledger hardening, CI automation |
| **Phase 3** | Analytics & Cross-Platform | Regional dashboards, Flutter background GPS, Playwright CI headless, national prep |

---

## Phase 1: MVP Local Core

**Timeline:** 2026-07-01 → 2026-09-30  
**Geography:** Bengkulu City pilot hub  
**Team focus:** Backend Guild + Web Guild + QA

### 1.1 Objectives

Establish a single-region, fully dockerized local development stack with production-grade NestJS REST endpoints, Prisma data model, BullMQ matching worker, Next.js merchant portal, and Flutter driver client scaffold — all verified by automated test suites.

### 1.2 Deliverables

| Workstream | Deliverable | Status |
|------------|-------------|--------|
| Infrastructure | Docker Compose: `postgres:15-alpine`, `redis:7-alpine`, named volumes `fleetflow_postgres_data` / `fleetflow_redis_data` | ✅ Shipped |
| API Core | `POST /v1/orders`, `GET /v1/orders/:id`, `POST /v1/auth/login`, `GET /v1/auth/me` | ✅ Shipped |
| Data Model | Prisma: `Merchant`, `Driver`, `Vehicle`, `Order`, `OrderTimeline`, `Transaction`, `User` | ✅ Shipped |
| Matching | Haversine 10 km filter, BullMQ `dispatch-queue`, atomic ledger on assign | ✅ Shipped |
| Security | `ApiKeyGuard`, `HybridAuthGuard`, JWT RBAC (6 roles), `GlobalExceptionFilter` | ✅ Shipped |
| Web Portal | Create order form (Formik + Yup), order tracker (3s poll), vehicle type selector | ✅ Shipped |
| Mobile | Flutter login + job offer screens, widget + integration tests | ✅ Shipped |
| QA | Unit, integration, Playwright E2E, infra smoke, `pnpm test:qa` scripts | ✅ Shipped |
| Docs | PRD, Backlog, Roadmap, QA_TESTING.md, API_SPEC.md | ✅ Shipped |
| DevEx | `pnpm prisma:sync` auto-migrate/generate/seed with lock release | ✅ Shipped |

### 1.3 Technical Milestones

**Week 1–2: Foundation**
- [x] pnpm monorepo workspace with `@fleetflow/shared`, `@fleetflow/api`, `@fleetflow/web`
- [x] Prisma schema migration `20260711180000_enterprise_fleetflow_schema`
- [x] Seed data: 3 merchants (1 low balance), 5 Jakarta-positioned drivers, 6 RBAC users
- [x] NestJS global prefix `v1`, Swagger at `/docs`, CORS for `:3001`

**Week 3–4: Dispatch Loop**
- [x] `PricingService` with vehicle-tier fares
- [x] `MatchingProcessor` state machine DRAFT → PENDING → MATCHING → ASSIGNED | CANCELLED
- [x] `OrderTimeline` audit entries at each transition
- [x] Merchant balance pre-check at create; debit at assign

**Week 5–6: Portal & Mobile**
- [x] Next.js App Router pages: `/`, `/orders/create`, `/orders/[id]`
- [x] `CreateOrderForm.tsx` with coordinate validation
- [x] `OrderTracker.tsx` vertical step UI
- [x] Flutter `LoginPage` + `JobOfferPage` with test keys

**Week 7–8: QA & Documentation Hub**
- [x] Playwright `e2e/portal.spec.ts` + `e2e/orders.spec.ts`
- [x] API `geo-matching.service.spec.ts`, `pricing.service.spec.ts`, `rbac.integration.spec.ts`
- [x] `fleetflow-infra/qa/smoke-stack.mjs`
- [x] `fleetflow-docs/` PRD, Backlog, Roadmap, `.cursorrules`

### 1.4 Phase 1 KPIs

| KPI | Target | Current |
|-----|--------|---------|
| Order create → ASSIGNED (driver exists, p95) | < 3s | ~2s local |
| Matching failure transparency | 100% CANCELLED orders have timeline note | ✅ |
| RBAC test coverage | All 6 roles have permission integration tests | Partial (matrix tested) |
| Zero manual Prisma lock steps | `pnpm prisma:sync` handles Studio/API locks | ✅ |

### 1.5 Phase 1 Risks

| Risk | Mitigation |
|------|------------|
| Windows Prisma EPERM on generate | `prisma-sync.mjs` retry + port kill |
| API + Next port conflicts | API `:3000`, Web `:3001` documented |
| Flutter API not yet wired to live backend | Phase 2 integration with `dio` auth |

---

## Phase 2: Multi-Warehouse Scalability

**Timeline:** 2026-10-01 → 2027-03-31  
**Geography:** Bengkulu + Palembang + Jambi hubs  
**Team focus:** Platform SRE + Backend Guild + Fleet Operations

### 2.1 Objectives

Scale from single-hub pilot to multi-warehouse regional deployment with capacity limits, optimized geo queries, fleet load balancing, hardened ledger concurrency, and automated API key lifecycle management.

### 2.2 Deliverables

| Workstream | Deliverable | Target Date |
|------------|-------------|-------------|
| Data Model | `Warehouse` entity with `capacityLimit`, `regionCode`, hub coordinates | 2026-10-15 |
| Capacity | Pre-enqueue warehouse load check; `429` on breach | 2026-10-30 |
| Fleet Balancing | Round-robin driver selection among equidistant candidates within 10 km | 2026-11-15 |
| Indexing | Composite indexes on `drivers(status)`, `orders(status, merchantId, createdAt)` | 2026-11-01 |
| Geo Optimization | Bounding-box pre-filter before Haversine; PostGIS evaluation spike | 2026-12-01 |
| API Keys | Key rotation endpoint, dual-key grace period, audit log | 2026-11-30 |
| Ledger | `SELECT FOR UPDATE` on merchant balance; `orderId` on all transactions | 2026-10-31 |
| BullMQ Ops | Dead-letter queue, `GET /v1/ops/queues/dispatch`, failed job alerts | 2026-12-15 |
| Fleet UI | Next.js `/fleet` map with driver pins, status filters | 2027-01-31 |
| CI/CD | GitHub Actions: `test:qa` on PR, `test:smoke` on main, `prisma:sync` in pipeline | 2026-11-15 |
| Mobile | Flutter `dio` wired to `POST /v1/auth/login` + assigned order fetch | 2027-02-28 |

### 2.3 Technical Milestones

**Q4 2026: Warehouse & Ledger Hardening**
- [ ] `Warehouse` Prisma model + migration
- [ ] `HEAD_OF_WAREHOUSE` capacity CRUD endpoints
- [ ] Idempotent matching processor (prevent double-assign on retry)
- [ ] Platform fee explicit column on `Order`
- [ ] Merchant low-balance webhook

**Q1 2027: Fleet & Indexing**
- [ ] `GET /v1/fleet/drivers` telemetry API
- [ ] Database index review with `EXPLAIN ANALYZE` on driver availability query
- [ ] Fleet Operator manual requeue: `POST /v1/orders/:id/requeue`
- [ ] Post-assignment lifecycle: `PICKED_UP`, `DELIVERED` endpoints
- [ ] Flutter live order polling widget

### 2.4 Phase 2 KPIs

| KPI | Target |
|-----|--------|
| Concurrent orders per hub | 50 ASSIGNED without degradation |
| Matching query p95 | < 50ms with indexes |
| Double-debit incidents | 0 |
| API key rotation downtime | 0 (dual-key grace) |
| CI pipeline pass rate | ≥ 98% on main |

### 2.5 Phase 2 Dependencies

- Phase 1 RBAC and matching loop stable in production pilot
- Docker Compose parity extended to staging environment
- Regional Manager onboarding for Palembang and Jambi hubs

---

## Phase 3: Analytics & Cross-Platform Expansion

**Timeline:** 2027-04-01 → 2027-09-30  
**Geography:** National hub-and-spoke (Java gateway + outer island spokes)  
**Team focus:** Data Platform + Web Guild + Mobile Guild

### 3.1 Objectives

Deliver executive and regional analytics dashboards, real-time fleet tracking for Regional Managers, Flutter background location telemetry for Driver Partners, and fully automated Playwright headless regression in CI — preparing for nationwide merchant onboarding.

### 3.2 Deliverables

| Workstream | Deliverable | Target Date |
|------------|-------------|-------------|
| Analytics | Regional Manager dashboard: order volume, cancellation rate, revenue by hub | 2027-05-31 |
| Live Tracking | SSE/WebSocket order position stream for ops portal | 2027-06-30 |
| Flutter GPS | Background location pings every 30s when `ON_TRIP` | 2027-05-15 |
| Playwright CI | Headless Chromium in GitHub Actions on every PR | 2027-04-30 |
| Reporting | Scheduled CSV/PDF regional SLA reports | 2027-07-31 |
| National Prep | Multi-region `regionCode` partitioning strategy document | 2027-08-31 |
| Payments | Stripe wallet top-up for merchants (live mode) | 2027-09-30 |
| Matching v2 | Optional Python matching microservice behind adapter | 2027-09-30 |

### 3.3 Technical Milestones

**Q2 2027: Dashboards & Telemetry**
- [ ] `GET /v1/reports/regional/orders` with date range and hub filter
- [ ] Next.js `/analytics/regional` charts (order funnel, matching SLA histogram)
- [ ] Flutter `geolocator` background service with battery-aware throttling
- [ ] Driver geofence exit alerts for Fleet Operators

**Q3 2027: Automation & National Scale**
- [ ] Playwright visual regression baselines in CI artifact store
- [ ] `test:qa:live` in nightly cron against staging stack
- [ ] Load test: 500 concurrent `POST /v1/orders` with k6
- [ ] Data residency review for Indonesia PDPA compliance
- [ ] Merchant self-service API key rotation UI

### 3.4 Phase 3 KPIs

| KPI | Target |
|-----|--------|
| Regional dashboard refresh | < 5s for 30-day window |
| Background GPS delivery rate | ≥ 95% pings received when ON_TRIP |
| Playwright CI duration | < 8 minutes per PR |
| National hub count | ≥ 8 warehouses |
| Merchant NPS (pilot cohort) | ≥ 40 |

### 3.5 Phase 3 Risks

| Risk | Mitigation |
|------|------------|
| Background GPS battery drain on Flutter | Adaptive ping interval, foreground service notification |
| Analytics query load on primary DB | Read replica for reporting queries |
| Cross-island latency for matching | Hub-local driver pools, no cross-hub matching |

---

## Cross-Phase Engineering Standards

These standards apply across all phases and are enforced via `.cursorrules` and CI gates:

1. **No pseudo-code** in production TypeScript or Dart files
2. **GlobalExceptionFilter** for all API error responses
3. **Contracts first** — breaking changes require `@fleetflow/shared` version bump
4. **Test pyramid** — every P0 backlog item maps to an automated test before ship
5. **Prisma migrations** — no `db push` in production; use `prisma migrate deploy`

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-07-11 | Initial roadmap: Phase 1 shipped items marked, Phase 2–3 planned |

---

**Next roadmap review:** 2026-10-01 (post Phase 1 exit review)
