# eHub Local Testing Protocol

This document is the repeatable protocol for testing the extracted eHub services locally, with particular focus on:

- destination-first consumer ride-hailing and eDelivery flows;
- H3 resolution-9 driver discovery;
- Redis reservations and concurrent accept races;
- H3-zone surge pricing;
- OSRM route previews and MapLibre/OSM map rendering;
- Photon destination autocomplete;
- gateway-authenticated HTTP and realtime SSE streams;
- Estay guest discovery, server quote, booking, and host workspace;
- merchant registration, verification document submission, admin document/status review, capability review, store staff invitations/RBAC, host-to-merchant linking, and the live eStay publish gate.

It is written for Windows + Docker Desktop, but the backend commands are platform-neutral.

> **Scope:** the extracted stack (`gateway`, `dispatch-service`, `auth-service`, `wallet-service`, `fulfillment-service`, `marketplace-b2b`, `marketplace-c2c`, `estay-service`, `merchant-service`, Postgres/PostGIS, Redis, OSRM, Photon). The legacy monolith in `services/eHubGo` is **not** required for these flows and currently does not build. Some consumer-app legacy screens still call the Render monolith via `EXPO_PUBLIC_API_URL`; those screens are out of scope here.

---

## 1. Test topology

Use a mix of clients rather than many physical phones.

| Client | Quantity | Purpose |
|---|---:|---|
| Physical rider phone | 1 | Real GPS, permissions, backgrounding, SSE reconnect |
| Physical driver phone | 1 | Real driver location updates, offer acceptance, proof photos |
| Android emulators | 2–4 | Extra drivers/riders with deterministic mock locations |
| Scripted client/harness | 1 | H3 radius, surge, concurrent accepts, load |

One backend stack runs on the development PC. Mobile devices are thin clients and connect to the gateway only.

```text
rider app ─┐
driver app ├── HTTP/JSON + SSE ──> gateway :9010 ──gRPC──> dispatch :9103
merchant ──┤                         │
admin ─────┤                         ├── auth :9101
emulators ─┘                         ├── wallet :9102
                                     ├── fulfillment :9104
                                     ├── marketplace-b2b :9105
                                     ├── marketplace-c2c :9106
                                     ├── estay :9107
                                     └── merchant :9108

dispatch ──> Postgres/PostGIS :5432
         ──> Redis :6379
         ──> OSRM :5000
         ──> Photon :2322
```

The apps must never call Postgres, Redis, OSRM, Photon, or service gRPC ports directly. All extracted-domain app traffic goes through `EXPO_PUBLIC_GATEWAY_URL`; the remaining legacy screens are a migration exception.

---

## 2. Prerequisites

### 2.1 Hardware and OS

- [ ] Windows 10/11 with WSL2 enabled.
- [ ] Docker Desktop running with Linux containers.
- [ ] At least 8 GB RAM; **16 GB recommended**. Photon uses a 1.5 GB Java heap and OSRM/Postgres/Redis/services add more.
- [ ] At least 10 GB free disk for Docker images, the OSRM Kenya graph, and Photon data.
- [ ] Phone and PC on the same Wi-Fi/LAN, or a tunnel/VPN plan (Tailscale, Cloudflare Tunnel, ngrok).

### 2.2 Required tooling

- [ ] `docker` and `docker compose` available.
- [ ] `psql` available on the host, or use `docker exec` into `ehub_postgres`.
- [ ] `redis-cli` available on the host, or use `docker exec` into `ehub_redis`.
- [ ] Node.js and Bun available for the Expo apps.
- [ ] Go toolchain for service tests (`go test`, `go vet`).
- [ ] Android Studio/emulator and/or Xcode/iOS Simulator.
- [ ] A custom Expo development build installed on physical devices (Expo Go cannot run MapLibre). Build once with `bunx expo run:android` / `run:ios`, or `eas build --profile development`.
- [ ] `adb` available for Android emulator location mocking.
- [ ] Firebase project or Firebase Auth emulator for sign-in.

### 2.3 Accounts and credentials

- [ ] A Firebase web app configuration for the consumer app:
  - `EXPO_PUBLIC_FIREBASE_API_KEY`
  - `EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN`
  - `EXPO_PUBLIC_FIREBASE_PROJECT_ID`
  - `EXPO_PUBLIC_FIREBASE_STORAGE_BUCKET`
  - `EXPO_PUBLIC_FIREBASE_MESSAGING_SENDER_ID`
  - `EXPO_PUBLIC_FIREBASE_APP_ID`
- [ ] A separate Firebase web app configuration for the driver app.
- [ ] At least one rider account and two driver accounts. Use distinct accounts per device.
- [ ] Driver accounts must have the `driver` role and a `drivers` row with a valid `vehicle_type_id`.
- [ ] For token-only local testing, the local JWT secret is `ehub-dev-secret-change-me` and `GO_ENV=development`. Never use this outside local development.
- [ ] Firebase Storage bucket/rules if proof-of-handover photo tests are in scope.

### 2.4 Ports

| Port | Service | Notes |
|---:|---|---|
| `5432` | Postgres/PostGIS | A native Postgres on this port will shadow the container |
| `6379` | Redis | A native Redis on this port will shadow the container |
| `2322` | Photon | Optional for H3-only tests; required for Photon autocomplete |
| `5000` | OSRM | Optional; route previews fall back to straight-line estimates when absent |
| `9001`/`9101` | Auth | HTTP/gRPC |
| `9002`/`9102` | Wallet | HTTP/gRPC |
| `9003`/`9103` | Dispatch | HTTP/gRPC |
| `9004`/`9104` | Fulfillment | HTTP/gRPC |
| `9007`/`9107` | Estay | HTTP/gRPC |
| `9008`/`9108` | Merchant | HTTP/gRPC |
| `9010` | Gateway | The only app-facing HTTP entry point for extracted domains |
| `8081` | Metro | Expo dev server; each app may use its own port if prompted |

Check for conflicts before starting:

```powershell
Get-NetTCPConnection -LocalPort 5432,6379,2322,5000,9001,9003,9007,9008,9010 -ErrorAction SilentlyContinue |
  Select-Object LocalAddress,LocalPort,OwningProcess
```

### 2.5 Network requirements

- [ ] Physical devices can reach the PC's LAN IP on `9010` and `8081`.
- [ ] Windows Firewall allows inbound TCP `9010` and `8081` on the private network.
- [ ] Android emulator uses `http://10.0.2.2:9010/api/v1/gateway`.
- [ ] iOS Simulator uses `http://localhost:9010/api/v1/gateway`.
- [ ] Physical devices use `http://<PC-LAN-IP>:9010/api/v1/gateway`.
- [ ] If using Expo `--tunnel`, remember it tunnels **Metro only**, not the gateway.
- [ ] For USB-connected Android devices, `adb reverse` is a fallback:

```powershell
adb reverse tcp:9010 tcp:9010
adb reverse tcp:8081 tcp:8081
```

Then the device can use `http://localhost:9010/api/v1/gateway`.

### 2.6 Test data prerequisites

- [ ] At least one rider user exists in `users`.
- [ ] At least two driver users exist in `users` with `user_roles.role = 'driver'`.
- [ ] Each driver has a `drivers` row and a compatible `vehicle_type_id`.
- [ ] Vehicle tiers are seeded (`vt-ev-bike`, `vt-motorbike`, `vt-economy`, `vt-normal`, `vt-xl`, `vt-luxury`, `vt-pickup`, `vt-lorry`).
- [ ] Drivers can be set `online` and publish locations.
- [ ] Test addresses are in Nairobi when the local Photon fixture is in use.
- [ ] No stale H3/surge keys from a previous run, or they are intentionally being tested.

After a driver signs in once (so the `users` row exists), link the driver profile:

```sql
-- Replace the email with the test driver's real Firebase email.
INSERT INTO user_roles (user_id, role)
SELECT id, 'driver'
FROM users
WHERE email = 'driver1@example.test'
ON CONFLICT DO NOTHING;

INSERT INTO drivers (user_id, name, status, vehicle_type_id, rating)
SELECT id, 'Test Driver 1', 'offline', 'vt-economy', 5.0
FROM users
WHERE email = 'driver1@example.test'
ON CONFLICT (user_id) DO UPDATE
SET name = EXCLUDED.name,
    vehicle_type_id = EXCLUDED.vehicle_type_id,
    status = EXCLUDED.status,
    rating = EXCLUDED.rating;
```

