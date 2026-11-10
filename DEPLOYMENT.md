# eHub Ecosystem - Production Deployment Strategy

This document outlines the complete production deployment strategy for the **eHub SuperApp** monorepo (`ehub_ecosystem`). 

The ecosystem uses a **strangler-fig architecture**: a legacy Go monolith (`eHubGo`), an internal **gRPC microservices mesh** fronted by a **Go API Gateway (BFF)**, a **strict multi-role PostGIS database**, and **multi-platform frontends** (Expo mobile apps and SvelteKit web portals).

---

## Architecture Topology

```
                                  PUBLIC INTERNET
                                         │
        ┌────────────────────────────────┴────────────────────────────────┐
        │                                                                 │
  Mobile Apps (iOS/Android)                                          Web Portals
  - apps/ehub (Consumer)                                             - apps/web (Marketplace)
  - apps/dispatch (Driver)                                           - apps/merchant (Vendor)
        │                                                            - apps/admin (Zero Trust)
        │                                                                 │
        └────────────────────────────────┬────────────────────────────────┘
                                         │ HTTPS
                                         ▼
                     ┌───────────────────────────────────────┐
                     │         Cloudflare WAF / CDN          │
                     └───────────────────┬───────────────────┘
                                         │
                 ┌───────────────────────┴───────────────────────┐
                 │ HTTPS                                         │ HTTPS
                 ▼                                               ▼
     ┌───────────────────────┐                       ┌───────────────────────┐
     │   services/gateway    │                       │   services/eHubGo     │
     │  (Port 9010 -> :443)  │                       │   (Legacy Monolith    │
     │  api.ehub.com/api/v1  │                       │   ehubgo.onrender.com)│
     └───────────┬───────────┘                       └───────────┬───────────┘
                 │                                               │
═════════════════╪═══════════════════════════════════════════════╪═════════════════
                 │  VPC / PRIVATE NETWORK (No Public Ingress)    │
                 ▼                                               │
   ┌─────────────────────────────┐                               │
   │  Internal gRPC Mesh         │                               │
   │  - auth-service       (:9101)                               │
   │  - wallet-service     (:9102)                               │
   │  - dispatch-service   (:9103)                               │
   │  - fulfillment-service(:9104)                               │
   │  - marketplace-b2b    (:9105)                               │
   │  - marketplace-c2c    (:9106)                               │
    │  - estay-service      (:9107)                               │
    │  - merchant-service   (:9108)                               │
   └─────────────┬───────────────┘                               │
                 │                                               │
                 ├───────────────────────┬───────────────────────┤
                 ▼                       ▼                       ▼
     ┌───────────────────────┐  ┌─────────────────┐  ┌───────────────────────┐
     │     Managed Redis     │  │ Safaricom /     │  │   Neon / Managed      │
     │  - H3 Spatial Cache   │  │ Firebase APIs   │  │   PostgreSQL + PostGIS│
     │  - Driver Locations   │  └─────────────────┘  │   - Multiple Roles    │
     │    (45s TTL)          │                       │   - Schema Contracts  │
     └───────────────────────┘                       └───────────────────────┘
```

---

## Phase 1: Database & Persistence Layer Setup

The database architecture uses a **shared PostgreSQL instance with strict role-level ownership contracts** (`infra/db/contracts.sql`).

### 1. Provision PostgreSQL with PostGIS 3.4+
- Use **Neon Serverless Postgres** or **AWS RDS / GCP Cloud SQL for PostgreSQL 15+**.
- **Critical Requirement**: The PostGIS extension must be enabled (`schema.sql` uses `GEOGRAPHY(POINT, 4326)`).

