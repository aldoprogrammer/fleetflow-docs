# FleetFlow Architecture

High-level system design for portfolio demos and technical interviews.

## System diagram

```mermaid
flowchart TB
  subgraph clients [Clients]
    Web["fleetflow-web<br/>Next.js :3001"]
    App["fleetflow-app<br/>Flutter driver"]
    B2B["Merchant B2B<br/>x-api-key"]
  end

  subgraph api [fleetflow-api :3000]
    Auth["Auth + JWT RBAC"]
    Orders["Orders + Pricing"]
    Fleet["Fleet / Drivers"]
    Match["BullMQ Worker<br/>Geo Matching"]
    Ledger["Ledger / Merchants"]
  end

  subgraph data [Data & Infra]
    PG[(PostgreSQL<br/>Prisma)]
    Redis[(Redis<br/>BullMQ)]
  end

  subgraph shared [fleetflow-shared]
    RBAC["Roles + Permissions"]
  end

  Web --> Auth
  Web --> Orders
  Web --> Fleet
  App --> Auth
  B2B --> Orders
  Auth --> PG
  Orders --> PG
  Orders --> Redis
  Redis --> Match
  Match --> PG
  Fleet --> PG
  Ledger --> PG
  api --> shared
  Web --> shared
```

## Order dispatch flow

```mermaid
sequenceDiagram
  participant U as Portal User
  participant W as fleetflow-web
  participant A as fleetflow-api
  participant Q as BullMQ
  participant M as Matching Worker
  participant DB as PostgreSQL

  U->>W: Create order + map coords
  W->>A: POST /orders/estimate
  A-->>W: price + distanceKm
  U->>W: Submit order
  W->>A: POST /orders
  A->>DB: DRAFT → PENDING
  A->>Q: dispatch-order job
  Q->>M: process job
  M->>DB: MATCHING → ASSIGNED or CANCELLED
  W->>A: GET /orders/:id poll
  A-->>W: timeline + driver
```

## Monorepo layout

| Package | Role |
|---------|------|
| `fleetflow-api` | NestJS REST, BullMQ matching, Prisma |
| `fleetflow-web` | Next.js portal, RBAC middleware, Playwright |
| `fleetflow-shared` | RBAC contracts shared by API + web |
| `fleetflow-infra` | Docker Compose (Postgres, Redis) |
| `fleetflow-app` | Flutter driver mobile (optional demo) |
| `fleetflow-docs` | QA guide, planning |

## RBAC (6 roles)

| Role | Core access |
|------|-------------|
| Super Admin | All modules |
| Regional Manager | Orders, fleet, merchants, ledger |
| Head of Warehouse | Orders, fleet, ledger |
| Fleet Operator | Orders, fleet, drivers |
| Merchant Admin | Create + track own orders |
| Driver Partner | Assigned orders only |

## Portfolio video script (~4 min)

1. **Architecture** — show this diagram (30s)
2. **Login** — Super Admin + Merchant Admin demo accounts (30s)
3. **RBAC** — merchant blocked on `/fleet` (20s)
4. **Create order** — map picker + price estimate → submit (60s)
5. **Tracker** — poll until `ASSIGNED`, show driver (45s)
6. **Ops** — Fleet control driver roster, merchants wallet (45s)
7. **QA** — mention Playwright + live API tests (20s)

## Stack

- **API:** NestJS, Prisma, PostgreSQL, BullMQ, Redis, JWT
- **Web:** Next.js 15, React 19, Zustand, Formik, Leaflet
- **QA:** Jest, Playwright, live-stack e2e