Repeat for each driver, choosing `vt-economy`, `vt-xl`, `vt-motorbike`, etc. A driver whose tier does not serve the requested service will not match.

---

## 3. Environment configuration

### 3.1 Consumer app (`apps/ehub/.env`)

```dotenv
# Legacy verticals only. Ride/delivery through the gateway does not need this.
EXPO_PUBLIC_API_URL=https://ehubgo.onrender.com/api/v1

# Gateway: Android emulator / iOS simulator / physical device
EXPO_PUBLIC_GATEWAY_URL=http://10.0.2.2:9010/api/v1/gateway

# MapLibre: optional self-hosted/vector style. Leave blank to use the OSM raster fallback.
EXPO_PUBLIC_MAPLIBRE_STYLE_URL=
# Fallback OSM raster tiles. Development only; keep attribution visible.
EXPO_PUBLIC_OSM_TILE_URL=https://tile.openstreetmap.org/{z}/{x}/{y}.png

# Firebase web config used by the app's auth client.
EXPO_PUBLIC_FIREBASE_API_KEY=
EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN=
EXPO_PUBLIC_FIREBASE_PROJECT_ID=
EXPO_PUBLIC_FIREBASE_STORAGE_BUCKET=
EXPO_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=
EXPO_PUBLIC_FIREBASE_APP_ID=
```

### 3.2 Driver app (`apps/dispatch/.env`)

```dotenv
EXPO_PUBLIC_GATEWAY_URL=http://10.0.2.2:9010/api/v1/gateway
EXPO_PUBLIC_MAPLIBRE_STYLE_URL=
EXPO_PUBLIC_OSM_TILE_URL=https://tile.openstreetmap.org/{z}/{x}/{y}.png

EXPO_PUBLIC_FIREBASE_API_KEY=
EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN=
EXPO_PUBLIC_FIREBASE_PROJECT_ID=
EXPO_PUBLIC_FIREBASE_STORAGE_BUCKET=
EXPO_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=
EXPO_PUBLIC_FIREBASE_APP_ID=
```

`EXPO_PUBLIC_*` values are baked into the bundle. Restart Expo with `--clear` after changing them.

### 3.3 Backend environment

The Compose stack supplies the local development values. The geo override sets:

```dotenv
PLACES_PROVIDER=photon
PHOTON_URL=http://photon:2322
PHOTON_COUNTRY_CODE=KE
OSRM_URL=http://osrm:5000
OSRM_TIMEOUT_SECONDS=5
```

These URLs are **container-internal**. Apps still use only the gateway.

Local auth defaults:

```dotenv
JWT_SECRET=ehub-dev-secret-change-me
GO_ENV=development
```

The gateway's local rate limits are per client IP:

```dotenv
RATE_LIMIT_DISCOVERY_PER_MINUTE=120
RATE_LIMIT_AUTH_PER_MINUTE=20
RATE_LIMIT_WINDOW_SECONDS=60
```

A scripted harness can hit `429` before dispatch is exercised. Raise these limits only for local load tests, or keep the harness under the quota.

---

## 4. First-time setup

Run from the repository root unless noted.

### 4.1 Prepare geo data

```powershell
cd infra

# Download/build the Kenya OSRM graph. This can take a long time.
powershell -NoProfile -ExecutionPolicy Bypass -File .\geo\prepare-osrm.ps1

# Generate the Nairobi Photon fixture used by local Docker.
powershell -NoProfile -ExecutionPolicy Bypass -File .\geo\prepare-photon.ps1
```

The local Photon fixture is intentionally Nairobi-only because Photon 1.3.0's 30-second bulk timeout is too short for the full Kenya dump on a small Docker VM. Use Nairobi addresses for autocomplete tests.

### 4.2 Start the backend

```powershell
cd infra

docker compose -f docker-compose.yml -f docker-compose.geo.yml up -d `
  postgres redis dispatch-service gateway osrm photon
```

Compose starts gateway dependencies (`auth-service`, `wallet-service`, `fulfillment-service`, `estay-service`, `merchant-service`) automatically. The monolith is opt-in under the `monolith` profile and is not needed.

Do not start marketplace services unless the test includes marketplace behavior.

### 4.3 Wait for health

```powershell
docker compose -f docker-compose.yml -f docker-compose.geo.yml ps
```

All required services should be `healthy`. Photon can take several minutes on first boot because it imports the fixture.

```powershell
curl http://localhost:9010/api/v1/gateway/health
curl "http://localhost:2322/api?q=Nairobi&limit=1"
curl "http://localhost:5000/route/v1/driving/36.8219,-1.2921;36.8319,-1.2821?overview=full&geometries=polyline"
```

### 4.4 Verify database contracts and seed data

```powershell
# Return to the repository root if the previous section left you in infra/.
cd ..

# The Estay host link is 13b_estay_host_merchant.sql, the marketplace catalog is
# 14_marketplace_catalog.sql, the merchant schema is 15_merchant.sql, the
# capability review metadata is 15b_merchant_capability_review.sql, the document
# review metadata is 15c_merchant_document_review.sql, the staff RBAC migration
# is 15d_merchant_staff_rbac.sql, and the contracts are 16_contracts.sql. All are
# idempotent.
docker exec -it ehub_postgres psql -U user -d ehub_db -f /docker-entrypoint-initdb.d/13b_estay_host_merchant.sql
docker exec -it ehub_postgres psql -U user -d ehub_db -f /docker-entrypoint-initdb.d/14_marketplace_catalog.sql
docker exec -it ehub_postgres psql -U user -d ehub_db -f /docker-entrypoint-initdb.d/15_merchant.sql
docker exec -it ehub_postgres psql -U user -d ehub_db -f /docker-entrypoint-initdb.d/15b_merchant_capability_review.sql
docker exec -it ehub_postgres psql -U user -d ehub_db -f /docker-entrypoint-initdb.d/15c_merchant_document_review.sql
docker exec -it ehub_postgres psql -U user -d ehub_db -f /docker-entrypoint-initdb.d/15d_merchant_staff_rbac.sql
docker exec -it ehub_postgres psql -U user -d ehub_db -f /docker-entrypoint-initdb.d/16_contracts.sql

# Verify the ownership contracts. Copy the file into the container first.
docker cp infra\db\verify_contracts.sql ehub_postgres:/tmp/verify_contracts.sql
docker exec -it ehub_postgres psql -U user -d ehub_db -f /tmp/verify_contracts.sql
```

If the Postgres container was created before the `13b`/`15b`/`15c`/`15d` mounts were
added, those files will not exist in the container. Recreate it with
`docker compose -f infra/docker-compose.yml up -d --force-recreate postgres`, or
copy the missing migrations into the container and apply them before the
contracts. A new empty volume applies everything automatically.

The contract check is expected to pass with 93 rows (79 catalogue-era rows plus
14 merchant grants). The catalog migration adds
`categories.sort_order/is_active`, `brands.is_active`, `product_models.is_active`,
`products.original_price/is_active/is_on_sale/discount_start_at/discount_end_at`,
`product_discounts.is_flash_sale/label`, and the `catalog_refresh_product_pricing`
function. `ehub_b2b` also gets `SELECT` on `businesses` so product detail can show
the seller name; it remains read-only on every table. Verify the effective-price path:

```powershell
docker exec -it ehub_postgres psql -U user -d ehub_db -c `
  "SELECT id, price, original_price, is_on_sale, is_flash_sale, discount_start_at, discount_end_at FROM products ORDER BY updated_at DESC LIMIT 5;"
```

Verify vehicle tiers and drivers:

```powershell
docker exec -it ehub_postgres psql -U user -d ehub_db -c `
  "SELECT id, name, tier_key, services FROM vehicle_types ORDER BY display_order;"

docker exec -it ehub_postgres psql -U user -d ehub_db -c `
  "SELECT d.id, u.email, d.name, d.status, d.vehicle_type_id
   FROM drivers d JOIN users u ON u.id = d.user_id;"
```

### 4.5 Start the apps

Both apps use MapLibre, so build the native development client once and then run Metro in dev-client mode. Expo Go cannot load the MapLibre native module.

