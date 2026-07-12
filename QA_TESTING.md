# FleetFlow QA & Testing Guide

Full business-logic coverage across API, web, shared RBAC, and infrastructure.

## Business domains covered

| Domain | API unit | API live e2e | Web unit | Web mock e2e | Web live e2e |
|--------|----------|--------------|----------|--------------|--------------|
| Shared RBAC | `shared-rbac.integration.spec.ts` | — | — | — | — |
| Auth login | `auth.service.spec.ts` | `business-logic.e2e` | — | `portal.spec.ts` | `auth.live.spec.ts` |
| RBAC guards | `permissions.guard.spec.ts` | `business-logic.e2e` | `access.test.ts` | `rbac.spec.ts` | `rbac.live.spec.ts` |
| Orders create | `orders.service.spec.ts` | `business-logic.e2e` | — | `orders.spec.ts` | `orders.live.spec.ts` |
| Matching | `matching.processor.spec.ts`, `geo-matching` | `orders-matching.e2e` | — | — | `orders.live.spec.ts` |
| Pricing | `pricing.service.spec.ts` | — | — | — | — |
| Merchants API | — | `business-logic.e2e` | — | — | `rbac.live.spec.ts` |
| Ledger settlement | — | `business-logic.e2e` | — | — | — |
| Health | `health.integration.spec.ts` | `stack-smoke.e2e` | — | `portal.spec.ts` | — |

**Mock e2e does NOT validate dispatch.** Live tests require infra + seed.

---

## Prerequisites

```bash
cd fleetflow-infra && docker compose up -d
pnpm prisma:seed
pnpm dev:api    # :3000
pnpm dev:web    # :3001 (for live Playwright)
```

Before live matching tests:

```bash
pnpm --filter @fleetflow/api run qa:reset-drivers
```

---

## Commands (monorepo root)

```bash
# Unit + integration — no live stack (CI default)
pnpm test:qa

# Web unit + mock Playwright (6 tests)
pnpm test:qa:web

# Live stack smoke
pnpm test:smoke

# Live API: auth, RBAC, merchants, matching, settlement
pnpm test:qa:live

# Live Playwright: auth + RBAC all roles + order matching
PLAYWRIGHT_LIVE_API=true pnpm --filter @fleetflow/web run test:e2e
```

---

## API (`fleetflow-api`)

### Unit

```bash
cd fleetflow-api && pnpm test:unit
```

- Auth, permissions guard, orders service
- Geo matching, pricing, matching processor

### Integration

```bash
pnpm test:integration
```

- Health HTTP, order read RBAC rules

### Live e2e

```bash
pnpm qa:reset-drivers
QA_RUN_LIVE_STACK=true pnpm test:e2e
```

Specs: `orders-matching.e2e.spec.ts`, `business-logic.e2e.spec.ts`, `stack-smoke.e2e.spec.ts`

### Redis + BullMQ manual QA

**Required:** run `pnpm run start:dev` in `fleetflow-api` first.

```powershell
# 1 order — smoke test: is queue + worker working?
pnpm qa:verify-redis-queue

# 50 orders at once (default) — fill the queue under load
pnpm qa:load-test-dispatch

# 100 orders, 25 per batch — change --total and --concurrency as needed
node scripts/load-test-dispatch.mjs --total=100 --concurrency=25

# Terminal 1: watch waiting/active live (leave running)
pnpm qa:watch-dispatch-queue

# Terminal 2: fire orders while watching terminal 1
node scripts/load-test-dispatch.mjs --total=100 --concurrency=25

# Stuck drivers — run before repeating load test
pnpm qa:reset-drivers
```

Details: [fleetflow-api/README.md](../fleetflow-api/README.md#redis--bullmq).

---

## Web (`fleetflow-web`)

### Unit

```bash
cd fleetflow-web && pnpm test
```

- RBAC route matrix (all 6 roles)
- API error helper

### Mock Playwright (default)

```bash
pnpm test:e2e
```

UI smoke only — mocked auth/orders where noted.

### Live Playwright

```bash
PLAYWRIGHT_LIVE_API=true pnpm test:e2e
# or
PLAYWRIGHT_LIVE_API=true pnpm test:e2e:live
```

Real login, all-role route gates, real order matching.

---

## Cursor skill

Read `.cursor/skills/fleetflow-qa-automation/SKILL.md` before adding or changing tests.

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Order CANCELLED in dev | `pnpm --filter @fleetflow/api run qa:reset-drivers` |
| Live e2e skipped | Set `QA_RUN_LIVE_STACK=true`, start API |
| Playwright live skipped | Set `PLAYWRIGHT_LIVE_API=true` |
| False-green mock tests | Run `pnpm test:qa:live` before release |
