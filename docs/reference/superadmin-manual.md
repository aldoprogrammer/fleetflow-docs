# FleetFlow User Operations Manual — Super Admin

**Document ID:** FF-MANUAL-SUPERADMIN-001  
**Role:** `SUPERADMIN`  
**Version:** 1.0.0  
**Last Updated:** 2026-07-11  
**Audience:** Platform administrators, system owners, security officers, SRE leads

---

## 1. Role Overview

The **Super Admin** (`SUPERADMIN`) is the highest-privilege role in the FleetFlow ecosystem. This role holds **unrestricted access** to all permissions across all tenants, regions, and warehouses. Super Admins govern platform-wide configuration, merchant and user provisioning, ledger corrections, API key lifecycle, infrastructure health, security policy enforcement, and emergency incident response.

Super Admin actions have system-wide blast radius. Every mutation must follow change management protocol with audit documentation.

### 1.1 Permission Matrix

`SUPERADMIN` is granted **ALL_PERMISSIONS** in `@fleetflow/shared`:

| Permission | Granted | Operational Meaning |
|------------|---------|---------------------|
| `orders:create` | Yes | Create orders on behalf of any merchant (emergency) |
| `orders:read:own` | Yes | Read own-scoped orders |
| `orders:read:assigned` | Yes | Read driver-assigned orders |
| `orders:read:all` | Yes | Read all orders globally |
| `orders:cancel` | Yes | Cancel any order |
| `users:manage` | Yes | Create, disable, reset all user accounts |
| `merchants:manage` | Yes | Provision merchants, rotate API keys |
| `drivers:manage` | Yes | Full driver fleet administration |
| `fleet:manage` | Yes | Global fleet and queue operations |
| `ledger:read` | Yes | Read all financial transactions |
| `ledger:manage` | Yes | Post manual credits, debits, reversals |

### 1.2 Seed Credentials (Development)

```
Email:    superadmin@fleetflow.dev
Password: FleetFlow!2026
Role:     SUPERADMIN
```

**Production requirement:** Super Admin accounts must use hardware-key MFA (Phase 2), IP allowlisting, and separate credentials from all other roles.

---

## 2. Authentication & Security Protocol

### 2.1 Login

**Endpoint:** `POST /v1/auth/login`

```json
{
  "email": "superadmin@fleetflow.dev",
  "password": "FleetFlow!2026",
  "role": "SUPERADMIN"
}
```

### 2.2 Security Constraints

| Control | Development | Production (Required) |
|---------|-------------|----------------------|
| MFA | Not enforced | TOTP + hardware key |
| IP allowlist | None | Office VPN + bastion only |
| Session duration | 7 days JWT | 4 hours with refresh |
| Password policy | Seed password | 16+ chars, rotation 90 days |
| Audit logging | Partial (Phase 2) | Every mutation logged |
| Separate account | Shared seed | Dedicated per admin, no sharing |

### 2.3 Principle of Least Privilege

Super Admin should **not** perform routine operations that other roles can handle:

| Task | Preferred Role |
|------|----------------|
| Daily dispatch monitoring | Fleet Operator |
| Warehouse capacity | Head of Warehouse |
| Regional SLA reports | Regional Manager |
| Merchant order creation | Merchant Admin |
| Driver job fulfillment | Driver Partner |

Use Super Admin only for provisioning, security, ledger corrections, and platform emergencies.

---

## 3. Platform Administration

### 3.1 User Account Management

**Responsibilities:**
- Create `User` records for all six roles
- Link `MERCHANT_ADMIN` → `merchantId`
- Link `DRIVER_PARTNER` → `driverId`
- Disable compromised accounts (`isActive = false`)
- Reset passwords on lockout

**User creation checklist:**

| Step | Action | Verification |
|------|--------|--------------|
| 1 | Create underlying entity (Merchant, Driver) if needed | Record exists in Prisma |
| 2 | Create `User` with correct `role` enum | Role matches job function |
| 3 | Set `passwordHash` via bcrypt (cost 12) | Login test succeeds |
| 4 | Link scope (`merchantId` or `driverId`) | Permission scoping test passes |
| 5 | Communicate credentials securely | Out-of-band channel (not email plaintext in prod) |
| 6 | Force password change on first login (Phase 2) | `mustChangePassword` flag |