```powershell
cd apps\ehub
bunx expo run:android        # or: bunx expo run:ios
bunx expo start --dev-client --lan --clear
```

```powershell
cd apps\dispatch
bunx expo run:android
bunx expo start --dev-client --lan --clear
```

A cloud option is `eas build --profile development` followed by `expo start --dev-client`. Use `--clear` after env changes or a Metro cache error. Do not alternate Node and Bun for the same Metro cache; their V8 cache formats differ. Rebuild the development client when native config/plugins change.

---

## 5. Pre-flight checklist

Do not begin scenario testing until all of these pass.

- [ ] Docker Desktop is running.
- [ ] `postgres`, `redis`, `auth-service`, `wallet-service`, `dispatch-service`, `fulfillment-service`, `estay-service`, `merchant-service`, and `gateway` are healthy.
- [ ] `GET http://localhost:9010/api/v1/gateway/health` returns `200`.
- [ ] Both apps are running in a development build (not Expo Go) and a MapLibre map renders tiles with attribution.
- [ ] OSRM health/route check returns a route with a non-empty polyline.
- [ ] Photon health/autocomplete check returns at least one Nairobi result.
- [ ] The rider app can sign in and reach the gateway.
- [ ] The driver app can sign in and reach the gateway.
- [ ] The driver app can go online and publish a location.
- [ ] `GET /dispatch/vehicle-types?service_type=ride` returns tiers.
- [ ] `GET /dispatch/nearby-drivers` sees at least one online test driver near the pickup.
- [ ] Database has no unexpected stale trips or reservations from a previous run.
- [ ] Physical devices can reach the gateway on the LAN IP.
- [ ] Firewall allows `9010` and `8081`.
- [ ] Test accounts are distinct per device.

If auth or gateway health fails, stop. Do not treat downstream failures as dispatch bugs.

---

## 6. Test protocol

### Phase 0 — Static checks

Run from the repository root before live testing when code changed.

```powershell
cd services\auth-service
go test ./...
go vet ./...

cd ..\dispatch-service
go test ./...
go vet ./...

cd ..\gateway
go test ./...
go vet ./...

cd ..\estay-service
go test ./...
go vet ./...

cd ..\merchant-service
go test ./...
go vet ./...

cd ..\..\packages\platform
go test ./...
go vet ./...

cd ..\..\proto
buf lint
cd ..\
```

```powershell
# Consumer typecheck. The Estay/MapLibre files are clean; food/grocery still has
# pre-existing createDispatchDeliveryRequest errors unrelated to this work.
cd apps\ehub
bunx tsc --noEmit
cd ..\..
```

```powershell
# Driver typecheck and lint, including the MapLibre route-map.
cd apps\dispatch
bunx tsc --noEmit
bunx expo lint src --no-cache
cd ..\..
```

```powershell
# Merchant SvelteKit host workspace, verification documents and vendor-approval UI.
cd apps\merchant
bun run check
bun run build
cd ..\..

# Admin SvelteKit verification console, document review and capability approval.
cd apps\admin
bun run check
bun run build
cd ..\..
```

Optional contract check:

```powershell
docker cp infra\db\verify_contracts.sql ehub_postgres:/tmp/verify_contracts.sql
docker exec -it ehub_postgres psql -U user -d ehub_db -f /tmp/verify_contracts.sql
```

Expected result: Go tests, vet, `buf lint`, `apps/dispatch` typecheck/lint, `apps/ehub` `bunx tsc --noEmit`, and `apps/merchant`/`apps/admin` `bun run check` and `bun run build` pass. The contract check is expected to pass with **93 rows** (79 catalogue-era rows plus 14 merchant grants). The food/grocery previews intentionally no longer call a non-existent standalone dispatch method; they show an explicit "ordering not connected" message until sample items are replaced with cart-backed products.

### Phase 1 — Backend smoke

1. Confirm gateway health.
2. Confirm OSRM route response includes `distance_meters`, `duration_seconds`, and `polyline`.
3. Confirm Photon autocomplete returns Nairobi suggestions for `q=Westlands`:

```powershell
curl "http://localhost:9010/api/v1/gateway/dispatch/places/autocomplete?q=Westlands&lat=-1.2921&lng=36.8219"
```

4. Confirm Redis responds:

```powershell
docker exec -it ehub_redis redis-cli ping
```

5. Confirm Postgres responds:

```powershell
docker exec -it ehub_postgres pg_isready -U user -d ehub_db
```

Expected: all dependencies reachable. If OSRM/Photon are down, mark route/autocomplete tests blocked but continue H3 tests.

### Phase 2 — Authentication and gateway authorization

1. Sign in on the rider app with a rider account.
2. Sign in on each driver device with a distinct driver account.
3. Confirm each app receives a session and can call an authenticated route.
4. Confirm a rider token cannot call a driver-only route:

```powershell
curl -i -X PUT "http://localhost:9010/api/v1/gateway/dispatch/driver/status" `
  -H "Authorization: Bearer $RIDER_TOKEN" `
  -H "Content-Type: application/json" `
  -d '{"status":"online"}'
```

Expected: `403` for the rider token. A driver token should succeed.

5. Confirm the driver app can go online and publish a location:

```powershell
curl -i -X PUT "http://localhost:9010/api/v1/gateway/dispatch/driver/location" `
  -H "Authorization: Bearer $DRIVER_TOKEN" `
  -H "Content-Type: application/json" `
  -d '{"lat":-1.2921,"lng":36.8219}'
```

The driver-location payload uses `lat` and `lng`, not `latitude`/`longitude`.

Expected: `200`, and the driver appears in the H3 index.

6. Confirm `/auth/sync` does not self-assign privileged roles:

```powershell
curl -X POST "http://localhost:9010/api/v1/gateway/auth/sync" `
  -H "Content-Type: application/json" `
  -d '{"id_token":"'$FIREBASE_ID_TOKEN'","role":"admin","include_session_token":true}'
```

Expected: the response succeeds but the returned user has no `admin`,
`executive_admin`, `staff` or `store_staff` role. Repeating with `role:"host"`
may add `host`; that role is product intent, not merchant verification.

### Phase 3 — Destination-first ride flow

Preconditions:

- Rider app is signed in.
- At least one eligible driver is online near the pickup.
- OSRM and Photon are healthy.

Steps:

1. Open eTaxi.
2. Confirm the destination search is shown **before** vehicle tiers.
3. Search for a Nairobi destination, e.g. `Westlands`.
4. Select a Photon suggestion.
5. Confirm the app calls `GET /dispatch/route` and MapLibre draws the road path as a GeoJSON line layer over OSM/vector tiles; the attribution is visible.
6. Confirm vehicle tiers appear only after the destination is selected.
7. Confirm fare quotes are in KES and include the live surge multiplier when pickup coordinates are supplied.
8. Select a tier and request the ride.
9. Confirm the rider enters the searching state.
10. Confirm the driver receives the offer and the trip appears in the driver feed.

Reference requests:

```powershell
curl "http://localhost:9010/api/v1/gateway/dispatch/route?from_lat=-1.2921&from_lng=36.8219&to_lat=-1.2647&to_lng=36.8028"

curl "http://localhost:9010/api/v1/gateway/dispatch/estimate-fare?distance_km=5.2&duration_minutes=14&vehicle_tier_key=economy&pickup_lat=-1.2921&pickup_lng=36.8219"
```

Create a trip:

```powershell
curl -i -X POST "http://localhost:9010/api/v1/gateway/dispatch/trips" `
  -H "Authorization: Bearer $RIDER_TOKEN" `
  -H "Content-Type: application/json" `
  -d '{
    "pickup_lat": -1.2921,
    "pickup_lng": 36.8219,
    "dropoff_lat": -1.2647,
    "dropoff_lng": 36.8028,
    "vehicle_type": "economy"
  }'
