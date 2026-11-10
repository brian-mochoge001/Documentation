# Database ownership contracts

Every extracted service connects to Postgres with its **own login role**, and
that role is granted only the table privileges its code actually uses. Ownership
is therefore enforced by the database, not by convention: a query that reaches
outside a service's contract fails with `permission denied` instead of silently
succeeding.

- The contract itself (roles, search paths, grants): [`contracts.sql`](contracts.sql)
- A verifier that fails if the live grants drift from the contract: [`verify_contracts.sql`](verify_contracts.sql)

## Why

All services currently share one database. Before this change every service
connected as the superuser/table owner, so any service could read or write
*any* table — including another domain's, and including by accident. That made
the service boundaries fictional: the code looked modular, but a stray
`UPDATE products` in the dispatch service would have succeeded.

Contracts make the boundaries real and make ownership explicit, without
requiring a faith-based refactor. They also map cleanly onto the end state: a
role with a search path over its own schema is one `pg_dump -n <schema>` away
from living in its own database.

## Roles

| Service | Role | Schema / `search_path` | Password (dev only) |
|---|---|---|---|
| `auth-service` | `ehub_auth` | `public` | `ehub-dev-auth` |
| `wallet-service` | `ehub_wallet` | `wallet` | `ehub-dev-wallet` |
| `dispatch-service` | `ehub_dispatch` | `public` | `ehub-dev-dispatch` |
| `fulfillment-service` | `ehub_fulfillment` | `public` | `ehub-dev-fulfillment` |
| `marketplace-b2b` | `ehub_b2b` | `public` | `ehub-dev-b2b` |
| `marketplace-c2c` | `ehub_c2c` | `public` | `ehub-dev-c2c` |
| `estay-service` | `ehub_estay` | `estay` | `ehub-dev-estay` |
| `merchant-service` | `ehub_merchant` | `public` | `ehub-dev-merchant` |
| `gateway` | - | no database | - |
| `eHubGo` (monolith) | `user` (owner) | `public` | legacy exception, see below |

The passwords are local development defaults and are **not** secrets. Each
service receives its role through `DATABASE_URL`; see `infra/docker-compose.yml`
for the compose values and the README for host-run values.

## The contract

`W` = write (INSERT/UPDATE), `R` = read. "Monolith" is the legacy co-writer that
is out of scope for this contract.

