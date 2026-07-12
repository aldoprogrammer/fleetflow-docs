# FleetFlow User Operations Manual — Head of Warehouse

**Document ID:** FF-MANUAL-HEAD-OF-WAREHOUSE-001  
**Role:** `HEAD_OF_WAREHOUSE`  
**Version:** 1.0.0  
**Last Updated:** 2026-07-11  
**Audience:** Warehouse facility managers, inbound/outbound supervisors, hub operations leads

---

## 1. Role Overview

The **Head of Warehouse** manages physical warehouse operations for a **single hub** within the FleetFlow network. This role enforces multi-tenant warehouse capacity limits, coordinates inbound parcel staging from multiple merchants, sorts and prioritizes outgoing dispatch packages, assigns bulk operational tasks to Fleet Operators, and oversees barcode-driven intake workflows.

Heads of Warehouse bridge physical logistics (packages, shelving, scanning) and digital dispatch (BullMQ queues, order states). They report to the **Regional Manager** and direct **Fleet Operators** during daily shift operations.

### 1.1 Permission Matrix

| Permission | Granted | Operational Meaning |
|------------|---------|---------------------|
| `orders:read:all` | Yes | View all orders routed through their hub |
| `orders:cancel` | Yes | Cancel pre-pickup orders (damaged parcel, merchant recall) |
| `fleet:manage` | Yes | Hub fleet utilization and capacity tools |
| `ledger:read` | Yes | Read-only hub financial summary |
| `drivers:manage` | No | Driver HR managed by Regional Manager |
| `merchants:manage` | No | Merchant onboarding managed regionally |
| `orders:create` | No | Merchants create orders |

### 1.2 Seed Credentials (Development)

```
Email:    warehouse.head@fleetflow.dev
Password: FleetFlow!2026
Role:     HEAD_OF_WAREHOUSE
```

---

## 2. Authentication & Hub Console Access

### 2.1 Login

**Endpoint:** `POST /v1/auth/login`

```json
{
  "email": "warehouse.head@fleetflow.dev",
  "password": "FleetFlow!2026",
  "role": "HEAD_OF_WAREHOUSE"
}
```

### 2.2 Hub Console (Phase 2)

| URL | Purpose |
|-----|---------|
| `/warehouse/dashboard` | Hub utilization overview |
| `/warehouse/inbound` | Incoming parcel staging queue |
| `/warehouse/outbound` | Dispatch-ready packages |
| `/warehouse/capacity` | Capacity limit configuration |
| `/warehouse/tasks` | Fleet Operator task assignments |

---

## 3. Warehouse Capacity Management

### 3.1 Capacity Model (Phase 2)

Each warehouse hub maintains capacity counters enforced at order enqueue time.

| Constraint | Default (Bengkulu Pilot) | Enforcement |
|------------|--------------------------|-------------|
| Max concurrent `ASSIGNED` orders | 50 | Order create pre-check |
| Max `MATCHING` queue depth | 20 | BullMQ enqueue guard |
| Max inbound staging parcels | 200 | Inbound scan gate |
| Max outbound dock slots | 10 | Physical loading bay |

### 3.2 Capacity Monitoring Procedure

**Every 2 hours during operating window (06:00–22:00 WIB):**

1. Open hub utilization dashboard.
2. Record current values:
   - `assignedCount` / `capacityLimit` (ASSIGNED orders)
   - `matchingQueueDepth` / `matchingQueueLimit`
   - `inboundStagingCount` / `inboundStagingLimit`
3. If any metric ≥ **85%** → activate throttling protocol (Section 3.3).
4. If any metric ≥ **95%** → halt new intake; notify Regional Manager.

### 3.3 Throttling Protocol

| Utilization | Action |
|-------------|--------|
| 85–94% | Notify Fleet Operator to accelerate outbound; defer non-urgent bulk uploads |
| 95–99% | Return `429 Too Many Requests` on new order enqueue for hub (Phase 2) |
| 100% | Emergency: cancel lowest-priority `PENDING` orders with merchant notification |

**429 response format (Phase 2):**
```json
{
  "success": false,
  "statusCode": 429,
  "message": "Warehouse WH-BKL-001 at capacity. Retry after 300 seconds.",
  "timestamp": "2026-07-11T10:00:00.000Z"
}
```
Header: `Retry-After: 300`

### 3.4 Capacity Configuration (Phase 2)

**Endpoint:** `PATCH /v1/warehouses/:id/capacity`

```json
{
  "capacityLimit": 75,
  "matchingQueueLimit": 25,
  "inboundStagingLimit": 250,
  "alertThresholdPercent": 85
}
```

Changes require Regional Manager approval above +20% adjustment.

---

## 4. Inbound Inventory Staging

### 4.1 Inbound Workflow Overview