```

Expected:

- `201 Created` with a trip id.
- Server-computed fare, not a client-supplied amount.
- Route line on both rider and driver maps.
- Driver offer appears without waiting for the poll interval when SSE is connected.
- No hardcoded destinations.

### Phase 4 — H3 discovery and radius expansion

H3 is inside `dispatch-service`; it is not a separate backend.

Matching properties:

- Resolution 9 cells, roughly 200 m edge length.
- Adaptive `gridDisk` search from 1 km to 3 km.
- Driver locations are stored in Redis with a **45-second TTL**.
- Drivers must be `online`, have a compatible tier, and publish fresh locations.

Place drivers at known distances from the rider pickup using emulator mock locations or scripted location updates:

```powershell
# Android emulator: longitude first, then latitude.
adb -s emulator-5554 emu geo fix 36.8219 -1.2921
adb -s emulator-5556 emu geo fix 36.8319 -1.2821
```

Query candidates:

```powershell
curl "http://localhost:9010/api/v1/gateway/dispatch/nearby-drivers?lat=-1.2921&lng=36.8219&service_type=ride&vehicle_type=economy&limit=10"
```

Test cases:

| Case | Setup | Expected |
|---|---|---|
| Near driver | Driver within 1 km | Appears first |
| Radius expansion | Driver at 1.5–2.5 km | Appears as search expands |
| Outside radius | Driver >3 km | Does not appear |
| Stale location | Stop updates for >45 s | Driver disappears from H3 index |
| Offline driver | Set status `offline` | Driver does not match |
| Wrong tier | Request XL with only economy driver | No match or `409` |
| Cross-cell boundary | Driver just across a cell edge but within radius | Still matched via `gridDisk` |

Inspect Redis:

```powershell
docker exec -it ehub_redis redis-cli --scan --pattern "dispatch:h3:r9:*"
docker exec -it ehub_redis redis-cli --scan --pattern "driver:loc:*"
docker exec -it ehub_redis redis-cli --scan --pattern "drivers:active"
```

### Phase 5 — H3-zone surge

Surge model:

- Demand is a Redis set per resolution-9 pickup cell.
- Supply is the live H3 driver-location set.
- Multiplier is `1 + 0.2 * demand/supply`, capped at `1.8`.
- Multiplier stays `1.0` when supply is zero.
- Demand expires after 20 minutes.
- A trip-to-cell mapping makes release idempotent on accept, cancel, and reaper paths.
- Surge is best-effort: Redis/index failures price at `1.0`.
- Surge currently applies to rides, not delivery.

Steps:

1. Put one or two eligible drivers in the same pickup cell and keep them online.
2. Create multiple rider requests from the same pickup cell using distinct rider accounts.
3. Inspect demand:

```powershell
docker exec -it ehub_redis redis-cli --scan --pattern "dispatch:surge:r9:*"
```

4. Request an estimate with pickup coordinates:

```powershell
curl "http://localhost:9010/api/v1/gateway/dispatch/estimate-fare?distance_km=5.2&duration_minutes=14&vehicle_tier_key=economy&pickup_lat=-1.2921&pickup_lng=36.8219"
```

5. Confirm the response includes a multiplier above `1.0`.
6. Confirm the multiplier caps at `1.8`.
7. Cancel a trip or let it complete.
8. Confirm demand is released and the multiplier returns toward `1.0`.
9. Repeat with no online drivers and confirm the multiplier remains `1.0`.

Expected: quotes and `CreateTrip` see the same live surge for the same pickup cell.

### Phase 6 — Concurrent accept race

Preconditions:

- Two driver devices are online and eligible for the same trip.
- Both can see the same offer.

Steps:

1. Rider creates one trip.
2. Both drivers tap **Accept** at nearly the same time.
3. Record both responses.

Expected:

- Exactly one driver wins.
- The other receives a conflict (`409`) or an equivalent "already taken" response.
- The trip has one assigned driver.
- No duplicate assignment exists in Postgres.
- Redis accept reservation uses `SET NX EX` with a short TTL.
- If Redis is unavailable, the Postgres row lock and partial unique index remain authoritative.

Inspect:

```powershell
docker exec -it ehub_postgres psql -U user -d ehub_db -c `
  "SELECT id, status, driver_id, vehicle_type_id, created_at
   FROM taxi_trips ORDER BY created_at DESC LIMIT 10;"
```

### Phase 7 — Ride lifecycle and billing/audit

For the winning driver:

1. Accept the trip.
2. Start the trip.
3. Publish location updates while in transit.
4. Complete the trip.
5. Confirm the rider sees the completed state.
6. Confirm the final fare is server-authoritative and in KES.
7. Confirm lifecycle/audit rows exist.
8. Confirm the driver returns to an available state.

Reference endpoints:

```text
POST /dispatch/trips/:id/accept
POST /dispatch/trips/:id/start
POST /dispatch/trips/:id/complete
POST /dispatch/trips/:id/cancel
GET  /dispatch/trips/:id
GET  /dispatch/ride-history
```

Expected: valid transitions only; invalid transitions are rejected. Cancellation releases surge demand and any reservation.

### Phase 8 — Realtime SSE

Driver offer stream:

```powershell
curl -N "http://localhost:9010/api/v1/gateway/dispatch/driver/stream" `
  -H "Authorization: Bearer $DRIVER_TOKEN"
```

Trip location stream (header auth):

```powershell
curl -N "http://localhost:9010/api/v1/gateway/dispatch/trips/$TRIP_ID/locations" `
  -H "Authorization: Bearer $RIDER_TOKEN"
```

EventSource/browser-style query token:

```text
GET /api/v1/gateway/dispatch/trips/$TRIP_ID/locations?token=$RIDER_TOKEN
```

Expected:

- `Content-Type: text/event-stream`.
- `: keep-alive` heartbeat every ~15 seconds.
- Driver receives an offer event when a trip is created.
- Rider/driver receive location events after the driver publishes.
- A stream for an unassigned trip returns `409`.
- A stream for a trip the caller is not party to is rejected.
- If the gateway Redis broker is unavailable, the stream returns `503`.
- Killing and restarting the app recovers through polling; Pub/Sub has no replay.

### Phase 9 — eDelivery destination-first flow

Preconditions:

- eDelivery is signed in.
- Photon and OSRM are healthy.
- At least one delivery-capable courier is online (`vt-ev-bike`, `vt-motorbike`, `vt-pickup`, or `vt-lorry`).

Steps:

1. Open eDelivery.
2. Confirm drop-off destination search is shown before courier tiers.
3. Search and select a Nairobi drop-off.
4. Confirm the delivery route polyline is drawn by MapLibre over OSM/vector tiles.
5. Confirm courier tiers appear only after destination selection.
6. Confirm quotes are in KES and returned cheapest-first.
7. Confirm nearby courier ETAs appear.
8. Confirm the quote uses the device pickup location or vendor coordinates as appropriate.

Reference requests:

```powershell
curl "http://localhost:9010/api/v1/gateway/dispatch/route?from_lat=-1.2921&from_lng=36.8219&to_lat=-1.2647&to_lng=36.8028"

curl "http://localhost:9010/api/v1/gateway/dispatch/delivery-fare?distance_km=4.5&duration_minutes=12&vehicle_type=motorbike"

curl "http://localhost:9010/api/v1/gateway/dispatch/nearby-drivers?lat=-1.2921&lng=36.8219&service_type=delivery&vehicle_type=motorbike&limit=10"
```

Known limitation: eDelivery currently confirms a **quote**. Dispatch's publish path requires an existing parent order/checkout group. Standalone ad-hoc package delivery creation is not implemented. Do not treat quote confirmation as a dispatch delivery row.

### Phase 9b — In-house catalogue: taxonomy, deals and flash sales

Preconditions:

- The marketplace catalog migration and `infra/db/contracts.sql` have been applied (section 4.4).
- `marketplace-b2b` and `gateway` are running.
- Optional: load `infra/db/demo_catalog.sql` into a local database to get a known tree and promotions.

Load the demo catalogue (owner role, never the B2B runtime role):

```powershell
docker cp infra\db\demo_catalog.sql ehub_postgres:/tmp/demo_catalog.sql
docker exec -it ehub_postgres psql -U user -d ehub_db -v ON_ERROR_STOP=1 -f /tmp/demo_catalog.sql
```

API checks:

