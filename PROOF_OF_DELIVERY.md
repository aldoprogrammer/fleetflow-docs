# Proof of Delivery / Departure SLA

Mandatory photo proofs for the FleetFlow driver app. Operations may override with a recorded reason.

## Web portal

Ops and assigned drivers can upload **Before pickup** / **After delivery** photos on the booking detail page. Driver app uploads sync to the same panel via SSE (`order.updated` reason `photos`) and create bell notifications (`DEPARTURE_PHOTO_UPLOADED` / `DELIVERY_PHOTO_UPLOADED`). Notification copy uses booking ref `#089065` style — see [REALTIME_SSE.md](./REALTIME_SSE.md#booking-reference-in-copy-089065).

---

## Driver SLA

| Action | Requirement |
|--------|-------------|
| Start Journey (`POST /orders/:id/pickup`) | Minimum 1 departure photo (`DEPARTURE`) |
| Complete Booking (`POST /orders/:id/deliver`) | Minimum 1 delivery photo (`DELIVERY`) |

Failure to provide required photos blocks the action.

## Operations SLA

| Action | Requirement |
|--------|-------------|
| Complete / advance booking | Photos optional |
| Manual override | Allowed with `overrideReason` |
| Incident resolution | Allowed; written to audit log |

## Storage

```text
Cloudinary
    └── fleetflow/
        ├── departures/
        │   └── booking-{id}/
        └── deliveries/
            └── booking-{id}/
```

Database tables: `order_photos`, `order_audit_events`.

## Audit events

| Action | When |
|--------|------|
| `DEPARTURE_PHOTO_UPLOADED` | Driver/ops uploads departure proof |
| `DELIVERY_PHOTO_UPLOADED` | Driver/ops uploads delivery proof |
| `JOURNEY_STARTED` | Pickup confirmed |
| `BOOKING_COMPLETED` | Delivery confirmed with photos |
| `MANUAL_COMPLETION_BY_OPS` | Ops advances without / with override |
| `COMPLETION_WITHOUT_PROOF_PHOTOS` | Ops completes without delivery photos |

Each event stores `user_id`, `role`, `booking_id` (`orderId`), `timestamp`, `photoUrls`, `overrideReason`.
