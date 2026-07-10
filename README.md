# FleetFlow — System Architecture Documentation

**Product:** FleetFlow (On-Demand Logistics & Courier Dispatch System)  
**Classification:** Enterprise Multi-Repo Ecosystem  
**Audience:** Principal Engineers, Platform Architects, Mobile & Backend Teams  
**Last Updated:** 2026-07-10

---

## 1. Executive Overview

FleetFlow is a highly scalable, on-demand logistics and courier dispatch platform. The ecosystem is organized as a **pnpm multi-package workspace** with clear ownership boundaries:

| Package | Role | Runtime |
|---------|------|---------|
| `fleetflow-shared` | Shared contracts, Zod schemas, DTOs, domain enums | TypeScript library |
| `fleetflow-api` | Core NestJS API gateway & domain services | Node.js 20+ |
| `fleetflow-web` | Operations / customer portal (Next.js) | React / Next.js |
| `fleetflow-app` | Driver & courier mobile client | Flutter / Dart |
| `fleetflow-infra` | Local HA replication & service topology | Docker Compose |
| `fleetflow-docs` | Architecture, contracts, topology (this package) | Markdown |

Cross-cutting principle: **contracts live in `fleetflow-shared`**. TypeScript consumers import Zod schemas and inferred types directly. Flutter consumes the same JSON wire format via generated Dart models and explicit mapping tables documented below.

---

## 2. Architectural Style

FleetFlow follows **Clean Architecture** with hexagonal ports/adapters:

```
┌─────────────────────────────────────────────────────────────┐
│                     Presentation Layer                       │
│   Next.js (Web)  │  Flutter (App)  │  NestJS Controllers    │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTP / WebSocket / JSON
┌───────────────────────────▼─────────────────────────────────┐
│                   Application / Use Cases                    │
│         Commands · Queries · Orchestration · Policies        │
└───────────────────────────┬─────────────────────────────────┘
                            │ Domain Ports
┌───────────────────────────▼─────────────────────────────────┐
│                      Domain Layer                            │
│     Entities · Value Objects · Domain Events · Invariants    │
└───────────────────────────┬─────────────────────────────────┘
                            │ Adapters
┌───────────────────────────▼─────────────────────────────────┐
│                   Infrastructure Layer                       │
│  Prisma/PostgreSQL · Redis · Bull Queues · Stripe · Maps     │
└─────────────────────────────────────────────────────────────┘
```

**DRY rules:**

- Validation schemas are defined once in `@fleetflow/shared` (Zod).
- NestJS pipes validate request bodies against those schemas.
- Flutter `dio` clients deserialize the same JSON keys; field names never diverge without a versioned contract change.
- Enums and status machines are single-sourced in shared contracts and mirrored in Dart.

---

## 3. Microservices Topology

### 3.1 Logical Services

| Service | Responsibility | Tech | Scaling Unit |
|---------|----------------|------|--------------|
| **API Gateway / Core API** (`fleetflow-api`) | Auth, orders, tracking, payments orchestration, OpenAPI | NestJS | Horizontal (stateless) |
| **Matching Engine** (`python-matching-service`) | Driver allocation, geo-scoring, capacity constraints | Python | Horizontal + sticky geo shards |
| **PostgreSQL** | System of record (orders, users, drivers, payments) | PostgreSQL 16 | Primary + replicas |
| **Redis** | Cache, sessions, Bull job queues, pub/sub for live tracking | Redis 7 | Cluster / sentinel in prod |
| **Web Portal** (`fleetflow-web`) | Ops dashboards, customer booking UI | Next.js | Edge + SSR nodes |
| **Mobile App** (`fleetflow-app`) | Driver login, job accept/reject, navigation | Flutter | Client |

### 3.2 Local High-Availability Topology (`fleetflow-infra`)

```
                    ┌──────────────────┐
                    │   fleetflow-web  │
                    │   (Next.js)      │
                    └────────┬─────────┘
                             │ HTTPS / JSON
         ┌───────────────────▼───────────────────┐
         │           fleetflow-api               │
         │     NestJS · Swagger · Bull           │
         └─────┬───────────┬───────────┬─────────┘
               │           │           │
               ▼           ▼           ▼
        ┌──────────┐ ┌─────────┐ ┌─────────────────────┐
        │ postgres │ │  redis  │ │ python-matching-     │
        │ (primary)│ │ cache + │ │ service              │
        │          │ │ queues  │ │ (allocation engine)  │
        └──────────┘ └─────────┘ └─────────────────────┘
```

**Data flow for a delivery job:**

1. Customer creates order via Web → `POST /v1/orders`.
2. API persists order (PostgreSQL), enqueues `MATCH_DRIVER` on Redis/Bull.
3. Matching service consumes job, scores nearby available drivers, returns assignment candidate.
4. API updates order status, notifies driver via WebSocket / push.
5. Driver accepts in Flutter app → `POST /v1/jobs/{id}/accept`.
6. Tracking events stream through Redis pub/sub; Web and App subscribe for live ETA.

---

## 4. API Contracts

### 4.1 Conventions