| Table | Contracted writer | Contracted readers | Notes |
|---|---|---|---|
| `wallet_accounts`, `wallet_transactions`, `wallet_ledger_entries`, `wallet_journal_lines`, `wallet_provider_events` | `wallet-service` | - | **Fully owned.** Schema `wallet`; no other service references these tables. |
| `estay.hosts`, `estay.listings`, `estay.listing_photos`, `estay.listing_amenities`, `estay.calendar_blocks`, `estay.bookings` | `estay-service` | `estay-service` | **Fully owned.** Schema `estay`; no other service reads or writes these tables. Only published listings are exposed publicly, through the Estay API/gateway. |
| `estay.amenities` | seeded by the bootstrap (`001_estay.sql`) | `estay-service` | **Reference data.** The migration seeds the 23 amenity keys/categories as the table owner; the runtime role holds `SELECT`. |
| `merchants`, `merchant_capabilities` | `merchant-service` | `merchant-service` | **Vendor registry and miniservice enrolments.** The runtime role can create/update merchants and enrol capabilities. Verification status is part of `merchants`. |
| `merchant_documents` | `merchant-service` | `merchant-service` | **Verification evidence references.** Owners add URL-backed documents; admins list them and record verified/rejected decisions with notes and reviewer metadata. The service stores the URL only, not the file bytes. |
| `merchant_verification_logs` | `merchant-service` | `merchant-service` | **Append-only audit.** The runtime role inserts one row per status decision and reads the history for the admin review detail. |
| `merchant_staff` | - | - | **Not exposed yet.** The migration creates it, but merchant-service has no staff endpoints, so the runtime role has no grant. |
| `users` | `auth-service` (+ monolith) | `dispatch-service` | `auth-service` owns identity/profile; `dispatch` needs display names. |
| `user_roles` | `auth-service` (+ monolith) | — | Role assignment on Firebase sync. |
| `drivers` | `dispatch-service` (`status`, `rating`, `last_location_at`) (+ monolith) | `dispatch-service` | Dispatch owns availability, the durable last position and refreshes the denormalized `rating` average when a ride/delivery rating is recorded; the monolith creates driver rows. |
| `orders` | `fulfillment-service`, `dispatch-service` (`status`) (+ monolith) | `fulfillment-service`, `dispatch-service` | Two extracted writers, both on `status`: fulfillment drives the state machine, dispatch publishes and tracks deliveries. |
| `deliveries` | `dispatch-service` | `dispatch-service` | **Delivery lifecycle + fares + offers + hand-over.** One row per vendor sub-order; dispatch drives `requested → assigned → picked_up → out_for_delivery → delivered → completed` (or `cancelled`), mirrors the group's `orders.status` in the same transaction, and stores the server-computed `fare_amount` plus the priced leg. `vendor_ready_at` is the merchant's packed-basket stamp; `PickUp` refuses until it is set. `expires_at` bounds how long the offer stays in the feed. `picked_up` is the courier holding the goods with a pickup photo; a merchant member's release moves it to `out_for_delivery`; `delivered` is the courier's photo-backed soft completion, and `completed` is the customer's confirmation. **Fully owned** — no other service reads or writes it. |
| `delivery_declines` | `dispatch-service` | `dispatch-service` | **Offer telemetry.** One row per (delivery, courier) that passed, with an optional reason. Insert/read only for dispatch; rows cascade with their delivery. **Fully owned.** |
| `delivery_proofs` | `dispatch-service` | `dispatch-service` | **Hand-over evidence.** Append-only pickup/dropoff photos per delivery, with the courier who took them and the merchant member who verified or rejected a pickup photo. One pending proof per (delivery, kind); a retake supersedes it. Stores the photo URL, not the bytes. **Fully owned.** |
| `service_ratings` | `dispatch-service` | `dispatch-service` | **Ride/delivery reputation.** One row per author per finished job: a 1-5 score, a `complaint`/`neutral`/`compliment` kind and a note. Either party rates the other; a ride must be `completed`, a delivery `delivered`/`completed`. Re-rating upserts the row. A driver rating also refreshes `drivers.rating` (a denormalized cache) in the same transaction. **Fully owned.** |
| `taxi_trips` | `dispatch-service` (+ monolith) | `dispatch-service` | **Ride lifecycle.** Dispatch creates trips and drives `requested → accepted → in_transit → completed` (or `cancelled`); the monolith still serves the legacy taxi routes. `driver_id` now stores `drivers.id`; the migration normalises legacy rows that held a user id. The offer clock (`expires_at`), cancellation attribution and the priced `distance_km`/`duration_minutes` live here. |
| `ride_declines` | `dispatch-service` | `dispatch-service` | **Ride offer telemetry.** One row per (trip, driver) that passed, with an optional reason. Insert/read only; rows cascade with their trip. **Fully owned.** |
| `addresses`, `businesses`, `business_staff`, `order_items`, `users` | monolith | `dispatch-service` | Read-only for dispatch. `business_staff` is read solely to authorize a merchant member (owner or active manager/cashier) for readiness, release and proof rejection; the merchant-service will own it later. |
| `vehicle_types` | seeded by the bootstrap (`003_vehicle_tiers.sql`) (+ monolith) | `dispatch-service` | **Vehicle taxonomy.** The monolith's table defined only a name and a seat count, so a rider's choice could not be matched. The bootstrap adds the class/cargo/services columns and the tier rows as the table owner; dispatch only reads them, so its grant is unchanged. |
| `products`, `categories`, `brands`, `product_models`, `businesses` | monolith (+ `catalog_refresh_product_pricing`) | `marketplace-b2b` | B2B is a read-only view over the catalogue and reads `businesses` for the seller name. The scheduled-promotion refresh is a `SECURITY DEFINER` function owned by the migration role; B2B holds only `SELECT` on those tables plus `EXECUTE` on that function, and the function rewrites `products.price`/`original_price` so the monolith cart charges the effective sale price. |
| `c2c_sellers`, `c2c_listings` | `marketplace-c2c` (+ monolith) | `marketplace-c2c` | C2C owns seller/listing content. |

