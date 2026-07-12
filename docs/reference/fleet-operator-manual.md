# FleetFlow User Operations Manual — Fleet Operator

**Document ID:** FF-MANUAL-FLEET-OPERATOR-001  
**Role:** `FLEET_OPERATOR`  
**Version:** 1.0.0  
**Last Updated:** 2026-07-11  
**Audience:** Dispatch coordinators, shift supervisors, real-time fleet operations staff

---

## 1. Role Overview

The **Fleet Operator** is the real-time dispatch coordinator for FleetFlow operations. This role monitors live driver telemetry, manages driver shift states (`AVAILABLE`, `ON_TRIP`, `OFFLINE`), intervenes in stalled BullMQ matching queues, performs manual dispatch overrides when automation fails, and maintains fleet vehicle maintenance logs.

Fleet Operators are the **first responders** for dispatch incidents. They work under **Head of Warehouse** direction for physical hub flow and report matching anomalies to **Regional Managers**.

### 1.1 Permission Matrix

| Permission | Granted | Operational Meaning |
|------------|---------|---------------------|
| `orders:read:all` | Yes | View every order regardless of merchant |
| `drivers:manage` | Yes | Change driver status, onboard support actions |
| `fleet:manage` | Yes | Telemetry dashboards, queue ops, maintenance logs |
| `orders:cancel` | No | Escalate to Head of Warehouse or Regional Manager |
| `ledger:read` | No | No financial report access |
| `merchants:manage` | No | No merchant configuration |

### 1.2 Seed Credentials (Development)

```
Email:    fleet.operator@fleetflow.dev
Password: FleetFlow!2026
Role:     FLEET_OPERATOR
```

---

## 2. Authentication & Operations Console

### 2.1 Login

**Endpoint:** `POST /v1/auth/login`

```json
{
  "email": "fleet.operator@fleetflow.dev",
  "password": "FleetFlow!2026",
  "role": "FLEET_OPERATOR"
}
```

### 2.2 Operations Console (Phase 2–3)

| URL | Purpose |
|-----|---------|
| `/fleet/map` | Real-time driver position map |
| `/fleet/drivers` | Driver list with status filters |
| `/fleet/queue` | BullMQ dispatch-queue monitor |
| `/fleet/orders` | All orders with status filters |
| `/fleet/maintenance` | Vehicle maintenance log |
| `/fleet/shifts` | Shift schedule management |

**Phase 1:** Fleet Operator uses API endpoints, Prisma Studio (read-only), and health checks until ops UI ships.

---

## 3. Real-Time Driver Telemetry Tracking

### 3.1 Telemetry Data Points

| Field | Source | Refresh Rate |
|-------|--------|--------------|
| `driver.id` | PostgreSQL `drivers` table | On query |
| `driver.fullName` | PostgreSQL | Static |
| `driver.status` | `AVAILABLE` \| `ON_TRIP` \| `OFFLINE` | Real-time (Phase 2: 30s push) |
| `driver.currentLat` | GPS sync (Phase 2) or last known | 30–60s |
| `driver.currentLng` | GPS sync (Phase 2) or last known | 30–60s |
| `vehicle.type` | Joined `vehicles` table | Static |
| `vehicle.plateNumber` | Joined `vehicles` table | Static |
| `activeOrderId` | Orders where `assignedDriverId = driver.id` and status not terminal | Real-time |

### 3.2 Fleet Map Operations (Phase 2)

**Map view features:**

1. **Driver pins** color-coded by status:
   - Green: `AVAILABLE`
   - Blue: `ON_TRIP`
   - Gray: `OFFLINE`
2. **Order pickup pins** for `PENDING` and `MATCHING` orders.
3. **10 km radius circle** around pending pickups showing eligible driver coverage.
4. **Click driver pin** → side panel: name, vehicle, status, last GPS update, active order.

### 3.3 Telemetry API (Phase 2)

**Endpoint:** `GET /v1/fleet/drivers?status=AVAILABLE&hubId=WH-BKL-001`

```json
{
  "drivers": [
    {
      "id": "uuid",
      "fullName": "Alex Rivera",
      "status": "AVAILABLE",
      "currentLat": -6.2012,
      "currentLng": 106.8175,
      "vehicleType": "BIKE",
      "plateNumber": "B-1001-FF",
      "lastLocationAt": "2026-07-11T10:15:30.000Z",
      "activeOrderId": null
    }
  ],
  "total": 1,
  "page": 1
}
```

