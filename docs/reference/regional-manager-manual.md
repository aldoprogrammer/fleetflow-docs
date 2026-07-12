# FleetFlow User Operations Manual — Regional Manager

**Document ID:** FF-MANUAL-REGIONAL-MANAGER-001  
**Role:** `REGIONAL_MANAGER`  
**Version:** 1.0.0  
**Last Updated:** 2026-07-11  
**Audience:** Regional operations directors, multi-hub logistics supervisors, SLA compliance officers

---

## 1. Role Overview

The **Regional Manager** governs FleetFlow operations across **multiple warehouse hubs** within a defined geographic region (e.g., Sumatra West: Bengkulu, Palembang, Jambi). This role holds cross-tenant visibility for order SLA reporting, driver fleet management, merchant onboarding oversight, and read-only financial ledger analysis — without direct wallet mutation authority.

Regional Managers bridge strategic planning (capacity, expansion) and tactical intervention (cancellation, driver onboarding, bottleneck resolution). They report to executive leadership and coordinate with **Heads of Warehouse** (per hub) and **Fleet Operators** (per shift).

### 1.1 Permission Matrix

| Permission | Granted | Operational Meaning |
|------------|---------|---------------------|
| `orders:read:all` | Yes | View all orders across all merchants in region |
| `orders:cancel` | Yes | Cancel pre-pickup orders for SLA or ops exceptions |
| `drivers:manage` | Yes | Onboard, suspend, reassign drivers |
| `fleet:manage` | Yes | Fleet-wide telemetry and capacity tools |
| `ledger:read` | Yes | Read-only merchant wallet and transaction reports |
| `merchants:manage` | Yes | Onboard merchants, manage API keys, assign branches |
| `orders:create` | No | Cannot create dispatch on behalf of merchants (Phase 2 delegation) |
| `ledger:manage` | No | Cannot post manual credits/debits |
| `users:manage` | No | Cannot create SUPERADMIN users |

### 1.2 Seed Credentials (Development)

```
Email:    regional.manager@fleetflow.dev
Password: FleetFlow!2026
Role:     REGIONAL_MANAGER
```

---

## 2. Authentication & Dashboard Access

### 2.1 Login Procedure

**Endpoint:** `POST /v1/auth/login`

```json
{
  "email": "regional.manager@fleetflow.dev",
  "password": "FleetFlow!2026",
  "role": "REGIONAL_MANAGER"
}
```

### 2.2 Dashboard Entry (Phase 3)

| URL | Purpose |
|-----|---------|
| `/analytics/regional` | Regional performance dashboard |
| `/analytics/cancellations` | MATCHING failure analysis |
| `/fleet/overview` | Cross-hub fleet map |
| `/merchants` | Tenant management console |
| `/reports/ledger` | Read-only financial reports |

**Phase 1:** Regional Manager uses API directly (`GET /v1/orders/:id` with JWT) and database reporting tools until analytics UI ships.

### 2.3 Session Security

- JWT Bearer token required for all dashboard API calls
- Sessions expire after 7 days — re-authenticate on expiry
- All actions will be audit-logged (Phase 2) with `actorId` and timestamp

---

## 3. Regional Performance Tracking Dashboards

### 3.1 Key Performance Indicators (KPIs)

| KPI | Formula | Target (Pilot) | Target (Regional) |
|-----|---------|----------------|-------------------|
| **Matching Success Rate** | `ASSIGNED / (ASSIGNED + CANCELLED_MATCHING)` | ≥ 85% | ≥ 92% |
| **Matching Latency (p95)** | `MATCHING` start → terminal state | < 3s | < 2s |
| **Cancellation Rate** | `CANCELLED / total orders` | < 15% | < 8% |
| **Fleet Utilization** | `ON_TRIP / (AVAILABLE + ON_TRIP)` | 40–70% | 50–75% |
| **Merchant Wallet Health** | Merchants with balance < 3× avg order price | < 10% | < 5% |
| **Hub Capacity Utilization** | `active orders / capacityLimit` | < 85% | < 80% |