Only `wallet_*`, dispatch's `deliveries`/`delivery_*`/`ride_declines`/
`service_ratings` tables, and the `estay` domain have a single writer today.
Everything else is still written by the monolith as well - that is the
remaining coupling, and it is tracked below rather than hidden.

## Verifying

```powershell
docker cp infra/db/verify_contracts.sql ehub_postgres:/tmp/verify_contracts.sql
docker exec ehub_postgres psql -U user -d ehub_db -v ON_ERROR_STOP=1 -f /tmp/verify_contracts.sql
```

The script compares the granted privileges (from `pg_class.relacl`) against the
declared contract and raises an exception on any mismatch, in both directions:

- **MISSING** — the contract promises a privilege the role does not hold.
- **EXTRA** — the role holds a privilege the contract does not declare.

It prints the effective matrix on success. Run it after changing any grant.

## Mock connections

[`mock-connections.env.example`](mock-connections.env.example) contains the
local development URLs for every role. [`demo_connections.ps1`](demo_connections.ps1)
prints the same set without exposing passwords and can prove the boundary:

```powershell
./infra/db/demo_connections.ps1
./infra/db/demo_connections.ps1 -Verify
```

`-Verify` connects as each role and prints its `search_path` plus table
privileges. Expected result: `ehub_auth` sees `users`, `dispatch` sees
`users`/`orders`, `wallet` sees only `wallet`, `estay` sees only `estay`, and the
marketplace roles see only their catalogue tables. The gateway has no
`DATABASE_URL`, and the owner URL is used only by migrations and the legacy
monolith.

For a remote target, pass the direct host and rotated passwords:

```powershell
./infra/db/demo_connections.ps1 -Target neon `
  -HostName ep-xxxx.c-3.eu-central-1.aws.neon.tech `
  -Database neondb `
  -Passwords @{ neondb_owner = '...'; ehub_auth = '...' }