| Concern | Standard |
|---------|----------|
| Base path | `/v1` |
| Content type | `application/json; charset=utf-8` |
| Auth | Bearer JWT (`Authorization: Bearer <token>`) |
| Idempotency | `Idempotency-Key` header on mutating payment/order creates |
| Errors | RFC 7807-inspired problem+json envelope |
| Pagination | Cursor-based: `?cursor=<opaque>&limit=20` |
| Timestamps | ISO-8601 UTC (`2026-07-10T03:00:00.000Z`) |
| IDs | UUID v4 strings |
| Money | Integer **minor units** (cents) + ISO 4217 `currency` |

### 4.2 Error Envelope

```json
{
  "type": "https://fleetflow.dev/errors/validation",
  "title": "Validation Failed",
  "status": 400,
  "detail": "pickup.latitude must be between -90 and 90",
  "instance": "/v1/orders",
  "traceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "errors": [
    {
      "path": "pickup.latitude",
      "code": "out_of_range",
      "message": "must be between -90 and 90"
    }
  ]
}
```

### 4.3 Core Resources

#### Auth

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/auth/login` | Email/password or phone OTP exchange → JWT pair |
| `POST` | `/v1/auth/refresh` | Rotate refresh token |
| `POST` | `/v1/auth/logout` | Revoke refresh token |

**Login request**

```json
{
  "email": "driver@fleetflow.dev",
  "password": "Str0ng!Pass",
  "role": "DRIVER"
}
```

**Login response**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "dGhpc2lzYXJlZnJlc2h0b2tlbg...",
  "expiresIn": 900,
  "tokenType": "Bearer",
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "driver@fleetflow.dev",
    "role": "DRIVER",
    "displayName": "Alex Rivera"
  }
}
```

#### Orders

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/orders` | Create delivery order |
| `GET` | `/v1/orders/{orderId}` | Fetch order by id |
| `GET` | `/v1/orders` | List orders (role-scoped) |
| `PATCH` | `/v1/orders/{orderId}/cancel` | Cancel if pre-pickup |

**Create order request**

```json
{
  "customerId": "550e8400-e29b-41d4-a716-446655440001",
  "pickup": {
    "latitude": -6.200000,
    "longitude": 106.816666,
    "addressLine": "Jl. Thamrin No. 1, Jakarta",
    "contactName": "Sender Co",
    "contactPhone": "+6281234567890"
  },
  "dropoff": {
    "latitude": -6.175110,
    "longitude": 106.865036,
    "addressLine": "Jl. Sudirman No. 52, Jakarta",
    "contactName": "Receiver Ltd",
    "contactPhone": "+6281298765432"
  },
  "parcel": {
    "weightKg": 2.5,
    "lengthCm": 30,
    "widthCm": 20,
    "heightCm": 15,
    "description": "Documents and samples"
  },
  "serviceLevel": "EXPRESS",
  "currency": "IDR",
  "quotedPriceMinor": 4500000
}
```

**Order status machine**

```
DRAFT → PENDING_MATCH → ASSIGNED → ACCEPTED → PICKED_UP → IN_TRANSIT → DELIVERED
                              ↘ REJECTED → PENDING_MATCH
         ANY (pre-pickup) → CANCELLED
         ANY → FAILED
```

#### Jobs (Driver)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/drivers/me/jobs/available` | Nearby offerable jobs |
| `POST` | `/v1/jobs/{jobId}/accept` | Driver accepts assignment |
| `POST` | `/v1/jobs/{jobId}/reject` | Driver rejects assignment |
| `POST` | `/v1/jobs/{jobId}/status` | Advance job lifecycle |

**Accept job request**

```json
{
  "driverId": "550e8400-e29b-41d4-a716-446655440010",
  "acceptedAt": "2026-07-10T03:15:00.000Z",
  "currentLocation": {
    "latitude": -6.201000,
    "longitude": 106.817000
  }
}
```

**Accept job response**

```json
{
  "jobId": "550e8400-e29b-41d4-a716-446655440020",
  "orderId": "550e8400-e29b-41d4-a716-446655440030",
  "status": "ACCEPTED",
  "pickup": {
    "latitude": -6.200000,
    "longitude": 106.816666,
    "addressLine": "Jl. Thamrin No. 1, Jakarta"
  },
  "dropoff": {
    "latitude": -6.175110,
    "longitude": 106.865036,
    "addressLine": "Jl. Sudirman No. 52, Jakarta"
  },
  "etaMinutes": 18
}
```

#### Payments (Stripe)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/payments/intents` | Create Stripe PaymentIntent |
| `POST` | `/v1/webhooks/stripe` | Stripe webhook ingress |

### 4.4 OpenAPI

`fleetflow-api` exposes Swagger UI at `/docs` and OpenAPI JSON at `/docs-json`. Contract changes must:

1. Update Zod schemas in `@fleetflow/shared`.
2. Regenerate NestJS DTO decorators / Swagger metadata.
3. Update Flutter Dart models and mapping tables in this document.
4. Bump API minor version for additive changes; major for breaking changes.

---

## 5. Cross-Language JSON → Dart Translation