### 2. Run Database Migrations & Deploy Gate
Run schema migrations in the exact order declared in `infra/db/push_to_neon.ps1`:
1. `services/eHubGo/schema.sql` (Core tables & RLS helper functions)
2. `services/eHubGo/migrations/001_ecommerce_filtering.sql`
3. `services/wallet-service/migrations/001_wallet.sql` (`wallet` schema)
4. `services/marketplace-c2c/migrations/001_c2c_listing_policies.sql`
5. `services/dispatch-service/migrations/002_ride_lifecycle.sql`
6. `services/dispatch-service/migrations/003_vehicle_tiers.sql` (Vehicle classes & seeded tiers)
7. `services/dispatch-service/migrations/004_delivery_lifecycle.sql`
8. `services/dispatch-service/migrations/005_delivery_fare_and_readiness.sql`
9. `services/dispatch-service/migrations/006_delivery_offers.sql`
10. `services/dispatch-service/migrations/007_delivery_proofs.sql`
11. `services/dispatch-service/migrations/008_service_ratings.sql`
12. `services/dispatch-service/migrations/009_ride_hardening.sql`
13. `services/estay-service/migrations/001_estay.sql` (`estay` schema + amenities)
14. `services/estay-service/migrations/002_host_merchant.sql` (nullable `estay.hosts.merchant_id`)
15. `services/marketplace-b2b/migrations/001_catalog_taxonomy_and_promotions.sql`
16. `services/merchant-service/migrations/001_merchant.sql` (merchant registry/capabilities/logs)
17. `services/merchant-service/migrations/002_capability_review.sql` (capability review metadata)
18. `services/merchant-service/migrations/003_document_review.sql` (document review metadata/indexes)
19. `infra/db/contracts.sql` (Creates least-privilege roles and grants)

```bash
# Automated deployment gate
./infra/db/push_to_neon.ps1 -NeonUrl "$DIRECT_DATABASE_URL" -LoadReferenceData
```

- **Connection Routing**:
  - Run DDL migrations against the **Direct Neon endpoint** (PgBouncer cannot run transactional DDL).
  - Point runtime services at the **Pooled endpoint** (`-pooler.region.neon.tech`) using their individual database roles.

### 3. Rotate Passwords for Production Roles
Never run `contracts.sql` passwords (`ehub-dev-<service>`) in production. Set strong credentials in your Secret Manager:
- `ehub_auth` (owns `users`, `user_roles`)
- `ehub_wallet` (owns `wallet.*`)
- `ehub_dispatch` (writes `drivers`, `taxi_trips`, `deliveries`, `delivery_*`, `service_ratings`)
- `ehub_fulfillment` (writes `orders.status`)
- `ehub_b2b` & `ehub_c2c` (catalog & P2P listings)
- `ehub_estay` (owns the `estay` schema)
- `ehub_merchant` (writes merchants, capabilities, verification documents and verification logs)

### 4. Provision Managed Redis 7+
- Deploy on **Upstash Redis**, **AWS ElastiCache**, or **Redis Cloud**.
- Needs persistent latency `< 5ms` from the `dispatch-service` because driver H3 indexing (`dispatch:h3:r9:*`), candidate searches and surge counters execute on every ride and delivery request.

---

## Phase 2: Containerization & Build Artifacts

### 1. Protobuf Compilation Gate
Before building Go or NestJS containers, run the Buf compiler in CI:
```bash
bun run generate_protos.ts
# Generates Go packages in packages/proto/
```

### 2. Multi-Stage Dockerfile for Go Microservices
The Go services reference internal modules (`packages/proto`, `packages/platform`, `packages/clients`) via `go.mod` `replace` directives.

*Example production Dockerfile (`services/dispatch-service/Dockerfile`):*
```dockerfile
FROM golang:1.26-alpine AS builder

RUN apk add --no-cache build-base git
WORKDIR /workspace

COPY packages/ packages/
COPY services/dispatch-service services/dispatch-service

WORKDIR /workspace/services/dispatch-service
RUN go mod download
RUN CGO_ENABLED=1 GOOS=linux go build -ldflags="-s -w" -o /bin/dispatch-service .

FROM alpine:3.20
RUN apk add --no-cache ca-certificates tzdata
WORKDIR /app
COPY --from=builder /bin/dispatch-service .

EXPOSE 9003 9103
USER nobody:nobody
ENTRYPOINT ["./dispatch-service"]
```

### 3. Multi-Stage Dockerfile for NestJS Microservices
*For `services/marketplace-b2b` and `marketplace-c2c`:*
```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY proto/ /proto/
COPY services/marketplace-b2b/package*.json ./
RUN npm ci
COPY services/marketplace-b2b/ ./
RUN npm run build && npm prune --production

FROM node:22-alpine
WORKDIR /app
COPY proto/ /proto/
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json

EXPOSE 9005 9105
USER node
CMD ["node", "dist/main.js"]
```

---

## Phase 3: Backend Orchestration & Private Service Mesh

