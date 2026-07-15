# FleetFlow Realtime (SSE + Redis)

**Document ID:** FF-RT-001  
**Audience:** Backend, Web, Flutter, QA, demo recording  
**Last updated:** 2026-07-15

How FleetFlow pushes **notifications** and **order updates** to web and Flutter without polling. This doc records what we shipped, what broke, how we fixed NestJS SSE, and how to verify it (including load-style checks).

## Related docs

- [DRIVER_NOTIFICATIONS.md](./DRIVER_NOTIFICATIONS.md) — inbox types, audience rules, bell/toast UX
- [PROOF_OF_DELIVERY.md](./PROOF_OF_DELIVERY.md) — proof photos + `order.updated` reason `photos`
- [DEMO_E2E.md](./DEMO_E2E.md) — full web ↔ API ↔ Flutter demo script

---

## Architecture

```text
User action (pickup, photo upload, assign, …)
        │
        ▼
Postgres (notifications row + order state)
        │
        ├── BullMQ NOTIFICATION_QUEUE → worker persists + publishToUser()
        │
        └── broadcastOrderUpdated() → publishToUser() per stakeholder
                    │
                    ▼
            Redis PUBLISH fleetflow:notifications:user:{userId}
                    │
                    ▼
            GET /v1/notifications/stream (SSE, one Redis SUB per connection)
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
   fleetflow-web            fleetflow-app
   RealtimeSync.tsx         NotificationSseClient (package:http)
```

| Layer | Role |
|-------|------|
| **Postgres** | Durable notification inbox (`notifications` table) |
| **BullMQ** | Async fan-out after write; retries on failure |
| **Redis pub/sub** | Ephemeral push to open SSE connections |
| **SSE** | Long-lived HTTP stream per logged-in client |

SSE is **not** a queue. Each browser/app tab holds one connection. Redis delivers only to subscribers that are connected at publish time (same as any pub/sub). Inbox rows remain queryable via `GET /v1/notifications` if the client was offline.

---

## Event types on the wire

| Payload | When | Client action |
|---------|------|---------------|
| `{ kind: "connected", userId }` | SSE opens | Ignore (handshake) |
| `{ kind: "heartbeat", at }` | Every 25s | Ignore (keep-alive) |
| Notification job payload | Bell events (`ORDER_ASSIGNED`, `ORDER_PICKED_UP`, photo types, …) | Toast + refresh inbox |
| `{ kind: "order.updated", orderId, reason }` | Order state / photos / assign | Refetch order detail & lists |

`reason` values: `status` \| `photos` \| `assigned`.

Channel name: `fleetflow:notifications:user:{userId}` (see `notificationUserChannel()` in API).

---

## Booking reference in copy (`#089065`)

User-facing booking IDs in toasts and notification bodies use **last 6 hex chars** of the UUID (no hyphens), uppercased, with `#` prefix.

| Full UUID | Display |
|-----------|---------|
| `a47f8cf1-0775-4511-a1cd-4f37194bd1ee` | `#BD1EEE` |
| (example) | `#089065` |

Implementation:

- API: `formatOrderReference()` in `fleetflow-api/src/orders/order-display.util.ts` — also exposed as `referenceCode` on order API responses
- Web: same helper in `fleetflow-web/src/lib/orders/display.ts`

Example toast: `Booking #089065 marked as picked up.`

---

## What we changed (2026-07-15)

### API (`fleetflow-api`)

| Area | Change |
|------|--------|
| **SSE encoding** | **Critical fix:** pass plain objects to `MessageEvent.data`, never `JSON.stringify()` (Nest double-encodes strings). See below. |
| **Heartbeat** | 25s `{ kind: "heartbeat" }` on every stream |
| **Logging** | `SSE subscribed to fleetflow:notifications:user:{id}` on subscribe |
| **Order realtime** | `broadcastOrderUpdated()` with reasons `status`, `photos`, `assigned` |
| **Proof photos** | `POST /v1/orders/:id/photos`, local `uploads/` + Cloudinary fallback, min 1KB validation |
| **Photo notifications** | `DEPARTURE_PHOTO_UPLOADED`, `DELIVERY_PHOTO_UPLOADED` |
| **Trip notifications** | Pickup/deliver notify driver + merchant + ops; exclude only the **actor** |
| **CORS** | `/uploads` static files get CORS headers in `main.ts` (Flutter web image preview) |
| **Docker** | `fleetflow_uploads_data` volume; set `API_PUBLIC_URL` for absolute photo URLs |

### Web (`fleetflow-web`)

| Area | Change |
|------|--------|
| **RealtimeSync** | SSE client; reconnect 3s after drop; waits for auth hydration |
| **Parser** | Tolerates legacy double-encoded SSE payloads |
| **Order panels** | Subscribe to `order.updated`; refetch proof gallery / tracker |
| **Fallback poll** | 8s poll on notification bell + order tracker while page open (safety net if SSE drops) |
| **Ops toasts** | Local toast on pickup/deliver + `emitOrderUpdated` |
| **Booking copy** | `formatOrderReference()` in toasts |

### Flutter (`fleetflow-app`)

| Area | Change |
|------|--------|
| **SSE transport** | `package:http` streaming instead of Dio (Dio stalled on web) |
| **Reconnect** | Backoff + teardown on token change |
| **Fallback poll** | 12s when SSE disconnected |
| **Proof photos** | Upload panel on trip detail; bytes upload on web |