```

## Applying to an existing database

`contracts.sql` is idempotent. On a fresh database it runs automatically as
`16_contracts.sql` during Postgres initialisation, after
`07_delivery_lifecycle.sql` creates the `deliveries` table it grants,
`08_delivery_fare_and_readiness.sql` adds its fare and readiness columns,
`09_delivery_offers.sql` adds the offer clock and `delivery_declines`,
`10_delivery_proofs.sql` adds the hand-over states and `delivery_proofs`,
`11_service_ratings.sql` adds `service_ratings`,
`12_ride_hardening.sql` adds the ride offer clock, cancellation columns and
`ride_declines`, `13_estay.sql` creates the `estay` schema and tables,
`13b_estay_host_merchant.sql` adds the host-to-merchant link,
`14_marketplace_catalog.sql` adds catalogue/promotions,
`15_merchant.sql` creates the merchant registry,
`15b_merchant_capability_review.sql` adds the capability review metadata, and
`15c_merchant_document_review.sql` adds the document review metadata and queue indexes.
For a database that already has data, apply the new migration first and then the
contracts:

```powershell
docker cp infra/db/contracts.sql ehub_postgres:/tmp/contracts.sql
docker exec ehub_postgres psql -U user -d ehub_db -v ON_ERROR_STOP=1 -f /tmp/contracts.sql
```

It creates the roles, sets their search paths and grants. If a database still
predates the `wallet` schema, it also relocates `wallet_*` from `public` into
`wallet` (the tables, indexes, foreign keys and the balance-check trigger move
with the table).

## Changing a contract

1. Change the service code first.
2. Add or remove the matching privileges in `contracts.sql`.
3. Mirror the change in the expected matrix in `verify_contracts.sql`.
4. Update the table above.
5. Re-apply `contracts.sql` and run `verify_contracts.sql`.

Steps 2 and 3 are deliberately separate copies: a change that does not update
both fails verification.

## Known deviations and limits

- **The monolith is outside the contract.** `eHubGo` connects as the table owner
  (`user`) and is not modified, so it retains unrestricted access. It is the
  reason several tables have two writers. This resolves as verticals move out of
  `eHubGo`; it cannot be fixed earlier.
- **RLS is not enforced yet for the services.** Every service role is granted
  `BYPASSRLS`. The monolith's row policies filter on session context
  (`get_app_user_id()` / `has_role()`) that the extracted services do not set,
  so enforcing RLS today would silently hide rows from them. Table-level
  ownership is enforced regardless. Dropping `BYPASSRLS` is the next security
  step; `marketplace-c2c` is the natural first candidate because it already
  propagates `app.current_user_id`. `dispatch-service`'s ride operations now do
  the same (`services/rides.go` sets `app.current_user_id` and
  `app.current_user_roles` on every transaction), so `taxi_trips` is a
  candidate too — once its reads stop joining tables whose policies the driver
  cannot satisfy.
- **`taxi_trips.driver_id` means `drivers.id`.** eHubGo's legacy accept path
  wrote the driver's *user* id into that column, which references
  `drivers(id)`. `migrations/002_ride_lifecycle.sql` normalises the existing rows
  and every reader still tolerates the old value, because a user id and a row id
  cannot be told apart in a single row.
- **`orders.status` has two extracted writers, and the dispatch one is not
  RLS-safe.** `fulfillment-service` drives the fulfillment state machine
  (`AssignDriver` → `assigned`, `UpdateOrderStatus`), and `dispatch-service` owns
  the delivery lifecycle in `deliveries` and mirrors each step onto the matching
  sub-order (`requested`, `assigned`, `in_transit` for both picked_up and
  out_for_delivery, `delivered` for delivered and completed, or back to
  `pending` when a delivery is cancelled before pickup) so the customer's order
  view keeps working. The overlap is narrow — dispatch writes `status` only — but
  note the courier transitions run under a **driver**-scoped context
  (`services/deliveries.go` sets `app.current_user_roles = 'driver'`), and the
  `orders` RLS policy covers the customer, the business owner and staff but
  **not drivers**. Because every service role currently has `BYPASSRLS` this
  works; dropping `BYPASSRLS` on `ehub_dispatch` would break the mirror. Making
  the mirror a `SECURITY DEFINER` function, or reading `deliveries` directly in
  the customer's order view, is the fix to apply before that step. The
  background reaper (`DeliveryService.ExpireStale`) writes the same mirror
  without any session context at all, so it is on the same list.
- **`deliveries` (with `delivery_declines`, `delivery_proofs`, `ride_declines`
  and `service_ratings`) is dispatch's fully-owned domain, alongside wallet's
  `wallet_*` tables and estay's `estay` schema.**
  Nothing outside `dispatch-service` reads or writes those tables, so they are
  the natural first candidates for a schema of their own when dispatch is split
  into its own database. `service_ratings` stores the id of the job it rates
  without a foreign key (the target table depends on `service`), so it survives
  the job's own retention.
- **Cross-domain reads are still SQL joins.** `dispatch-service` reads orders,
  addresses, businesses and users in single statements. Replacing those with API
  calls or read models is a separate workstream; the contract makes the
  dependency visible but does not remove it.
- **`vehicle_types` is reference data, written by the bootstrap.** Dispatch reads
  it and matches on the columns `003_vehicle_tiers.sql` adds, but does not own the
  rows: the migration runs as the table owner during initialisation, and the
  dispatch runtime role keeps `SELECT`. A deployment that changes a tier's class
  or capacity therefore ships SQL, not a service call — acceptable while the
  catalogue is small, and the point at which it stops being acceptable is when
  operators need to edit tiers without a deploy.
- **One database, one instance.** Only wallet and estay have schemas of their
  own. Splitting further requires the monolith to release co-owned tables first.