```
Merchant creates order → Driver ASSIGNED → Driver arrives at hub
  → Parcel received at dock → Barcode scan → Staging shelf
  → Sort by outbound route → Driver pickup confirmation → PICKED_UP
```

### 4.2 Physical Staging Layout

| Zone | Label | Purpose |
|------|-------|---------|
| A | Express (< 2h SLA) | Immediate outbound priority |
| B | Standard | Normal dispatch queue |
| C | Bulk / TRUCK | Large parcels, palletized goods |
| D | Hold / Exception | Damaged, address dispute, merchant hold |

### 4.3 Inbound Receiving Procedure

1. Driver or merchant courier arrives at warehouse dock.
2. Warehouse clerk verifies order ID against `ASSIGNED` orders list.
3. **Barcode scan** parcel label (Phase 2 — see Section 7).
4. System confirms `order.id` matches scan → status prep for `PICKED_UP`.
5. Physical condition check:
   - **Accept:** Place on staging shelf in appropriate zone (A/B/C).
   - **Reject:** Move to Zone D; Head of Warehouse cancels order with reason "damaged at intake".
6. Update inbound staging count on dashboard.

### 4.4 Multi-Tenant Inbound Sorting

When multiple merchants deliver to the same hub:

| Sort Key | Priority |
|----------|----------|
| SLA tier (Express first) | 1 |
| `vehicleTypeRequired` (BIKE parcels batched) | 2 |
| Delivery zone cluster (minimize driver travel) | 3 |
| `createdAt` (FIFO within same tier) | 4 |

**Sorting procedure:**
1. Group scanned parcels by delivery zone cluster (geohash prefix).
2. Assign each group to outbound dock slot.
3. Notify Fleet Operator of batch ready for driver assignment confirmation.

### 4.5 Inbound Exception Handling

| Exception | Action | System Update |
|-----------|--------|---------------|
| Parcel damaged | Photo + Zone D hold | `orders:cancel` with note |
| Wrong order ID scanned | Reject intake | No status change; return to sender |
| Oversized for vehicle type | Contact Merchant Admin | Upgrade to TRUCK or cancel |
| Merchant recall | Merchant confirms | `orders:cancel` pre-pickup |
| Duplicate scan | Alert clerk | Investigate duplicate order |

---

## 5. Outbound Dispatch Coordination

### 5.1 Outbound Ready Criteria

An order is outbound-ready when:
1. Status is `ASSIGNED`
2. Parcel is physically staged (barcode scanned at inbound)
3. Assigned driver is `ON_TRIP` or `AVAILABLE` and en route
4. Dock slot allocated

### 5.2 Driver Pickup at Dock

1. Driver arrives at assigned dock slot.
2. Clerk scans parcel barcode → `PATCH /v1/orders/:id/pickup` (Phase 2).
3. Driver confirms parcel count matches assignment.
4. Status transitions `ASSIGNED` → `PICKED_UP`.
5. Driver departs for delivery coordinates.

### 5.3 End-of-Day Clearance

- [ ] Zero parcels remaining in Zone A (express must clear daily)
- [ ] Zone D exceptions documented and merchants notified
- [ ] Inbound staging count = 0
- [ ] Capacity counters reset for next operating day
- [ ] Fleet Operator confirms all `ON_TRIP` drivers accounted for

---

## 6. Assigning Bulk Tasks to Fleet Operators

### 6.1 Task Types

| Task ID | Description | Typical Assignee | Priority |
|---------|-------------|------------------|----------|
| `T-DISPATCH` | Process outbound dock assignments | Fleet Operator on dispatch desk | High |
| `T-REQUEUE` | Requeue stalled `PENDING` orders | Fleet Operator with API access | Critical |
| `T-DRIVER-SYNC` | Verify all on-shift drivers show `AVAILABLE` | Fleet Operator mobile | Medium |
| `T-SCAN-AUDIT` | Reconcile barcode scans vs system orders | Warehouse clerk + Fleet Operator | Daily |
| `T-CAPACITY` | Report hub utilization to Regional Manager | Head of Warehouse | Every 2h |
| `T-MAINT` | Vehicle inspection logging | Fleet Operator | Weekly |

### 6.2 Task Assignment Procedure

1. Head of Warehouse opens `/warehouse/tasks` console (Phase 2).
2. Create task: type, description, priority, deadline.
3. Assign to Fleet Operator on current shift.
4. Fleet Operator acknowledges task in ops mobile view.
5. On completion, operator marks done with notes.
6. Head of Warehouse reviews completion audit.

### 6.3 Shift Handover Template