### 3.2 Dashboard Panels (Phase 3)

**Panel 1 — Order Volume Funnel**
```
DRAFT → PENDING → MATCHING → ASSIGNED → PICKED_UP → DELIVERED
                              ↘ CANCELLED
```
Visualize counts at each stage for selected date range and hub filter.

**Panel 2 — Matching SLA Histogram**
Distribution of milliseconds from `MATCHING` timeline entry to `ASSIGNED` or `CANCELLED`. Flag outliers > 5000ms.

**Panel 3 — Vehicle Type Demand vs Supply**
| Vehicle | Orders Requested (24h) | Drivers AVAILABLE | Gap |
|---------|------------------------|-------------------|-----|
| BIKE | 142 | 8 | Surplus |
| CAR | 89 | 3 | **Deficit** |
| TRUCK | 12 | 1 | Balanced |

**Panel 4 — Revenue Summary (Read-Only)**
- Total order value (`ASSIGNED` + `DELIVERED`)
- Platform fee aggregate (10%)
- Driver payout aggregate (90%)

### 3.3 Report Export (Phase 2)

**Endpoint:** `GET /v1/reports/regional/orders?from=YYYY-MM-DD&to=YYYY-MM-DD&hubId=WH-BKL-001`

Export formats: CSV, PDF (Phase 3).

**Scheduled digest:** Weekly email every Monday 08:00 WIB with prior week KPIs (Phase 2).

---

## 4. Driver Capacity Load-Balancing

### 4.1 Load-Balancing Objectives

1. **Minimize `CANCELLED` orders** due to no driver in 10 km radius.
2. **Balance vehicle type supply** against merchant demand patterns.
3. **Prevent hub overload** by distributing drivers across warehouse zones.
4. **Maintain partner earnings equity** — avoid over-reliance on single high-performing driver.

### 4.2 Supply-Demand Analysis Procedure

**Daily (08:00 WIB):**

1. Pull 24-hour order volume by `vehicleTypeRequired` per hub.
2. Count `AVAILABLE` drivers by `vehicle.type` per hub.
3. Calculate **supply gap** = orders − (drivers × estimated trips per driver per day).
4. If gap > 0 for any vehicle type, initiate driver recruitment or inter-hub transfer.

**Example analysis (Bengkulu hub):**

| Vehicle | Avg Daily Orders | AVAILABLE Drivers | Trips/Driver/Day | Capacity | Gap |
|---------|------------------|-------------------|------------------|----------|-----|
| BIKE | 80 | 4 | 15 | 60 | +20 deficit |
| CAR | 40 | 3 | 10 | 30 | +10 deficit |
| TRUCK | 5 | 1 | 3 | 3 | +2 deficit |

**Action:** Recruit 2 BIKE partners; convert 1 CAR order overflow to BIKE where parcel weight permits (merchant communication required).

### 4.3 Inter-Hub Driver Transfer (Phase 2)

1. Identify surplus drivers at Palembang hub (HIGH availability, LOW order volume).
2. Issue temporary reassignment to Bengkulu hub for 7-day window.
3. Update `driver.warehouseId` → Bengkulu.
4. Monitor matching success rate improvement.
5. Return driver to home hub after surge period.

### 4.4 Dynamic Incentive Zones (Phase 3)

Define geofenced zones with elevated driver payout multiplier (e.g., 95% instead of 90%) during peak demand windows to attract `AVAILABLE` drivers into high-order-density areas.

---

## 5. Resolving Dispatch Bottlenecks

### 5.1 Bottleneck Classification

