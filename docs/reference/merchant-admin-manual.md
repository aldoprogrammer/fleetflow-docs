# FleetFlow User Operations Manual — Merchant Admin

**Document ID:** FF-MANUAL-MERCHANT-ADMIN-001  
**Role:** `MERCHANT_ADMIN`  
**Version:** 1.0.0  
**Last Updated:** 2026-07-11  
**Audience:** B2B shipper operations staff, ERP integration engineers, tenant onboarding coordinators

---

## 1. Role Overview

The **Merchant Admin** is the primary operational user for a FleetFlow B2B tenant (Merchant). This role manages dispatch intake, monitors order lifecycle from creation through driver assignment, maintains prepaid wallet balance, and coordinates multi-tenant onboarding when a shipper joins a regional warehouse network.

Merchant Admins operate within **strict tenant isolation**: they can create orders and read only orders belonging to their linked `merchantId`. They cannot view other merchants' data, modify driver assignments directly, or access regional fleet telemetry.

### 1.1 Permission Matrix

| Permission | Granted | Operational Meaning |
|------------|---------|---------------------|
| `orders:create` | Yes | Submit single or bulk dispatch orders |
| `orders:read:own` | Yes | View orders scoped to own `merchantId` |
| `orders:read:all` | No | Cannot view other tenants' orders |
| `orders:cancel` | No | Must request cancellation via ops (Phase 2 self-service) |
| `ledger:read` | No | Wallet balance visible via merchant profile only |
| `merchants:manage` | No | Cannot modify tenant configuration |

### 1.2 Typical Personnel

- Dispatch coordinator at Acme Commerce Jakarta
- ERP integration owner connecting WMS to FleetFlow API
- Branch operations lead for a merchant with multiple warehouse pickup points

---

## 2. Authentication & Access

### 2.1 Dashboard Login (JWT)

**Endpoint:** `POST /v1/auth/login`

```json
{
  "email": "merchant.admin@acme-commerce.id",
  "password": "FleetFlow!2026",
  "role": "MERCHANT_ADMIN"
}
```

**Response:** Bearer `accessToken` (7-day default expiry), user profile with `merchantId` and `permissions` array.

**Profile check:** `GET /v1/auth/me` with header `Authorization: Bearer <token>`.

### 2.2 Machine-to-Machine (API Key)

For ERP, WMS, or scripted integrations, use the merchant `x-api-key` header instead of JWT.

| Merchant (seed) | API Key | Wallet Balance (IDR) |
|-----------------|---------|----------------------|
| Acme Commerce Jakarta | `ff_live_merchant_acme_7f3c9a2e` | 5,000,000 |
| Enterprise Retail Nusantara | `ff_live_merchant_enterprise_9c2a5d1b` | 12,500,000 |
| Startup Logistics ID | `ff_live_merchant_startup_1b4d8e6f` | 25,000 |

**Header:** `x-api-key: ff_live_merchant_acme_7f3c9a2e`

API key authentication grants exactly `orders:create` and `orders:read:own`. Keys must never be committed to source control or shared across tenants.

### 2.3 Web Portal Entry Points

| URL | Purpose |
|-----|---------|
| `http://localhost:3001/` | Operations portal home |
| `http://localhost:3001/orders/create` | Single order creation form |
| `http://localhost:3001/orders/{id}` | Live order tracker (3-second poll) |

---

## 3. Multi-Tenant Onboarding

Multi-tenant onboarding is the process of registering a new B2B Merchant tenant, assigning warehouse branch affiliation, provisioning API credentials, and funding the prepaid wallet before first dispatch.

### 3.1 Onboarding Workflow (Coordinated with SUPERADMIN / Regional Manager)

**Phase 1 (current):** Onboarding is performed by platform administrators via database seed or admin tooling. Merchant Admin receives credentials after provisioning.

**Phase 2 (planned):** Self-service merchant registration with Regional Manager approval.