```
Hub: WH-BKL-001
Shift: 06:00–14:00 → 14:00–22:00
Handover time: 13:45 WIB

Inbound staging: 12 parcels (Zone B: 8, Zone C: 4)
Outbound pending: 5 ASSIGNED awaiting driver pickup
Zone D exceptions: 1 (order ff-xxx, damaged — cancelled)
Capacity: ASSIGNED 38/50, MATCHING queue 6/20
Active issues: Redis slow since 12:30 — Fleet Op monitoring
Tasks assigned to next shift: T-DISPATCH (3 pending), T-SCAN-AUDIT
```

---

## 7. Barcode Scanning Workflows

### 7.1 Barcode Standards (Phase 2)

| Label Type | Format | Content |
|------------|--------|---------|
| Order label | Code 128 | `FF-{orderId}-{checksum}` |
| Merchant manifest | QR Code | JSON: `{ merchantId, externalRef, batchId }` |
| Shelf location | Code 39 | `ZONE-{A|B|C|D}-SHELF-{nnn}` |
| Dock slot | Code 39 | `DOCK-{nn}` |

### 7.2 Scanning Hardware

- Handheld barcode scanners (USB/Bluetooth) connected to warehouse workstation
- Phase 3: Flutter warehouse companion app on rugged tablets

### 7.3 Inbound Scan Workflow

1. Clerk scans order barcode on parcel.
2. System calls `POST /v1/warehouse/inbound-scan`:

```json
{
  "orderId": "uuid",
  "scannedAt": "2026-07-11T08:30:00.000Z",
  "warehouseId": "WH-BKL-001",
  "dockId": "DOCK-03",
  "condition": "ACCEPTED"
}
```

3. System validates:
   - Order exists and status is `ASSIGNED`
   - Order routed to this warehouse hub
   - Inbound staging capacity not exceeded
4. On success: display shelf assignment (e.g., `ZONE-B-SHELF-042`).
5. On failure: red alert with reason (wrong hub, wrong status, capacity full).

### 7.4 Outbound Scan Workflow

1. Driver presents ID at dock.
2. Clerk scans parcel barcode.
3. System calls `POST /v1/warehouse/outbound-scan`:
4. Validates driver `assignedDriverId` matches order.
5. Status → `PICKED_UP`; driver departs.

### 7.5 Daily Scan Reconciliation

**Task `T-SCAN-AUDIT` procedure:**

1. Export scan log for shift: `inbound_scans` + `outbound_scans`.
2. Compare against `ASSIGNED` and `PICKED_UP` order sets.
3. Discrepancies:
   - **Scanned but no order:** Investigate mislabeled parcel
   - **Order ASSIGNED but no scan:** Locate physical parcel in staging
4. Document reconciliation report; submit to Regional Manager weekly.

---

## 8. Fleet Utilization (Hub Scope)

### 8.1 Hub Fleet Dashboard

| Metric | Formula | Healthy Range |
|--------|---------|---------------|
| Available drivers | Count `AVAILABLE` at hub | ≥ 3 per vehicle type |
| On-trip ratio | `ON_TRIP / total hub drivers` | 40–70% |
| Avg match distance | Mean `matchDistanceKm` for hub orders | < 5 km |
| Pickup wait time | `PICKED_UP` timestamp − `ASSIGNED` timestamp | < 30 minutes |

### 8.2 Coordination with Fleet Operator

Head of Warehouse directs physical flow; Fleet Operator manages digital driver states. Joint decisions required for:

- Surge staffing (call in offline drivers)
- Vehicle type rebalancing (shift CAR driver to BIKE zone)
- Emergency order cancellation affecting dock schedule

---

## 9. Financial Visibility (Read-Only)

Head of Warehouse has `ledger:read` for hub-level summaries only.

| Report | Access | Use |
|--------|--------|-----|
| Hub daily order value | Yes | Capacity planning |
| Individual merchant balances | No | Regional Manager scope |
| Driver payouts | No | Fleet Operator / Regional scope |
| Platform fee totals | Aggregate only | Executive reporting |

---

## 10. Troubleshooting

| Symptom | Action |
|---------|--------|
| Capacity dashboard shows 0 but dock is full | Force refresh; verify scan logs syncing |
| Barcode scan rejected | Confirm order status is `ASSIGNED` and correct hub |
| Merchant parcel not in system | Verify Merchant Admin created order; check `PENDING` queue |
| Zone D overflowing | Escalate to Regional Manager; notify affected merchants |
| Fleet Operator not responding to task | Phone escalation; reassign task to backup operator |

---

## 11. Glossary

| Term | Definition |
|------|------------|
| **Staging** | Physical holding area for inbound parcels before outbound |
| **Zone D** | Exception/hold area for damaged or disputed parcels |
| **Capacity limit** | Maximum concurrent orders hub can process |
| **Dock slot** | Physical loading bay for driver pickup |
| **Scan reconciliation** | Matching physical scans to system order states |

---

**Document owner:** Platform Operations · **Next review:** 2026-08-11
