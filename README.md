# eHub Ecosystem

A monorepo for the **eHub SuperApp** — an urban commerce and mobility platform combining ride-hailing, food/grocery delivery, on-demand services, a B2C/B2B marketplace, short-stay accommodation (Estay), and a unified wallet.

The ecosystem is split into thin **frontends** (`apps/`), independent **backend services** (`services/`), shared **gRPC contracts** (`proto/`), and local **infrastructure** (`infra/`).

For the full system design — service ownership, request/auth flows, Estay lifecycle, H3/geo, MapLibre, vendor onboarding, and the rationale for each language and datastore — read [`Architecture.md`](Architecture.md).

> © 2026 Infinnity Developers. "The city, in your pocket."

---

## Repository Layout

```
ehub_ecosystem/
├── apps/          # Frontends (mobile + web)
├── services/      # Backend services (Go + Node/NestJS)
├── packages/      # Shared Go modules (proto = gRPC bindings, platform = auth, clients = gRPC clients)
├── proto/         # gRPC/protobuf service contracts
├── infra/         # docker-compose for local Postgres + Redis + services
├── generate_protos.ts  # buf codegen helper (→ packages/proto)
└── index.ts       # Bun entry stub
```

Workspaces are declared in the root `package.json` (`apps/*`, `services/*`, `packages/*`).

---

## Architecture at a Glance

> **Migration in progress.** This repo is mid-migration from a centralized backend (**eHubGo**) to independent, modular services so that a failure in one domain does not take down the whole system. eHubGo is the **legacy monolith being decomposed** (strangler-fig style), and the extracted services listed below are the **target architecture**.