Because `packages/clients` establishes **plaintext gRPC connections** (`insecure.NewCredentials()`), **backend services must never be exposed directly to the public internet**.

### Hosting Options

#### Recommended Option A: Container Orchestration (AWS ECS / EKS or GCP Cloud Run / GKE)
- Deploy services into a **Private VPC Subnet**.
- Use internal DNS (`auth-service.internal:9101`, etc.).
- Deploy `services/gateway` as the **only service in the Public Subnet** behind an Application Load Balancer (ALB).

#### Recommended Option B: PaaS with Private Networking (Fly.io or Render Private Services)
- Deploy backend services as **Private Services** (reachable only via private DNS).
- Deploy `gateway` as a **Public Web Service** mapping port `9010` to HTTPS.

### Gateway Production Settings
Configure `services/gateway`:
```env
PORT=9010
GO_ENV=production
JWT_SECRET=<32+ byte random hex string matching auth-service>
REQUEST_TIMEOUT_SECONDS=10
AUTH_GRPC_ADDR=auth-service:9101
WALLET_GRPC_ADDR=wallet-service:9102
DISPATCH_GRPC_ADDR=dispatch-service:9103
FULFILLMENT_GRPC_ADDR=fulfillment-service:9104
MARKETPLACE_B2B_GRPC_ADDR=marketplace-b2b:9105
MARKETPLACE_C2C_GRPC_ADDR=marketplace-c2c:9106
ESTAY_GRPC_ADDR=estay-service:9107
MERCHANT_GRPC_ADDR=merchant-service:9108
```

### Estay Production Settings

Estay calls the merchant registry to enforce the live host onboarding/approval
gate. Configure it with the internal merchant gRPC address:

```env
PORT=9007
GRPC_PORT=9107
GO_ENV=production
JWT_SECRET=<the same shared session key>
DATABASE_URL=<ehub_estay runtime URL>
ESTAY_SERVICE_FEE_PERCENT=5
MERCHANT_GRPC_ADDR=merchant-service:9108
MERCHANT_TIMEOUT_SECONDS=3
```

Outside development, Estay refuses to start without `MERCHANT_GRPC_ADDR`; the
publish path also fails closed if the registry is unavailable. The merchant
service remains the only owner of verification and capability state.

`/auth/sync` self-assigns only `user`, `customer`, `vendor`, `driver`, `host`
and `c2c_seller`. Grant `admin`, `executive_admin`, `staff` and `store_staff`
through an operational process; the verification queue, document decisions,
merchant status and capability review endpoints require `admin` or
`executive_admin`. `apps/admin` signs in through Firebase but does not request a
privileged role; it refuses to store a session unless the database-assigned
roles include an admin role.

---

## Phase 4: Web Applications Deployment (SvelteKit)

The web apps in `apps/` (`web`, `admin`, `merchant`) use SvelteKit 2 and Svelte 5.

| App | Target Audience | Recommended Platform | Security / Routing |
|---|---|---|---|
| `apps/web` | Public Marketplace | **Cloudflare Pages** or **Vercel** | CDN caching on static assets; SSR for SEO listings. |
| `apps/merchant` | Vendors / Sellers | **Cloudflare Pages** | Authenticated via Firebase + Gateway session. |
| `apps/admin` | Internal Operations | **Cloudflare Pages** | Firebase sign-in plus database-assigned admin role; **Cloudflare Zero Trust (Access)** is still mandatory for network-level protection. |

### Build & Deploy:
1. Switch `@sveltejs/adapter-auto` to `@sveltejs/adapter-cloudflare` or `@sveltejs/adapter-node`.
2. Set server-side runtime environment variables (they are not `PUBLIC_`, since the browser must not talk to the gateway directly):
   - `GATEWAY_URL=https://api.ehub.com/api/v1/gateway`
   - `FIREBASE_API_KEY=...` for Firebase email/password sign-in
   - leave `ALLOW_DEV_SESSION` unset in production
3. `apps/merchant` uses the `ehub_session` cookie; `apps/admin` uses its own
   `ehub_admin_session` cookie so both can run on localhost without colliding.
   Both apps are SvelteKit BFFs and call only the gateway.

---

## Phase 5: Mobile App Delivery Pipeline (Expo / EAS)

The mobile apps (`apps/ehub` and `apps/dispatch`) are built on Expo with New Architecture enabled.