| Type | Symptom | Root Cause | Primary Resolver |
|------|---------|------------|------------------|
| **Matching deficit** | High `CANCELLED` with timeline note "no driver within 10 km" | Driver shortage or wrong vehicle type | Regional Manager + Fleet Operator |
| **Queue stall** | Orders stuck in `PENDING` > 5s | BullMQ worker down, Redis outage | Fleet Operator |
| **Warehouse capacity** | `429` responses on order create (Phase 2) | Hub at `capacityLimit` | Head of Warehouse |
| **Wallet blockage** | Merchant orders fail at create with `400` balance | Merchant underfunded | Regional Manager → merchant outreach |
| **GPS staleness** | Drivers AVAILABLE but never matched | Stale coordinates (Phase 2) | Fleet Operator → driver notification |

### 5.2 Bottleneck Resolution Playbook

**Scenario A — High MATCHING cancellation rate (> 20%)**

1. Query cancelled orders last 24h; group by `vehicleTypeRequired` and pickup zone.
2. Overlay AVAILABLE driver map for same period.
3. If driver deficit confirmed → activate driver recruitment pipeline.
4. If drivers present but distant → evaluate 10 km radius adequacy for zone (executive escalation for policy change).
5. Communicate ETA to affected merchants via ops channel.

**Scenario B — Orders stuck in PENDING**

1. Verify `GET /v1/health/ready` returns `ready`.
2. Escalate to Fleet Operator to inspect BullMQ `dispatch-queue` depth.
3. If Redis down → coordinate with SRE for `docker compose restart redis`.
4. If worker crashed → restart `fleetflow-api` container.
5. Requeue affected orders (Phase 2: `POST /v1/orders/:id/requeue`).

**Scenario C — Single merchant flooding queue**

1. Identify merchant with > 50 orders in `PENDING` within 10 minutes.
2. Contact Merchant Admin — confirm intentional bulk upload.
3. If accidental → cancel batch with `orders:cancel` and advise CSV split.
4. If intentional → verify wallet balance covers total estimated cost.

### 5.3 Escalation Matrix

| Severity | Criteria | Response Time | Escalate To |
|----------|----------|---------------|-------------|
| P0 | Ledger double-debit suspected | 15 minutes | SUPERADMIN |
| P1 | Queue stalled > 5 minutes | 30 minutes | Fleet Operator + SRE |
| P2 | Cancellation rate > 25% for 1 hour | 2 hours | Executive review |
| P3 | Single merchant complaint | 4 hours | Regional Manager resolution |

---

## 6. Cross-Tenant Analytics Reports

### 6.1 Report Types

| Report | Endpoint (Phase 2) | Frequency | Audience |
|--------|-------------------|-----------|----------|
| Regional Order Summary | `/v1/reports/regional/orders` | Daily | Regional Manager |
| Cancellation Analysis | `/v1/reports/regional/cancellations` | Weekly | Regional Manager + executives |
| Merchant Wallet Summary | `/v1/reports/regional/ledger` | Monthly | Regional Manager (read-only) |
| Driver Performance | `/v1/reports/regional/drivers` | Weekly | Fleet Operators |
| Hub Utilization | `/v1/reports/regional/warehouses` | Daily | Heads of Warehouse |

### 6.2 Cross-Tenant Data Handling

Regional Managers may view **aggregated and per-tenant** operational data but must comply with data handling policy:

| Data Type | Access | Restrictions |
|-----------|--------|--------------|
| Order counts by status | Full | None |
| Order addresses | Full | No external sharing without merchant consent |
| Merchant wallet balance | Read-only aggregate | No public disclosure of individual balances |
| Merchant API keys | Manage (rotate) | Never display full key in reports — last 4 chars only |
| Driver PII (phone, name) | Manage scope | GDPR/PDPA compliant storage |
| End customer PII | None in Phase 1 | Not collected in current order model |

### 6.3 Sample Cancellation Report Structure

```csv
date,hub_id,merchant_id,vehicle_type,pickup_zone,cancelled_count,total_count,cancel_rate,primary_reason
2026-07-11,WH-BKL-001,merchant-acme,CAR,central-jakarta,12,80,15.0%,no_driver_10km
2026-07-11,WH-BKL-001,merchant-enterprise,BIKE,south-jakarta,2,45,4.4%,no_driver_10km
```