| Step | Actor | Action | System Record |
|------|-------|--------|---------------|
| 1 | Regional Manager | Submit merchant application with company name, email, tax ID | `Merchant` draft |
| 2 | SUPERADMIN | Approve tenant, generate unique `apiKey` | `Merchant.apiKey` |
| 3 | SUPERADMIN | Assign initial wallet credit or payment link | `Transaction` CREDIT |
| 4 | Merchant Admin | Complete first login, rotate password | `User` linked to `merchantId` |
| 5 | Merchant Admin | Configure default pickup branch (Phase 2) | `Warehouse` affiliation |
| 6 | Merchant Admin | Run test order in sandbox hub | Order `DRAFT` → `ASSIGNED` |

### 3.2 Tenant Isolation Rules

1. Each `Merchant` row owns a unique `apiKey` and `email`.
2. All `Order` rows carry `merchantId` foreign key — queries always filter by this field for Merchant Admin.
3. Ledger `Transaction` rows with `merchantId` are visible only to platform roles with `ledger:read`; Merchant Admin sees balance via merchant profile, not raw ledger export (Phase 2 merchant ledger UI).

### 3.3 Branch Configuration (Phase 2)

When operating across multiple warehouse hubs (e.g., Bengkulu main hub + Palembang branch), Merchant Admin will configure:

| Setting | Description | Example |
|---------|-------------|---------|
| `defaultWarehouseId` | Primary pickup staging hub | Bengkulu Hub `WH-BKL-001` |
| `allowedWarehouseIds` | Branches permitted for dispatch | `[WH-BKL-001, WH-PLM-002]` |
| `pickupCutoffTime` | Latest same-day intake | `17:00 WIB` |

Until Phase 2 ships, all orders use free-form pickup coordinates without warehouse binding.

---

## 4. Warehouse Branch Configuration (Operational Reference)

Merchant Admins do not manage warehouse capacity directly (that is **Head of Warehouse** responsibility), but must understand branch constraints when planning dispatch volume.

### 4.1 Branch Selection at Order Create

When creating an order, specify pickup coordinates within the branch service area. The matching engine uses pickup `(pickupLat, pickupLng)` for Haversine driver search — not warehouse ID in Phase 1.

**Best practice:** Maintain an internal lookup table mapping branch addresses to validated latitude/longitude pairs. Use the portal coordinate fields or API payload directly.

### 4.2 Branch Capacity Awareness

| Signal | Merchant Admin Action |
|--------|----------------------|
| Orders stuck in `MATCHING` > 10s | Contact Fleet Operator; likely driver shortage in radius |
| `429` warehouse capacity (Phase 2) | Defer bulk upload; retry after `Retry-After` seconds |
| High `CANCELLED` rate | Review vehicle type selection — ensure `BIKE`/`CAR`/`TRUCK` matches available fleet |

---

## 5. Order Creation — Single Dispatch

### 5.1 Web Portal Procedure

1. Navigate to `/orders/create`.
2. Select **Vehicle type required:** `BIKE`, `CAR`, or `TRUCK`.
3. Enter **Pickup address** (minimum 8 characters).
4. Enter **Delivery address** (minimum 8 characters).
5. Verify or adjust **Coordinates** (latitude −90 to 90, longitude −180 to 180).
6. Click **Create order**.
7. On success, portal redirects to `/orders/{id}` tracker.

### 5.2 API Procedure

**Endpoint:** `POST /v1/orders`

**Headers:**
```
x-api-key: ff_live_merchant_acme_7f3c9a2e
Content-Type: application/json
```

**Body:**
```json
{
  "vehicleTypeRequired": "CAR",
  "pickupAddress": "Jl. Thamrin No. 1, Jakarta Pusat",
  "deliveryAddress": "Jl. Sudirman No. 52, Jakarta Selatan",
  "pickupLat": -6.2,
  "pickupLng": 106.816666,
  "deliveryLat": -6.17511,
  "deliveryLng": 106.865036
}
```

### 5.3 Pricing Preview Logic

Price is computed server-side — merchants cannot set arbitrary prices.

| Vehicle | Base Fare (IDR) | Per-km Rate (IDR) |
|---------|-----------------|-------------------|
| BIKE | 15,000 | 2,500 |
| CAR | 35,000 | 4,500 |
| TRUCK | 90,000 | 9,000 |

**Formula:** `price = baseFare + ceil(haversineDistanceKm) × perKmRate`

### 5.4 Balance Pre-Check