### 3.4 Telemetry Monitoring Cadence

| Activity | Frequency | Action Threshold |
|----------|-----------|------------------|
| Scan fleet map | Every 15 minutes during shift | Any hub zone with zero AVAILABLE drivers |
| Check GPS freshness | Every 30 minutes | `lastLocationAt` > 5 minutes stale while AVAILABLE |
| Review ON_TRIP duration | Hourly | Trip > 2 hours without `DELIVERED` (Phase 2) |
| Match distance audit | Daily | Avg `matchDistanceKm` > 7 km indicates poor positioning |

### 3.5 Stale GPS Response Procedure

1. Identify driver with stale coordinates (AVAILABLE but old `lastLocationAt`).
2. Send push notification: "Please open FleetFlow app to refresh location."
3. If no response in 10 minutes → call driver.
4. If still stale → set status `OFFLINE` until app confirms fresh GPS.
5. Log incident in shift notes.

---

## 4. Managing Driver Shifts & Active States

### 4.1 Driver Status State Machine

```
OFFLINE → AVAILABLE → ON_TRIP → AVAILABLE → OFFLINE
              ↑                      |
              └──── DELIVERED ───────┘
```

| Transition | Trigger | Who Initiates |
|------------|---------|---------------|
| → `AVAILABLE` | Shift start, trip complete | Driver app or Fleet Operator |
| → `ON_TRIP` | Matching processor assigns order | System (automatic) |
| → `OFFLINE` | Break, end of shift, maintenance | Driver app or Fleet Operator |
| `ON_TRIP` → `AVAILABLE` | Delivery complete (Phase 2) | Driver app |

### 4.2 Shift Schedule Management (Phase 2)

| Shift | Hours (WIB) | Minimum Staff |
|-------|-------------|---------------|
| Morning | 06:00–14:00 | 2 BIKE, 1 CAR, 1 Fleet Operator |
| Afternoon | 14:00–22:00 | 3 BIKE, 2 CAR, 1 TRUCK, 1 Fleet Operator |
| Night | 22:00–06:00 | On-call only (Bengkulu pilot: no night ops) |

### 4.3 Shift Start Procedure

1. Review scheduled drivers for shift.
2. Call/text each driver: confirm attendance.
3. As drivers arrive, verify status → `AVAILABLE` in system.
4. Confirm GPS permissions active on driver devices.
5. Report fleet count to Head of Warehouse: "Shift start: 4 BIKE, 2 CAR AVAILABLE."

### 4.4 Manual Status Override

**Endpoint (Phase 2):** `PATCH /v1/fleet/drivers/:id/status`

```json
{
  "status": "OFFLINE",
  "reason": "Vehicle breakdown — flat tire",
  "operatorNote": "Fleet Op Jane — shift 14:00"
}
```

**When to override (without driver action):**

| Situation | Override To | Reason Code |
|-----------|-------------|-------------|
| Driver phone dead | `OFFLINE` | `NO_DEVICE_CONTACT` |
| Vehicle breakdown | `OFFLINE` | `VEHICLE_MAINTENANCE` |
| Driver no-show | `OFFLINE` | `NO_SHOW` |
| System stuck ON_TRIP after delivery | `AVAILABLE` | `MANUAL_TRIP_RESET` |
| Emergency reassignment needed | `AVAILABLE` | `OPS_OVERRIDE` |

**Audit requirement (Phase 2):** Every manual override logs `operatorId`, `timestamp`, `reason`.

### 4.5 Shift End Procedure

1. Verify no drivers stuck in `ON_TRIP` (escalate incomplete deliveries).
2. Confirm all departing drivers set to `OFFLINE`.
3. Report final fleet count to Head of Warehouse.
4. Complete shift handover notes (open queue issues, maintenance flags).

---

## 5. BullMQ Matching Queue Intervention

### 5.1 Queue Architecture

| Component | Value |
|-----------|-------|
| Queue name | `dispatch-queue` |
| Job name | `dispatch-order` |
| Worker | `MatchingProcessor` |
| Broker | Redis 7 (port 6379) |
| Retry policy | 3 attempts, exponential backoff 2000ms base |
| Job ID format | `dispatch-order:{orderId}` |

### 5.2 Normal Processing Flow

1. Merchant creates order → status `PENDING` → job enqueued.
2. Worker picks up job (< 1s typical).
3. Status → `MATCHING` → Haversine search → `ASSIGNED` or `CANCELLED`.
4. Job completes; `removeOnComplete: true` deletes job record.

