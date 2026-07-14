# FleetFlow Notifications (Web + Flutter)

**Document ID:** FF-NOTIF-001  
**Audience:** Backend, Flutter, Web, QA, demo recording  
**Last updated:** 2026-07-14

Realtime order-lifecycle alerts for **drivers (Flutter)** and **portal users (web)**. Uses **Postgres** (inbox), **BullMQ** (async delivery), and **Redis pub/sub** (SSE push).

## Lifecycle events

| Event | Type | Recipients |
|-------|------|------------|
| Matcher assigns driver | `ORDER_ASSIGNED` | Driver + merchant users + ops roles |
| Pickup confirmed | `ORDER_PICKED_UP` | Driver + merchant + ops |
| Delivery confirmed | `ORDER_DELIVERED` | Driver + merchant + ops |
| Cancelled (no match / balance) | `ORDER_CANCELLED` | Merchant + ops |

Ops roles: `SUPERADMIN`, `REGIONAL_MANAGER`, `HEAD_OF_WAREHOUSE`, `FLEET_OPERATOR`.

## Web UX

- Header **bell** with unread badge (poll every 4s)
- In-app **toast** bottom-right when assign / pickup / deliver / cancel arrives (same feel as Flutter SnackBar)
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

## Manual test

1. Rebuild API; re-login merchant / ops / driver
2. Create BIKE order → bell shows **Driver assigned**
3. Confirm pickup → **Parcel picked up**
4. Confirm delivery → **Delivery completed**

## Related

- [DEMO_E2E.md](./DEMO_E2E.md)
- `fleetflow-api/API_SPEC.md`