Before enqueueing dispatch, the API verifies `merchant.balance >= price`. If insufficient:

```json
{
  "success": false,
  "statusCode": 400,
  "message": "Insufficient merchant balance. Required 48250, available 25000.",
  "timestamp": "2026-07-11T09:30:00.000Z"
}
```

**Important:** Balance is **not debited** at create time. Debit occurs only when order reaches `ASSIGNED`.

---

## 6. Bulk Order CSV Manifest Upload (Phase 2)

> **Status:** Planned for Phase 2. Documented here as operational target workflow.

### 6.1 CSV Format Specification

| Column | Required | Type | Example |
|--------|----------|------|---------|
| `external_ref` | Yes | string | `PO-2026-00142` |
| `vehicle_type` | Yes | enum | `CAR` |
| `pickup_address` | Yes | string | `Jl. Thamrin No. 1` |
| `pickup_lat` | Yes | float | `-6.2` |
| `pickup_lng` | Yes | float | `106.816666` |
| `delivery_address` | Yes | string | `Jl. Sudirman No. 52` |
| `delivery_lat` | Yes | float | `-6.17511` |
| `delivery_lng` | Yes | float | `106.865036` |
| `branch_code` | No | string | `WH-BKL-001` |

### 6.2 Upload Procedure (Phase 2)

1. Navigate to `/orders/bulk-upload`.
2. Select CSV file (max 500 rows per batch in pilot).
3. System validates each row against schema and cumulative wallet requirement.
4. Preview total estimated cost vs available balance.
5. Confirm upload — orders created in `DRAFT` → `PENDING` sequentially.
6. BullMQ enqueues one `dispatch-order` job per row.
7. Download result report: `order_id`, `external_ref`, `status`, `error_message`.

### 6.3 Bulk Upload Error Handling

| Error | Resolution |
|-------|------------|
| Row validation failure | Fix CSV row; failed rows do not block valid rows (Phase 2) |
| Insufficient total balance | Top up wallet before resubmitting full batch |
| Warehouse capacity `429` | Split batch across time windows |

**Phase 1 workaround:** Script `POST /v1/orders` per row via API key with rate limit of 10 req/sec.

---

## 7. BullMQ Queue Status Tracking

Every successfully created order enqueues a job on the **`dispatch-queue`** with job name **`dispatch-order`**.

### 7.1 Order State Machine (Merchant-Visible)

```
DRAFT → PENDING → MATCHING → ASSIGNED → PICKED_UP → DELIVERED
                              ↘ CANCELLED
```

| Status | Meaning for Merchant Admin |
|--------|---------------------------|
| `DRAFT` | Priced, persisted (transient — immediately advances) |
| `PENDING` | Queued for BullMQ worker pickup |
| `MATCHING` | Worker active; searching AVAILABLE drivers within 10 km |
| `ASSIGNED` | Driver matched; wallet debited; driver en route |
| `CANCELLED` | No driver found in radius, or ops cancellation |
| `PICKED_UP` | Driver collected parcel (Phase 2 endpoint) |
| `DELIVERED` | Delivery confirmed (Phase 2 endpoint) |

### 7.2 Live Tracker (Web Portal)

The **Order Tracker** at `/orders/{id}` polls `GET /v1/orders/:id` every **3 seconds** until status is `DELIVERED` or `CANCELLED`.

**Vertical step UI mapping:**

| UI Step | Order Statuses |
|---------|----------------|
| Order Placed | `DRAFT`, `PENDING` |
| Finding Driver | `MATCHING` |
| Driver Dispatched | `ASSIGNED` |
| Trip Active | `PICKED_UP` |
| Completed | `DELIVERED` |

### 7.3 Timeline Audit Trail

Every status change appends an `OrderTimeline` row:

```json
{
  "id": "uuid",
  "status": "MATCHING",
  "note": "Searching for available driver within 10 km radius.",
  "createdAt": "2026-07-11T09:30:02.000Z"
}
```

Merchant Admin should use timeline notes as primary diagnostic when escalating to Fleet Operator.

### 7.4 Queue SLA Expectations

| Metric | Target |
|--------|--------|
| `PENDING` → `MATCHING` | < 1 second (worker pickup) |
| `MATCHING` → terminal state | < 3 seconds (p95) |
| Full create → `ASSIGNED` | < 5 seconds (p95, driver available) |