**Expected total duration:** < 3 seconds (p95).

### 5.3 Queue Health Monitoring

**Step 1 — API health check:**
```bash
curl http://localhost:3000/v1/health/ready
# Expected: { "status": "ready", "service": "fleetflow-api" }
```

**Step 2 — Redis connectivity:**
```bash
docker exec fleetflow-redis redis-cli ping
# Expected: PONG
```

**Step 3 — Queue depth (Phase 2):**
```
GET /v1/ops/queues/dispatch
```
```json
{
  "queue": "dispatch-queue",
  "waiting": 3,
  "active": 1,
  "completed": 1247,
  "failed": 2,
  "delayed": 0
}
```

### 5.4 Stalled Queue Diagnosis

| Symptom | Diagnosis | Resolution |
|---------|-----------|------------|
| `waiting` count growing, `active` = 0 | Worker not consuming | Restart `fleetflow-api` container |
| `active` stuck > 30s | Worker hung on DB query | Check Postgres connections; restart API |
| `failed` count increasing | Repeated matching errors | Inspect failed job payload in Redis; check logs |
| Orders `PENDING` > 5s, queue empty | Enqueue failure | Verify Redis; check API logs for BullMQ connection error |
| All jobs `completed` but orders still `PENDING` | Status update failure | Database transaction issue — escalate SUPERADMIN |

### 5.5 Manual Requeue Procedure (Phase 2)

When order is stuck in `PENDING` or `MATCHING` without terminal resolution:

**Endpoint:** `POST /v1/orders/:id/requeue`  
**Auth:** `FLEET_OPERATOR` JWT

1. Verify order status is `PENDING` or `MATCHING` (not `ASSIGNED`).
2. Check failed job log for prior attempts.
3. Execute requeue → new `dispatch-order` job enqueued.
4. Monitor order timeline for `MATCHING` → terminal state within 5s.
5. If second requeue fails → escalate to Regional Manager.

**Phase 1 workaround:** Contact SUPERADMIN to manually restart API worker and verify Redis.

### 5.6 Manual Driver Assignment Override (Phase 2)

When automated Haversine matching selects suboptimal driver or fails incorrectly:

**Endpoint:** `POST /v1/orders/:id/assign`

```json
{
  "driverId": "target-driver-uuid",
  "operatorNote": "Manual assign — closest automated driver had vehicle breakdown",
  "overrideReason": "DRIVER_BREAKDOWN"
}
```

**Preconditions:**
- Target driver is `AVAILABLE`
- Vehicle type matches `order.vehicleTypeRequired`
- Driver within 10 km radius (warning if override exceeds — requires supervisor approval)

### 5.7 Dead-Letter Queue (Phase 2)

Jobs exhausted after 3 retries move to dead-letter queue `dispatch-queue-dlq`.

**DLQ procedure:**
1. Review DLQ daily at shift start.
2. For each failed job: inspect `orderId`, error message, attempt count.
3. Fix root cause (DB, Redis, invalid order data).
4. Requeue from DLQ or cancel order with merchant notification.

---

## 6. Fleet Maintenance Logging

### 6.1 Maintenance Log Purpose

Track vehicle condition, scheduled service, and breakdown incidents to ensure driver safety and matching reliability. Poorly maintained vehicles cause `OFFLINE` spikes and matching deficits.

### 6.2 Maintenance Record Schema (Phase 2)

**Endpoint:** `POST /v1/fleet/vehicles/:id/maintenance`

```json
{
  "vehicleId": "uuid",
  "type": "BREAKDOWN",
  "description": "Flat tire — front right. Driver Alex Rivera.",
  "reportedAt": "2026-07-11T14:30:00.000Z",
  "reportedBy": "fleet-operator-user-id",
  "estimatedRepairHours": 4,
  "driverId": "driver-uuid",
  "actionTaken": "Driver set OFFLINE. Backup driver assigned to pending orders."
}
```

| Maintenance Type | Description |
|------------------|-------------|
| `SCHEDULED_SERVICE` | Oil change, tire rotation, inspection |
| `BREAKDOWN` | Unplanned failure during shift |
| `ACCIDENT` | Collision or damage incident |
| `INSPECTION_PASS` | Routine safety check passed |
| `INSPECTION_FAIL` | Vehicle removed from fleet until repaired |