```powershell
# Full tree, including arbitrary depth and product counts
curl "http://localhost:9010/api/v1/gateway/marketplace/categories"

# Category subtree (Electronics should include Smartphones/Laptops products)
curl "http://localhost:9010/api/v1/gateway/marketplace/products?category_id=demo-electronics&include_descendants=true&limit=20"

# Brand and model filters
curl "http://localhost:9010/api/v1/gateway/marketplace/brands?category_id=demo-smartphones"
curl "http://localhost:9010/api/v1/gateway/marketplace/models?brand_id=demo-brand-samsung&category_id=demo-smartphones"
curl "http://localhost:9010/api/v1/gateway/marketplace/products?brand_id=demo-brand-samsung&model_id=demo-model-galaxy-s24"

# Normal deal (scheduled discount that is not a flash sale) and live flash sale
curl "http://localhost:9010/api/v1/gateway/marketplace/products?normal_sale=true&sort=discount"
curl "http://localhost:9010/api/v1/gateway/marketplace/products?flash_sale=true"
```

Expected:

1. `products[].price` is the effective (sale) price; `original_price` is the list price and is greater when `is_on_sale` is true.
2. `normal_sale=true` returns only products with `is_on_sale = true` and `is_flash_sale = false`; every live flash-sale product has `discount_start_at <= now < discount_end_at`, a non-empty `discount_end_at`, and `is_flash_sale = true`.
3. The home screen shows a live countdown for the earliest flash-sale end; the flash-sale screen and product cards count down to each product's real end time.
4. Category cards open `/shop/category/:id`; the category screen shows breadcrumbs, child categories, brand chips, model chips (after a brand is selected), price/sort filters and a paginated grid.
5. All-categories renders the full tree (Electronics > Smartphones, Fashion > Men > Shoes), not just roots.
6. Search matches product names, descriptions, category names, brand names and model names; the filter sheet can narrow by category, brand, model, price, on-sale, normal-deal and flash-sale.
7. Product detail shows the category path, brand/model chips, effective price, original price and the sale countdown.

Verify the pricing refresh from SQL:

```powershell
docker exec -it ehub_postgres psql -U user -d ehub_db -c `
  "SELECT p.id, p.price, p.original_price, p.is_on_sale, p.is_flash_sale, p.discount_start_at, p.discount_end_at FROM products p WHERE p.id LIKE 'demo-%' ORDER BY p.id;"
```

Then prove the window is real by expiring the demo flash discount and refreshing:

```powershell
docker exec -it ehub_postgres psql -U user -d ehub_db -c `
  "UPDATE product_discounts SET end_at = now() - interval '1 second' WHERE label = 'Demo flash sale'; SELECT public.catalog_refresh_product_pricing(); SELECT id, price, original_price, is_on_sale, is_flash_sale FROM products WHERE id = 'demo-product-galaxy-s24';"
```

Expected: the Galaxy S24 returns to its list price and `is_on_sale`/`is_flash_sale` become false. Re-run `infra/db/demo_catalog.sql` to restore the demo window.

Known limits:

- Brand and model are separate foreign keys, not category nodes; the UI shows the category path plus brand/model chips.
- `product_discounts` is the source of truth for promotions; `products.discount_percentage`/`is_flash_sale` are refreshed derived values.
- The consumer app still uses the monolith for cart/checkout; the migration keeps `products.price` aligned so those legacy paths charge the sale price.

### Phase 9c — Estay discovery, quote, booking and host workspace

Preconditions:

- `estay-service`, `merchant-service` and `gateway` are running and healthy.
- `MERCHANT_GRPC_ADDR=merchant-service:9108` is set on `estay-service`.
- The Estay migrations (`001_estay.sql`, `002_host_merchant.sql`) and `infra/db/contracts.sql` have been applied (section 4.4), so `ehub_estay` can reach the `estay` schema.
- The host owns a merchant store that is `approved`, `is_active = true`, and has an `estay` capability in `active` state. Complete Phase 9d first.
- The host workspace is linked with `PUT /estay/host/merchant`.
- At least one host account has created and published a listing with a photo and coordinates. Use the mobile host screens (`ehost-create.tsx`) or the merchant host workspace.
- A second account is available as the guest.

Public discovery and quote (no token):

```powershell
$GW = "http://localhost:9010/api/v1/gateway"

curl "$GW/estay/amenities"
curl "$GW/estay/listings?query=Nairobi&guests=2&limit=10"
curl "$GW/estay/listings?lat=-1.2921&lng=36.8219&radius_km=10&sort=distance&limit=10"
curl "$GW/estay/listings/$STAY_ID"
curl "$GW/estay/listings/$STAY_ID/availability?from=2026-12-01&to=2026-12-10"
curl "$GW/estay/listings/$STAY_ID/quote?check_in=2026-12-01&check_out=2026-12-03&guests=2"
```

Expected:

1. The list contains only published stays; the near-me query is filtered by radius and ordered by distance.
2. `amenity_keys` is repeatable; a listing must have every requested key.
3. The quote is in KES and equals `nightly_price * nights + cleaning_fee + service_fee`; the client never sends a total.
4. `quote` works without a session; `POST /estay/bookings` does not.

Authenticated guest flow:

```powershell
curl -X POST "$GW/estay/bookings" `
  -H "Authorization: Bearer $GUEST_TOKEN" -H "Content-Type: application/json" `
  -d '{"listing_id":"'$STAY_ID'","check_in":"2026-12-01","check_out":"2026-12-03","guests":2,"guest_note":"Late arrival"}'

curl "$GW/estay/bookings" -H "Authorization: Bearer $GUEST_TOKEN"
curl "$GW/estay/bookings/$BOOKING_ID" -H "Authorization: Bearer $GUEST_TOKEN"
```

Host flow:

```powershell
curl "$GW/estay/host/profile" -H "Authorization: Bearer $HOST_TOKEN"

curl -X PUT "$GW/estay/host/merchant" `
  -H "Authorization: Bearer $HOST_TOKEN" -H "Content-Type: application/json" `
  -d '{"merchant_id":"'$MERCHANT_ID'"}'

curl "$GW/estay/host/listings" -H "Authorization: Bearer $HOST_TOKEN"
curl "$GW/estay/host/bookings?status=pending" -H "Authorization: Bearer $HOST_TOKEN"
curl -X POST "$GW/estay/host/bookings/$BOOKING_ID/confirm" -H "Authorization: Bearer $HOST_TOKEN"
```

Expected:

1. The guest sees the booking in `pending`, then `confirmed` after the host confirms.
2. A second overlapping booking for the same listing/dates fails (service check and GiST exclusion), including after a host blackout block.
3. The guest can cancel before check-in; the host cannot confirm a cancelled booking.
4. The stay detail screen shows the MapLibre map, photo carousel, amenities and server quote; the booking sheet sends the selected dates/guests and refreshes the guest booking list.
5. Legacy `/properties/:id` and `/properties/:id/book` links redirect to `/listings/stay/:id` and do not call the monolith.

Additional onboarding checks:

1. Unlink the store while a listing is published: expect HTTP 409 (gRPC `Aborted`).
2. Pause/archive the published listings, then unlink: expect success and `merchant_id` empty on `GET /estay/host/profile`.
3. With no store linked, publish a complete draft: expect HTTP 403 with a message asking for an approved vendor store.
4. Link a store that belongs to another account: expect HTTP 403; the other store must not become linked.
5. Set the store to `pending` or `suspended`, or the eStay capability to `pending`/`suspended`, then publish: expect HTTP 403 and no status change.

Known limits: no payment capture/refund, no reviews, no host-guest messaging, and photos are URL-only. The merchant status/capability approval workflow is now also available in `apps/admin`; Phase 9e covers that console and the document-review flow.

### Phase 9d — Merchant registration, approval and the eStay publish gate

Preconditions:

- `merchant-service`, `estay-service` and `gateway` are running.
- `merchant-service` uses `ehub_merchant` (not the owner URL).
- You have two signed-in test accounts: a vendor and an unrelated second user.
- You have an administrator. `/auth/sync` cannot self-assign `admin`; grant it
  operationally, then sign in again so the new session token contains the role:

```powershell
docker exec -it ehub_postgres psql -U user -d ehub_db -c `
  "INSERT INTO user_roles (user_id, role) VALUES ('<ADMIN_USER_ID>', 'admin') ON CONFLICT DO NOTHING;"
```

Vendor flow:

```powershell
$GW = "http://localhost:9010/api/v1/gateway"

curl -X POST "$GW/merchants" `
  -H "Authorization: Bearer $VENDOR_TOKEN" -H "Content-Type: application/json" `
  -d '{"name":"Test Stay Vendor","description":"Local test","phone_number":"+254700000000","email":"vendor@example.test","address":"Nairobi"}'