TypeScript (Zod) is the **source of truth**. Flutter mirrors the wire JSON with `freezed` / hand-written immutable models. Rules:

1. **camelCase** JSON keys on the wire for both platforms.
2. Dart types map as follows:

| JSON / Zod | TypeScript | Dart |
|------------|------------|------|
| `string` (uuid) | `string` | `String` |
| `string` (datetime) | `string` (ISO) | `DateTime` (parsed UTC) |
| `number` (int money) | `number` | `int` |
| `number` (float geo) | `number` | `double` |
| `boolean` | `boolean` | `bool` |
| `enum` | string union | `enum` with same names |
| `object` | interface | class / Freezed |
| `array` | `T[]` | `List<T>` |
| `null` optional | `T \| null` | `T?` |

3. **Enums** must use identical uppercase identifiers:

```text
OrderStatus: DRAFT | PENDING_MATCH | ASSIGNED | ACCEPTED | PICKED_UP | IN_TRANSIT | DELIVERED | CANCELLED | FAILED | REJECTED
UserRole: CUSTOMER | DRIVER | DISPATCHER | ADMIN
ServiceLevel: STANDARD | EXPRESS | SAME_DAY
```

4. **Example Dart mapping** for login response:

```dart
class AuthTokens {
  const AuthTokens({
    required this.accessToken,
    required this.refreshToken,
    required this.expiresIn,
    required this.tokenType,
    required this.user,
  });

  final String accessToken;
  final String refreshToken;
  final int expiresIn;
  final String tokenType;
  final UserSummary user;

  factory AuthTokens.fromJson(Map<String, dynamic> json) {
    return AuthTokens(
      accessToken: json['accessToken'] as String,
      refreshToken: json['refreshToken'] as String,
      expiresIn: json['expiresIn'] as int,
      tokenType: json['tokenType'] as String,
      user: UserSummary.fromJson(json['user'] as Map<String, dynamic>),
    );
  }

  Map<String, dynamic> toJson() => {
        'accessToken': accessToken,
        'refreshToken': refreshToken,
        'expiresIn': expiresIn,
        'tokenType': tokenType,
        'user': user.toJson(),
      };
}
```

5. **Geo points** always serialize as:

```json
{ "latitude": -6.2, "longitude": 106.816666 }
```

Never use nested `lat`/`lng` aliases in production payloads.

6. **Failure translation:** Flutter `Either<Failure, T>` (dartz) maps HTTP problem+json `status` + `detail` into domain `Failure` subtypes (`ValidationFailure`, `UnauthorizedFailure`, `NetworkFailure`, `ServerFailure`).

---

## 6. Shared Package Contract Surface (`@fleetflow/shared`)

Consumers:

- NestJS: import schemas for `ZodValidationPipe`, domain enums, event payload types.
- Next.js: import types for React Query hooks and Formik initial values.
- Flutter: treat OpenAPI + this README as the Dart generation input; do not invent parallel field names.

Recommended module layout inside `fleetflow-shared`:

```
src/
  index.ts
  auth/
  orders/
  jobs/
  payments/
  geo/
  common/   # pagination, problem+json, money
```

---

## 7. Security & Compliance Baseline

- TLS everywhere outside the Docker internal network.
- JWT access tokens short-lived (≤ 15 minutes); refresh tokens rotated and stored hashed.
- Stripe secrets and DB credentials only via environment / secret manager — never committed.
- PII fields (phone, address) encrypted at rest where required by jurisdiction.
- Audit log for order status transitions and payment events.
- Rate limiting on auth and matching endpoints.

---

## 8. Observability

| Signal | Tooling (target) |
|--------|------------------|
| Logs | Structured JSON, `traceId` correlation |
| Metrics | Prometheus / OpenTelemetry |
| Traces | OTLP → collector |
| Uptime | Health: `/health/live`, `/health/ready` |

---

## 9. Local Development Bootstrap

```bash
# From repository root
pnpm install
pnpm --filter @fleetflow/shared build
docker compose -f fleetflow-infra/docker-compose.yml up -d
pnpm --filter @fleetflow/api run start:dev
pnpm --filter @fleetflow/web run dev
```

Flutter:

```bash
cd fleetflow-app
flutter pub get
flutter test
flutter test integration_test/app_test.dart
```

---

## 10. Repository Ownership Matrix

| Path | Owner | Change Control |
|------|-------|----------------|
| `fleetflow-shared` | Platform / Contracts Guild | Breaking changes require RFC |
| `fleetflow-api` | Backend Guild | OpenAPI review on PR |
| `fleetflow-web` | Web Guild | Design system + a11y checks |
| `fleetflow-app` | Mobile Guild | Contract sync checklist |
| `fleetflow-infra` | Platform SRE | Compose parity with staging |
| `fleetflow-docs` | Architecture Board | Versioned with releases |

---

## 11. Non-Goals (Current Scaffold)

- Multi-region active-active failover (documented for later phases).
- Full gRPC mesh between matching and API (HTTP/JSON stub first).
- Native iOS/Android module plugins beyond Flutter packages listed in `pubspec.yaml`.

---

**FleetFlow** — contracts first, scale second, chaos never.