### 3.2 Role Assignment Matrix

| Role | `merchantId` | `driverId` | Typical Email Pattern |
|------|-------------|------------|----------------------|
| `SUPERADMIN` | null | null | `superadmin@fleetflow.dev` |
| `REGIONAL_MANAGER` | null | null | `regional.manager@fleetflow.dev` |
| `HEAD_OF_WAREHOUSE` | null | null | `warehouse.head@fleetflow.dev` |
| `FLEET_OPERATOR` | null | null | `fleet.operator@fleetflow.dev` |
| `MERCHANT_ADMIN` | required | null | `*@{merchant-domain}` |
| `DRIVER_PARTNER` | null | required | `driver.*@fleetflow.dev` |

---

## 4. Merchant & API Key Provisioning

### 4.1 New Merchant Onboarding (Super Admin Steps)

1. **Create Merchant record:**
   ```
   companyName, email (unique), balance (initial), apiKey (unique)
   ```
2. **Generate API key** using cryptographically secure random string:
   - Format: `ff_live_{merchant_slug}_{8_char_hex}`
   - Example: `ff_live_merchant_acme_7f3c9a2e`
3. **Post initial wallet CREDIT** transaction if pre-funded:
   ```json
   {
     "merchantId": "uuid",
     "amount": 5000000,
     "type": "CREDIT",
     "description": "Initial merchant wallet top-up"
   }
   ```
4. **Create MERCHANT_ADMIN user** linked to `merchantId`.
5. **Deliver credentials** to Regional Manager for merchant handoff.
6. **Verify** test order: `POST /v1/orders` with new API key → `ASSIGNED` or `CANCELLED`.

### 4.2 API Key Rotation Procedure

1. Generate new key: `ff_live_{merchant_slug}_{new_hex}`.
2. **Dual-key grace period (Phase 2):** Both old and new keys active for 72 hours.
3. Notify Merchant Admin to update ERP integration.
4. Confirm zero traffic on old key for 24 hours (access log review).
5. Revoke old key: replace `Merchant.apiKey` or disable old key record (Phase 2).
6. Document rotation in change log.

### 4.3 API Key Emergency Revocation

If key compromise suspected:

1. **Immediately** replace `Merchant.apiKey` in database.
2. Invalidate active sessions (Phase 2 token revocation list).
3. Notify Merchant Admin and Regional Manager.
4. Review `orders` and `transactions` for anomalous activity in compromise window.
5. File security incident report.

---

## 5. Financial Ledger Administration

### 5.1 Ledger Overview

Super Admin has `ledger:manage` — the only role that can post manual financial corrections.

| Transaction Type | Typical Use |
|------------------|-------------|
| `CREDIT` | Merchant wallet top-up, driver payout correction |
| `DEBIT` | Merchant charge correction, clawback |

### 5.2 Automated Ledger (Do Not Duplicate)

On every `ASSIGNED` order, the matching processor automatically:

1. `DEBIT` merchant `order.price`
2. `CREDIT` driver `order.price × 0.90`
3. Platform retains `order.price × 0.10`

**Never manually post transactions for normal assignments** — this creates duplicate entries.

### 5.3 Manual Credit Procedure (Merchant Top-Up)

1. Receive payment confirmation from finance team.
2. Verify merchant identity and payment reference.
3. Post CREDIT:
   ```json
   {
     "merchantId": "uuid",
     "amount": 10000000,
     "type": "CREDIT",
     "description": "Bank transfer ref: TRF-2026-0711-001"
   }
   ```
4. Update `Merchant.balance` atomically in same Prisma transaction.
5. Notify Merchant Admin that wallet is funded.

### 5.4 Manual Reversal Procedure (Post-Assignment Cancellation)

When order cancelled after `ASSIGNED` (rare, ops exception):

1. Verify cancellation authority (Regional Manager or Head of Warehouse initiated).
2. In single Prisma `$transaction`:
   - `CREDIT` merchant `order.price` (refund)
   - `DEBIT` driver `order.price × 0.90` (clawback) — or adjust pending payout
   - Set order `CANCELLED` with timeline note referencing reversal
   - Set driver `AVAILABLE`
3. Document incident ID in all transaction `description` fields.

### 5.5 Ledger Reconciliation (Daily)

