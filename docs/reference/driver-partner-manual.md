# FleetFlow User Operations Manual — Driver Partner

**Document ID:** FF-MANUAL-DRIVER-PARTNER-001  
**Role:** `DRIVER_PARTNER`  
**Version:** 1.0.0  
**Last Updated:** 2026-07-11  
**Audience:** Independent courier partners, fleet drivers, mobile operations staff

---

## 1. Role Overview

The **Driver Partner** is an independent courier operating under FleetFlow's dispatch network. Each partner is linked to exactly one `Vehicle` (BIKE, CAR, or TRUCK), publishes live GPS coordinates, receives automated assignments via the Haversine matching engine, fulfills deliveries, and accumulates wallet credits at **90% of order price** per completed assignment.

Driver Partners operate under **strict assignment scoping**: they can read only orders where `assignedDriverId` matches their own `driverId`. They cannot create merchant orders, view other drivers' jobs, or access regional analytics.

### 1.1 Permission Matrix

| Permission | Granted | Operational Meaning |
|------------|---------|---------------------|
| `orders:read:assigned` | Yes | View orders assigned to self only |
| `orders:create` | No | Cannot create dispatch orders |
| `orders:read:all` | No | Cannot view unassigned or others' orders |
| `drivers:manage` | No | Cannot change other drivers' status |
| `fleet:manage` | No | No fleet-wide telemetry access |

### 1.2 Driver Status Lifecycle

| Status | Meaning | Matching Eligibility |
|--------|---------|---------------------|
| `AVAILABLE` | On shift, ready for dispatch | **Eligible** — included in Haversine search |
| `ON_TRIP` | Actively fulfilling assigned order | **Not eligible** — excluded from matching |
| `OFFLINE` | Off shift, break, or maintenance | **Not eligible** — excluded from matching |

---

## 2. Authentication & Mobile Access

### 2.1 Flutter App Login

**Application:** `fleetflow-app` (Flutter Driver Partner client)

**Seed credentials (development):**
```
Email:    driver.partner@fleetflow.dev
Password: FleetFlow!2026
Role:     DRIVER_PARTNER
```

**API login (underlying):** `POST /v1/auth/login`

```json
{
  "email": "driver.partner@fleetflow.dev",
  "password": "FleetFlow!2026",
  "role": "DRIVER_PARTNER"
}
```

Response includes `accessToken`, `driverId`, and `permissions: ["orders:read:assigned", "notifications:read"]`.

### 2.2 App Navigation Structure

| Route | Screen | Purpose |
|-------|--------|---------|
| `/login` | `LoginPage` | Email/password authentication (`DRIVER_PARTNER`) |
| `/active` | `ActiveJobsPage` | Assigned / in-progress trips (polls every 5s) |
| `/history` | `HistoryPage` | Delivered / cancelled archive |
| `/notifications` | `NotificationsPage` | Unread / inbox assign alerts (SSE + poll) |
| `/profile` | `ProfilePage` | Session user + sign out |
| `/orders/:id` | `OrderDetailPage` | Map, navigate, confirm pickup / deliver |

Full live demo (merchant web → match → Flutter → web tracker): see [DEMO_E2E.md](../../DEMO_E2E.md).  
Notification design: [DRIVER_NOTIFICATIONS.md](../../DRIVER_NOTIFICATIONS.md).

### 2.3 Session Management

- JWT access token default expiry: **7 days**
- On token expiry, app redirects to login
- Do not share device or credentials with other partners — each `User` links to one `driverId`

---

## 3. Flutter Mobile App Navigation

### 3.1 First Launch Setup

1. Install FleetFlow Driver app on iOS or Android device.
2. Grant **location permissions** (Always Allow recommended for background sync — Phase 2).
3. Grant **camera permissions** for proof-of-delivery photos (Phase 2).
4. Log in with Driver Partner credentials.
5. Verify vehicle assignment displays correctly (plate number, type).

### 3.2 Login Screen Operations