### 6.4 Merchant Onboarding Analytics

Track funnel for new merchant tenants:

| Stage | Metric |
|-------|--------|
| Application submitted | Count |
| Approved by SUPERADMIN | Count, avg approval time |
| First test order | Count, time from approval |
| First ASSIGNED order | Count, matching success |
| 30-day retention | Merchants with ≥ 10 orders/month |

---

## 7. Merchant & Driver Management

### 7.1 Merchant Onboarding (Regional Manager Actions)

1. Collect merchant application: company name, email, expected monthly volume, primary vehicle types.
2. Submit to SUPERADMIN for `apiKey` provisioning and initial wallet credit.
3. Assign merchant to default warehouse branch.
4. Schedule Merchant Admin training on portal and API.
5. Monitor first-week order success rate.

### 7.2 Driver Onboarding (Regional Manager Actions)

1. Verify driver identity documents and vehicle registration.
2. Create `Vehicle` record (plate, type, capacityKg).
3. Create `Driver` record linked to vehicle and warehouse hub.
4. Create `User` with role `DRIVER_PARTNER` linked to `driverId`.
5. Conduct app training: login, AVAILABLE status, job acceptance, PoD (Phase 2).
6. Monitor first 10 assignments for SLA compliance.

### 7.3 Order Cancellation Authority

Regional Managers may cancel orders with `orders:cancel` permission.

**Cancellation procedure:**
1. Verify order is pre-pickup (`ASSIGNED` but not `PICKED_UP` — Phase 2 status check).
2. Document reason: merchant recall, damaged parcel, ops exception.
3. Execute cancellation → `CANCELLED` timeline entry with operator note.
4. If post-assignment cancellation (Phase 2): coordinate wallet reversal with SUPERADMIN.

---

## 8. Regional Expansion Playbook (Bengkulu → National)

### 8.1 New Hub Launch Checklist

- [ ] Site warehouse facility; record `hubLat`, `hubLng`, `capacityLimit`
- [ ] Deploy FleetFlow infrastructure (Postgres, Redis) or connect to regional DB shard
- [ ] Recruit minimum viable fleet: 3 BIKE, 2 CAR, 1 TRUCK partners
- [ ] Onboard 2+ merchant tenants with funded wallets
- [ ] Run 50 test orders; verify matching success ≥ 85%
- [ ] Appoint Head of Warehouse and Fleet Operator for hub
- [ ] Enable Regional Manager dashboard filter for new hub

### 8.2 Regional Governance Cadence

| Meeting | Frequency | Participants | Outputs |
|---------|-----------|--------------|---------|
| Hub standup | Daily 07:30 | HoW, Fleet Op, Regional Mgr | Bottleneck actions |
| SLA review | Weekly | Regional Mgr, executives | KPI report |
| Fleet planning | Monthly | Regional Mgr, recruitment | Driver hiring plan |
| Merchant review | Quarterly | Regional Mgr, key merchants | Volume forecast |

---

## 9. Troubleshooting

| Symptom | Action |
|---------|--------|
| Cannot see orders from specific merchant | Verify merchant assigned to your region's hub |
| `403` on cancellation | Confirm order is in cancellable state |
| Dashboard data stale | Check date range filter; verify API connectivity |
| Ledger report mismatch | Escalate to SUPERADMIN with transaction IDs |
| Driver shows wrong hub | Update `warehouseId` assignment |

---

## 10. Glossary

| Term | Definition |
|------|------------|
| **Cross-tenant** | Spanning multiple Merchant organizations |
| **Hub** | Physical warehouse operations center |
| **Matching deficit** | Orders cancelled due to no driver in 10 km |
| **Supply gap** | Driver capacity shortfall vs order demand |
| **Platform fee** | 10% FleetFlow revenue per assigned order |

---

**Document owner:** Product Architecture · **Next review:** 2026-08-11
