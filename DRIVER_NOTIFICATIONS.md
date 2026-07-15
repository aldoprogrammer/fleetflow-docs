# FleetFlow Notifications (Web + Flutter)

**Document ID:** FF-NOTIF-001  
**Audience:** Backend, Flutter, Web, QA, demo recording  
**Last updated:** 2026-07-15

Realtime order-lifecycle alerts for **drivers (Flutter)** and **portal users (web)**. Uses **Postgres** (inbox), **BullMQ** (async delivery), and **Redis pub/sub** (SSE push).

For SSE architecture, the NestJS double-encoding fix, and stress testing → **[REALTIME_SSE.md](./REALTIME_SSE.md)**.

## Booking reference in messages

Toasts and notification bodies use **`#` + last 6 hex chars** of the order UUID (no hyphens), e.g. `#089065`. API: `formatOrderReference()`; web: `@/lib/orders/display`.

## Lifecycle events

| Event | Type | Recipients |
|-------|------|------------|
| Matcher assigns driver | `ORDER_ASSIGNED` | Driver + merchant users + ops roles |
| Pickup confirmed | `ORDER_PICKED_UP` | Driver + merchant + ops (exclude actor) |
| Delivery confirmed | `ORDER_DELIVERED` | Driver + merchant + ops (exclude actor) |
| Cancelled (no match / balance) | `ORDER_CANCELLED` | Merchant + ops |
| Before-pickup photo uploaded | `DEPARTURE_PHOTO_UPLOADED` | Other side (web ↔ driver) + ops |
| After-delivery photo uploaded | `DELIVERY_PHOTO_UPLOADED` | Other side (web ↔ driver) + ops |

Ops roles: `SUPERADMIN`, `REGIONAL_MANAGER`, `HEAD_OF_WAREHOUSE`, `FLEET_OPERATOR`.

## Web UX

- Header **bell** with unread badge (SSE push; 8s fallback poll)
- In-app **toast** bottom-right when assign / pickup / deliver / cancel / photo arrives
- Sidebar **Notifications** → `/notifications`
- Optional browser `Notification` when permission granted
- **Re-login** after deploy so JWT picks up `notifications:read`

## API

Requires `notifications:read`.

| Method | Path |
|--------|------|
| GET | `/v1/notifications` |
| GET | `/v1/notifications/unread-count` |
| GET | `/v1/notifications/stream` |
| PATCH | `/v1/notifications/:id/read` |
| POST | `/v1/notifications/read-all` |

## Audience rules (pickup / deliver / photos)

| Who acts | Who gets notified |
|----------|-------------------|
| Assigned driver (Flutter) | Merchant + ops on web; driver excluded |
| Ops / admin (web) | Driver + merchant + other ops; clicking user excluded (local toast instead) |
| Proof photo upload | Other side + ops; uploader excluded |

Assignment (`ORDER_ASSIGNED`) still notifies both sides (matcher acts, not a user button).

## Live status sync

Uses **SSE** (`GET /v1/notifications/stream`) over Redis pub/sub — not HTTP polling as primary path.

| Event | Payload | Client action |
|-------|---------|---------------|
| Notification | `notificationId`, `orderId`, … | Toast / banner + refresh inbox |
| Order updated | `kind: order.updated`, `orderId`, `reason` | Refetch order detail + lists |

`order.updated` is published on assign, matching, pickup, deliver, cancel, and proof-photo upload.

### Quick verification

1. Rebuild API (`docker compose up -d --build fleetflow-api`); re-login merchant / ops / driver
2. `curl -N` on `/v1/notifications/stream` — see [REALTIME_SSE.md](./REALTIME_SSE.md)
3. Create BIKE order → bell shows **Driver assigned**
4. Upload proof photo → other session gallery updates
5. Confirm pickup → **Parcel picked up** with `#XXXXXX` in body

## Related

- [REALTIME_SSE.md](./REALTIME_SSE.md)
- [PROOF_OF_DELIVERY.md](./PROOF_OF_DELIVERY.md)
- [DEMO_E2E.md](./DEMO_E2E.md)
- `fleetflow-api/API_SPEC.md`