**UI elements (test keys for QA):**
- `brand_title` — FleetFlow branding
- `email_field` — Partner email
- `password_field` — Secure password entry
- `login_button` — Submit authentication

**Validation rules:**
- Empty email or password → error: "Email and password are required."
- Invalid credentials → `401` from API, displayed via error banner

### 3.3 Job Offer Screen Operations

After successful login, partner navigates to **Available job** screen.

**Displayed information:**
- `job_offer_title` — "Delivery offer"
- `pickup_address` — Merchant pickup location
- `dropoff_address` — Delivery destination
- `eta_minutes` — Estimated time of arrival (Phase 2 live calculation)
- `accept_job_button` — Confirm job acceptance

**Acceptance flow (current):**
1. Tap **Accept delivery job**.
2. Loading spinner displays during processing.
3. Success banner: **Job accepted** (`job_accepted_banner`).

**Phase 2 acceptance flow:**
1. Tap accept → `PATCH /v1/orders/:id/accept` confirms partner acknowledgment.
2. Order status remains `ASSIGNED` until pickup scan.

### 3.4 In-Trip Navigation (Phase 2)

1. Active trip screen opens with Google Maps integration (`google_maps_flutter`).
2. Route polyline from current position → pickup → delivery.
3. External navigation handoff to Google Maps / Waze via deep link.
4. Status bar shows order ID, merchant name, parcel notes.

---

## 4. Haversine 10 km Matching Logic

### 4.1 How Assignment Works

Driver Partners do not manually bid on orders. FleetFlow's **MatchingProcessor** (BullMQ worker) automatically selects the closest eligible driver.

**Eligibility criteria (all must be true):**
1. `Driver.status === AVAILABLE`
2. `Vehicle.type === Order.vehicleTypeRequired`
3. Haversine distance from `(driver.currentLat, driver.currentLng)` to `(order.pickupLat, order.pickupLng)` ≤ **10.0 km**

**Selection rule:** Among eligible drivers, the one with the **minimum distance** wins. Distance stored as `order.matchDistanceKm` (3 decimal places).

### 4.2 Haversine Formula Reference

```
a = sin²(Δφ/2) + cos(φ1) · cos(φ2) · sin²(Δλ/2)
distanceKm = 2 × 6371 × asin(√a)
```

Where φ = latitude in radians, λ = longitude in radians, Earth radius = 6371 km.

### 4.3 Matching Window Timing

| Phase | Duration | Partner Experience |
|-------|----------|-------------------|
| Order `PENDING` | < 1s | Order not yet visible to partner |
| Order `MATCHING` | 1–3s typical | Partner not yet assigned |
| Order `ASSIGNED` | Instant on match | Job appears in app; status → `ON_TRIP` |
| No match | < 3s | Order `CANCELLED` — partner never sees it |

### 4.4 Maximizing Match Probability

| Action | Effect |
|--------|--------|
| Set status `AVAILABLE` at shift start | Enters matching pool |
| Position within 10 km of high-demand zones | Higher selection frequency for closest-distance rule |
| Match vehicle type to zone demand | TRUCK orders only match TRUCK vehicles |
| Avoid unnecessary `OFFLINE` periods | Reduces eligible time |
| Keep GPS coordinates fresh (Phase 2) | Accurate distance calculation |

### 4.5 Match Failure Scenarios (Partner Not Selected)

| Scenario | System Behavior |
|----------|-----------------|
| Partner is `ON_TRIP` | Excluded from pool until trip completes |
| Partner is `OFFLINE` | Excluded from pool |
| Partner outside 10 km radius | Excluded — closer driver selected |
| Vehicle type mismatch | Excluded — order requires different type |
| Another driver is closer | Not selected despite being within radius |

---

## 5. Live GPS Background Sync (Phase 2)

> **Status:** Phase 2 deliverable. Phase 1 uses seed coordinates updated at assignment time.

### 5.1 Purpose

Accurate `(currentLat, currentLng)` is critical for Haversine matching. Background GPS sync ensures Fleet Operator telemetry and matching engine use real-time position, not stale coordinates.