1. Query all `ASSIGNED` + `DELIVERED` orders for past 24h.
2. Sum `order.price` → expected merchant DEBIT total.
3. Sum driver CREDIT → expected `total × 0.90`.
4. Compare against `transactions` table aggregates.
5. Discrepancy ≠ 0 → P0 incident; halt new assignments until resolved.

### 5.6 Ledger Dashboard (Phase 2)

**Endpoint:** `GET /v1/admin/ledger/summary?date=YYYY-MM-DD`

| Metric | Description |
|--------|-------------|
| `totalOrderValue` | Sum of assigned order prices |
| `totalMerchantDebits` | Sum of DEBIT transactions |
| `totalDriverCredits` | Sum of driver CREDIT transactions |
| `platformFeeTotal` | `totalOrderValue × 0.10` |
| `unmatchedEntries` | Orders without corresponding transactions |

---

## 6. Infrastructure & System Health

### 6.1 Local Development Stack

```bash
cd fleetflow-infra && docker compose up -d    # Postgres + Redis
cd ../fleetflow-api && pnpm prisma:sync       # Migrate + seed
pnpm dev                                       # API :3000 + Web :3001
```

### 6.2 Health Endpoints

| Endpoint | Expected | Action if Failing |
|----------|----------|-------------------|
| `GET /v1/health/live` | `{ status: "ok" }` | Restart API process |
| `GET /v1/health/ready` | `{ status: "ready" }` | Check Postgres + Redis |

### 6.3 Smoke Test

```bash
pnpm test:smoke    # From monorepo root
```

Validates API liveness, readiness, and web portal HTTP 200.

### 6.4 Database Operations

| Operation | Command | When |
|-----------|---------|------|
| Schema sync | `pnpm prisma:sync` | After schema change |
| Schema watch | `pnpm prisma:watch` | Active development |
| Studio GUI | `pnpm prisma:studio` | Data inspection |
| Full reset | `docker compose down -v` + `prisma:sync` | Corrupted local DB |

**Production:** Use `prisma migrate deploy` only — never `db push`.

### 6.5 Redis & BullMQ Administration

| Check | Command |
|-------|---------|
| Redis ping | `docker exec fleetflow-redis redis-cli ping` |
| Queue depth (Phase 2) | `GET /v1/ops/queues/dispatch` |
| Flush queue (emergency) | `redis-cli DEL bull:dispatch-queue:*` — **destructive**; requeue orders after |

---

## 7. Security Incident Response

### 7.1 Incident Classification

| Severity | Examples | Response Time |
|----------|----------|---------------|
| P0 | Ledger double-debit, API key leak, cross-tenant data exposure | 15 minutes |
| P1 | Queue stalled > 10 min, RBAC bypass suspected | 1 hour |
| P2 | Elevated cancellation rate, single merchant data issue | 4 hours |
| P3 | Non-critical config drift | Next business day |

### 7.2 Cross-Tenant Data Exposure

1. Immediately disable affected endpoint or account.
2. Identify scope: which merchants' data was exposed, time window.
3. Preserve logs before rotation.
4. Notify affected merchants per PDPA requirements.
5. Root cause analysis within 48 hours.
6. Deploy fix + regression test.

### 7.3 RBAC Bypass Investigation

1. Reproduce with specific role JWT and target endpoint.
2. Inspect `HybridAuthGuard` → `PermissionsGuard` chain.
3. Verify `@RequirePermissions` decorator on controller method.
4. Check `userCanReadOrder` scoping logic.
5. Add integration test before closing incident.

---

## 8. Multi-Tenant & Multi-Region Governance

### 8.1 Tenant Isolation Verification

**Monthly audit checklist:**

- [ ] Merchant A API key cannot read Merchant B orders (integration test)
- [ ] `MERCHANT_ADMIN` JWT scoped to own `merchantId` only
- [ ] `DRIVER_PARTNER` reads only `assignedDriverId` orders
- [ ] No API key or JWT returns `Transaction` data without `ledger:read` permission
- [ ] Swagger docs do not expose seed credentials in production deployment

### 8.2 Regional Expansion Authority

Super Admin approves:
- New warehouse hub creation
- Regional Manager account provisioning
- Database capacity scaling
- Production deployment windows

### 8.3 Warehouse Limit Configuration