curl "$GW/merchants/me" -H "Authorization: Bearer $VENDOR_TOKEN"
curl "$GW/merchants/$MERCHANT_ID/manage" -H "Authorization: Bearer $VENDOR_TOKEN"

curl -X POST "$GW/merchants/$MERCHANT_ID/capabilities" `
  -H "Authorization: Bearer $VENDOR_TOKEN" -H "Content-Type: application/json" `
  -d '{"service_type":"estay"}'
```

Admin approval:

```powershell
curl -X PUT "$GW/merchants/$MERCHANT_ID/status" `
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" `
  -d '{"status":"approved","reason":"KYC complete"}'

curl -X PUT "$GW/merchants/$MERCHANT_ID/capabilities/estay/status" `
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" `
  -d '{"status":"active","reason":"eStay listing standards met"}'

curl "$GW/merchants/$MERCHANT_ID/capabilities" -H "Authorization: Bearer $VENDOR_TOKEN"
```

Security and PII checks:

1. With the unrelated second user, call `GET /merchants/:id/manage`,
   `POST /merchants/:id/capabilities`, `PUT /merchants/:id/status` and
   `PUT /merchants/:id/capabilities/estay/status`. Expect `403`/`404`; no
   mutation is allowed.
2. Call public `GET /merchants/:id` without a token. Expect a 200 storefront
   profile with `owner_id`, `phone_number` and `email` omitted (or empty).
3. Call `/auth/sync` with `role: "admin"`, `role: "staff"` or
   `role: "store_staff"` as a normal user. Expect no privileged role in the
   returned session. `host`/`vendor` may be self-assigned but are not
   verification.

Expected:

1. Registration returns a merchant owned by the caller with `verification_status = pending`.
2. `GET /merchants/me` lists only the caller's merchants; `/manage` returns the
   full contact record to the owner.
3. The capability starts `pending`; enrolling `estay` does not by itself allow a
   host to publish. The vendor UI shows `Awaiting approval`, not `Active`.
4. After the admin approval, the capability is `active` with `reviewed_by`,
   `reviewed_at` and the reviewer note; `GET /merchants/:id` still hides PII.
5. Publish with the store linked and approved: succeeds. Set the store to
   `pending`/`suspended`, or the capability to `pending`/`suspended`, then
   publish another complete draft: expect HTTP `403` and no status change.
6. Restore `approved`/`active` to keep the rest of Phase 9c runnable.

Known gaps: store staff invitations/RBAC are covered separately in Phase 9f.
The document/review/admin console flow is covered in Phase 9e; this phase
exercises the older status-first path through the gateway.

Optional automated live smoke test:

With `merchant-service` and `estay-service` running, the gateway handler package
has an opt-in integration test that drives the whole Phase 9d flow through the
real gateway handlers and gRPC clients. It uses the development JWT secret, so
run it only against a disposable local database:

```powershell
cd services\gateway
$env:EHUB_PHASE9D_LIVE = "1"
$env:MERCHANT_GRPC_ADDR = "127.0.0.1:9108"
$env:ESTAY_GRPC_ADDR = "127.0.0.1:9107"
go test ./handlers -run TestPhase9DLive -count=1 -v
```

It covers registration, owner-scoped reads, public PII stripping, linking while
pending, publish rejection before approval, admin approval, successful publish,
the live recheck after suspension, the unlink conflict, and cleanup to paused
and unlinked state. It does not replace the manual `/auth/sync` role checks or
the running gateway `main` process; run those through the gateway as above. It
creates a test merchant, host and listings in the target database.

### Phase 9e — Verification documents and admin console

Preconditions:

- The Phase 9d stack is running, and `merchant-service` uses `ehub_merchant`.
- You have an owner token and an admin token (`admin` or `executive_admin`).
  The admin must sign in again after the role grant so the token carries it.
- `apps/merchant` is configured with `GATEWAY_URL` and, for token-only local
  testing, `ALLOW_DEV_SESSION=true`.
- `apps/admin` has its own `.env` copied from `.env.example`: set `GATEWAY_URL`
  and either `FIREBASE_API_KEY` or `ALLOW_DEV_SESSION=true`.

Owner document flow:

```powershell
$GW = "http://localhost:9010/api/v1/gateway"

# A fresh store with no evidence cannot be submitted.
curl -X POST "$GW/merchants/$MERCHANT_ID/verification/submit" `
  -H "Authorization: Bearer $VENDOR_TOKEN"

curl -X POST "$GW/merchants/$MERCHANT_ID/documents" `
  -H "Authorization: Bearer $VENDOR_TOKEN" -H "Content-Type: application/json" `
  -d '{"type":"business_license","url":"https://example.test/evidence/business-license.pdf"}'

curl -X POST "$GW/merchants/$MERCHANT_ID/documents" `
  -H "Authorization: Bearer $VENDOR_TOKEN" -H "Content-Type: application/json" `
  -d '{"type":"id_proof","url":"https://example.test/evidence/id.pdf"}'

curl "$GW/merchants/$MERCHANT_ID/documents" `
  -H "Authorization: Bearer $VENDOR_TOKEN"

curl -X POST "$GW/merchants/$MERCHANT_ID/verification/submit" `
  -H "Authorization: Bearer $VENDOR_TOKEN"
```

Admin review through the API:

```powershell
curl "$GW/admin/verification/queue?status=under_review&q=Test%20Stay" `
  -H "Authorization: Bearer $ADMIN_TOKEN"

curl "$GW/admin/verification/$MERCHANT_ID" `
  -H "Authorization: Bearer $ADMIN_TOKEN"

curl -X POST "$GW/admin/verification/$MERCHANT_ID/documents/$DOCUMENT_ID/review" `
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" `
  -d '{"status":"verified","notes":"Matches the registered business name"}'

curl -X PUT "$GW/merchants/$MERCHANT_ID/status" `
  -H "Authorization: Bearer $ADMIN_TOKEN" -H "Content-Type: application/json" `
  -d '{"status":"approved","reason":"Documents verified"}'
```

Console flow:

1. Open `apps/merchant`, sign in as the owner, open the store and add a URL
   document. Confirm the list shows its type, status and reviewer note.
2. Submit for review. Confirm the status becomes `under_review` and the submit
   button is no longer offered for that state.
3. Open `apps/admin` on its own port and sign in. A non-admin account must be
   refused before a session cookie is stored.
4. Filter the queue by `under_review` and search by merchant name. Open the
   store review and confirm owner contact details, document links, capabilities
   and verification audit are visible.
5. Verify one document and reject another. Confirm the store status does **not**
   change by itself, and reviewer id/time/notes appear on the document.
6. Return a document to `pending`; confirm `verified_at` is cleared. Approval of
   the store remains a separate status action.
7. Approve the store and the eStay capability, then publish a complete listing;
   the Phase 9c/9d publish gate must still pass.

Expected:

1. `POST /merchants/:id/verification/submit` returns `412` when there is no
   document, and moves `pending`, `documents_requested` or `rejected` to
   `under_review` when at least one exists.
2. Document types are restricted to `id_proof`, `business_license`,
   `tax_certificate`, `permit` and `other`; non-HTTP(S) URLs are rejected with
   `400`.
3. Queue, review-detail and document-decision routes return `403` for an owner
   or ordinary user, at both the gateway and merchant-service.
4. Documents and verification history never appear on public
   `GET /merchants/:id`; a non-owner cannot list another store's documents.
5. Document review changes only `merchant_documents`; the store status changes
   only through `PUT /merchants/:id/status`, which appends an audit row.

Cleanup: the evidence URLs are placeholder records only. There is no object
storage to clean up, but pause/unlink any listings created for the publish
check.

### Phase 9f — Store staff invitations and scoped access

Preconditions:

- The Phase 9d/9e stack is running, `merchant-service` uses `ehub_merchant`,
  and `15d_merchant_staff_rbac.sql` plus the contracts have been applied.
- You have an owner token for an existing store and a separate signed-in user
  (`STAFF_USER_ID`/`STAFF_TOKEN`) who is not the owner and has no global `staff`
  role. `/auth/sync` can issue this ordinary session.
- You have a third `OTHER_TOKEN` for a user who was **not** invited.

Owner invites and reads the roster:

```powershell
$GW = "http://localhost:9010/api/v1/gateway"

