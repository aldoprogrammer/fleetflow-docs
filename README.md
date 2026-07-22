# FleetFlow Documentation Hub

> **FleetFlow Production Board** · [View Kanban Board →](https://github.com/users/aldoprogrammer/projects/2)

**Product:** FleetFlow — On-Demand Logistics & Courier Dispatch System  
**Classification:** Enterprise Multi-Repo Ecosystem  
**Audience:** Principal Engineers, Product Architects, QA Engineers, Mobile & Backend Guilds  
**Last Updated:** 2026-07-15

---

## Purpose of This Hub

`fleetflow-docs/` is the **single source of truth** for FleetFlow product intent, engineering planning, AI-assisted development rules, and quality verification. Every specification in this folder maps directly to code in sibling packages (`fleetflow-api`, `fleetflow-web`, `fleetflow-app`, `fleetflow-infra`, `fleetflow-shared`) and to automated test scripts executable from the monorepo root.

Read this index first, then drill into the document that matches your role.

### Start here for a live / video demo

→ **[PROOF_OF_DELIVERY.md](./PROOF_OF_DELIVERY.md)** — Driver departure/delivery photo SLA + ops override.

→ **[DEMO_E2E.md](./DEMO_E2E.md)** — Web create order → matcher assigns Alex → Flutter pickup/deliver → web tracker updates live.

→ **[DRIVER_NOTIFICATIONS.md](./DRIVER_NOTIFICATIONS.md)** — Driver assign push + unread inbox (BullMQ + Redis SSE).

→ **[REALTIME_SSE.md](./REALTIME_SSE.md)** — NestJS SSE fix, Redis pub/sub, proof-photo sync, stress testing.

---

## Folder Structure

```
fleetflow-docs/
├── README.md                      ← You are here (documentation index)
├── DEMO_E2E.md                    ← Video / live demo: web ↔ API ↔ Flutter
├── DRIVER_NOTIFICATIONS.md        ← Driver assign alerts (BullMQ + Redis + Flutter)
├── REALTIME_SSE.md                ← SSE architecture, NestJS fix, verification & stress tests
├── PRD.md                         ← Product requirements & acceptance criteria
├── QA_TESTING.md                  ← Full QA pyramid & test commands
├── ARCHITECTURE.md                ← System diagram + portfolio video script
├── .cursorrules                   ← Cursor AI workspace rules (copy to monorepo root)
│
└── docs/
    ├── planning/
    │   ├── BACKLOG.md             ← Epics, sprint tasks, checklists
    │   └── ROADMAP.md             ← Phase 1–3 timeline & milestones
    │
    └── reference/
        ├── merchant-admin-manual.md    ← MERCHANT_ADMIN operations guide
        ├── driver-partner-manual.md    ← DRIVER_PARTNER mobile & matching guide
        ├── regional-manager-manual.md  ← REGIONAL_MANAGER analytics & SLA guide
        ├── head-of-warehouse-manual.md ← HEAD_OF_WAREHOUSE hub operations guide
        ├── fleet-operator-manual.md    ← FLEET_OPERATOR dispatch & telemetry guide
        └── superadmin-manual.md        ← SUPERADMIN platform administration guide
```

### Sibling Package Documentation

| Package | Local Docs | Scope |
|---------|------------|-------|
| `fleetflow-api/` | `README.md`, `API_SPEC.md` | REST endpoints, Prisma, auth, matching |
| `fleetflow-web/` | `README.md`, [web visual preview](../fleetflow-web/docs/web-version-visual-preview/WEB_VERSION_VISUAL_PREVIEW.md) | Next.js portal, Playwright, UI gallery |
| `fleetflow-app/` | `README.md`, [mobile visual preview](../fleetflow-app/docs/mobile-version-visual-preview/MOBILE_VERSION_VISUAL_PREVIEW.md) | Flutter driver client |
| `fleetflow-infra/` | `README.md` | Docker Compose, smoke QA |
| `fleetflow-shared/` | `README.md` | Zod contracts, RBAC permissions |

---

## How to Read the Specifications

### For Product & Architecture

1. Start with **[PRD.md](./PRD.md)** — business scale (Bengkulu → national), persona journeys, Haversine matching specs, ledger rules, performance/security metrics.
2. Read **[docs/planning/ROADMAP.md](./docs/planning/ROADMAP.md)** — what shipped in Phase 1, what is planned for multi-warehouse scale and analytics.
3. Groom tasks from **[docs/planning/BACKLOG.md](./docs/planning/BACKLOG.md)** — epics with checkbox status (`[x]` shipped, `[ ]` remaining).

### For Backend Engineers

1. **PRD §3** — matching state machine, BullMQ worker flow, Prisma transaction block for ledger.
2. **`fleetflow-api/API_SPEC.md`** — request/response contracts, validation limits, error envelope.
3. **BACKLOG Epic 2 & 3** — dispatch loop hardening and wallet bookkeeping tasks.
4. Run `pnpm test:qa` after every change.

### For Web Engineers

1. **PRD §2.1** — Merchant Admin journey (create order → tracker poll).
2. **`fleetflow-web/docs/COMPONENTS.md`** — reusable UI catalog (`AppLink`, skeletons, overlays).
3. **`fleetflow-web/docs/WEB_PATTERNS.md`** — auth, RBAC middleware, loading strategy.
4. **`fleetflow-web/e2e/`** — Playwright specs as executable acceptance criteria.
5. **BACKLOG Epic 4.2** — Playwright and Jest tasks.

### For Mobile Engineers

1. **PRD §2.2** — Driver Partner journey (assignment, job acceptance).
2. **`fleetflow-app/test/widget/`** and **`integration_test/`** — widget and E2E tests.
3. **BACKLOG Epic 4.3** — Flutter reactive connection tasks.

### For QA Engineers

1. **[QA_TESTING.md](./QA_TESTING.md)** — complete test pyramid, commands, CI order, troubleshooting.
2. **Verification Matrix** (below) — PRD requirement → test file mapping.
3. Run `pnpm test:qa:live` before release candidates (requires live stack).

### For Cursor AI Sessions

1. Copy or symlink **[.cursorrules](./.cursorrules)** to the monorepo root as `.cursorrules`.
2. Use project skills in **[`.cursor/skills/`](../.cursor/skills/README.md)** for repetitive workflows (Prisma, web UI, RBAC portal, API endpoints).
3. Cursor will enforce: no pseudo-code, strict TypeScript, `GlobalExceptionFilter` errors, Prisma transaction ledger rules, and dev command shortcuts.

---

## Document Reference

| Document | ID | Description |
|----------|-----|-------------|
| [PRD.md](./PRD.md) | FF-PRD-001 | Product Requirements Document |
| [BACKLOG.md](./docs/planning/BACKLOG.md) | FF-BACKLOG-001 | Product backlog with epic checklists |
| [ROADMAP.md](./docs/planning/ROADMAP.md) | FF-ROADMAP-001 | Phase 1–3 timeline blueprint |
| [QA_TESTING.md](./QA_TESTING.md) | FF-QA-001 | QA automation guide |
| [.cursorrules](./.cursorrules) | FF-CURSOR-001 | Cursor AI code assistance rules |
| [API_SPEC.md](../fleetflow-api/API_SPEC.md) | FF-API-001 | REST API technical specification |
| [COMPONENTS.md](../fleetflow-web/docs/COMPONENTS.md) | FF-WEB-UI-001 | Web reusable component catalog |
| [WEB_PATTERNS.md](../fleetflow-web/docs/WEB_PATTERNS.md) | FF-WEB-ARCH-001 | Portal auth, RBAC, loading patterns |
| [Cursor Skills](../.cursor/skills/README.md) | FF-SKILLS-001 | Agent skills for repetitive tasks |

### User Operations Manuals

| Role | Manual | Seed Email (dev) |
|------|--------|------------------|
| `MERCHANT_ADMIN` | [merchant-admin-manual.md](./docs/reference/merchant-admin-manual.md) | `merchant.admin@acme-commerce.id` |
| `DRIVER_PARTNER` | [driver-partner-manual.md](./docs/reference/driver-partner-manual.md) | `driver.partner@fleetflow.dev` |
| `REGIONAL_MANAGER` | [regional-manager-manual.md](./docs/reference/regional-manager-manual.md) | `regional.manager@fleetflow.dev` |
| `HEAD_OF_WAREHOUSE` | [head-of-warehouse-manual.md](./docs/reference/head-of-warehouse-manual.md) | `warehouse.head@fleetflow.dev` |
| `FLEET_OPERATOR` | [fleet-operator-manual.md](./docs/reference/fleet-operator-manual.md) | `fleet.operator@fleetflow.dev` |
| `SUPERADMIN` | [superadmin-manual.md](./docs/reference/superadmin-manual.md) | `superadmin@fleetflow.dev` |

All seed accounts use password `FleetFlow!2026` in local development.

---

## Verification Matrix

This matrix connects **PRD acceptance criteria** to **automated test artifacts**. A requirement is considered verified when its mapped test passes in CI or local `pnpm test:qa*`.

### Matching & Dispatch

| PRD Requirement | Implementation | Test |
|-----------------|----------------|------|
| Haversine 10 km radius filter | `geo-matching.service.ts` | `fleetflow-api/src/orders/matching/geo-matching.service.spec.ts` |
| Closest driver selection | `findClosestDriverWithinRadius` | `geo-matching.service.spec.ts` |
| No driver → CANCELLED | `matching.processor.ts` | Manual + Phase 2 processor integration test |
| Vehicle type hard filter | `MatchingProcessor` Prisma query | `geo-matching.service.spec.ts` |
| Order pricing by vehicle tier | `pricing.service.ts` | `fleetflow-api/src/orders/pricing/pricing.service.spec.ts` |
| DRAFT → PENDING → MATCHING → ASSIGNED | `orders.service.ts` + processor | `fleetflow-web/e2e/orders.spec.ts` (mocked flow) |

### Security & RBAC

| PRD Requirement | Implementation | Test |
|-----------------|----------------|------|
| API key merchant isolation | `HybridAuthGuard` | `fleetflow-api/test/integration/rbac.integration.spec.ts` |
| MERCHANT_ADMIN read own orders | `userCanReadOrder` | `rbac.integration.spec.ts` |
| DRIVER_PARTNER read assigned orders | `userCanReadOrder` | `rbac.integration.spec.ts` |
| FLEET_OPERATOR read all orders | `userCanReadOrder` | `rbac.integration.spec.ts` |
| Permission matrix correctness | `@fleetflow/shared` RBAC | `rbac.integration.spec.ts` |
| Consistent error envelope | `GlobalExceptionFilter` | `fleetflow-web/src/lib/api/orders.test.ts` |

### Financial Ledger

| PRD Requirement | Implementation | Test |
|-----------------|----------------|------|
| Pre-create balance check | `OrdersService.createOrder` | API manual + insufficient balance seed key |
| Debit only at ASSIGNED | `matching.processor.ts` `$transaction` | Phase 2 integration test (backlog) |
| Driver credit at 90% | `matching.processor.ts` | Phase 2 integration test (backlog) |
| Low-balance merchant rejection | Seed `ff_live_merchant_startup_1b4d8e6f` | Playwright + live API manual |

### Web Portal

| PRD Requirement | Implementation | Test |
|-----------------|----------------|------|
| Create order form validation | `CreateOrderForm.tsx` | `fleetflow-web/e2e/portal.spec.ts` |
| Create → tracker redirect | `CreateOrderForm.tsx` | `fleetflow-web/e2e/orders.spec.ts` |
| 3-second order polling | `OrderTracker.tsx` | `e2e/orders.spec.ts` (ASSIGNED visible) |
| API error message display | `extractApiErrorMessage` | `fleetflow-web/src/lib/api/orders.test.ts` |

### Mobile (Flutter)

| PRD Requirement | Implementation | Test |
|-----------------|----------------|------|
| Login validation | `LoginPage` | `fleetflow-app/test/widget/login_page_test.dart` |
| Login → jobs navigation | `LoginPage` | `login_page_test.dart` |
| Job acceptance UI | `JobOfferPage` | `test/widget/job_offer_page_test.dart` |
| Full driver flow | `FleetFlowApp` routes | `integration_test/app_test.dart` |

### Infrastructure & Health

| PRD Requirement | Implementation | Test |
|-----------------|----------------|------|
| API liveness | `GET /v1/health/live` | `fleetflow-api/test/integration/health.integration.spec.ts` |
| API readiness | `GET /v1/health/ready` | `health.integration.spec.ts` |
| Live stack smoke | Docker Compose + dev servers | `fleetflow-infra/qa/smoke-stack.mjs` |
| Live API + Web e2e | Running stack | `fleetflow-api/test/e2e/stack-smoke.e2e.spec.ts` |

---

## QA Quick Commands

Run from **monorepo root**:

```bash
# CI-friendly (no live stack required)
pnpm test:qa              # API unit + integration
pnpm test:qa:web          # Web Jest + Playwright E2E
pnpm test:qa:flutter      # Flutter widget + integration

# Requires API (:3000) + Web (:3001) running
pnpm test:smoke           # infra smoke script
pnpm test:qa:live         # smoke + API live e2e
```

**Prerequisites for live tests:**

```bash
cd fleetflow-infra && docker compose up -d
cd ../fleetflow-api && pnpm prisma:sync
pnpm dev                  # API + Web in parallel
```

Full details: **[QA_TESTING.md](./QA_TESTING.md)**

---

## Ecosystem Package Map

| Package | Role | Runtime |
|---------|------|---------|
| `fleetflow-shared` | Zod contracts, RBAC `PERMISSIONS`, `USER_ROLES` | TypeScript library |
| `fleetflow-api` | NestJS REST, Prisma, BullMQ matching, JWT + API key | Node.js 20+ (:3000) |
| `fleetflow-web` | Next.js merchant portal, order create, live tracker | Native host (:3001) |
| `fleetflow-app` | Flutter Driver Partner client | iOS / Android / Desktop |
| `fleetflow-infra` | PostgreSQL 15, Redis 7, smoke QA | Docker Compose |
| `fleetflow-docs` | This documentation hub | Markdown |

---

## Reading Order by Role

| Role | Read in order |
|------|---------------|
| **Product Manager** | README → PRD → ROADMAP → BACKLOG |
| **Backend Engineer** | README → PRD §3 → API_SPEC → BACKLOG Epic 2–3 → QA_TESTING |
| **Frontend Engineer** | README → PRD §2.1 → QA_TESTING §Web → BACKLOG Epic 4.2 |
| **Mobile Engineer** | README → PRD §2.2 → fleetflow-app README → BACKLOG Epic 4.3 |
| **QA Engineer** | README → QA_TESTING → Verification Matrix above → BACKLOG Epic 4 |
| **DevOps / SRE** | README → fleetflow-infra README → ROADMAP Phase 2 → QA smoke |
| **Cursor AI** | `.cursorrules` → PRD → API_SPEC |
| **Merchant Admin** | [merchant-admin-manual.md](./docs/reference/merchant-admin-manual.md) |
| **Driver Partner** | [driver-partner-manual.md](./docs/reference/driver-partner-manual.md) |
| **Regional Manager** | [regional-manager-manual.md](./docs/reference/regional-manager-manual.md) |
| **Head of Warehouse** | [head-of-warehouse-manual.md](./docs/reference/head-of-warehouse-manual.md) |
| **Fleet Operator** | [fleet-operator-manual.md](./docs/reference/fleet-operator-manual.md) |
| **Super Admin** | [superadmin-manual.md](./docs/reference/superadmin-manual.md) |

---

## Governance

| Activity | Owner | Artifact |
|----------|-------|----------|
| Product scope change | Product Architecture | `PRD.md` version bump |
| Sprint grooming | Engineering Leads | `BACKLOG.md` checkbox updates |
| Phase gate review | Architecture Board | `ROADMAP.md` exit criteria |
| Test coverage gap | QA Guild | `QA_TESTING.md` + Verification Matrix |
| AI coding standards | Platform Guild | `.cursorrules` |

**Breaking contract changes** require `@fleetflow/shared` update, `API_SPEC.md` revision, and BACKLOG item in Epic 1 or 2.

---

## Related Links

- [FleetFlow Production Kanban](https://github.com/users/aldoprogrammer/projects/2)
- API Swagger (local): `http://localhost:3000/docs`
- Prisma Studio (local): `http://localhost:5555`
- Web Portal (local): `http://localhost:3001`

---

**FleetFlow** — contracts first, scale second, chaos never.