If SLA breached, collect `order.id` and timeline timestamps before contacting support.

### 7.5 Merchant Queue Visibility (Phase 2)

Planned endpoint: `GET /v1/merchants/me/queue-status` returning count of orders in `PENDING` and `MATCHING` for own tenant.

---

## 8. Financial Ledger & Platform Fee

### 8.1 Wallet Model

Merchants maintain a **prepaid wallet** (`Merchant.balance` in IDR). All dispatch costs draw from this balance upon successful driver assignment.

### 8.2 Fee Structure on Assignment

When order reaches `ASSIGNED`, a single Prisma transaction block executes:

| Entry | Party | Amount | Type |
|-------|-------|--------|------|
| Dispatch charge | Merchant | `order.price` | `DEBIT` |
| Driver payout | Driver Partner | `order.price × 0.90` | `CREDIT` |
| Platform fee | FleetFlow (implicit) | `order.price × 0.10` | Revenue |

**Example:** Order price IDR 100,000
- Merchant debited: 100,000
- Driver credited: 90,000
- Platform retains: 10,000

### 8.3 Ledger Records

Each financial event creates a `Transaction` row:

```json
{
  "id": "uuid",
  "merchantId": "merchant-uuid",
  "driverId": null,
  "amount": 100000,
  "type": "DEBIT",
  "description": "Dispatch settlement for order {orderId}",
  "createdAt": "2026-07-11T09:30:05.000Z"
}
```

Merchant Admin cannot query raw ledger in Phase 1. Balance is reflected on merchant record after assignment.

### 8.4 Wallet Top-Up (Phase 2)

| Method | Procedure |
|--------|-----------|
| Bank transfer | Regional Manager confirms → SUPERADMIN posts CREDIT |
| Stripe PaymentIntent | Self-service top-up in portal (Phase 3) |

### 8.5 Reconciliation Checklist (Monthly)

1. Export assigned orders for billing period (Phase 2 report).
2. Sum `order.price` for all `ASSIGNED` + `DELIVERED` orders.
3. Compare against wallet DEBIT transactions.
4. Discrepancy > 0 → escalate to SUPERADMIN with order IDs.

---

## 9. ERP Integration Guide

### 9.1 Idempotency Pattern

Use `external_ref` in Phase 2 bulk CSV or maintain a local mapping table keyed by ERP order ID to FleetFlow `order.id` to prevent duplicate dispatch on retry.

### 9.2 Polling vs Webhooks

**Phase 1:** Poll `GET /v1/orders/:id` until terminal state.  
**Phase 2:** Register webhook `POST https://merchant-erp.com/fleetflow/events` for `order.assigned`, `order.cancelled`, `order.delivered`.

### 9.3 Error Response Handling

All API errors return:

```json
{
  "success": false,
  "statusCode": 400,
  "message": "Human-readable description",
  "timestamp": "ISO-8601"
}
```

Implement retry with exponential backoff for `500` and `503`. Do not retry `400` balance errors without wallet top-up.

---

## 10. Troubleshooting

| Symptom | Likely Cause | Action |
|---------|--------------|--------|
| `401 Invalid API key` | Wrong or revoked key | Verify `x-api-key` with SUPERADMIN |
| `400 Insufficient balance` | Wallet below order price | Top up wallet |
| Order `CANCELLED` immediately | No AVAILABLE driver within 10 km matching vehicle type | Retry with different vehicle type or wait for fleet availability |
| Tracker stuck on "Finding Driver" | Worker delay or Redis outage | Contact Fleet Operator |
| `403` on order read | Accessing another merchant's order ID | Verify order belongs to your tenant |

---

## 11. Glossary

| Term | Definition |
|------|------------|
| **Merchant** | B2B tenant with wallet and API key |
| **Haversine** | Great-circle distance formula for driver matching |
| **BullMQ** | Redis-backed job queue powering dispatch matching |
| **OrderTimeline** | Immutable audit log of order status changes |
| **Platform fee** | 10% of order price retained by FleetFlow |

---

**Document owner:** Product Architecture · **Next review:** 2026-08-11