### 1. Setup Expo Application Services (EAS)
Create `eas.json` in `apps/ehub` and `apps/dispatch`:
```json
{
  "cli": {
    "version": ">= 12.0.0"
  },
  "build": {
    "production": {
      "channel": "production",
      "env": {
        "EXPO_PUBLIC_GATEWAY_URL": "https://api.ehub.com/api/v1/gateway",
        "EXPO_PUBLIC_API_URL": "https://ehubgo.onrender.com/api/v1"
      },
      "android": {
        "buildType": "app-bundle"
      },
      "ios": {
        "simulator": false
      }
    }
  }
}
```

> **MapLibre build requirement:** both mobile apps use `@maplibre/maplibre-react-native` through an Expo config plugin. Expo Go cannot load the native module, so every profile must build a **custom development/production binary**. Add `EXPO_PUBLIC_MAPLIBRE_STYLE_URL` (vector style) or `EXPO_PUBLIC_OSM_TILE_URL` (raster fallback) to the build profile when you are not using the default development tile server.

### 2. Build and Release Execution
```bash
eas login
eas credentials
eas build --platform all --profile production
eas submit --platform all --profile production
```

### 3. Over-The-Air (OTA) Updates
```bash
eas update --channel production --message "Hotfix release"
```

---

## Phase 6: External Integrations & Webhook Ingress

1. **Safaricom Daraja M-Pesa**: Expose secure HTTPS webhooks for callbacks (`/api/v1/wallet/daraja/callback`, `/result`, `/timeout`) with token verification (`DARAJA_CALLBACK_TOKEN`).
2. **Firebase Authentication**: Mount service account JSON via secret manager (`FIREBASE_SERVICE_ACCOUNT_PATH`).
3. **Photon and OSRM**: deploy **Photon** (OSM-based places autocomplete/details) and **OSRM** (road distance, duration and polyline) as private services. Set `PLACES_PROVIDER=photon`, `PHOTON_URL`, `PHOTON_COUNTRY_CODE=KE`, `OSRM_URL` and `OSRM_TIMEOUT_SECONDS`. Google Places remains an optional provider only when `PLACES_PROVIDER=google` and `GOOGLE_PLACES_API_KEY` is set; the maps themselves use MapLibre and never require a Google Maps SDK key.
4. **Map tiles**: serve a self-hosted or commercial vector style (or raster tile template) for MapLibre and point the mobile builds at it with `EXPO_PUBLIC_MAPLIBRE_STYLE_URL` / `EXPO_PUBLIC_OSM_TILE_URL`. Keep the OpenStreetMap attribution visible; the public `tile.openstreetmap.org` endpoint is development-only.

---

## Phase 7: Deployment Sequence Checklist

```
Step 1: Database Setup
 ├─ Provision Neon PostGIS instance
 ├─ Execute ./infra/db/push_to_neon.ps1 -NeonUrl <DIRECT_URL> -LoadReferenceData
 ├─ Verify contracts: verify_contracts.sql
 └─ Rotate passwords for all 8 service roles in Postgres

Step 2: Cache & Queue Setup
 ├─ Provision Managed Redis 7+
 └─ Test connection latency and memory eviction policies

Step 3: Internal Microservices Deployment
 ├─ Build and push Docker images to registry (ECR / GCR / GHCR)
 ├─ Deploy auth-service, wallet-service, dispatch-service, fulfillment-service
 ├─ Deploy marketplace-b2b, marketplace-c2c, estay-service, merchant-service
 └─ Confirm all gRPC ports (:9101-:9108) pass internal health checks

Step 4: API Gateway (BFF) Deployment
 ├─ Deploy services/gateway with public ingress
 ├─ Configure TLS certificate (Let's Encrypt / Cloudflare)
 └─ Test end-to-end: curl https://api.ehub.com/api/v1/gateway/health

Step 5: Frontends
 ├─ Deploy SvelteKit web apps (web, admin, merchant) to Cloudflare Pages
 ├─ Build and submit mobile apps (eHub & Dispatch) via EAS
 └─ Verify production endpoints in mobile apps:
     • EXPO_PUBLIC_GATEWAY_URL -> https://api.ehub.com/api/v1/gateway
     • EXPO_PUBLIC_API_URL     -> https://ehubgo.onrender.com/api/v1
```