### Not yet documented elsewhere

- DevTools showing **“failed to load response”** on `/stream` is **normal** for long-lived SSE (not a bug).
- Fallback polls are intentional until SSE is proven stable in production; can be tuned or removed later.
- List panels (`OperationsOrdersPanel`, `DriverAssignedOrdersPanel`) may still show truncated UUID prefix in tables — booking detail and notifications use `#XXXXXX`.

---

## NestJS SSE fix (why realtime was silent)

**Symptom:** Web bell and proof gallery never updated live; API logs showed **no** `SSE subscribed` lines; clients received escaped JSON strings they could not parse.

**Cause:** `@Sse()` / RxJS `MessageEvent` — Nest **JSON-stringifies** the `data` field again. Passing `data: JSON.stringify(payload)` produced:

```text
data: "{\"kind\":\"connected\",...}"
```

Clients expected a single JSON object, not a string containing JSON.

**Fix** (`notification-realtime.service.ts`):

```typescript
// Correct — pass objects only
subscriber.next({ data: payload } as MessageEvent);

// Wrong — do not do this
subscriber.next({ data: JSON.stringify(payload) } as MessageEvent);
```

Redis messages are `JSON.parse()`’d once, then pushed as objects. Web parser still accepts legacy double-encoded frames from older API images.

**After code change:** rebuild API container:

```bash
docker compose up -d --build fleetflow-api
```

---

## How to test SSE

### 1. Smoke test (curl)

Login (or use seed token), then:

```bash
curl.exe -N -H "Authorization: Bearer <access_token>" -H "Accept: text/event-stream" http://localhost:3000/v1/notifications/stream
```

**Pass:** first event looks like:

```text
data: {"kind":"connected","userId":"<uuid>"}
```

**Fail:** `data: "{\"kind\":\"connected\"...` (quoted string) → API still double-encoding; rebuild with fixed code.

Heartbeat every ~25s:

```text
data: {"kind":"heartbeat","at":"2026-07-15T11:00:00.000Z"}
```

### 2. API logs

With a portal tab open:

```text
SSE subscribed to fleetflow:notifications:user:<uuid>
Published realtime event to fleetflow:notifications:user:<uuid>
```

### 3. Manual E2E (two sessions)

| Step | Ops (web) | Driver (Flutter) |
|------|-----------|------------------|
| 1 | Login `fleet.operator@fleetflow.dev` | Login `driver.partner@fleetflow.dev` |
| 2 | Open booking detail | Open same trip |
| 3 | Upload departure photo | Gallery updates without refresh |
| 4 | Confirm pickup | Other side gets bell + `#XXXXXX` in body |
| 5 | Upload delivery photo | `order.updated` reason `photos` |
| 6 | Confirm delivery | Toast on non-actor session |

Password (demo): `FleetFlow!2026`

Re-login after deploy so JWT includes `notifications:read`.

### 4. Stress / volume (compare to BullMQ)

BullMQ stress = enqueue many jobs and watch queue depth. **SSE stress** = many publishes while multiple clients stay subscribed.

**A. Redis publish flood (pub/sub only)**

```bash
# Replace USER_ID; run while curl SSE is connected
for i in $(seq 1 100); do
  redis-cli PUBLISH "fleetflow:notifications:user:USER_ID" \
    "{\"kind\":\"order.updated\",\"orderId\":\"test\",\"reason\":\"status\"}"
done
```

Expect: curl prints 100 `order.updated` events; no API crash.

**B. Multiple SSE clients**

Open 5 browser tabs (or 5 curl processes) with the same user token. Trigger one pickup. All tabs should receive the same notification within seconds.

**C. Rapid business actions**

Script or manual burst: upload 10 photos in a row on one booking. Each upload should enqueue notification + `order.updated` (`photos`). Bell count and gallery should match (fallback poll may smooth over a missed frame).

**D. Disconnect resilience**

Kill API container briefly. Clients should reconnect (web 3s, Flutter backoff). Unread count recovers via inbox GET or fallback poll.

### 5. Automated QA

From monorepo root:

```bash
pnpm test:qa
```

Covers notification permissions and order flows; does not replace live SSE curl check.

---

## File map

| Package | File |
|---------|------|
| API | `src/notifications/notification-realtime.service.ts` |
| API | `src/notifications/notifications.service.ts` |
| API | `src/orders/orders.service.ts` (`broadcastOrderUpdated`) |
| API | `src/orders/order-display.util.ts` (`formatOrderReference`) |
| Web | `src/components/realtime/RealtimeSync.tsx` |
| Web | `src/lib/realtime/notification-stream.ts` |
| Web | `src/lib/realtime/bus.ts` |
| Flutter | `lib/src/core/network/notification_sse_client.dart` |
| Flutter | `lib/src/presentation/providers/notifications_provider.dart` |

---

## Troubleshooting

| Symptom | Check |
|---------|--------|
| No live updates | curl smoke test; API rebuilt? |
| No `SSE subscribed` logs | Client not connecting — auth, CORS, or wrong base URL |
| Images broken on Flutter web | CORS on `/uploads`; file size ≥ 1KB |
| Bell updates but gallery stale | `order.updated` listener on `OrderProofPhotosPanel` / tracker |
| Only actor sees change | Expected for excluded user; other sessions should notify |