# The response includes staff.invitation_token exactly once for sharing.
curl -X POST "$GW/merchants/$MERCHANT_ID/staff" `
  -H "Authorization: Bearer $OWNER_TOKEN" -H "Content-Type: application/json" `
  -d '{"user_id":"<STAFF_USER_ID>","role":"manager"}'

curl "$GW/merchants/$MERCHANT_ID/staff" `
  -H "Authorization: Bearer $OWNER_TOKEN"
```

Invitee accepts and sees the store:

```powershell
curl -X POST "$GW/staff/invitations/accept" `
  -H "Authorization: Bearer $STAFF_TOKEN" -H "Content-Type: application/json" `
  -d '{"token":"<INVITATION_TOKEN>"}'

curl "$GW/merchants/me" -H "Authorization: Bearer $STAFF_TOKEN"
curl "$GW/merchants/$MERCHANT_ID/manage" -H "Authorization: Bearer $STAFF_TOKEN"
```

Owner changes role/status:

```powershell
curl -X PUT "$GW/merchants/$MERCHANT_ID/staff/<STAFF_ID>" `
  -H "Authorization: Bearer $OWNER_TOKEN" -H "Content-Type: application/json" `
  -d '{"role":"cashier","status":"active"}'
```

Security checks:

1. Before acceptance, the invited user's `/merchants/me` must not include the
   store, and `/merchants/:id/manage` must return `403`/`404`.
2. Accept with `OTHER_TOKEN` and the same token: expect `412`; the invitation
   remains outstanding. Accept twice with `STAFF_TOKEN`: the second attempt
   fails with `412`.
3. With `STAFF_TOKEN`, call `POST /merchants/:id/documents`,
   `POST /merchants/:id/verification/submit`, `POST /merchants/:id/staff` and
   `PUT /merchants/:id/staff/:staffId`. Expect `403`/`404`; no mutation occurs.
4. `GET /merchants/:id/manage` with `STAFF_TOKEN` returns `access_role:
   "manager"`, `is_owner: false`, and the scoped permissions; after the owner
   changes the role to `cashier`, the permissions become `view_orders` and
   `approve_pickup`.
5. Set the member to `suspended`; `/merchants/me` and `/manage` must stop
   listing/allowing the store. The row remains for audit, with no invitation
   token.
6. The owner cannot invite their own user id; expect `400`. An active duplicate
   invite returns `409`.
7. Public `GET /merchants/:id` remains PII-sanitized for staff and non-members
   alike.

Expected: staff access is limited to the invited store while active, owners
retain all management rights, and no global role is created by accepting an
invitation. Dispatch's `business_staff` authorization is unchanged; staff roles
do not yet grant access to the dispatch merchant queue until the storefront
mapping exists.

Cleanup: suspend test members rather than deleting rows. The invitation tokens
are development values; rotate any live token by re-inviting the user.

### Phase 10 — Failure and fallback behavior

| Failure | Expected behavior |
|---|---|
| OSRM unavailable | Route preview keeps pins; pricing/ETA uses straight-line 24 km/h fallback |
| Photon unavailable | Autocomplete fails visibly; no hardcoded destination list |
| Redis unavailable | Accept reservations fail open; Postgres remains authoritative; realtime returns `503` |
| Surge lookup fails/times out | Fare prices at multiplier `1.0`; request still succeeds |
| Driver stops location updates | H3 entry expires after 45 s |
| No eligible driver | Rider receives a clear no-driver/`409` response |
| Duplicate accept | One winner, one conflict |
| SSE disconnect | Polling recovers the authoritative state |
| Rate limit exceeded | Gateway returns `429`; wait for the window or raise local limits |
| `EXPO_PUBLIC_MAPLIBRE_STYLE_URL` unset | MapLibre uses the OSM raster fallback (development only) |
| MapLibre style URL invalid/unreachable | Map tiles fail; check URL/network and attribution, then rebuild/restart Metro with `--clear` |
| Estay overlapping dates | Booking is rejected by the service and the GiST exclusion constraint; no second active booking |
| Merchant mutation from a non-owner | Rejected with `403`/`404`; no mutation occurs |
| Document URL is not HTTP(S) or the type is unknown | `400` (`InvalidArgument`); no document row is written |
| Submit verification with no documents | `412` (`FailedPrecondition`); store status is unchanged |
| Document review for an unknown document | `404` (`NotFound`); no store status change |
| Admin review without `admin`/`executive_admin` | Rejected with `403` by both the gateway and merchant-service |
| Staff invite for the owner or a blank user id | `400` (`InvalidArgument`); no staff row |
| Duplicate invite for an active staff member | `409` (`AlreadyExists`); existing membership unchanged |
| Invitation accepted by the wrong user or reused | `412` (`FailedPrecondition`); membership stays `invited` or remains consumed |
| Staff calls an owner-only verification/staff route | `403` (`PermissionDenied`) or `404`; no mutation |
| Suspended staff reads the store | `403`/`404`; the store disappears from `/merchants/me` |
| Estay publish without a linked store | `403` (`PermissionDenied`) with a message to link an approved vendor store; listing stays a draft |
| Estay publish with pending/suspended verification or capability | `403`; listing stays a draft; the live merchant check is used, not a cached snapshot |
| Unlink a store with a published listing | `409` (`Aborted`); pause/archive first |

### Phase 11 — Cleanup

1. Cancel or complete all open test trips.
2. Set test drivers offline.
3. Confirm no test trips remain in an active state.
4. Optionally clear local H3/surge keys:

```powershell
docker exec -it ehub_redis redis-cli --scan --pattern "dispatch:h3:r9:*" |
  ForEach-Object { docker exec -it ehub_redis redis-cli DEL $_ }

docker exec -it ehub_redis redis-cli --scan --pattern "dispatch:surge:r9:*" |
  ForEach-Object { docker exec -it ehub_redis redis-cli DEL $_ }
```

5. For a full local reset, remove volumes and let the init scripts run again:

```powershell
cd infra
docker compose -f docker-compose.yml -f docker-compose.geo.yml down -v
```

Warning: `down -v` deletes the local Postgres and Photon volumes. Use it only when a clean database is intended.

---

## 7. Observability commands

### Service logs

```powershell
cd infra
docker compose -f docker-compose.yml -f docker-compose.geo.yml logs -f gateway dispatch-service
```

### Container status

```powershell
docker compose -f docker-compose.yml -f docker-compose.geo.yml ps
```

### Redis

```powershell
docker exec -it ehub_redis redis-cli ping
docker exec -it ehub_redis redis-cli --scan --pattern "dispatch:h3:r9:*"
docker exec -it ehub_redis redis-cli --scan --pattern "dispatch:surge:r9:*"
docker exec -it ehub_redis redis-cli --scan --pattern "driver:loc:*"
docker exec -it ehub_redis redis-cli GET "driver:loc:<driver-id>"
docker exec -it ehub_redis redis-cli SMEMBERS "dispatch:surge:r9:demand:<cell>"
```

### Postgres

```powershell
docker exec -it ehub_postgres psql -U user -d ehub_db -c `
  "SELECT id, status, driver_id, vehicle_type_id, distance_km, duration_minutes, created_at
   FROM taxi_trips ORDER BY created_at DESC LIMIT 20;"

docker exec -it ehub_postgres psql -U user -d ehub_db -c `
  "SELECT id, user_id, name, status, vehicle_type_id, last_location
   FROM drivers ORDER BY updated_at DESC;"
```

Use the owner/superuser URL only for local inspection and migrations. Each service must keep using its own least-privilege role.

---

## 8. Pass/fail criteria

A test run is successful only when:

- [ ] All pre-flight checks pass.
- [ ] Destination-first ride flow selects a destination before showing tiers.
- [ ] Photon autocomplete returns real suggestions; no hardcoded destinations.
- [ ] OSRM route line renders on both rider and driver maps.
- [ ] Fares and quotes are in KES and server-authoritative.
- [ ] H3 matching respects 1–3 km expansion, tier, status, and 45-second TTL.
- [ ] Surge rises with demand, caps at 1.8, and releases on accept/cancel.
- [ ] Concurrent accept yields exactly one winner.
- [ ] Ride lifecycle transitions and audit/billing records are correct.
- [ ] Driver offer SSE and trip location SSE deliver events; polling recovers after disconnect.
- [ ] eDelivery shows destination first, route polyline, KES tiers, and nearby courier ETAs.
- [ ] MapLibre renders route/stay maps in both apps from a development build; no Google Maps SDK is required.
- [ ] Estay discovery filters and near-me radius work; the server quote matches the booked total.
- [ ] Overlapping Estay bookings are rejected; host confirm/decline and guest cancel work.
- [ ] Merchant registration, owner-scoped management, admin approval and capability review are exercised.
- [ ] An owner can add URL-backed documents, cannot submit without evidence, and the store reaches `under_review`.
- [ ] The admin console lists the queue, shows merchant PII/documents/audit, records document decisions, and refuses non-admin sessions.
- [ ] Document decisions do not change the store status automatically; status approval remains explicit and audited.
- [ ] An owner invites a user, the invitee accepts from their own session, and the store appears in `/merchants/me`.
- [ ] Staff can read the store record/capabilities but cannot manage staff, documents, verification or capabilities.
- [ ] Suspending a staff member removes store access while retaining the row; wrong-user and reused tokens are rejected.
- [ ] A host with no approved/active eStay capability cannot publish; the listing stays a draft and the error is user-visible.
- [ ] Public merchant reads contain no owner/contact PII or documents; owners can still read `/merchants/:id/manage`.
- [ ] `/auth/sync` refuses to self-assign `admin`, `executive_admin`, `staff` or `store_staff`.
- [ ] Failure modes degrade as documented, without corrupting state.
- [ ] No service is called directly by the apps for extracted domains; all traffic goes through the gateway.
- [ ] No test data remains in an active state.

A run is **blocked**, not failed, when a prerequisite is missing (Docker down, Firebase credentials absent, Photon still importing, OSRM graph not built, or devices cannot reach the gateway).

---

## 9. Known limitations and out of scope

- `services/eHubGo` does not compile from an import cycle and is intentionally out of scope. It is also behind the `monolith` Compose profile.
- Consumer-app legacy screens still call the Render monolith through `EXPO_PUBLIC_API_URL`. Ride/delivery flows through the gateway do not.
- Standalone ad-hoc package delivery is not implemented; eDelivery confirms a quote. The food/grocery miniservice previews also use sample items, so their checkout now explains that ordering is not connected rather than creating a delivery without a parent order.
- Delivery route geometry is not persisted; only distance/duration are stored.
- Surge applies to rides, not delivery.
- Pub/Sub has no replay/resume; polling is the recovery path.
- Local Photon uses a Nairobi fixture; destinations outside Nairobi may not autocomplete.
- Public OSM tiles are development-only and require visible attribution. Set `EXPO_PUBLIC_MAPLIBRE_STYLE_URL` to a production vector style for real traffic.
- MapLibre is a native module, so these apps require an Expo development build; Expo Go cannot run them. There is no Google Maps SDK or API key in either app.
- Firebase email/password runtime testing needs real Firebase web/API credentials. Token-only local work can use the Auth emulator with `FIREBASE_AUTH_EMULATOR_HOST=127.0.0.1:9099` and `GCLOUD_PROJECT=ehub-dev`.
- Driver proof-photo testing depends on Firebase Storage bucket/rules configuration.
- Merchant evidence is URL-backed only. Managed uploads (signed URLs, MIME/size checks, malware scanning, retention) are not implemented.
- Store staff invitations/RBAC are available through the gateway and `apps/merchant`. Dispatch's merchant queue still authorizes through `business_staff`; there is no `businesses` ↔ `merchants` mapping yet, so staff membership does not grant dispatch access.
- The workspace has no root git repository; test artifacts and changes are uncommitted.

---

## 10. Troubleshooting

### Devices cannot reach the gateway

- Confirm the PC's LAN IP: `ipconfig`.
- Use `http://<LAN-IP>:9010/api/v1/gateway`, not `localhost`.
- Allow `9010` and `8081` through Windows Firewall.
- Confirm the phone and PC are on the same network and client isolation is off.
- Try USB `adb reverse tcp:9010 tcp:9010` and `http://localhost:9010/api/v1/gateway`.
- Remember Expo `--tunnel` does not tunnel the gateway.

### Gateway returns `429`

The public discovery/auth endpoints are rate-limited per client IP. Wait for the window, reduce the harness rate, or raise `RATE_LIMIT_DISCOVERY_PER_MINUTE` / `RATE_LIMIT_AUTH_PER_MINUTE` for local load tests.

### Driver is online but never matches

- Confirm `drivers.status = 'online'`.
- Confirm a `drivers` row exists for the user.
- Confirm `vehicle_type_id` serves the requested service and has enough capacity.
- Confirm the driver app is publishing locations; Redis TTL is 45 seconds.
- Confirm the pickup and driver are within 3 km.
- Inspect `dispatch:h3:r9:*` and `driver:loc:*` in Redis.

### Photon autocomplete is empty

- Confirm Photon is healthy.
- Confirm the query is a Nairobi place when using the local fixture.
- Confirm `PLACES_PROVIDER=photon` and `PHOTON_URL=http://photon:2322` in the dispatch container.
- First import can take several minutes; wait for health.

### OSRM route is missing

- Confirm `OSRM_URL=http://osrm:5000` in dispatch.
- Confirm the Kenya graph was built and `ehub_osrm` is healthy.
- If OSRM is intentionally absent, expect pins and straight-line 24 km/h estimates, not a polyline.

### MapLibre map is blank or Expo Go fails

- MapLibre is a native module: Expo Go will fail at import or show a missing
  native module. Build/install the development client with
  `bunx expo run:android` / `run:ios` (or `eas build --profile development`).
- Confirm `@maplibre/maplibre-react-native` is in `app.json` `plugins` and you
  rebuilt after changing native config.
- With no `EXPO_PUBLIC_MAPLIBRE_STYLE_URL`, the app should show the OSM raster
  fallback. Confirm the device can reach `EXPO_PUBLIC_OSM_TILE_URL` or your
  self-hosted style/tile host.
- Keep the OSM attribution visible. Public OSM tiles are not production
  infrastructure.
- Restart Metro with `--clear` after changing env values; `EXPO_PUBLIC_*` is
  baked at bundle time.

### Metro cache error

`Unable to deserialize cloned data due to invalid or unsupported version` is usually a stale Metro cache, not app code.

```powershell
Remove-Item -Recurse -Force "$env:TEMP\metro-file-map-*" -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force "$env:TEMP\metro-cache" -ErrorAction SilentlyContinue
```

Restart Expo with `--clear`. Use Node or Bun consistently; do not alternate them for the same cache.

### Port already in use

A native Postgres/Redis on `5432`/`6379` shadows the container. Stop the native service or change the host port and update the connection URLs.

---

## 11. Test report template

Copy this into the test record for each run.

```text
Date:
Tester:
Workspace state / commit (if available):
Backend stack: core / core+geo
OSRM: healthy / unavailable
Photon: healthy / unavailable
Devices:
  Rider:
  Driver(s):
  Emulators:
Test data:
  Rider accounts:
  Driver accounts / tiers:

Results:
  Phase 0 static checks: PASS / FAIL / BLOCKED
  Phase 1 backend smoke: PASS / FAIL / BLOCKED
  Phase 2 auth/authorization: PASS / FAIL / BLOCKED
  Phase 3 destination-first ride: PASS / FAIL / BLOCKED
  Phase 4 H3 discovery: PASS / FAIL / BLOCKED
  Phase 5 surge: PASS / FAIL / BLOCKED
  Phase 6 accept race: PASS / FAIL / BLOCKED
  Phase 7 lifecycle/billing: PASS / FAIL / BLOCKED
  Phase 8 realtime SSE: PASS / FAIL / BLOCKED
  Phase 9 eDelivery: PASS / FAIL / BLOCKED
  Phase 9b catalogue: PASS / FAIL / BLOCKED
  Phase 9c Estay: PASS / FAIL / BLOCKED
  Phase 9d merchant onboarding: PASS / FAIL / BLOCKED
  Phase 10 failure/fallback: PASS / FAIL / BLOCKED
  Phase 11 cleanup: PASS / FAIL / BLOCKED

Failures / evidence:
  - Test case:
    Expected:
    Actual:
    Logs / Redis / SQL evidence:

Blockers:
Follow-ups:
```