- **eHubGo** is the original core backend monolith ("the high-performance engine") that still powers most verticals and is the primary API consumed by the mobile apps.
- Several focused services have been (partially) extracted from it: **auth**, **dispatch**, **wallet**, **fulfillment**, the **marketplace** (B2B/C2C) services, **Estay** (accommodation), and the **merchant** registry (vendor verification and capabilities).
- Mobile clients authenticate with **Firebase**, then call an `/auth/sync` endpoint to obtain a **session JWT** used against the backend services.
- Services share a **Postgres** instance and **Redis** (for rate limiting, location/geo indexes, accept reservations, surge demand and realtime Pub/Sub), but each service connects with its own least-privilege database role — see [Database ownership contracts](#database-ownership-contracts). `infra/docker-compose.yml` runs the whole local stack.

---

## Apps (Frontends)

| App | Path | Stack | Purpose | State |
|---|---|---|---|---|
| **eHub** | `apps/ehub` | Expo SDK 58 / React Native 0.88.1, expo-router, React Query, Zustand, Firebase, MapLibre React Native, i18n (6 langs) | Consumer super-app: eTaxi, food/grocery, 20+ mini-services, in-house catalog with deep categories and scheduled flash sales, cart/checkout, wallet, delivery with H3 + OSRM ETA, destination-first Photon search, surge-aware ride quotes, destination-first eDelivery courier quotes, Estay discovery/detail/booking/host screens, "stays near me" and MapLibre route/stay maps | Large, largely functional |
| **Dispatch** | `apps/dispatch` | Expo SDK 58 / React Native 0.88.1, expo-router, Firebase, MapLibre React Native | Driver/courier console: go online/offline, receive pushed and polled ride & delivery offers, drive both lifecycles (accept → start/complete a ride; accept → pickup photo → wait for merchant release → dropoff photo for a delivery), review finished records, and follow the pickup/dropoff route over OpenStreetMap with MapLibre | Functional (~2 screens) |
| **Web** | `apps/web` | SvelteKit 2, Svelte 5, Tailwind 4, Paraglide i18n | Intended public web portal / marketplace | Scaffold (default SvelteKit template) |
| **Admin** | `apps/admin` | SvelteKit 2, Svelte 5, Tailwind 4 | Internal verification console: admin Firebase sign-in with role checks, merchant review queue, document decisions, merchant status history, and mini-service capability approval | **Verification console functional** |
| **Merchant** | `apps/merchant` | SvelteKit 2, Svelte 5, Tailwind 4, Storybook, Playwright | Estay host workspace: host profile, vendor-store link, verification document links, approval state, store staff invitations/roles, multiple listings, photos, amenities, pricing, calendar and booking requests. Also the future home of every other miniservice's merchant management | **Host workspace functional** (Estay onboarding + vendor approval + staff RBAC) |

The mobile apps reach the backend through typed API clients in `apps/ehub/packages/services/` and `apps/dispatch/src/services/driver-api.ts`. Extracted-service calls (auth, wallet, dispatch, marketplace, estay) go through the **gateway** via `EXPO_PUBLIC_GATEWAY_URL`; the still-unmigrated verticals continue to hit `eHubGo` via `EXPO_PUBLIC_API_URL`.

### Mobile maps (MapLibre + OpenStreetMap)

Both mobile apps use `@maplibre/maplibre-react-native` and the OpenStreetMap/OSM data stack. There is no Google Maps SDK, Google API key, or `react-native-maps` dependency in either app.

- `apps/ehub/packages/components/TaxiMap.tsx` draws ride and delivery maps with declarative `<GeoJSONSource>` + `<Layer>` route lines, `<Marker>` pickup/drop-off/driver pins, and a `<Camera>` that fits the route.
- `apps/ehub/packages/components/estay/StayMap.tsx` renders a stay location or a set of listing pins for discovery.
- `apps/dispatch/src/components/route-map.tsx` renders the driver's dashed approach line and active trip line in the same way; it is non-interactive so the surrounding console still scrolls.
- Web builds resolve `TaxiMap.web.tsx` / `StayMap.web.tsx` fallbacks instead of trying to load the native renderer.

1. The consumer taxi screen opens on a destination-first “Where to?” search. Photon autocomplete returns a `place_id`; `GET /dispatch/place/:id` resolves it to coordinates. The vehicle tiers stay hidden until the rider has chosen a destination.
2. As soon as the rider selects a destination, the consumer app calls `GET /dispatch/route?from_lat=…&from_lng=…&to_lat=…&to_lng=…`. Dispatch asks OSRM for `overview=full&geometries=polyline` and returns `distance_meters`, `duration_seconds` and the encoded polyline (precision 5). The same endpoint is used by the driver console, so both apps draw the identical road path.
3. The app decodes the polyline into `{latitude, longitude}` points and converts them to GeoJSON `[longitude, latitude]` coordinates. MapLibre draws the route as a `LineString` layer; the consumer app uses `@mapbox/polyline`, and the driver app has a small local decoder in `src/utils/polyline.ts`.
4. The consumer app fits the camera to the pickup → destination route with `Camera.fitBounds`. The driver console draws a dashed driver → pickup approach plus the pickup → dropoff trip line, and refreshes the approach at most every 30 seconds while the driver is moving.
5. Once the route is known, the consumer app calls `GET /dispatch/estimate-fare` with the pickup coordinates for every tier, so the tier cards show the same live H3-zone surge that `POST /dispatch/trips` will charge. Nearby cars and pickup ETAs come from dispatch; the legacy simulator feed is not used by this screen.
6. If OSRM is not configured or is unreachable, the destination pin and the straight-line distance/duration fallback remain; the fare comparison still works, but no route line is drawn. The create-trip response still carries the polyline when routing is available.
7. eDelivery uses the same destination-first contract: the customer searches a drop-off with Photon, the OSRM path is drawn on the map, and only then are courier tiers shown with `GET /dispatch/delivery-fare` prices and nearby-courier ETAs. This is an ad-hoc quote/preview screen; dispatch publishes a courier when an existing checkout group is requested.
8. Estay uses the same tile and map stack. Guests can search a destination, use “stays near me” (`latitude`/`longitude` + `radius_km`), and open a listing with its location map; hosts get the same map when creating and editing a stay.

The shared style lives in `apps/ehub/packages/components/maps/style.ts` and `apps/dispatch/src/components/map-style.ts`. If `EXPO_PUBLIC_MAPLIBRE_STYLE_URL` is set, MapLibre loads that vector style; otherwise the app falls back to an OpenStreetMap raster style whose template can be overridden with `EXPO_PUBLIC_OSM_TILE_URL`. The public OSM tile service is development-only and rate-limited; production should use a self-hosted or commercial vector tile style and keep the “© OpenStreetMap contributors” attribution visible.

MapLibre is a native module. It runs in an Expo **development build** (`bunx expo run:android` / `run:ios`, or `eas build --profile development`), not in Expo Go. Both apps declare the `@maplibre/maplibre-react-native` config plugin in `app.json` and use Continuous Native Generation, so do not hand-edit `android/` or `ios/`.

---

## Services (Backends)

| Service | Path | Stack | Port | Purpose | State |
|---|---|---|---|---|---|
| **eHubGo** | `services/eHubGo` | Go 1.26, Gin, SQLC, Postgres, Redis, Casbin RBAC, NATS, OSRM, H3, Swagger | 9000 | Core monolith: identity, e-commerce engine, and all verticals (taxi, food, grocery, laundry, health, cinema, bus, flights, jobs, bills, B2B, C2C, property booking, delivery, driver WebSocket) | Large / production-grade, no Go tests |
| **auth-service** | `services/auth-service` | Go, Gin, Firebase Admin SDK, Postgres, Viper, Zap | 9001 | Firebase → Postgres user sync, role assignment, session JWT issuance | Complete |
| **wallet-service** | `services/wallet-service` | Go, Gin, Postgres, Safaricom Daraja (M-Pesa) | 9002 | Wallet accounts, ledger/journal entries, idempotent transactions, M-Pesa STK Push / B2C payouts | Complete + tests + migrations |
| **dispatch-service** | `services/dispatch-service` | Go, Gin, Redis, Uber H3, Photon/Google Places, OSRM | 9003 | Driver discovery (H3 resolution 9 adaptive 1–3 km + Haversine), places autocomplete, nearby businesses, ride and delivery fare estimates with optional OSRM road legs, a bounded H3 demand/supply surge, a public point-to-point `GET /dispatch/route` that returns the road distance, duration and encoded polyline for mobile map rendering, **per-tier delivery fares** with minimums, **both lifecycles**: the taxi ride (request → accept → start → complete) and the delivery (requested → assigned → *merchant ready* → picked_up + pickup photo → *cashier release* → out_for_delivery → delivered + dropoff photo → *customer confirms* → completed, one record per vendor sub-order), each with database-enforced reservation plus a Redis `SET NX EX` accept mutex, realtime offer/location events over Redis Pub/Sub, and **ratings for both** (1-5 with a complaint/neutral/compliment and a note, one per party per job) | Complete + tests |
| **fulfillment-service** | `services/fulfillment-service` | Go, Gin, Postgres | 9004 | Order fulfillment state machine (assign driver, validated status transitions) | Complete + tests |
| **marketplace-b2b** | `services/marketplace-b2b` | NestJS 10, `pg` | 9005 | In-house catalog: arbitrary-depth categories, brands/models, featured/deal/flash-sale listings, and scheduled discounts with a real start/end window | Complete (read-only catalog + pricing refresh) |
| **marketplace-c2c** | `services/marketplace-c2c` | NestJS 10, `pg`, custom JWT guard, Postgres RLS | 9006 | Peer-to-peer listings (max 5/seller, image limits) with row-level-security migration | Complete |
| **estay-service** | `services/estay-service` | Go, Gin, Postgres (`estay` schema), gRPC | 9007 | Airbnb-style stays: host profiles, vendor-store linking, multiple listings, photos, amenities, pricing, host calendar/blackouts, guest discovery by destination/host/amenity/guest count plus “near me” radius and distance sorting, availability, server-priced booking quotes, booking requests with double-booking protection and KES totals, and a live merchant/capability publish gate | Complete + tests + schema + contracts |
| **merchant-service** | `services/merchant-service` | Go, gRPC, Postgres (`public`), `ehub_merchant` role | 9008 | Vendor registry: merchant registration, owner-scoped management, miniservice capability enrolment, URL-backed verification documents, owner submission, admin document decisions, verification queue/audit, capability approval, owner-issued staff invitations/roles, staff access reads, and the eStay publish gate's live source of truth. Dispatch still authorizes its merchant queue through the monolith's `business_staff` because no storefront mapping exists yet | Complete core + verification + staff RBAC |
| **gateway** | `services/gateway` | Go, Gin, Redis, `packages/clients` | 9010 | HTTP/JSON entry point and BFF: validates the session, applies per-IP public quotas, fans out to the services over gRPC, aggregates, and relays dispatch's Redis Pub/Sub offers/locations as SSE streams | Complete + tests |
| **travel-service** | `services/travel-service` | NestJS 10 (declared) | — | Intended travel domain | Scaffold (`package.json` only) |
| **accommodation-service** | `services/accommodation-service` | NestJS 10 (declared) | - | Legacy placeholder; superseded by `estay-service` | Scaffold (`package.json` only) |
| **lifestyle-service** | `services/lifestyle-service` | NestJS 10 (declared) | — | Intended lifestyle domain | Scaffold (`package.json` only) |
| **templates** | `services/templates` | Go `net/http` + NestJS | — | Starter templates for new services | Template |

### Key backend concepts

- **Auth flow:** Firebase ID token → `POST /api/v1/gateway/auth/sync` → session JWT (HS256, `{user_id, roles}`) issued once and revalidated by every service. `/auth/sync` self-assigns only `user`, `customer`, `vendor`, `driver`, `host` and `c2c_seller`; `admin`, `executive_admin`, `staff` and `store_staff` must be granted operationally. The gateway is the app-facing path; the auth service also exposes a direct HTTP/gRPC surface for internal use.
- **eHubGo** exposes public routes (catalog, pricing, places, nearby drivers, etc.) plus protected routes gated by Casbin RBAC and Firebase JWT middleware, and generated Swagger under `/admin/swagger`.
- **dispatch-service** owns driver availability and the live driver-location H3 index in Redis; **wallet-service** uses a balanced debit/credit ledger.
- **gRPC:** the extracted Go services expose their API over gRPC as well as HTTP/JSON (see [Protobuf Contracts](#protobuf-contracts-proto)); both transports share the same underlying logic. `packages/clients` provides typed clients, and `services/gateway` exposes them over HTTP/JSON.
- **Service data boundaries:** the monolith keeps the table-owner role while it still co-writes a table, so extracted services share `public` only for tables the monolith has not released. A new service gets its own schema from day one when no monolith path writes it. `wallet` and `estay` are already fully isolated schemas; dispatch's `deliveries`/`delivery_*`/`ride_declines`/`service_ratings` are fully owned but still live in `public`, and `taxi_trips`, `orders`, `drivers` and `users` stay transitional until the monolith routes retire. New services must not add cross-service SQL joins; reads go through the owning service's API/gRPC contract, even while the storage is shared.
- **Domain events:** when a cross-service consumer first appears, the producing service emits through a transactional outbox in the same database transaction as the state change. The monolith does not emit events, and no service publishes directly to a broker from inside a request; a relay drains the outbox. Estay starts isolated (its own schema and, if it needs notifications, its own outbox) rather than joining the dispatch lifecycle.
- Local setup guides: `services/DISPATCH_LOCAL_SETUP.md`, `services/MARKETPLACE_LOCAL_SETUP.md`, `services/estay-service/README.md` and `apps/merchant/README.md`.

### Vendor onboarding and the eStay publish gate

The merchant registry and Estay are separate services. The flow is:

1. A signed-in vendor registers a store with `POST /merchants`. It starts in
   `pending` verification.
2. The vendor enrolls eStay with
   `POST /merchants/:id/capabilities {"service_type":"estay"}`. The capability
   also starts `pending`.
3. The vendor adds evidence links with
   `POST /merchants/:id/documents {"type":"id_proof","url":"https://..."}` and
   submits the store with `POST /merchants/:id/verification/submit`. Submission
   moves `pending`, `documents_requested` or `rejected` to `under_review` and
   requires at least one document. The service stores URLs only; file upload
   storage is still an external concern.
4. An administrator reviews the queue in `apps/admin` or through
   `GET /admin/verification/queue`, opens
   `GET /admin/verification/:id`, and records per-document decisions with
   `POST /admin/verification/:id/documents/:documentId/review`. The reviewer then
   sets the store state with `PUT /merchants/:id/status {"status":"approved"}`
   and activates capabilities with
   `PUT /merchants/:id/capabilities/estay/status {"status":"active"}`.
   All admin routes require `admin` or `executive_admin`; `/auth/sync` cannot
   self-assign those roles.
5. The host links the workspace to the owned store with
   `PUT /estay/host/merchant {"merchant_id":"..."}`. Linking is allowed while
   the store is pending, so drafts can be prepared.
6. Publishing a stay (`POST /estay/host/listings/:id/status {"status":"published"}`)
   re-checks the live merchant record. The store must be owned by the caller,
   `verification_status = approved`, `is_active = true`, and its `estay`
   capability must be `active`. Otherwise the service returns a policy error
   (gRPC `PermissionDenied`, HTTP 403) and the listing stays a draft.
7. A store cannot be unlinked while any listing is published. Pause or archive
   those listings first.

The public storefront `GET /merchants/:id` never returns `owner_id`,
`phone_number` or `email`; owners and active staff read the full record from
`GET /merchants/:id/manage`, which also returns the caller's `access_role`,
`is_owner` flag and store permissions. Estay stores only the nullable
`estay.hosts.merchant_id` link and does not copy verification state.

### Store staff and role access

Store membership is separate from global `user_roles`. An owner invites an
existing eHub user with `POST /merchants/:id/staff {"user_id":"...","role":"manager|cashier|delivery"}`.
The response contains a one-time invitation token; the invited user signs in,
opens `/invitations/accept` in `apps/merchant` (or calls
`POST /staff/invitations/accept {"token":"..."}`), and the store appears in
their `/merchants/me` list. The owner can list the roster with
`GET /merchants/:id/staff` and change a member's role or `active`/`suspended`
state with `PUT /merchants/:id/staff/:staffId`. The owner is never stored as a
staff row and cannot be suspended through this API.

Roles expand to scoped permissions: `manager` can view/manage orders and approve
pickups; `cashier` can view orders and approve pickups; `delivery` has read-only
order visibility. Only the owner can manage staff, verification, capabilities or
the store profile. The permissions are published now for the future merchant
order dashboard. Dispatch still authorizes its current merchant queue against
the monolith's `businesses`/`business_staff`, so merchant-service staff do not
yet grant dispatch access until a storefront mapping is added.

`apps/admin` is the reviewer-facing console. It uses its own httpOnly session
cookie, exchanges Firebase sign-in at `/auth/sync` without requesting a
privileged role, and refuses to store the session unless the database-assigned
roles include `admin` or `executive_admin`. The console calls only the gateway.

A fresh local database has no administrator. Grant the role directly with the
owner/superuser connection, then sign in again so the new session token includes
it:

```sql
INSERT INTO user_roles (user_id, role)
VALUES ('<ADMIN_USER_ID>', 'admin')
ON CONFLICT DO NOTHING;
```

Publishing without an approved/active eStay capability is expected to return
HTTP 403 and leave the listing in `draft`; this is the policy gate, not a
transient error.

---

## Protobuf Contracts (`proto/`)

Shared service definitions live under `proto/<service>/<service>.proto`:

| Contract | Service |
|---|---|
| `proto/auth/auth.proto` | auth-service |
| `proto/dispatch/dispatch.proto` | dispatch-service |
| `proto/fulfillment/fulfillment.proto` | fulfillment-service |
| `proto/wallet/wallet.proto` | wallet-service |
| `proto/marketplace/marketplace.proto` | marketplace-b2b / marketplace-c2c |
| `proto/estay/estay.proto` | estay-service |
| `proto/merchant/merchant.proto` | merchant-service |
| `proto/spatial/spatial.proto` | geospatial helpers (currently in eHubGo) |

The contracts mirror each service's real API (they are no longer placeholder RPCs). The dispatch contract covers the full HTTP surface of `dispatch-service` (driver discovery/fares, point-to-point OSRM routing, places autocomplete/details, nearby businesses, driver tasks, and both the taxi ride and the delivery lifecycle) and the marketplace contract mirrors every field the read API returns. Go bindings are generated with **buf** into the shared Go module `packages/proto` (module path `ehub/proto`), which every Go service imports via a `replace` directive:

```bash
bun run generate_protos.ts   # or: cd proto && buf generate
```

Generated packages: `ehub/proto/auth`, `ehub/proto/dispatch`, `ehub/proto/fulfillment`, `ehub/proto/wallet`, `ehub/proto/marketplace`, `ehub/proto/estay`, `ehub/proto/merchant`, `ehub/proto/spatial`.

> Requires `buf` plus `protoc-gen-go` and `protoc-gen-go-grpc` on `PATH`:
> `go install github.com/bufbuild/buf/cmd/buf@latest google.golang.org/protobuf/cmd/protoc-gen-go@latest google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest`

Each backend service now serves **both** HTTP/JSON and gRPC from the same process (dual-serving), so callers can migrate incrementally:

| Service | HTTP | gRPC |
|---|---|---|
| auth-service | 9001 | 9101 |
| wallet-service | 9002 | 9102 |
| dispatch-service | 9003 | 9103 |
| fulfillment-service | 9004 | 9104 |
| marketplace-b2b | 9005 | 9105 |
| marketplace-c2c | 9006 | 9106 |
| estay-service | 9007 | 9107 |
| merchant-service | 9008 | 9108 |
| gateway | 9010 | — |

The gRPC port is configurable via `GRPC_PORT`. The Go services use the generated bindings from `packages/proto`; the NestJS marketplace services load the same `.proto` at runtime with `@grpc/proto-loader`.

> **Local port convention.** Backends use the **9000s for HTTP** and the **9100s for gRPC**, deliberately away from the frontends' local ports (Vite/SvelteKit `5173+`, Expo/Metro `8081+`). The `8000s` are avoided because Metro would clash with `auth-service`.

### Service authentication

Every service validates the session JWT with one shared implementation:

- **Go services** import `packages/platform/auth` (module `ehub/platform`), which pins **HS256**, rejects unsigned/`alg=none` tokens, and exposes:
  - `Validator.Parse` / `Validator.Generate` for token handling;
  - `GinMiddleware` / `GinSession` for HTTP routes (reads the `Authorization: Bearer` header and the `session_token` cookie);
  - `UnaryInterceptor` for gRPC, which authenticates the `authorization` / `session_token` metadata and can exempt public methods.
- **marketplace-c2c** mirrors the same rules in `src/security/session.ts`, shared by its HTTP guard and its gRPC `CreateListing` handler.

Authenticated handlers derive the caller's identity from the token rather than trusting request fields (a `user_id` in the payload must match the token). Public RPCs — fare estimates, driver discovery, catalog reads, Estay discovery/quote, the merchant storefront read, and `auth.SyncUser`/`ValidateToken` — are exempt; everything else requires a valid token. `/auth/sync` only self-assigns participant roles; operational/admin roles must be granted outside the request.

### Service-to-service gRPC and the gateway

`services/gateway` is the single HTTP/JSON entry point. It holds no business logic: it validates the session JWT, applies a per-IP fixed-window quota to public auth and discovery routes, then calls the owning service over gRPC using `packages/clients`, forwarding the caller's token so each service re-validates it (defence in depth). The default budgets are 120 discovery requests/minute and 20 auth requests/minute per client IP, backed by Redis so replicas share the counter. A Redis outage fails open and logs once; the owning service still enforces its own limits.

The gateway also subscribes to dispatch's Redis Pub/Sub channels and relays them as Server-Sent Events: `GET /dispatch/driver/stream` for a driver's offers and `GET /dispatch/trips/:id/locations` for a trip's driver position. The trip stream calls `GetTrip` first, so a stranger cannot subscribe to a driver's location. These two routes accept `?token=` because EventSource cannot set an Authorization header; all other routes keep header/cookie auth. A query token can appear in access logs, so browser clients should use a short-lived token or a dedicated stream ticket; the native driver app sends the bearer header. The gateway still holds no dispatch state: it only authorizes the channel and forwards the event bytes.

All routes are prefixed with `/api/v1/gateway`:

| Area | Public | Authenticated |
|---|---|---|
| auth | `POST /auth/sync`, `POST /auth/validate` | — |
| wallet | — | `GET /wallet/balance`, `POST /wallet/accounts`, `POST /wallet/topup`, `POST /wallet/payout`, `GET /wallet/transactions` |
| dispatch | `POST /dispatch/find-driver`, `GET /dispatch/estimate-fare`, `GET /dispatch/route` (OSRM distance/duration/polyline between two points), `GET /dispatch/delivery-fare`, `GET /dispatch/nearby-drivers`, `GET /dispatch/vehicle-types`, `GET /dispatch/nearby-businesses`, `GET /dispatch/places/autocomplete`, `GET /dispatch/place/:id` | **Rider:** `POST /dispatch/trips` (fare computed by dispatch), `GET /dispatch/trips/:id`, `POST /dispatch/trips/:id/cancel` (optional `{"reason":"..."}`), `GET /dispatch/ride-history`, `POST /dispatch/delivery-requests`, `GET /dispatch/deliveries/:id`, `POST /dispatch/deliveries/:id/cancel`, `POST /dispatch/deliveries/:id/confirm` (customer confirms receipt), `GET /dispatch/orders/:id/deliveries`. **Merchant:** `POST /dispatch/deliveries/:id/ready` (packed-basket stamp), `GET /dispatch/merchant/deliveries` (owner/cashier queue, `?statuses=picked_up&business_id=...`), `POST /dispatch/deliveries/:id/release` (cashier verifies the pickup photo and marks it out for delivery), `POST /dispatch/deliveries/:id/reject-pickup` (reject the photo; the courier retakes it). The service checks the caller owns the business or is active manager/cashier staff. **Driver role:** `GET /dispatch/ride-requests`, `POST /dispatch/trips/:id/accept`, `POST /dispatch/trips/:id/decline` (optional `{"reason":"..."}`), `POST /dispatch/trips/:id/start`, `POST /dispatch/trips/:id/complete`, `GET /dispatch/delivery-requests`, `POST /dispatch/deliveries/:id/accept`, `POST /dispatch/deliveries/:id/decline`, `POST /dispatch/deliveries/:id/pickup` (`{"photo_url":"..."}` required), `POST /dispatch/deliveries/:id/complete` (`{"photo_url":"..."}` required), `GET /dispatch/delivery-history`, `PUT /dispatch/driver/location`, `PUT /dispatch/driver/status`, `GET /dispatch/driver/tasks`. **Ratings (either party, once the job is finished):** `POST /dispatch/trips/:id/rating` (`{"score":1-5,"kind":"complaint\|neutral\|compliment","comment":"..."}`), `GET /dispatch/trips/:id/ratings`, `POST /dispatch/deliveries/:id/rating`, `GET /dispatch/deliveries/:id/ratings`, `GET /dispatch/ratings/summary?subject_user_id=&subject_role=driver\|customer` |
| fulfillment | — | `GET /fulfillment/orders/:id/status`, `POST /fulfillment/orders/:id/assign-driver`, `POST /fulfillment/orders/:id/status` |
| marketplace | `GET /marketplace/products` (category/brand/model/search/on-sale/sort filters), `GET /marketplace/products/:id`, `GET /marketplace/categories`, `GET /marketplace/brands`, `GET /marketplace/models`, `GET /marketplace/listings`, `GET /marketplace/listings/:id` | `POST /marketplace/listings` |
| estay | `GET /estay/amenities`, `GET /estay/listings` (`query`/destination and host-name text, city/country, `host_id`, guests, bedrooms, beds, bathrooms, price range, property type, paired dates, instant-book, `amenity_keys`, `lat`/`lng` + `radius_km`, `sort=distance\|price_asc\|price_desc\|rating`, paging), `GET /estay/listings/:id`, `GET /estay/listings/:id/availability`, `GET /estay/listings/:id/quote?check_in=&check_out=&guests=` (public server-priced preview in KES) | **Host:** `GET`/`PUT /estay/host/profile`, `PUT /estay/host/merchant` (link/unlink an owned vendor store), `GET`/`POST /estay/host/listings`, `GET`/`PUT /estay/host/listings/:id`, `POST /estay/host/listings/:id/status`, `POST /estay/host/listings/:id/photos`, `DELETE /estay/host/listings/:id/photos/:photoID`, `POST /estay/host/listings/:id/cover`, `PUT /estay/host/listings/:id/amenities`, `GET /estay/host/listings/:id/calendar`, `POST /estay/host/listings/:id/blocks`, `DELETE /estay/host/listings/:id/blocks/:blockID`, `GET /estay/host/bookings`, `POST /estay/host/bookings/:id/confirm`, `POST /estay/host/bookings/:id/decline`. **Guest:** `POST /estay/bookings`, `GET /estay/bookings`, `GET /estay/bookings/:id`, `POST /estay/bookings/:id/cancel`. |
| merchant | `GET /merchants/:id` (storefront profile; owner/contact PII stripped) | **Owner:** `POST /merchants` (register a vendor), `GET /merchants/me`, `GET /merchants/:id/manage` (owners and active staff; returns `access_role`, `is_owner` and permissions), `POST`/`GET /merchants/:id/capabilities` (miniservice enrolment; staff may read), `POST`/`GET /merchants/:id/documents` (verification evidence URLs), `POST /merchants/:id/verification/submit`, `POST`/`GET /merchants/:id/staff`, `PUT /merchants/:id/staff/:staffId`. **Invited user:** `POST /staff/invitations/accept`. **Admin (`admin`/`executive_admin`):** `PUT /merchants/:id/status`, `PUT /merchants/:id/capabilities/:service/status`, `GET /admin/verification/queue`, `GET /admin/verification/:id`, `POST /admin/verification/:id/documents/:documentId/review` |
| aggregate | - | `GET /me/home` |

Dispatch also exposes two Server-Sent Event streams outside the table: `GET /dispatch/driver/stream` (driver role; ride and delivery offers) and `GET /dispatch/trips/:id/locations` (a party to the trip; the assigned driver's live position). Both accept `?token=` for EventSource clients; the polled feeds remain the recovery path.

`GET /me/home` is a BFF aggregate: it fans out to the wallet (and, when `lat`/`lng` are supplied, dispatch) concurrently and returns the combined payload, reporting any section that failed without failing the whole request.

Service addresses come from `AUTH_GRPC_ADDR`, `WALLET_GRPC_ADDR`, `DISPATCH_GRPC_ADDR`, `FULFILLMENT_GRPC_ADDR`, `MARKETPLACE_B2B_GRPC_ADDR`, `MARKETPLACE_C2C_GRPC_ADDR`, `ESTAY_GRPC_ADDR` and `MERCHANT_GRPC_ADDR`, defaulting to the local gRPC ports (see `packages/clients`). Estay itself also needs `MERCHANT_GRPC_ADDR` and `MERCHANT_TIMEOUT_SECONDS` so it can enforce the host approval gate; publishing fails closed if the registry is absent.

---

## Local Infrastructure (`infra/`)

`infra/docker-compose.yml` starts a full local stack:

- **postgis/postgis:15-3.4** (`ehub_postgres`, port 5432, db `ehub_db`) with a healthcheck. eHubGo's schema declares four `GEOGRAPHY(POINT, 4326)` columns, so the plain `postgres` image cannot create it.
  - On first boot (empty data volume) the container loads, in order, `services/eHubGo/schema.sql`, the eHubGo attributes migration, the wallet migration, the c2c listing policies, the dispatch ride-lifecycle, vehicle-tier, delivery-lifecycle, fare/readiness, offer-clock, proof-of-handover, ratings and ride-hardening migrations, the estay schema and host-merchant-link migrations, the marketplace catalog taxonomy/promotions migration, the merchant-service schema, capability-review and document-review migrations, and finally `infra/db/contracts.sql` (the ownership contracts). To re-bootstrap, remove the volume: `docker compose down -v`.
  - `services/eHubGo/migrations/002_add_is_featured_to_products.sql` is intentionally not applied (it drops `is_flash_sale`, which `schema.sql` and `SearchProducts` still use).
- **redis:7** (`ehub_redis`, port 6379) — note that a native Redis already listening on 6379 will shadow this published port on the host, so host-run services may silently talk to the wrong server.
- `auth-service` (9001), `wallet-service` (9002), `dispatch-service` (9003), `fulfillment-service` (9004), `estay-service` (9007), `merchant-service` (9008) and `gateway` (9010), each `go run`-ing from its mounted service directory against the shared Postgres/Redis.
- `dispatch-service` is the only Go service that needs cgo: `uber/h3-go` has no pure-Go fallback. Its container installs `gcc`/`musl-dev` and sets `CGO_ENABLED=1`, so the first start compiles the H3 C sources and is noticeably slower than the other services.
- **`ehubgo`** (9000) is opt-in behind a compose profile, because the monolith does not build in its current state (`handlers` ⇄ `handlers/ehost` import cycle) and nothing in the extracted stack depends on it.
- `marketplace-b2b` (9005) and `marketplace-c2c` (9006), the NestJS services (`npm install` + `nest build` on first start; their `node_modules` live in named volumes so the host install is untouched). B2B runs a minute-by-minute pricing refresh so a scheduled discount starts/ends without waiting for a request; the function is `SECURITY DEFINER` and the runtime role only holds `EXECUTE`. Load `infra/db/demo_catalog.sql` as the database owner for a local deep tree, brands/models, a weekly deal and live flash sales.
- Every service exposes a healthcheck, and dependents wait on `service_healthy` before starting.

For the end-to-end local test protocol — prerequisites, device networking, H3 discovery, surge, concurrent accepts, realtime streams, eDelivery, catalogue, Estay and the MapLibre development build — see [`TESTING.md`](TESTING.md). For the production topology, database rollout, container builds, gateway/EAS configuration and integration secrets, see [`DEPLOYMENT.md`](DEPLOYMENT.md).

```bash
cd infra
docker compose up                      # extracted services + gateway
docker compose --profile monolith up   # also attempt the eHubGo monolith
```

Environment variables use development defaults (`JWT_SECRET=ehub-dev-secret-change-me`, etc.) — **change these outside local development.**

### Local OSRM and Photon

The default compose stack does not run the geo providers. To exercise real road
routes and address autocomplete locally:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File infra/geo/prepare-osrm.ps1
powershell -NoProfile -ExecutionPolicy Bypass -File infra/geo/prepare-photon.ps1
docker compose -f infra/docker-compose.yml -f infra/docker-compose.geo.yml `
  up -d postgres redis dispatch-service gateway osrm photon
powershell -NoProfile -ExecutionPolicy Bypass -File infra/geo/smoke.ps1
```

`prepare-osrm.ps1` builds a Kenya MLD graph once. `prepare-photon.ps1` downloads
the Kenya JSON dump and writes a bounded Nairobi fixture, because Photon 1.3.0's
hard-coded 30-second bulk timeout is too short for the full dump on a small
Docker Desktop VM. The smoke test verifies OSRM, Photon, and the public gateway
route endpoint with the same Nairobi coordinates the mobile apps use. See
[`infra/geo/README.md`](infra/geo/README.md) for reset and fallback details.

### Database ownership contracts

All services share one Postgres instance, but each connects with its **own least-privilege role** instead of the superuser. The grants are declared in [`infra/db/contracts.sql`](infra/db/contracts.sql) and enforced by Postgres: a service that touches a table outside its contract gets `permission denied`, not silent success. The rationale, the full table-level matrix (who writes what, and who may read it), and the procedure for changing a contract live in [`infra/db/OWNERSHIP.md`](infra/db/OWNERSHIP.md). For a runnable demonstration of the per-service URLs and role isolation, use [`infra/db/mock-connections.env.example`](infra/db/mock-connections.env.example) and `infra/db/demo_connections.ps1 -Verify`.

| Service | Role | `DATABASE_URL` (dev) |
|---|---|---|
| auth-service | `ehub_auth` | `postgres://ehub_auth:ehub-dev-auth@localhost:5432/ehub_db?sslmode=disable` |
| wallet-service | `ehub_wallet` | `postgres://ehub_wallet:ehub-dev-wallet@localhost:5432/ehub_db?sslmode=disable` |
| dispatch-service | `ehub_dispatch` | `postgres://ehub_dispatch:ehub-dev-dispatch@localhost:5432/ehub_db?sslmode=disable` |
| fulfillment-service | `ehub_fulfillment` | `postgres://ehub_fulfillment:ehub-dev-fulfillment@localhost:5432/ehub_db?sslmode=disable` |
| marketplace-b2b | `ehub_b2b` | `postgres://ehub_b2b:ehub-dev-b2b@localhost:5432/ehub_db?sslmode=disable` |
| marketplace-c2c | `ehub_c2c` | `postgres://ehub_c2c:ehub-dev-c2c@localhost:5432/ehub_db?sslmode=disable` |
| estay-service | `ehub_estay` | `postgres://ehub_estay:ehub-dev-estay@localhost:5432/ehub_db?sslmode=disable` |
| merchant-service | `ehub_merchant` | `postgres://ehub_merchant:ehub-dev-merchant@localhost:5432/ehub_db?sslmode=disable` |

`wallet-service` owns a dedicated `wallet` schema and `estay-service` owns a dedicated `estay` schema (no other service references those tables). `merchant-service` owns the vendor registry tables (`merchants`, `merchant_capabilities`, `merchant_documents`, `merchant_verification_logs`, `merchant_staff`); its runtime role has create/read/update access to document evidence and staff rows, and append/read access to the audit. Staff rows are suspended rather than deleted so the membership history remains inspectable. The remaining tables still live in `public` next to the monolith, which connects as the table owner and is deliberately outside the contract - that is why several tables still have two writers. A service that gains a write is a contract change: `dispatch-service` owns the taxi ride lifecycle in `taxi_trips`, its decline telemetry in `ride_declines`, the delivery lifecycle in `deliveries`, and their shared `service_ratings`, which is why it holds `SELECT, INSERT` on the first two and `SELECT, INSERT, UPDATE` on the latter two.

`deliveries` — with its `delivery_declines` and `delivery_proofs` companions — is one of the fully-owned extracted domains (alongside `estay`). Dispatch creates one row per **vendor sub-order** (the food/grocery/liquor verticals are merchant-sourced, so each merchant ships its own basket and a multi-vendor checkout is several independent courier jobs) and drives `requested → assigned → picked_up → out_for_delivery → delivered → completed`, or `cancelled` before pickup. Two partial unique indexes enforce the invariants in the database rather than in the service: one live delivery per sub-order and one live delivery per courier (including a courier who is out for delivery). Each step mirrors the sub-order's `orders.status` in the same transaction, so the consumer's order view keeps working. The row also stores the server-computed courier fee (`fare_amount`, priced when the delivery is published) and the merchant's `vendor_ready_at` stamp; `PickUp` is refused until the merchant has stamped the basket ready. The hand-over itself is evidence-backed: `PickUp` requires a pickup photo, and only the owner or an active manager/cashier may verify it and release the order (`out_for_delivery`); `Complete` requires a dropoff photo and soft-completes the order (`delivered`), and the customer's `confirm` records receipt (`completed`). A delivered order that the customer does not confirm auto-confirms after 24 hours. The photos live in `delivery_proofs` (URLs only); a rejected pickup photo can be retaken without changing the status. An offer is time-boxed by `expires_at`, and a courier's pass is recorded in `delivery_declines`, so a declined job is hidden from that courier but stays open to the rest. A background reaper cancels requests that aged out (re-opening the sub-order and clearing the fee) and returns accepted-but-uncollected deliveries to the feed after ten minutes, releasing the courier. See `infra/db/OWNERSHIP.md` for the `orders` co-writer note this creates.

`service_ratings` records the reputation of a finished job. Either party leaves one rating per job — the rider rates the driver or vice versa; the customer rates the courier or vice versa — as a 1-5 score with a `complaint`/`neutral`/`compliment` kind and a note. A rating is allowed only when the job is finished (`completed` for a ride; `delivered` or `completed` for a delivery), and the service checks that the caller was a party under a row lock. Re-rating upserts the same row. The subject's `driver` summary spans rides and deliveries, and recording a driver rating refreshes `drivers.rating`, the monolith's denormalized average, in the same transaction; `service_ratings` stays the source of truth. This is the record only — complaint escalation and moderation are planned, not built.

**Estay owns the whole accommodation vertical.** A host is created lazily on their first host action and can list any number of properties. A listing carries its location (street address, city, region, coordinates), specifications (guests, bedrooms, beds, bathrooms), nightly price and cleaning fee in KES, minimum/maximum nights and an instant-book flag. Photos and amenities are separate child records so an edit never has to replace the whole listing; the bootstrap seeds 23 amenities (Wi-Fi, kitchen, parking, pool, etc.), and publishing requires a full address, coordinates, a positive nightly price and at least one photo. Listings move through `draft → published → paused → archived`.

Guests search public listings by destination/host text, guests, bedrooms/beds/bathrooms, amenities, price range, property type, optional paired check-in/check-out dates and instant-book, or by a lat/lng + radius “near me” query with distance sorting. The mobile apps use the gateway's Estay API for discovery, detail, quotes and booking, and the detail page shows the stay on a MapLibre/OSM map. A public `GET /estay/listings/:id/quote` returns the same server-priced total the booking will charge, so the client never estimates the total itself. Bookings move `pending → confirmed → completed`, with `declined` and `cancelled` alternatives; a guest may cancel only before check-in. The service computes and snapshots the nightly price, cleaning fee, platform service fee (5% by default, `ESTAY_SERVICE_FEE_PERCENT`) and total in KES when the request is created — the client never sets a price. Double-booking is prevented in the database by a GiST exclusion constraint on the listing and `daterange(check_in, check_out, '[)')` for pending/confirmed rows, backed by listing-row locking in the service, and host blackout ranges are checked in the same path. A background reaper completes stays whose check-out has passed.

`vehicle_types` is reference data, not a service's own table: the bootstrap seeds the tiers and adds the class/cargo/services columns (`003_vehicle_tiers.sql`) as the table owner, and `dispatch-service` keeps `SELECT`. A driver is matched to a request by class and capacity, so `drivers.vehicle_type_id` must point at a seeded tier — an unknown value means the driver cannot be matched to anything. Delivery requests are matched the same way, with the load choosing the class: `POST /dispatch/delivery-requests` takes a `cargo_weight_kg` (or an explicit `vehicle_type`) and dispatch uses the cheapest tier that can carry it — motorbike, then pickup, then lorry. A load no tier can carry is `400`, not an empty courier list. Publishing to a class with no courier online is refused too (`409`), because the sub-order would otherwise be stranded with no way to withdraw it.

Delivery fares share the ride engine (`services/pricing.go`): base fare + distance + time, with road/traffic/weather/fuel/peak adjustments, but priced from **per-tier rate cards** and floored at a **per-tier minimum** (KES 70 bike, 85 motorbike, 300 pickup, 600 lorry). The spread is deliberate - a lorry burns far more fuel per kilometre than a bike and must not be priced like one. Dispatch computes the fare server-side when it publishes, stores it on the delivery and mirrors it onto the sub-order's `delivery_fee`; `GET /dispatch/delivery-fare` and `rpc EstimateDeliveryFare` use the same function to quote the tiers cheapest-first before checkout. The delivery publish path is router-aware when `OSRM_URL` is set: it uses the OSRM road distance and duration for the pickup -> dropoff leg, and falls back to straight-line distance at an assumed 24 km/h when routing is unavailable. The live demand signal is the H3-zone surge described below plus the deterministic time-of-day peak; weather and traffic default to clear/free-flow.

**Ride fares are also server-authoritative.** `POST /dispatch/trips` prices the leg with the same engine and per-tier ride rate cards (`ev_bike`, `motorbike`, `economy`, `normal`, `xl`, `luxury`; the legacy `ECONOMY`/`COMFORT`/`XL` keys still resolve). When `OSRM_URL` is set, the distance and duration come from OSRM and the create response includes the encoded polyline; otherwise the service keeps the straight-line fallback. A client-supplied `amount` is ignored, so a rider cannot set the price. A ride offer is time-boxed by `expires_at` (15 minutes), a `requested` trip expires through the replica-safe reaper, and an `accepted` trip that is not started within ten minutes is re-offered to the feed and the driver released. A driver can pass on an offer (`POST /dispatch/trips/:id/decline`, recorded in `ride_declines`); the feed only shows online drivers whose last position is less than ten minutes old and who are not busy or on an active job, and `Accept` re-checks online status, vehicle class/capacity and cross-table busyness under a row lock. Accepting a ride or delivery first claims two 15-second Redis `SET NX EX` keys (one per job and one per driver), so concurrent accepts fail fast across replicas; the reservation is released after the Postgres transaction and a Redis outage fails open because the row lock and partial unique indexes remain the final authority. `taxi_trips` stores the priced `distance_km`/`duration_minutes`, the cancel attribution (`cancelled_by`, `cancel_reason`) and the offer clock.

**H3-zone surge** is layered on the time-of-day peak. Demand is a Redis set per H3 resolution-9 pickup cell and supply is the live H3 driver-location set; the multiplier is `1 + 0.2 × demand/supply`, capped at `1.8`, and stays `1.0` when the cell has no supply. A committed request adds demand, accept/cancel release it, the reaper releases expired offers and re-records re-opened accepts, and the demand keys expire after 20 minutes. Lookup fails open to `1.0` with a 250 ms timeout, so a Redis outage never blocks a booking. `CreateTrip` and an estimate with pickup coordinates use the same multiplier, and `FareBreakdown` exposes `surge_multiplier` and `surge_surcharge` in KES.

`contracts.sql` is idempotent: apply it to an existing database with `docker cp infra/db/contracts.sql ehub_postgres:/tmp/` and then `docker exec ehub_postgres psql -U user -d ehub_db -v ON_ERROR_STOP=1 -f /tmp/contracts.sql`. `infra/db/verify_contracts.sql` then fails if the live grants drift from the declared contract, in both directions:

```powershell
docker cp infra/db/verify_contracts.sql ehub_postgres:/tmp/verify_contracts.sql
docker exec ehub_postgres psql -U user -d ehub_db -v ON_ERROR_STOP=1 -f /tmp/verify_contracts.sql
```

### Deploying to Neon

`infra/db/push_to_neon.ps1` applies the same SQL, in the same order, to a Neon database and then runs the contract verifier as a deploy gate:

```powershell
$env:NEON_DATABASE_URL = "postgresql://neondb_owner:PASSWORD@ep-xxxx.us-east-2.aws.neon.tech/neondb?sslmode=require"
./infra/db/push_to_neon.ps1 -NeonUrl $env:NEON_DATABASE_URL
./infra/db/push_to_neon.ps1 -NeonUrl $env:NEON_DATABASE_URL -LoadReferenceData
```

It needs no SQL client install: it uses a `psql` from `PATH` if there is one, otherwise a throwaway container from the PostGIS image the local stack already uses. It refuses a target that looks local unless you pass `-AllowLocal`. Use the **direct** Neon endpoint for schema changes (the `-pooler` host is PgBouncer, which rejects some DDL) and the pooled endpoint for the services' `DATABASE_URL`.

- `schema.sql` is the only DDL the monolith contributes, and it needs exactly one extension: it declares `postgis` itself and uses nothing from `postgis_tiger_geocoder` / `postgis_topology`, which exist locally only because the `postgis` image installs them at initdb.
- `-LoadReferenceData` copies `categories`, `brands` and `product_models` from the local `ehub_db`, in that order, with `ON CONFLICT DO NOTHING` so a repeat run is safe. A self-referencing table (`categories.parent_id`) needs its rows to arrive parent-first; `pg_dump` warns when the source has circular foreign keys, and deeper trees may need a manual load. `vehicle_types` is not copied by default — the schema files seed the bookable tiers, so copying the local rows would push development fixtures over the deployed catalogue. Pass `-ReferenceTables` to override.
- The roles arrive carrying the development passwords from `contracts.sql` (`ehub-dev-<service>`). Rotate them in Neon before exposing the database, and keep production credentials out of the repository. If Neon rejects the `BYPASSRLS` clause when creating the roles, remove that clause — but be aware that row-level security will then start filtering the extracted services, which is the open item described in `infra/db/OWNERSHIP.md`.

---

## Packages (`packages/`)

- `packages/proto` — the shared Go module `ehub/proto` containing generated gRPC/protobuf bindings for every contract. Imported by the Go services via a `replace` directive.
- `packages/platform` — the shared Go module `ehub/platform` for cross-cutting service code. Currently `auth`: session-JWT validation plus the HTTP middleware and gRPC interceptor used by every Go service.
- `packages/clients` — the shared Go module `ehub/clients`: typed gRPC clients for every service, plus the connection plumbing and the client interceptor that propagates the caller's session token. Used by `services/gateway` and available to any service that needs to call another.
- `packages/shared-utils` — currently **empty**; reserved for shared code across services/apps.

---

## Getting Started

### Prerequisites

- **Bun** (root tooling: `bun.lock`) and/or **npm** (per-app lockfiles exist)
- **Go 1.26+** for the Go services
- **Docker** for local Postgres/Redis (or run your own)
- **Node.js** for the NestJS services and SvelteKit apps
- **buf** + `protoc-gen-go` / `protoc-gen-go-grpc` (only if regenerating protobuf code)

### Install dependencies

```bash
bun install
```

### Run the backend stack

```bash
cd infra
docker-compose up
```

### Run an app

```bash
# Install once from the repository root (Bun is the working package manager here).
bun install

# Consumer mobile app: build once for the MapLibre native module, then use the dev client.
cd apps/ehub
bunx expo run:android        # or: bunx expo run:ios
bunx expo start --dev-client --clear

# Driver console: same development-build requirement.
cd apps/dispatch
bunx expo run:android
bunx expo start --dev-client --clear

# Public web portal (SvelteKit scaffold)
cd apps/web && bun install && bun run dev

# Admin verification console (Vite: pass --port if 5173 is in use)
cd apps/admin && bun install && bun run dev

# Merchant host workspace (Estay)
cd apps/merchant && bun install && bun run dev
```

`npm install` currently fails in this workspace with a `Cannot read properties of null (reading 'matches')` resolver error; use Bun. The mobile apps cannot run in Expo Go because MapLibre is a native module. `bunx expo run:*` performs Continuous Native Generation and builds the native project; do not commit or hand-edit generated `android/`/`ios/` directories. For a cloud build, use `eas build --profile development`.

The merchant and admin apps need their server-side environment (see
`apps/merchant/.env.example` and `apps/admin/.env.example`): `GATEWAY_URL` plus
either `FIREBASE_API_KEY` or `ALLOW_DEV_SESSION=true`. Both are SvelteKit BFFs —
the browser only holds an httpOnly session cookie, and all gateway calls are made
server-side. The admin app additionally requires the signed-in account to have
an `admin` or `executive_admin` role before it stores that cookie.

### Environment variables

The mobile apps read `EXPO_PUBLIC_*` variables (see `apps/ehub/.env.example` and `apps/dispatch/.env.example`), including:

- `EXPO_PUBLIC_GATEWAY_URL` — the gateway (`/api/v1/gateway`); used for auth, wallet, dispatch, marketplace and estay
- `EXPO_PUBLIC_API_URL` — eHubGo base (`/api/v1`), still used by the consumer app for the un-extracted verticals
- `EXPO_PUBLIC_INIT_TOKEN` — app startup gate in `apps/ehub/app/_layout.tsx`
- `EXPO_PUBLIC_FIREBASE_*` — Firebase web config for driver sign-in; `EXPO_PUBLIC_FIREBASE_STORAGE_BUCKET` is also the target for pickup/dropoff proof uploads. Storage rules must allow an authenticated driver to write under `dispatch-proofs/<uid>/<job-id>/`.
- `EXPO_PUBLIC_MAPLIBRE_STYLE_URL` — optional MapLibre vector style URL used by both apps. When unset, the app falls back to an OSM raster style (no Google key).
- `EXPO_PUBLIC_OSM_TILE_URL` — OpenStreetMap raster tile template used by the fallback style for `TaxiMap`, `StayMap` and the driver route map; defaults to `https://tile.openstreetmap.org/{z}/{x}/{y}.png` for development. Use a production tile provider or a self-hosted server for real traffic.

Backend services read `DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, `GO_ENV`, `PORT`, `GRPC_PORT`, and service-specific keys (e.g. `GOOGLE_PLACES_API_KEY`, Daraja M-Pesa credentials, `ESTAY_SERVICE_FEE_PERCENT`). `DATABASE_URL` must use the service's own database role (see [Database ownership contracts](#database-ownership-contracts)) — connecting as another role, or as the superuser, bypasses the contract. Dispatch also reads `DELIVERY_REQUEST_TTL_SECONDS` (900), `DELIVERY_ACCEPT_TTL_SECONDS` (600), `DELIVERY_REAPER_INTERVAL_SECONDS` (60), `DELIVERY_AUTO_CONFIRM_SECONDS` (86400), `RIDE_REQUEST_TTL_SECONDS` (900), `RIDE_ACCEPT_TTL_SECONDS` (600), `RIDE_REAPER_INTERVAL_SECONDS` (60), and the optional geo providers: `PLACES_PROVIDER` (`google` or `photon`), `PHOTON_URL`, `PHOTON_COUNTRY_CODE` (KE), `OSRM_URL` and `OSRM_TIMEOUT_SECONDS` (3). When `OSRM_URL` is empty, ride legs and ETAs fall back to straight-line estimates; the reapers run in every replica and arbitrate with `FOR UPDATE SKIP LOCKED`. The gateway also reads `REDIS_URL`, `RATE_LIMIT_DISCOVERY_PER_MINUTE` (120), `RATE_LIMIT_AUTH_PER_MINUTE` (20) and `RATE_LIMIT_WINDOW_SECONDS` (60); set either per-minute limit to 0 to disable that bucket. The same `REDIS_URL` backs the gateway's SSE subscriptions; when Redis is unavailable the streams return `503` and clients keep polling.

---

## Status Summary

- **Functional today:** `eHubGo`, `auth-service`, `dispatch-service`, `wallet-service`, `fulfillment-service`, `marketplace-b2b`, `marketplace-c2c`, `estay-service`, `merchant-service` (vendor registry + verification + staff RBAC), `gateway`, `apps/ehub`, `apps/dispatch`, `apps/merchant` (Estay host workspace + store staff) and `apps/admin` (verification console).
- **Service wiring:** `packages/clients` provides typed gRPC clients, and `services/gateway` (HTTP `9010`) is the single entry point that fans requests out to the services over gRPC (including the `GET /me/home` aggregate). The mobile apps call the gateway for the extracted domains (auth, wallet, dispatch, marketplace, estay, merchant) and only reach `eHubGo` for the un-migrated verticals.
- **gRPC ready:** `packages/proto` (generated Go bindings) and gRPC servers on `auth-service` (9101), `wallet-service` (9102), `dispatch-service` (9103), `fulfillment-service` (9104), `marketplace-b2b` (9105), `marketplace-c2c` (9106), `estay-service` (9107) and `merchant-service` (9108), dual-serving alongside their HTTP APIs and session-authenticated via the shared `packages/platform/auth` interceptor (public read/login RPCs excepted).
- **Scaffolds / stubs:** `apps/web`, `services/travel-service`, `services/accommodation-service` (superseded by `estay-service`), `services/lifestyle-service`, `packages/shared-utils`, `services/templates`.
- **Notable gaps:** `eHubGo` still contains the taxi/ecommerce/C2C/property logic that the extracted services are meant to replace (dual implementations during migration) and is not yet wired to the extracted services (the apps reach it directly for those verticals); `eHubGo` has no Go tests. Rides and deliveries now price themselves server-side; rides use OSRM road distance, ETA and polyline when `OSRM_URL` is configured (with a straight-line fallback), and delivery publish uses the same router for its pickup -> dropoff leg (also with a straight-line 24 km/h fallback). Ride and delivery matching is by vehicle class and tier capacity, but the driver's real seat count is a property of the tier rather than of the vehicle, and the delivery load is supplied per request (`order_items` carries no weight, so a cart cannot yet sum it). The merchant order app is still deferred — readiness and the cashier release queue are backend-only until it ships — while the driver app now captures and uploads pickup/dropoff photos to Firebase Storage and the consumer order screen has the customer confirmation action. That upload relies on Firebase Storage rules allowing an authenticated driver to write its own `dispatch-proofs/` prefix; there is no server-issued upload URL or retention policy yet. Order totals keep the order's stored currency (legacy rows use `Ksh`), while fare amounts and stay prices are always quoted in KES. Dispatch drives its own `deliveries`/`taxi_trips` tables but mirrors delivery progress onto `orders.status` from a driver-scoped transaction context that the `orders` RLS policy does not cover — correct while every service role keeps `BYPASSRLS`, and a prerequisite to fix before it is dropped (see `infra/db/OWNERSHIP.md`). Estay's host workspace and the consumer guest flow are live, including radius/distance discovery and server-priced quotes, but booking payments/refunds, guest reviews, host-guest messaging, notifications, moderation and cancellation fees are still planned; listing photos are URL-only (no upload pipeline or retention policy), and Estay radius search currently uses a Haversine calculation in the service rather than a PostGIS GiST index. `merchant-service` now enforces owner/admin authorization on verification and capability mutations and owner-only staff management; staff roles expand to order permissions, but dispatch's merchant queue still reads the monolith's `business_staff`, so merchant-service staff do not yet grant dispatch access until a storefront mapping exists. `/auth/sync` accepts a caller-supplied allowed role; Estay host identity is a user-scoped `estay.hosts.user_id`, not an approved merchant/business ID, so the merchant-app "eStay capability" is currently a soft `owner_id`/name lookup rather than an enforced link. `eHubGo`'s legacy `properties` tables remain untouched and are not read or written by `estay-service`. There are no domain events/outbox or courier payout ledger yet, and the marketplace and consumer web frontends still need implementation.