### 5.2 Configuration

| Setting | Value | Notes |
|---------|-------|-------|
| Ping interval (ON_TRIP) | 30 seconds | Higher frequency during active delivery |
| Ping interval (AVAILABLE) | 60 seconds | Battery-optimized idle polling |
| Ping interval (OFFLINE) | None | No GPS transmission |
| Accuracy threshold | ≤ 50 meters | Discard readings with poor accuracy |
| Battery saver | Reduce to 120s below 15% battery | Automatic throttling |

### 5.3 API Endpoint (Phase 2)

**Endpoint:** `PATCH /v1/drivers/me/location`

```json
{
  "currentLat": -6.2012,
  "currentLng": 106.8175,
  "accuracyMeters": 12.5,
  "recordedAt": "2026-07-11T10:15:30.000Z"
}
```

### 5.4 Background Service Behavior (Flutter)

1. Foreground service notification displayed on Android during `AVAILABLE` and `ON_TRIP`.
2. iOS uses `UIBackgroundModes: location` with blue status bar indicator.
3. Location batching: queue up to 5 pings offline; flush on connectivity restore.
4. Geofence alert if partner exits assigned hub boundary while `AVAILABLE` (Fleet Operator notified).

### 5.5 GPS Troubleshooting

| Symptom | Action |
|---------|--------|
| Not receiving assignments | Verify status is `AVAILABLE`; check GPS permission |
| Stale position on ops map | Force location refresh in app settings |
| High battery drain | Enable battery saver mode in app; reduce ON_TRIP screen brightness |

---

## 6. Proof-of-Delivery (PoD) Uploads (Phase 2)

### 6.1 When PoD Is Required

Proof-of-delivery is mandatory for all `DELIVERED` orders before driver returns to `AVAILABLE`.

### 6.2 PoD Capture Procedure

1. Arrive at delivery coordinates; confirm within 100m geofence.
2. Tap **Mark Delivered** on active trip screen.
3. Camera opens — photograph parcel at delivery point (minimum 1 photo).
4. Optional: capture recipient signature on screen.
5. Enter delivery notes (e.g., "Left with security guard").
6. Submit → `PATCH /v1/orders/:id/deliver` with multipart upload.

### 6.3 PoD Data Model (Phase 2)

```json
{
  "orderId": "uuid",
  "deliveredAt": "2026-07-11T11:00:00.000Z",
  "photos": ["https://storage.fleetflow.id/pod/uuid/1.jpg"],
  "recipientName": "Security Desk",
  "notes": "Left with security guard",
  "deliveryLat": -6.17511,
  "deliveryLng": 106.865036
}
```

### 6.4 PoD Quality Standards

| Requirement | Standard |
|-------------|----------|
| Photo resolution | Minimum 1280×720 |
| Photo clarity | Parcel label readable where applicable |
| Geolocation | Delivery coordinates within 200m of `deliveryLat/Lng` |
| Timestamp | Server-validated; no backdating |

### 6.5 PoD Failure Handling

| Failure | Resolution |
|---------|------------|
| Upload timeout | Auto-retry 3 times; cache locally until success |
| Geofence mismatch | Contact Fleet Operator for manual delivery confirmation |
| Recipient unavailable | Follow merchant delivery instructions; note in PoD |

---

## 7. Wallet & Instant Withdrawal Mechanics

### 7.1 Earnings Model

On each `ASSIGNED` order, partner receives **90% of order price** as a `CREDIT` transaction.

**Example:**
- Order price: IDR 50,000
- Partner credit: IDR 45,000
- Platform fee: IDR 5,000 (not visible in partner wallet)

### 7.2 Wallet Balance (Phase 2 UI)

**Endpoint:** `GET /v1/drivers/me/wallet`

```json
{
  "driverId": "uuid",
  "availableBalance": 1250000,
  "pendingBalance": 45000,
  "currency": "IDR",
  "lastCreditAt": "2026-07-11T10:30:00.000Z"
}
```