Super Admin sets global defaults; Head of Warehouse adjusts within approved bounds:

| Parameter | Global Default | Override Authority |
|-----------|----------------|------------------|
| `capacityLimit` | 50 | Head of Warehouse (±20%), Super Admin (>20%) |
| `matchingQueueLimit` | 20 | Super Admin only |
| Haversine radius | 10 km | Super Admin only (policy change) |
| Platform fee | 10% | Super Admin + executive approval |

---

## 9. Deployment & Release Management

### 9.1 Pre-Release Checklist

- [ ] `pnpm test:qa` passes (unit + integration)
- [ ] `pnpm test:qa:web` passes (Jest + Playwright)
- [ ] `pnpm typecheck` clean
- [ ] `pnpm prisma:deploy` tested on staging
- [ ] `API_SPEC.md` updated for contract changes
- [ ] `BACKLOG.md` checkboxes updated
- [ ] Seed data not deployed to production

### 9.2 Release Procedure

1. Tag release: `v1.x.x`
2. Deploy API container with zero-downtime rolling update (Phase 2).
3. Run `prisma migrate deploy` against production database.
4. Verify `pnpm test:smoke` against production endpoints.
5. Monitor BullMQ queue depth for 30 minutes post-deploy.
6. Announce release to Regional Managers.

### 9.3 Rollback Procedure

1. Revert API container to previous tag.
2. If migration was applied: run down migration (must exist) or forward-fix.
3. Requeue any orders stuck in `PENDING` during outage.
4. Post-incident review within 24 hours.

---

## 10. Data Management & Seeding

### 10.1 Development Seed Data

`pnpm prisma:sync` (includes seed) creates:

| Entity | Count | Notes |
|--------|-------|-------|
| Merchants | 3 | 1 low balance, 2 sufficient |
| Drivers + Vehicles | 5 | Jakarta coordinates, mixed types |
| RBAC Users | 6 | One per role |
| Ledger transactions | 5 | Sample CREDIT/DEBIT entries |

### 10.2 Production Data Rules

- Never run seed script in production.
- Never copy production data to development without PII anonymization.
- Database backups: daily full, hourly WAL (Phase 2 production).

---

## 11. Monitoring & Alerting (Phase 2–3)

| Alert | Condition | Channel |
|-------|-----------|---------|
| Queue stall | `waiting` > 10 for > 60s | PagerDuty P1 |
| Health check fail | `/health/ready` ≠ 200 for 3 consecutive | PagerDuty P0 |
| Ledger mismatch | Daily reconciliation ≠ 0 | Email + PagerDuty P0 |
| High cancellation | Rate > 25% for 1 hour | Slack ops channel |
| API error rate | 5xx > 1% for 5 minutes | PagerDuty P1 |

---

## 12. Troubleshooting (Platform Level)

| Symptom | Action |
|---------|--------|
| Prisma EPERM on generate | Run `pnpm prisma:sync` (auto-kills Studio/API locks) |
| All roles get 403 | Check JWT secret rotation; verify `JWT_SECRET` env |
| BullMQ not processing | Restart API; verify Redis; check worker logs |
| Balance mismatch | Halt assignments; run ledger reconciliation |
| Cross-tenant leak report | P0 — disable endpoint; begin incident response |

---

## 13. Glossary

| Term | Definition |
|------|------------|
| **SUPERADMIN** | Highest-privilege platform administrator |
| **ledger:manage** | Permission to post manual financial transactions |
| **Dual-key grace** | Overlap period during API key rotation |
| **Reconciliation** | Verifying order values match transaction totals |
| **Platform fee** | 10% FleetFlow revenue per assigned order |

---

## 14. Related Role Manuals

| Role | Manual |
|------|--------|
| Merchant Admin | [merchant-admin-manual.md](./merchant-admin-manual.md) |
| Driver Partner | [driver-partner-manual.md](./driver-partner-manual.md) |
| Regional Manager | [regional-manager-manual.md](./regional-manager-manual.md) |
| Head of Warehouse | [head-of-warehouse-manual.md](./head-of-warehouse-manual.md) |
| Fleet Operator | [fleet-operator-manual.md](./fleet-operator-manual.md) |

---

**Document owner:** Platform Security & SRE · **Next review:** 2026-08-11