### 6.3 Maintenance Workflow

1. Driver or Fleet Operator reports issue.
2. Fleet Operator creates maintenance log entry.
3. Set affected driver to `OFFLINE`.
4. Notify Head of Warehouse if inbound/outbound capacity impacted.
5. Track repair progress; update log with resolution timestamp.
6. On repair complete: inspection pass → driver returns `AVAILABLE`.

### 6.4 Preventive Maintenance Schedule

| Vehicle Type | Service Interval | Inspection Interval |
|--------------|------------------|---------------------|
| BIKE | Every 3,000 km | Weekly |
| CAR | Every 5,000 km | Weekly |
| TRUCK | Every 10,000 km | Bi-weekly |

Fleet Operator receives automated reminders 7 days before service due (Phase 3).

### 6.5 Maintenance Impact on Matching

When vehicle is in maintenance:
1. Linked driver must be `OFFLINE`.
2. Vehicle excluded from matching pool.
3. Regional Manager notified if fleet deficit exceeds threshold.
4. Maintenance log referenced in cancellation analysis reports.

---

## 7. Order Monitoring & Support

### 7.1 Order List Filters

Fleet Operator views all orders with filters:

| Filter | Use Case |
|--------|----------|
| `status=MATCHING` | Active matching in progress |
| `status=PENDING` | Awaiting worker pickup — stall detection |
| `status=ASSIGNED` + age > 30min | Driver not picking up at warehouse |
| `status=CANCELLED` + today | Daily cancellation review |
| `vehicleTypeRequired=TRUCK` | Low-supply vehicle monitoring |

### 7.2 Order Detail Investigation

**Endpoint:** `GET /v1/orders/:id`

Review `timeline` array for diagnostic sequence:

```json
[
  { "status": "DRAFT", "note": "Order draft created and priced.", "createdAt": "..." },
  { "status": "PENDING", "note": "Order queued for driver matching.", "createdAt": "..." },
  { "status": "MATCHING", "note": "Searching for available driver within 10 km radius.", "createdAt": "..." },
  { "status": "CANCELLED", "note": "No available driver within 10 km matching window.", "createdAt": "..." }
]
```

Time deltas between entries reveal bottleneck location.

---

## 8. Incident Response Runbook

### 8.1 P0 — Queue Completely Stalled

1. Confirm via `GET /v1/health/ready` and Redis `PING`.
2. `docker compose restart fleetflow-api` (if dockerized) or `pnpm dev:api` restart.
3. Monitor queue depth for 5 minutes.
4. Requeue all `PENDING` orders older than 60 seconds.
5. Notify Regional Manager and affected merchants if outage > 5 minutes.

### 8.2 P1 — Single Order Stuck

1. Check order timeline.
2. Verify matching processor logs for `orderId`.
3. Requeue single order.
4. If fails → manual assignment override (Phase 2).

### 8.3 P2 — Driver Complaint

1. Look up driver by name or ID.
2. Review active order and timeline.
3. Coordinate with Head of Warehouse for physical issues.
4. Document in shift notes.

---

## 9. Daily Shift Checklist

### 9.1 Shift Start
- [ ] Login to ops console
- [ ] Verify `GET /v1/health/ready` = ready
- [ ] Check BullMQ queue depth = 0 waiting
- [ ] Confirm scheduled drivers are `AVAILABLE`
- [ ] Review DLQ for overnight failures (Phase 2)
- [ ] Brief with Head of Warehouse on inbound staging

### 9.2 During Shift
- [ ] Monitor fleet map every 15 minutes
- [ ] Respond to stalled `PENDING` orders within 5 minutes
- [ ] Process maintenance reports within 30 minutes
- [ ] Execute tasks assigned by Head of Warehouse

### 9.3 Shift End
- [ ] Zero orders in `PENDING` or `MATCHING` (or hand off with notes)
- [ ] All drivers accounted for (`OFFLINE` or completing final trip)
- [ ] Maintenance log updated
- [ ] Handover notes to next Fleet Operator

---

## 10. Glossary

| Term | Definition |
|------|------------|
| **BullMQ** | Redis-backed job queue for dispatch matching |
| **DLQ** | Dead-letter queue for failed jobs after max retries |
| **Requeue** | Re-submit dispatch job for stalled order |
| **Telemetry** | Live driver GPS and status data |
| **Haversine** | 10 km radius matching formula |

---

**Document owner:** Platform SRE · **Next review:** 2026-08-11