| Field | Meaning |
|-------|---------|
| `availableBalance` | Withdrawable now |
| `pendingBalance` | Assigned but not yet `DELIVERED` |

### 7.3 Instant Withdrawal Procedure (Phase 2)

1. Open **Wallet** tab in Flutter app.
2. Tap **Withdraw funds**.
3. Enter amount (minimum IDR 50,000; maximum = `availableBalance`).
4. Select registered bank account (one-time KYC required).
5. Confirm withdrawal → `POST /v1/drivers/me/withdrawals`.
6. Instant payout via payment rail (Phase 3) or T+0 batch (Phase 2 pilot).

### 7.4 Withdrawal Status Tracking

| Status | Meaning |
|--------|---------|
| `REQUESTED` | Submitted, awaiting processing |
| `PROCESSING` | Payment rail in progress |
| `COMPLETED` | Funds sent to bank |
| `FAILED` | Rejected — funds returned to wallet |

### 7.5 Withdrawal Limits

| Limit | Value |
|-------|-------|
| Minimum withdrawal | IDR 50,000 |
| Maximum per transaction | IDR 5,000,000 |
| Daily withdrawal cap | IDR 10,000,000 |
| KYC required above | IDR 1,000,000 cumulative |

---

## 8. Daily Operations Checklist

### 8.1 Shift Start

- [ ] Log in to Flutter app
- [ ] Set status to `AVAILABLE` (automatic on login in Phase 2)
- [ ] Verify GPS permissions granted
- [ ] Confirm vehicle type matches physical vehicle
- [ ] Check wallet balance and pending withdrawals

### 8.2 During Shift

- [ ] Accept assigned jobs within 60 seconds (Phase 2 SLA)
- [ ] Navigate to pickup; confirm parcel integrity
- [ ] Update status to `PICKED_UP` at pickup (Phase 2)
- [ ] Navigate to delivery; capture PoD
- [ ] Mark `DELIVERED`; verify return to `AVAILABLE`

### 8.3 Shift End

- [ ] Complete or hand off any active `ON_TRIP` order
- [ ] Set status to `OFFLINE`
- [ ] Review daily earnings in wallet
- [ ] Log out of app

---

## 9. API Reference (Driver-Scoped)

### 9.1 Read Assigned Order

**Endpoint:** `GET /v1/orders/:id`  
**Auth:** `Authorization: Bearer <token>` (DRIVER_PARTNER)

Returns full order with timeline when `assignedDriverId === user.driverId`. Returns `403` otherwise.

### 9.2 Response Fields (Key)

| Field | Description |
|-------|-------------|
| `status` | Current order state |
| `pickupAddress` / `deliveryAddress` | Human-readable locations |
| `pickupLat/Lng` | Pickup coordinates for navigation |
| `deliveryLat/Lng` | Delivery coordinates |
| `price` | Order price (partner earns 90%) |
| `matchDistanceKm` | Distance at time of matching |
| `timeline` | Full status audit trail |

---

## 10. Troubleshooting

| Symptom | Likely Cause | Action |
|---------|--------------|--------|
| No jobs appearing | Status `OFFLINE` or `ON_TRIP` | Set `AVAILABLE` |
| Job disappears | Order `CANCELLED` by ops | Check with Fleet Operator |
| Login fails | Wrong role selected | Ensure role `DRIVER_PARTNER` in login |
| `403` on order view | Order assigned to different driver | Only view own assignments |
| Wallet balance mismatch | Pending vs available confusion | Wait for `DELIVERED` confirmation |

---

## 11. Glossary

| Term | Definition |
|------|------------|
| **Haversine** | Great-circle distance used for 10 km matching radius |
| **ON_TRIP** | Driver status during active fulfillment |
| **PoD** | Proof-of-delivery photo/signature evidence |
| **Platform fee** | 10% retained by FleetFlow from order price |
| **matchDistanceKm** | Distance from driver to pickup at assignment time |

---

**Document owner:** Mobile Guild · **Next review:** 2026-08-11
