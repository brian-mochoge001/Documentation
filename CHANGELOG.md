# Changelog

All notable changes to the **eHub Ecosystem** monorepo are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project does not yet follow semantic versioning per release; entries are
grouped by delivery.

> **Note on `services/eHubGo`:** it is the legacy monolith used as a migration
> reference. No changes were made to it in any of the work below.

---

## [Unreleased] - 2026-11-09 - Merchant staff invitations and store RBAC

Store owners can now invite an existing eHub user to one merchant with a
scoped role. The invitee accepts a one-time token from their own session, and
the store appears in their `/merchants/me` list. Staff can read the private
store record and capabilities; only the owner can manage staff, verification,
capabilities or the store profile. The permissions are published for the future
merchant order dashboard, while dispatch keeps its current `business_staff`
authorization until a `businesses` ↔ `merchants` mapping exists.

- **Proto** - `InviteMerchantStaff`, `ListMerchantStaff`,
  `UpdateMerchantStaff` and `AcceptMerchantStaffInvitation`; `StaffMember`
  carries role/status/acceptance/permissions and `GetManagedMerchantResponse`
  now returns `access_role`, `is_owner` and `permissions`. Merchant timestamps
  were also added to the wire contract.
- **merchant-service** - `services/staff.go` implements owner-scoped invites,
  roster reads, role/status updates, token acceptance, `GetMerchantForMember`
  and `GetMerchantsForUser`. `manager` gets `view_orders`, `manage_orders` and
  `approve_pickup`; `cashier` gets `view_orders` and `approve_pickup`;
  `delivery` is read-only. Invitations are one-time bound to the invited user;
  suspending clears the token; staff rows are never hard-deleted.
- **Gateway** - owner routes `POST`/`GET /merchants/:id/staff`,
  `PUT /merchants/:id/staff/:staffId` and the invited-user route
  `POST /staff/invitations/accept`. `GET /merchants/:id/manage` is now available
  to active staff and returns their access role.
- **Database** - `merchant-service/migrations/004_staff_rbac.sql` adds
  `status`, `accepted_at`, lifecycle indexes and a status check; it is mounted
  as `15d_merchant_staff_rbac.sql` and added to `infra/db/push_to_neon.ps1`.
  `ehub_merchant` receives `SELECT, INSERT, UPDATE` on `merchant_staff`;
  `verify_contracts.sql` now expects 93 rows.
- **Merchant app** - the store page shows staff role, adds owner-only invite and
  roster controls, hides verification/management controls from staff, and adds
  `/invitations/accept` for the one-time token. The nav exposes the acceptance
  page.
- **Consumer app** - removed the unresolved `createDispatchDeliveryRequest`
  calls from the food and grocery previews. Those screens still use sample items
  with synthetic ids, so they now say ordering is not connected instead of
  pretending a courier was booked without a checkout-backed parent order.
  `apps/ehub` typechecks cleanly.
- **Validation** - `go test ./...`/`go vet ./...` pass in merchant-service;
  gateway handler tests cover staff invite/list/update and invitation
  acceptance; `buf lint` passes; `apps/merchant` `bun run check` and
  `bun run build` pass; `apps/ehub` `bunx tsc --noEmit` passes.
- **Deferred** - applying `15d` to a live database, rerunning
  `verify_contracts.sql` for the expected 93 rows, the Compose smoke test and
  manual staff/invite checks remain part of the deferred manual validation
  phase.

---

## [Planned] — Not yet implemented

A standing list of expected work that is designed or partially built but not yet
shipped. It is not a promise of order.

### Apps and merchant operations

- **Merchant app, remaining miniservices** (`apps/merchant`, SvelteKit): the
  Estay host workspace, verification-document submission and staff
  invitations/RBAC are shipped; still planned are a multi-business switcher,
  miniservice enrolment polish, and the live order dashboard. Start with polling
  `GET /dispatch/merchant/deliveries?statuses=picked_up`, then move to SSE.
- **`merchant-service` hardening**: ownership/admin authorization, URL-backed
  document review and staff invitations/RBAC are now in place. Still planned:
  managed uploads (signed URLs, MIME/size checks, retention) and `order_events`
  (sales attribution). Dispatch's `business_staff` read should become an RPC
  after a business-to-merchant mapping exists, and the global `staff` role
  should stop being used for merchant access.
- **Dispatch staff authorization** - bridge `businesses` to `merchants` and
  authorize readiness/release/proof rejection through merchant-service staff
  roles instead of the monolith's `business_staff` and global `staff` role.
- **Cashier release screen**: show the pickup photo and the order, then release
  or reject. The backend endpoints exist; the UI does not.
- **Proof upload hardening**: the driver app now captures pickup/dropoff photos and uploads them to Firebase Storage, but there is no server-issued signed upload URL, retake screen or retention policy. The cashier/pickup queue still has no UI.
- **Ratings UI**: the backend records ride/delivery ratings but neither app prompts for one yet. Planned: a post-trip card, driver/courier reputation on match cards, and a merchant view of delivery complaints.
- **Driver contact details**: the active-task feed exposes the customer name but not a phone number or recipient instructions. Planned: add the ride passenger's phone and the delivery recipient's name/phone (from the shipping address) to `DriverTask`, with a call action and an explicit privacy policy; do not leak full order history.

### Estay and accommodation

- **Booking payments and refunds**: bookings currently record a server-priced
  total but no money moves. Planned: charge on confirmation through
  `wallet-service`/Daraja, hold the deposit, refund on cancellation under a
  published policy, and show the payment state on both host and guest sides.
- **Guest reviews and host replies**: `listings.rating`/`review_count` exist, but
  no review is written yet. Planned: one review per completed stay, host
  response, aggregate refresh, and moderation.
- **Messaging and notifications**: host-guest threads, booking-request,
  confirmation, cancellation and check-in reminders over push/email/SMS.
- **Photo upload pipeline**: listings accept absolute URLs only. Planned:
  server-issued signed uploads, image validation/resizing, ordering/alt text,
  and a retention policy.
- **PostGIS radius index**: public discovery now supports lat/lng radius and
  distance ranking with a Haversine calculation in Go; add a generated
  `geography` column and GiST index when listing volume outgrows the scan.
- **Calendar depth**: iCal import/export, per-date pricing and minimum-stay
  overrides, seasonal/weekend rates, and a multi-listing calendar view.
- **Trust and safety**: guest identity verification, damage deposits, host
  verification badges, complaint escalation, and cancellation-fee enforcement.
- **Host payouts and tax**: payout ledger, service-fee invoicing, and Kenya
  tourism-levy handling.
- **Verified vendor onboarding follow-ups**: the host-to-merchant link, live
  publish gate, URL-backed document review, admin verification console and
  store staff invitations/RBAC are shipped. Still planned: managed uploads and
  the dispatch staff-authorization bridge.

### Notifications and live updates

- **Push/email/SMS for the confirm prompt** — a delivered order prompts the
  customer to confirm. No notification provider is wired in yet.
- **Merchant order-queue SSE** — dispatch offers and driver positions already use
  gateway SSE; the merchant dashboard still polls and has no live queue.
- **Push notifications** (FCM/APNs) for offers, releases, rejections and
  confirmations.
- **Platform-owned object storage / upload service** for proof photos and rating
  attachments. The driver app now uploads to Firebase Storage and sends the URL;
  there is no signed-upload endpoint, virus/content check or retention policy.

### Ratings and trust

- **Complaint escalation/moderation queue** — a low score or `kind=complaint`
  currently just sits in `service_ratings`. Planned: a review queue, responses
  from the rated party, and policy actions.
- **Rating reasons/tags** (late, rude, damaged, careful, punctual) and an
  optional photo attachment per rating.
- **Abuse controls** — rate limiting, spam checks and the ability to hide a
  rating pending review.
- **Ratings in matching** — surface `RatingSummary` on match/offer cards and
  factor it into ranking; today it is a display-only lookup.

### Dispatch and pricing

- **Fare negotiation** — bounded name-your-fare, now that the server is the fare authority.
- **Ride fare breakdown in the app** — show the server quote and its components before booking; today the app only renders the returned total.
- **Surge controls and visibility** — the bounded H3 multiplier is live, but
  there is no per-zone cap override, admin view or rider-facing surge badge yet.
- **Realtime stream hardening** — offers and driver positions are pushed over
  gateway SSE, but Pub/Sub has no replay/resume, status changes are not pushed,
  and the consumer app still polls the legacy location endpoint.
- **Push notifications** — FCM/APNs for offers when the app is backgrounded;
  SSE only reaches a foreground connection.
- **Multi-vendor batching** — one courier collecting several sub-orders from the
  same merchant or nearby merchants.
- **Courier earnings via `wallet-service`** — no payout/earnings ledger for
  completed deliveries yet.
- **Product `weight_kg` and cart-summed load** — blocked by the no-monolith rule;
  `products`/`order_items` live in `services/eHubGo`.
- **Reconcile `fulfillment-service` order-status writes** with dispatch's mirror.
- **Delivery ETA in the courier feed** (only distance is shown today).

### Hardening

- **RLS follow-through** — every service role keeps `BYPASSRLS` deliberately.
  Dropping it and relying on the policies is a future security step; the
  `orders.status` mirror from dispatch is the main gap.
- **Retention/deletion policy** for proofs, ratings and finished deliveries.
- **TypeScript/lint baseline**: `apps/ehub` still has two pre-existing
  food/grocery ad-hoc delivery errors (`createDispatchDeliveryRequest`); they
  are unrelated to the Estay/MapLibre work but must be fixed before making the
  typecheck a required CI gate. `apps/dispatch` now typechecks and lints
  cleanly, including the MapLibre route map.

---

## [Unreleased] - 2026-11-09 - Merchant verification documents and admin review console

Merchant verification is now a complete owner/admin workflow. Owners attach
URL-backed identity and business evidence, submit their store for review, and
admins work a verification queue with per-document decisions, status history and
capability approval. Verification decisions remain separate from document
decisions, so a rejected document cannot silently approve a store.

### Added

- **Verification document API** - `AddMerchantDocument`,
  `ListMerchantDocuments`, `SubmitVerification`, `ReviewMerchantDocument`,
  `ListVerificationQueue` and `GetVerificationReview` in
  `proto/merchant/merchant.proto`, with matching direct HTTP and gateway routes.
  Supported types are `id_proof`, `business_license`, `tax_certificate`,
  `permit` and `other`; only absolute HTTP(S) URLs are accepted.
- **Owner submission flow** - `POST /merchants/:id/verification/submit` moves
  `pending`, `documents_requested` or `rejected` to `under_review` only when at
  least one document exists, and appends an audit row.
- **Admin review queue** - filterable queue with document counts and last
  evidence time, plus a review detail containing merchant PII, documents,
  capabilities and verification history. All routes require `admin` or
  `executive_admin`.
- **Admin console** - `apps/admin` is now a functional SvelteKit BFF with its own
  `ehub_admin_session` cookie, Firebase/dev-token sign-in, role enforcement
  before the cookie is stored, a verification dashboard, queue, document
  decisions, merchant status updates, capability approval and audit history.
- **Merchant app verification UI** - the merchant detail page lists evidence,
  accepts document URLs, shows reviewer notes, and submits the store for review.
- **Database** - `merchant-service/migrations/003_document_review.sql` adds
  `reviewed_by`, `updated_at` and queue indexes. It is mounted as
  `15c_merchant_document_review.sql` and added to `infra/db/push_to_neon.ps1`.
- **Contracts** - `ehub_merchant` gains `SELECT`/`INSERT`/`UPDATE` on
  `merchant_documents` and `SELECT` on `merchant_verification_logs`;
  `verify_contracts.sql` expects the four additional grants.

### Changed

- `GetVerificationReview` is the only read that exposes merchant PII and
  verification documents; the public `GET /merchants/:id` remains sanitized and
  has no document route.
- Document review records the reviewer, note, verification time and update time,
  but does not change the merchant's store-level status automatically.
- The admin console uses `/auth/sync` with the ordinary `user` role and checks
  the returned role set, so privileged roles still cannot be self-assigned.

### Validation

- `go test ./...` passes in `merchant-service`; `go test ./handlers` passes in
  `gateway`, including new tests for document upload forwarding,
  admin-only document review and the admin-only verification queue.
- `buf generate` and `buf lint` pass.
- `apps/admin` and `apps/merchant` `bun run check` report 0 errors and 0 warnings.
- `apps/admin` and `apps/merchant` production builds pass.
- Full Compose startup, migration/seed execution, contract verification and the
  live gateway smoke test remain deferred to the manual test phase.

### Known limits

- Evidence is URL-backed; signed uploads, MIME/size validation, malware scanning
  and retention are still pending. The service stores the URL only.
- Staff invitations/RBAC and merchant order events remain out of scope.

---

## [Unreleased] - 2026-11-07 - Vendor onboarding and the eStay publish gate

A host can no longer publish a stay just by being signed in. Estay now links a
host workspace to one owned merchant store, and publication re-checks the live
merchant verification and eStay capability over gRPC. Merchant mutations are
owner-scoped, verification/capability decisions are admin-only, and the public
merchant read no longer exposes contact PII.

### Added

- **Estay host-to-merchant link** -
  `PUT /estay/host/merchant` (direct HTTP and gateway) and
  `LinkHostMerchant` in `proto/estay`. Linking is allowed while the store is
  pending; unlinking is refused while any listing is published.
- **Live publish gate** - `SetListingStatus(published)` calls the merchant
  registry through `MerchantDirectory` and requires store ownership,
  `verification_status = approved`, `is_active = true`, and an `estay`
  capability with `status = active`. `ErrNotApproved` maps to gRPC
  `PermissionDenied` and HTTP 403 with a user-readable reason. A missing
  registry fails closed.
- **Estay config** - `MERCHANT_GRPC_ADDR` and `MERCHANT_TIMEOUT_SECONDS`.
  Production startup requires the registry address.
- **Merchant-service owner/admin API** - `GetManagedMerchant`, owner-scoped
  `GetMerchantForOwner`, owner-scoped capability enrollment/listing, admin-only
  `UpdateCapabilityStatus`, and verification-status validation. Capabilities now
  carry `reviewed_by`, `reviewed_at` and `review_note`.
- **Gateway routes** - `GET /merchants/:id/manage` for owners,
  `PUT /merchants/:id/capabilities/:service/status` for admins, admin gating on
  `PUT /merchants/:id/status`, and `PUT /estay/host/merchant`.
- **Merchant app** - the host profile page links a store and shows
  verification/eStay capability state; the merchant detail page shows
  `pending`/`active`/`suspended` capability states and reviewer notes; the
  listing draft callout explains the publish requirement.
- **Mobile host hint** - `apps/ehub` shows a "vendor store not linked" banner
  and understands `merchant_id` on the host profile.
- **Database** - `estay-service/migrations/002_host_merchant.sql` and
  `merchant-service/migrations/002_capability_review.sql`, mounted as
  `13b_estay_host_merchant.sql` and `15b_merchant_capability_review.sql`, and
  added to `infra/db/push_to_neon.ps1`.

### Changed

- `/auth/sync` now self-assigns only `user`, `customer`, `vendor`, `driver`,
  `host` and `c2c_seller`. `admin`, `executive_admin`, `staff` and
  `store_staff` must be granted operationally. Requested roles are normalized
  before insertion.
- The public `GET /merchants/:id` response is sanitized at the gateway:
  `owner_id`, `phone_number` and `email` are cleared. Owners use
  `/merchants/:id/manage`.
- Merchant gRPC `GetMerchant` is the only public merchant RPC; the gateway's
  public storefront route is allow-listed in the interceptor.
- `packages/platform/auth` now stores the raw session token alongside the
  claims (`WithSessionToken` / `SessionTokenFromContext`) for HTTP and gRPC, so
  Estay can forward the caller's identity to merchant-service.
- Compose starts Estay after merchant-service and passes the registry address.

### Validation

- `go test ./...` and `go vet ./...` pass in `auth-service`, `estay-service`,
  `merchant-service`, `gateway` and `packages/platform`.
- New tests cover the live merchant/capability decision, the gateway's public
  PII stripping, and the admin-only capability approval route.
- `buf lint` passes after splitting `GetManagedMerchantRequest`/`Response` from
  the public `GetMerchant` messages.
- `apps/merchant` `bun run check` reports 0 errors and 0 warnings.
- `apps/ehub` `tsc --noEmit` still reports only the two known pre-existing
  food/grocery `createDispatchDeliveryRequest` errors.
- The opt-in gateway live test (`EHUB_PHASE9D_LIVE=1`) passed against isolated
  local PostgreSQL with `merchant-service` and `estay-service`: registration,
  owner-scoped reads, public PII stripping, linking while pending, 403 publish
  before approval, admin approval, successful publish, the live suspension
  recheck, the 409 unlink conflict, and paused/unlinked cleanup. The test is
  documented in `TESTING.md`.
- Docker startup, the full Compose migration/seed run, contract verification
  and the running `gateway` `main` smoke test remain deferred to the manual test
  phase.

---

## [Unreleased] - 2026-11-05 - Estay guest flow, MapLibre maps and merchant registry

Estay now has a complete consumer path, and both mobile apps render maps with
MapLibre instead of `react-native-maps`, removing the Google Maps SDK/API-key
dependency. The vendor registry is also present behind the gateway; its
ownership/admin and public-PII gaps were addressed in the 2026-11-07 entry.

### Added

- **Estay discovery filters** — the service and gateway support destination/host
  text search, `host_id`, amenities, bedrooms/beds/bathrooms, guests, price,
  property type, paired dates, instant-book, `lat`/`lng` + `radius_km`, and
  `sort=distance|price_asc|price_desc|rating`. Nearby results are filtered and
  ordered with Haversine distance.
- **Public booking quote** — `GET /estay/listings/:id/quote?check_in=&check_out=&guests=`
  returns the same server-priced KES total used by booking creation. The Estay
  gRPC quote method is exempted from the session interceptor so the gateway can
  serve it before sign-in.
- **Consumer Estay screens** — `apps/ehub` now has gateway-backed discovery
  (`ehost.tsx`), stay detail with photo carousel, amenities, MapLibre map,
  availability, quote and booking (`listings/stay/[id].tsx`), plus host profile,
  listing creation, calendar and booking-request screens (`ehost-*.tsx`).
- **Reusable Estay components** — `StayCard`, `StaySearchSheet`,
  `StayBookingSheet`, `StayMap` and shared formatting/search utilities.
- **Legacy stay redirects** — `/properties/:id` and `/properties/:id/book` now
  redirect to `/listings/stay/:id`; merchant stay links use the same route.
- **MapLibre on mobile** — both `apps/ehub` and `apps/dispatch` use
  `@maplibre/maplibre-react-native` with declarative GeoJSON sources/layers,
  markers and camera fitting. There is no `react-native-maps` dependency and no
  Google Maps key.
- **Shared OSM fallback style** — `EXPO_PUBLIC_MAPLIBRE_STYLE_URL` can point at a
  self-hosted/vector style; otherwise an OSM raster style is used, with
  `EXPO_PUBLIC_OSM_TILE_URL` overriding the tile template.
- **Web map fallbacks** — `TaxiMap.web.tsx`, `StayMap.web.tsx` and the dispatch
  web route map render a location/route summary instead of importing the native
  renderer.
- **Merchant registry service** — `merchant-service` owns merchants,
  capabilities and verification logs; the gateway exposes register/list-my/
  capability/status routes.
- **Merchant database contract** — `ehub_merchant` gets `ehub-dev-merchant`, a
  `public` search path, and least-privilege grants on merchants/capabilities/logs.
  `infra/db/contracts.sql`, `verify_contracts.sql`, `OWNERSHIP.md` and compose
  use the role.

### Changed

- `infra/docker-compose.yml` adds `merchant-service` on `9008/9108`, grants the
  merchant DB role, and gives the gateway `MERCHANT_GRPC_ADDR` plus a
  merchant-service health dependency.
- `merchant-service` now has a checked-in `go.sum` and builds cleanly; an unused
  `database/sql` import was removed.
- `infra/db/push_to_neon.ps1` now applies the catalogue and merchant migrations
  before contracts and prints the merchant role.
- The gateway Estay route surface adds the public quote endpoint and the new
  listing filters; the merchant routes are registered under the authenticated
  group (except `GET /merchants/:id`).
- `apps/merchant` stay items now resolve through Estay `searchStayListings` by
  owner ID/name instead of the legacy `getBusinessProperties`.
- Estay's README documents its filters, quote, schema and runtime role.
- `Architecture.md` now covers the platform context, request/auth flows, service
  ownership, H3/PostGIS/OSRM/Photon, the Estay lifecycle, vendor onboarding and
  the MapLibre decision. `DEPLOYMENT.md` now lists the full 16-step migration
  order, all eight runtime roles, `9008/9108` for merchant-service, and
  Photon/OSRM instead of Google Places for the default geo stack.
- `apps/dispatch` typecheck cleanup: the React Native color-scheme fallback,
  web tab `ref` cast, and Firebase v12 auth persistence initialization are fixed.
  The driver session JWT remains persisted in SecureStore, so Firebase only needs
  to live for the sign-in exchange.

### Security and known gaps (closed on 2026-11-07 unless noted)

- `/auth/sync` no longer self-assigns `admin`, `executive_admin`, `staff` or
  `store_staff`; the 2026-11-07 entry records the restriction.
- Merchant capability/status routes are now owner-scoped and admin-gated, and
  the public `GET /merchants/:id` response is sanitized by the gateway.
- Estay now enforces a merchant link and an approved/active eStay capability on
  publish, using a live merchant-service check.
- Still open: `merchant_documents` and `merchant_staff` have no endpoints or
  runtime grants yet.

### Tests

- `go test ./...` passes in `services/estay-service`, `services/gateway` and
  `services/merchant-service`; `buf lint` passes.
- `apps/ehub` typecheck still has two pre-existing food/grocery ad-hoc delivery
  errors outside the Estay/MapLibre files; everything changed here typechecks.
  `apps/dispatch` typechecks and lints cleanly, including the MapLibre
  `route-map.tsx` and the Firebase auth cleanup.
- Docker startup, migrations, seed data, contract verification and live gateway
  smoke tests remain deferred to manual testing.

---

## [Unreleased] - 2026-10-31 - In-house catalogue: deep taxonomy and scheduled promotions

The consumer storefront only had flat categories, a hardcoded flash-sale countdown,
and a `products.is_flash_sale` boolean with no real sale window. This ships a
catalogue the app can browse from category to brand to model, with normal deals
and flash sales that start and end on a real schedule.

### Added

- **Catalogue migration** — `services/marketplace-b2b/migrations/001_catalog_taxonomy_and_promotions.sql` adds arbitrary-depth categories (`sort_order`, `is_active`, parent-scoped name uniqueness), brand/model visibility flags, product `original_price`/`is_active`/`is_on_sale`/discount-window columns, KES currency normalisation, `product_discounts.is_flash_sale`/`label`, a category-path helper, and the `catalog_refresh_product_pricing` function. Compose mounts it as `14_marketplace_catalog.sql` before `15_contracts.sql`.
- **Effective pricing** — the refresh function selects the strongest active promotion (flash first, then the largest discount), writes the effective price back to `products.price`, keeps the list price in `products.original_price`, and stamps `discount_start_at`/`discount_end_at`. The legacy cart/checkout therefore charges the same price the storefront shows.
- **Scheduled refresh** — `marketplace-b2b` refreshes pricing on startup, every minute, and opportunistically before catalogue reads, so a promotion starts or ends without waiting for an operator action.
- **Gateway catalogue API** — `GET /marketplace/products` now accepts `category_id`, `include_descendants`, `brand_id`, `model_id`, `q`/`search`, `on_sale`, `normal_sale`, `flash_sale`, `featured`, `min_price`, `max_price` and `sort`; new public `GET /marketplace/brands` and `GET /marketplace/models` routes expose brand/model navigation.
- **Deep catalogue UI** — home, catalogue, category, all-categories, flash-sale, product-detail and search screens now use category IDs, render the full tree, and expose brand/model/price/promotion/sort filters. Product cards show effective price, original price, discount percentage and a live sale countdown.
- **Demo data** — optional `infra/db/demo_catalog.sql` seeds Electronics > Smartphones/Laptops, Fashion > Men > Shoes, brands, models, a featured product, a weekly deal and two live flash sales for local verification.

### Changed

- `marketplace-b2b` reads brands/models with category-subtree scoping and returns pagination totals; the gRPC contract adds `ListBrands`/`ListModels` and the new product/category fields.
- `ehub_b2b` keeps read-only table access and receives `EXECUTE` on the `SECURITY DEFINER` pricing refresh. It also gets `SELECT` on `businesses` so product detail can show the seller name; the ownership verifier now expects 79 rows.
- `apps/ehub` category routes use `/shop/category/:id` instead of the category name, and the product detail screen shows the category path, brand/model chips, sale window and related products.

### Known limits

- The UI shows the category path (e.g. Electronics / Smartphones) plus separate brand/model chips; brands and models are not duplicated as category nodes. This reuses the existing `brands`/`product_models` foreign keys.
- After the migration, `product_discounts` is the source of truth for promotions. `products.discount_percentage`/`is_flash_sale` are recalculated denormalized values, so scheduled deals should be managed in `product_discounts`.
- `apps/dispatch` lint remains blocked by the workspace `es-abstract` resolver failure; `apps/ehub` typecheck passes.
- Docker and live-database validation are intentionally deferred to the manual test phase. The migration, demo seed and live gateway smoke tests have not yet been executed against a running database.

### Tests

- `apps/ehub/node_modules/.bin/tsc.exe --noEmit -p apps/ehub/tsconfig.json` passes.
- `apps/ehub/node_modules/.bin/tsc.exe --noEmit --ignoreDeprecations 6.0 -p services/marketplace-b2b/tsconfig.json` passes; the service's local Nest build also passes.
- `go build ./...`, `go test ./...` and `go vet ./...` pass in `services/gateway`, `packages/proto`, `packages/clients` and `packages/platform`, including new catalogue filter and brand/model route tests.
- `buf generate` regenerated the marketplace Go bindings; `buf lint` passes.
- ESLint `--quiet` passes on the changed catalogue screens, components, clients and utilities.

---

## [Unreleased] - 2026-10-30 - Consumer eDelivery: destination-first courier quotes

The delivery miniservice opened a legacy shop-to-user checkout that checked a delivery area and fee against the monolith, with no drop-off search, no road path, and no courier-tier comparison. This brings eDelivery onto the same Photon/OSRM and dispatch-pricing contract as the taxi screen.

### Added

- **Destination-first drop-off search.** `eDelivery` now opens a Photon-backed “Where to deliver?” flow. Suggestions come from `GET /dispatch/places/autocomplete`, details resolve through `GET /dispatch/place/:id`, and the courier tiers stay hidden until a drop-off is chosen.
- **Road route and courier comparison.** Selecting a drop-off fetches the OSRM path through `GET /dispatch/route` and prices every delivery tier through `GET /dispatch/delivery-fare`. The map draws the same precision-5 polyline as the ride screen, with dispatch-backed nearby couriers and pickup ETAs.
- **Delivery map markers.** `TaxiMap` accepts `pickupTitle` / `destinationTitle` and a `delivery` driver mode, so the package flow labels the pickup and drop-off correctly and renders nearby couriers with a truck icon instead of a taxi.
- **Ad-hoc entry points.** “Send Package”, “Pick Up” and “Check Rates” now use the device pickup location instead of a hardcoded Nairobi point, falling back to the location hook's documented Nairobi default when permission is unavailable.
- **Local test protocol.** `TESTING.md` records the prerequisites, environment and device networking setup, and the end-to-end protocol for destination-first rides/eDelivery, H3 discovery, surge, concurrent accepts, lifecycle, and realtime SSE. `README.md` links to it from the local infrastructure section.

### Changed

- `apps/ehub/app/miniservices/edelivery.tsx` delegates the selected-delivery screen to the new `DeliveryBookingFlow` component. The quote confirmation reports the tier, KES fare, road distance and trip time.
- The delivery flow uses the same 24 km/h straight-line fallback as dispatch when OSRM is unavailable, so the tier list remains usable without a route line.

### Known limits

- This screen confirms a **quote**, not a dispatch delivery. Dispatch publishes courier rows for an existing checkout group (`POST /dispatch/delivery-requests` with a parent order), and the ad-hoc package flow has no order to attach; creating standalone package orders remains planned.
- Public OSM tiles remain development-only; set `EXPO_PUBLIC_OSM_TILE_URL` for production.

### Tests

- `apps/ehub/node_modules/.bin/tsc.exe --noEmit -p apps/ehub/tsconfig.json` passes.

---

## [Unreleased] - 2026-10-29 - Consumer taxi: destination-first Photon/OSRM booking

The consumer taxi screen showed vehicle tiers and a fallback pickup before the rider had chosen where to go, so the search bar was secondary and the tier list could not show a price for the actual leg. This changes the screen to a Bolt-style destination-first flow and uses the gateway's Photon autocomplete, OSRM route, and surge-aware fare estimate before showing the tiers.

### Added

- **Destination-first search.** The screen opens on a Photon-backed “Where to?” search. Suggestions come from `GET /dispatch/places/autocomplete`; no place list is bundled. Selecting a suggestion resolves details through `GET /dispatch/place/:id`, and only then reveals the ride tiers.
- **Road-aware tier comparison.** After a destination is chosen, the screen fetches the OSRM polyline through `GET /dispatch/route` and prices every tier through `GET /dispatch/estimate-fare` with the pickup coordinates, so the live H3-zone surge is included before booking. Each tier shows seats, the nearest online car's ETA, and the estimated KES fare.
- **Dispatch-backed map cars.** `TaxiMap` accepts `nearbyDriverLocations` and `showInfo`, so the consumer map renders the same cars the booking flow queried instead of polling the legacy monolith, and hides the simulator status box on this screen.
- **Client contract.** `api.estimateRideFare` / `gatewayClient.estimateRideFare` return `RideFareEstimates`, and `FareBreakdown` now carries the optional `surge_multiplier` / `surge_surcharge` fields.

### Changed

- The vehicle picker is hidden until a destination is selected. The search card shows pickup and destination as separate rows, with a clear action and a cancel/back guard while a ride is active.
- Pickup loads first and the tier catalogue loads in the background, so the destination search is never blocked by tier loading. A tier failure shows an unavailable message instead of an endless spinner.
- If OSRM is unavailable, the screen keeps the pins and uses a straight-line distance at 24 km/h for the fare comparison and route summary, matching dispatch's fallback.

### Tests

- `bunx tsc --noEmit -p apps/ehub/tsconfig.json` passes for the consumer app.

### Known limits

- The consumer app still tracks the assigned driver through the legacy driver-location endpoint; the gateway trip-location SSE integration remains a follow-up.
- Public OSM tiles remain development-only; set `EXPO_PUBLIC_OSM_TILE_URL` for production.

---

## [Unreleased] - 2026-10-26 - Dispatch: H3-zone surge and realtime offers

The dispatch service could already match drivers and price a road-aware leg, but the fare had no live demand signal and drivers only learned about new work on the next five-second poll. This adds a bounded H3-zone surge, Redis Pub/Sub events, gateway SSE streams, and a driver-app subscription that refreshes the authoritative feed as soon as an offer is pushed.

### Added

- **H3-zone surge.** Demand is a Redis set per H3 resolution-9 pickup cell; supply is the live H3 driver-location set. The multiplier is `1 + 0.2 × demand/supply`, capped at `1.8`, and stays `1.0` when the cell has no online supply. A trip→cell mapping makes release idempotent, demand expires after 20 minutes, and a Redis/index failure fails open to `1.0` with a 250 ms lookup timeout.
- **Surge lifecycle.** A committed ride records demand; accept and cancel release it; the reaper releases expired offers and re-records re-opened accepts. `CreateTrip` applies the multiplier, and `EstimateFare` / `GET /dispatch/estimate-fare` include it when optional pickup coordinates are supplied, so the quote and charge share one calculation.
- **Surge in the contract.** `FareBreakdown` now carries `surge_multiplier` and `surge_surcharge`; the dispatch proto and generated `packages/proto/dispatch` bindings were regenerated.
- **Redis Pub/Sub events.** Dispatch publishes `ride_offer` and `delivery_offer` events to eligible drivers after the booking transaction commits, and `driver_location` events after both location stores accept an update. Location events use a separate channel from offers so a rider cannot receive another customer's offers.
- **Gateway SSE streams.** `GET /dispatch/driver/stream` (driver role) pushes offers; `GET /dispatch/trips/:id/locations` authorizes with `GetTrip` and pushes the assigned driver's position. Both accept `?token=` for browser EventSource clients; all other routes keep header/cookie auth. A query token can appear in access logs, so browser clients should use a short-lived token or a dedicated stream ticket. Heartbeats run every 15 seconds.
- **Driver app push.** `apps/dispatch` subscribes with `expo/fetch` streaming, sends the bearer header (not a URL token), and refreshes the authoritative request/task feed on each offer event. The five-second poll remains the recovery path, so a dropped stream cannot hide work.
- Tests cover surge demand/supply ratios, the 1.8 cap, zero-supply fallback, release idempotency, demand expiry, fail-open pricing, estimate surge propagation, offer fan-out, Redis publishing, SSE broker forwarding/cancellation, and the gateway stream handlers (driver-role auth, query-token forwarding, the no-driver `409`, missing-broker `503`, and assigned-driver channel selection).

### Changed

- The public estimate endpoint accepts optional `pickup_lat`/`pickup_lng` (gateway aliases `lat`/`lng`) and prices the live zone multiplier when supplied. Omitting them keeps the previous time-of-day-peak-only quote.
- Realtime notifications are best-effort and bounded: offer fan-out has a 2-second budget, location publish has 250 ms, and a Redis failure is logged rather than failing the booking or location write.

### Known limits

- Pub/Sub has no replay or resume. A driver that reconnects relies on the polled feed to recover missed offers, and the trip-location stream returns `409` until a driver is assigned.
- The consumer app still polls the legacy driver-location endpoint; the gateway trip-location stream is ready for it, but the UI integration is a follow-up.
- Delivery and trip status changes are not pushed yet; only offers and driver positions are on the stream.

---

## [Unreleased] - 2026-10-23 - Mobile route tracing over OpenStreetMap

Ride pricing and driver ETAs already used OSRM when configured, but the rider had no route line when choosing a destination, the driver console had no map at all, and the consumer map used the platform's Google/Apple basemap. This adds a point-to-point route endpoint and renders the returned polyline over OpenStreetMap in both apps, so the consumer sees the path before booking and the driver can follow the approach and the trip.

### Added

- **Route RPC and gateway endpoint.** `rpc GetRoute` and the public
  `GET /dispatch/route?from_lat=…&from_lng=…&to_lat=…&to_lng=…` return
  `distance_meters`, `duration_seconds` and the OSRM encoded polyline
  (precision 5). The dispatch gRPC server uses the same `services.Router` that
  prices ride legs, so quotes and map geometry cannot diverge.
- **Consumer route preview.** Selecting a Photon/Google destination resolves its
  coordinates, fetches the route immediately, decodes it with
  `@mapbox/polyline`, draws a red destination pin and fits the camera to the
  pickup → destination path. The bottom sheet shows the road distance and
  duration before the rider confirms.
- **Driver route card.** `apps/dispatch` now uses `react-native-maps` with OSM
  tiles for the first active task: pickup/dropoff markers, a dashed driver →
  pickup approach line (refreshed at most every 30 seconds) and the pickup →
  dropoff trip line. A local decoder lives in `src/utils/polyline.ts`; web uses
  a placeholder because `react-native-maps` is native-only.
- `EXPO_PUBLIC_OSM_TILE_URL` for both apps, defaulting to the public
  `https://tile.openstreetmap.org/{z}/{x}/{y}.png` template, with visible
  “© OpenStreetMap contributors” attribution.
- gRPC route tests for successful geometry, missing coordinates, no route,
  invalid coordinates and an unavailable provider.
- A wildcard `*.css` ambient declaration so the driver app's declared TypeScript
  6 compiler accepts the existing side-effect `global.css` import.
- **Local geo validation stack.** `infra/docker-compose.geo.yml` adds OSRM and
  Photon without changing the default compose profile.
  `infra/geo/prepare-osrm.ps1` builds the Kenya MLD graph;
  `infra/geo/prepare-photon.ps1` plus `filter_photon.py` build a bounded Nairobi
  Photon fixture; `infra/geo/smoke.ps1` exercises OSRM, Photon and the public
  gateway route endpoint with the same coordinates the apps use.
- **Redis accept reservations.** `dispatch-service` now claims a job key and a
  driver key with `SET NX EX` before a ride or delivery accept enters Postgres.
  The job key rejects concurrent accepts of the same offer; the driver key
  rejects one courier accepting a ride and a delivery at once across replicas.
  The keys expire after 15 seconds and are compare-and-deleted after the
  transaction. Redis errors fail open because Postgres row locks and partial
  unique indexes remain the final authority.
- Reservation tests cover job conflicts, driver conflicts, TTL expiry, stale
  token release, fail-open behavior, and the ride/delivery error mapping.
- **Gateway public quotas.** Public auth and discovery routes now use a
  Redis-backed fixed-window limiter: 120 discovery requests/minute and 20 auth
  requests/minute per client IP over a 60-second window. Responses expose
  `X-RateLimit-*`; a rejected request returns `429` with `Retry-After`. Redis
  errors fail open and log once, and the Redis keys are SHA-256 hashes of the
  client IP rather than the address itself. The limits are configurable with
  `RATE_LIMIT_DISCOVERY_PER_MINUTE`, `RATE_LIMIT_AUTH_PER_MINUTE` and
  `RATE_LIMIT_WINDOW_SECONDS`.
- Rate-limiter tests cover window expiry, per-client separation, hashed keys,
  the `429` headers, and fail-open behavior.
- **H3 resolution 9 adaptive driver search.** Telemetry and cell keys move from
  resolution 8 to 9 (`dispatch:h3:r9:*`). The finder starts with a 1 km
  `gridDisk`, expands to 2 km and 3 km only until it has enough eligible drivers,
  and still applies an exact Haversine filter. A dense pickup area stops at the
  first ring; a sparse one pays for up to two more Redis/Postgres reads. Old
  resolution-8 sets expire with the 45-second location TTL, so no manual Redis
  migration is needed.
- H3 tests cover 1 km and 3 km disk coverage at Nairobi and across latitudes;
  finder tests cover expansion to an eligible outer ring, early stop when the
  limit is met, and expansion to 3 km for a larger result set.
- **OSRM delivery pricing.** Publishing a delivery now uses the configured
  router for the pickup -> dropoff road distance and duration. The published
  fare, the mirrored sub-order `delivery_fee`, and the pre-checkout quote all
  share the same calculation. OSRM failure, a missing dropoff, or an unset
  `OSRM_URL` falls back to the straight-line 24 km/h leg. Delivery route
  geometry is not persisted yet; only distance and duration are stored.
- Delivery fare tests cover road-route pricing and the straight-line fallback.

### Changed

- `TaxiMap` renders `<UrlTile>` instead of the native basemap on Android and
  overlays OSM tiles on iOS. The destination marker and camera framing now
  include the selected destination even before a route arrives.
- The driver console tracks its own position locally (in addition to publishing
  it) so the route map can draw the approach without a second round-trip.
- The route endpoint is public, like the other read-only discovery endpoints,
  because the rider previews a destination before creating a trip.
- The Compose `dispatch-service` definition now installs `gcc`/`musl-dev` and
  sets `CGO_ENABLED=1`. `uber/h3-go` is a cgo binding with no pure-Go fallback,
  so the Alpine Go image could not compile it without a C toolchain. The first
  container start is slower because the H3 C sources are compiled.

### Fixed

- `infra/geo/smoke.ps1` now braces the final coordinate (`${toLatText}?…`);
  PowerShell otherwise parsed `$toLatText?overview` as one variable name and
  sent a malformed OSRM query.
- Local geo instructions use `powershell -NoProfile -ExecutionPolicy Bypass`
  instead of `pwsh`, which is not installed in this environment.

### Notes

- OSRM remains optional. Without `OSRM_URL`, `GET /dispatch/route` returns
  `Unavailable`; the consumer keeps the destination pin and the straight-line
  ETA fallback, and the driver map shows markers without a line.
- The public OSM tile service is rate-limited and is for development only.
  Production must point `EXPO_PUBLIC_OSM_TILE_URL` at MapTiler, Stadia,
  Thunderforest or a self-hosted tile server.
- On Android, `react-native-maps` uses the Google Maps SDK as its rendering
  surface even with `mapType="none"`. Expo Go provides a key; a standalone
  build needs the `react-native-maps` config plugin with
  `androidGoogleMapsApiKey`, or the map component can be swapped to MapLibre for
  a fully Google-free renderer. The visible tiles and route data are
  OpenStreetMap either way.
- The route polyline is not persisted on `taxi_trips`; clients re-fetch it by
  coordinates. Persisting the accepted route for audit is a future step.
- `apps/dispatch` typechecks clean. `apps/ehub` still has its pre-existing
  TypeScript baseline (missing `react-native`/`expo-router` declarations in this
  workspace); the new files use the same map APIs as the existing component.
- `services/eHubGo` was not modified.
- **Live local validation passed.** With the Kenya OSRM graph and the Nairobi
  Photon fixture, `GET /dispatch/route` from
  `36.8172,-1.2864` to `36.8025,-1.2635` returned a `3,377.3 m` /
  `201.8 s` route with a 295-character polyline. Photon returned Nairobi
  features through both `GET /dispatch/places/autocomplete` and
  `GET /dispatch/place/:id`. The consumer's `@mapbox/polyline` decoder and the
  driver's local decoder both produced 99 points whose endpoints matched the
  OSRM waypoints to five decimal places.
- Photon 1.3.0 uses a hard-coded 30-second bulk timeout and flushes up to
  10,000 documents per request. The full Kenya dump exceeded that timeout on
  the local Docker VM, so `prepare-photon.ps1` creates a 1,800-place central
  Nairobi fixture. The container falls back to the full dump when the fixture
  is absent, but that fallback may time out on a small VM. This is a local
  development fixture, not production geocoding coverage.

---

## [Unreleased] - 2026-10-22 - Dispatch: Photon places and OSRM road legs

Ride matching and pricing now have optional production geo providers. The
dispatch service could already return H3-indexed drivers and price a ride
server-side, but its ETA and fare leg were straight-line estimates and its only
places provider was Google. This adds a Photon adapter and an OSRM route client
behind configuration, while keeping the existing fallback paths for local
development and provider outages.

### Added

- **Photon places provider** (`PLACES_PROVIDER=photon`, `PHOTON_URL`). It uses
  the same `PlacesSearch` interface as Google Places, applies an optional
  `lat`/`lon` bias and `PHOTON_COUNTRY_CODE` (default `KE`), and maps GeoJSON
  features to the existing autocomplete contract. Because Photon has no stable
  place-details endpoint, the service encodes the selected feature into an
  opaque `place_id` and `GetPlace` decodes it locally.
- **OSRM route client** (`OSRM_URL`, `OSRM_TIMEOUT_SECONDS`). It requests
  `overview=full&geometries=polyline` and returns road distance, duration and
  polyline, with explicit `ErrNoRoute`/`ErrRoutingUnavailable` errors.
- **Router-aware driver ETAs.** `find-driver` and `nearby-drivers` replace the
  straight-line ETA with the OSRM road duration for returned candidates.
- Tests for Photon mapping/token round-trips, OSRM URLs and error mapping, the
  route-leg fallback, and the road-ETA replacement.

### Changed

- `POST /dispatch/trips` prices the OSRM road distance and duration, not the
  straight-line leg, and returns the encoded polyline on the create response.
  The polyline is response-only; it is not persisted on `taxi_trips`.
- `PLACES_PROVIDER` defaults to Google. A configured `PHOTON_URL` selects
  Photon when the provider is unset, so existing Google deployments are
  unchanged.
- Google Places remains supported; the gateway and proto contracts are
  unchanged.

### Notes

- H3 telemetry and discovery use resolution 9 with an adaptive 1-3 km
  `gridDisk`. Per-zone surge and the pushed offer stream are deliberately
  separate changes.

---

## [Unreleased] - 2026-10-20 - Estay: isolated accommodation service, host workspace and booking lifecycle

Accommodation was the last sizeable consumer vertical still represented only by the monolith's legacy `properties` tables, and the `services/accommodation-service` scaffold had never been implemented. The audit's next step was Estay: a host lists several places with location, specifications, photos and a KES price; guests discover them by destination and dates; and the host manages everything from the merchant web app.

The vertical now ships as an independent Go service with its own `estay` schema, its own least-privilege database role, a gRPC contract, gateway routes and a working SvelteKit host workspace. `services/eHubGo` was not modified and Estay neither reads nor writes its property tables.

### Added

- `proto/estay/estay.proto` and generated Go bindings in `packages/proto/estay`: public `ListAmenities`, `ListListings`, `GetListing` and `GetListingAvailability`; host profile, listing, status, photo, amenity, calendar and booking RPCs; guest booking, history, lookup and cancellation RPCs.
- `services/estay-service`, a dual HTTP/gRPC Go service on **9007/9107**:
  - lazy host profiles (display name, bio) created on first host operation;
  - any number of listings per host, with location, coordinates, guests, bedrooms, beds, bathrooms, nightly price, cleaning fee, minimum/maximum nights and instant-book;
  - dedicated photo and amenity mutations (so editing one does not replace the listing), with a 30-photo cap, one cover photo and absolute `http(s)` URL validation;
  - host calendar with blackout ranges, public availability lookups, and booking/block overlap checks;
  - public search by destination text, city/country, guests, price range, property type, instant-book, paired check-in/check-out dates and paging;
  - booking requests with server-computed KES totals and a 5% default platform fee (`ESTAY_SERVICE_FEE_PERCENT`), optional instant-book confirmation, host confirm/decline, guest cancellation before check-in, and an hourly reaper that completes stays after check-out.
- `services/estay-service/migrations/001_estay.sql`: the `estay` schema (hosts, listings, photos, 23 seeded amenities, listing amenities, calendar blocks, bookings), indexes, `updated_at` triggers, and the `bookings_no_overlap` GiST exclusion constraint over `listing_id` and `daterange(check_in, check_out, '[)')` for pending/confirmed rows. Mounted as `13_estay.sql`, before the contracts (`14_contracts.sql`).
- `ehub_estay` role with `search_path=estay` and table-level grants in `infra/db/contracts.sql`; `verify_contracts.sql` now expects **78** privilege rows.
- Gateway `/estay` routes (public discovery plus host and guest operations) backed by `packages/clients`' Estay client and `ESTAY_GRPC_ADDR`.
- `apps/merchant` is now a working Estay host workspace:
  - SvelteKit BFF auth: Firebase email/password → gateway `/auth/sync` with `role=host` → httpOnly `ehub_session` cookie; a development-token fallback under `ALLOW_DEV_SESSION` or `vite dev`;
  - same-origin enforcement for mutating requests;
  - dashboard, listings list, create/edit form, photo manager, amenities, availability/blackout calendar and booking request confirm/decline, plus host profile;
  - all backend calls are server-side through `GATEWAY_URL`; the browser never holds the session token.
- `.env.example` and a rewritten `apps/merchant/README.md` for the host workspace; `services/estay-service/README.md` documents the schema, environment and direct API.
- `infra/db/mock-connections.env.example` and `infra/db/demo_connections.ps1`: a printable per-service connection set (same host/database, one role each) plus a `-Verify` role-isolation check against a running database.

### Changed

- `infra/docker-compose.yml` starts `estay-service`, publishes 9007/9107, uses the `ehub_estay` role, passes `ESTAY_GRPC_ADDR` to the gateway, and orders the new migration before the contracts.
- `infra/db/push_to_neon.ps1` applies the Estay migration before contracts and lists the `ehub_estay` role in its credential output.
- README architecture, service/port tables, gateway routes, migration order, database-role table, ownership notes and status summary now describe Estay and the merchant host workspace; the empty `accommodation-service` is marked as superseded.
- Direct HTTP models now serialize with snake_case JSON tags, matching the gateway payloads; host profile, listing, amenity, photo, calendar and booking fields are consistent across both transports.

### Verification

- `buf lint` and `buf generate` pass; `go build ./...`, `go vet ./...` and `go test ./...` pass in `estay-service`, `gateway` and `packages/clients`/`packages/proto`.
- Service unit tests cover listing normalization and publish rules, stay-date validation (including the half-open range and 365-night cap), photo URL rules, KES quote rounding, fee clamps, domain-error mapping and the presence of the overlap constraint in the migration.
- `001_estay.sql` was applied to the local Postgres, followed by `contracts.sql`; `verify_contracts.sql` printed the full matrix and passed with **78** rows, including the 20 `ehub_estay` privileges.
- `infra/db/demo_connections.ps1 -Verify` connected as every service role and showed the expected isolation: auth sees `users`, dispatch sees `users`/`orders`, marketplace roles see only their catalogue tables, wallet sees only `wallet`, and estay sees only `estay`.
- A live direct-API smoke created a host profile, listing, photo, amenities and publish, found it in search, created a booking, confirmed it as host, read the calendar and cancelled it as guest.
- Through the gateway, the same listing was listed and fetched; a 3-night booking priced at **KES 27,975** (25,500 subtotal + 1,200 cleaning + 1,275 service fee); a second overlapping request for the same dates was refused with **409**; and the first booking was cancelled as guest.
- The merchant app's server-side calls were exercised against the gateway: development-token login set the httpOnly cookie, and the dashboard, listings, create page, listing detail, bookings and profile returned 200; save/status, amenities, add/delete block and add/delete photo actions all round-tripped through the gateway.
- `npm run check` in `apps/merchant` reports **0 errors and 0 warnings**, and `npm run build` completes.

### Notes

- Estay deliberately ignores the monolith's `properties` tables. The consumer mobile app still uses the legacy property screens; wiring `apps/ehub` to `/estay` is planned.
- Photos are URLs, not uploaded bytes; a signed-upload pipeline and retention policy are planned.
- Booking totals are recorded, but no payment, refund, review, messaging or notification flow exists yet. These are listed under `[Planned]`.
- `services/eHubGo` was not modified.

---

## [Unreleased] - 2026-10-20 - Dispatch hardening: authoritative ride fares, offer clocks and proof upload

The dispatch lifecycle worked, but too much of its safety depended on the client. A rider could name the fare, a ride offer never expired, a driver who went quiet held the reservation, and the driver feed trusted a position that might be days old. The delivery side had the same freshness gap, while the new hand-over flow required a photo the driver app could not yet take. The audit called these P0: fix them before starting estay.

Rides now price themselves, every offer has a clock, a pass and a stale accept are recorded, feeds require a live position, and the driver app captures the proof photos the hand-over backend demands. A delivered order also confirms itself after the customer has had a day to respond.

### Added

- `services/dispatch-service/migrations/009_ride_hardening.sql` — `taxi_trips.expires_at`, `cancelled_at`, `cancelled_by`, `cancel_reason`, `distance_km` and `duration_minutes`; `ride_declines` (one pass per driver per trip, with an optional reason); `drivers.last_location_at`; and partial indexes for the expiry and stale-accept sweeps. Idempotent, mounted as `12_ride_hardening.sql` before the contracts (`13_contracts.sql`).
- **Server-authoritative ride fares.** `services/pricing.go` gains per-tier ride rate cards (`ev_bike`, `motorbike`, `economy`, `normal`, `xl`, `luxury`) and `EstimateRideFare`. `POST /dispatch/trips` prices the leg, stores the amount and distance/duration, and ignores any client-supplied `amount`. The legacy uppercase keys (`ECONOMY`, `COMFORT`, `XL`) still resolve.
- **Ride offers have a lifecycle.** `expires_at` is 15 minutes by default; `POST /dispatch/trips/:id/decline` records a pass; `CancelTrip` accepts a reason and records who cancelled; `RideService.ExpireStale` cancels expired requests and re-offers an `accepted` trip whose driver did not start within ten minutes, releasing the driver. The reaper is replica-safe with `FOR UPDATE SKIP LOCKED`.
- **Feeds require presence.** Ride and delivery feeds only consider drivers who are `online` and whose position is less than ten minutes old, excluding busy drivers and jobs the caller has declined. `Accept` re-checks online status, vehicle class/capacity and cross-table busyness under the row lock, and refuses a vehicle that cannot serve the booked tier with `ErrVehicleCannotServe` (412 / `FailedPrecondition`).
- `DELIVERY_AUTO_CONFIRM_SECONDS` (86400): the delivery reaper turns a `delivered` order into `completed` after 24 hours if the customer has not confirmed, stamping `customer_confirmed_at`, so a silent customer does not leave the job open forever.
- `RIDE_REQUEST_TTL_SECONDS` (900), `RIDE_ACCEPT_TTL_SECONDS` (600) and `RIDE_REAPER_INTERVAL_SECONDS` (60).
- **Driver app proof capture.** `expo-image-picker` and Firebase Storage are wired: `captureProofPhoto` takes the pickup/dropoff photo, uploads it under `dispatch-proofs/<user>/<job>/`, and returns the URL that `pickupDelivery`/`completeDelivery` send. The app now waits for merchant release (`picked_up` → `out_for_delivery`) instead of calling complete from pickup, and offers a dropoff photo only after release.
- **Driver app decline.** Rides now call `POST /dispatch/trips/:id/decline` instead of hiding the card locally, and the feed reads the gateway's `requests` envelope (it previously read a `trips` key that protojson never emits).
- **Consumer confirmation.** The order tracking screen renders `picked_up`, `out_for_delivery`, `delivered` and `completed` and calls `POST /dispatch/deliveries/:id/confirm`.

### Changed

- `Delivery.merchant_id` is exposed over proto and JSON, and `GET /dispatch/merchant/deliveries` accepts an optional `business_id` so an owner with several businesses can scope the cashier queue.
- Proof photos must be absolute `http(s)` URLs (maximum 2048 characters). A `file:`, `data:` or relative value is rejected before the row is written.
- Delivery accepts validate the courier's vehicle against the booked tier, matching the ride accept path.
- A ride without a preferred tier prices at economy but remains visible to any ride-capable vehicle, preserving the old "no preference" behaviour.
- `apps/ehub`'s `Delivery` type now carries the full hand-over shape (proofs, `merchant_id`, the new timestamps and terminal states), and `dispatchClient`/`api` expose `confirmDelivery`. The taxi screen no longer sends a fare; dispatch is the sole fare authority.
- The driver app no longer mirrors its position to the monolith and no longer requires `EXPO_PUBLIC_API_URL`; dispatch is the only source for ride and delivery work.
- README now records the service boundary decision: shared `public` only while the monolith co-writes a table, a new service gets its own schema from day one, cross-service reads go through APIs, and the first cross-service consumer introduces a transactional outbox rather than a direct broker write. Estay starts isolated.
- `infra/db/contracts.sql` grants `ehub_dispatch` `SELECT, INSERT, UPDATE` on `ride_declines`; `verify_contracts.sql` expects 58 rows. Compose and `push_to_neon.ps1` include migrations 008 and 009.

### Fixed

- The driver app requested `data.trips` from the ride feed, so a healthy feed looked empty. It now reads `data.requests`.
- `out_for_delivery` was missing from the driver app's active statuses and labels, so a released order disappeared between pickup and dropoff.
- The delivery reaper's stray `delivered` rows and the ride reaper's stale accepts are now both swept; previously a driver who accepted and stopped responding held the reservation indefinitely.
- `ErrVehicleCannotServe` fell through the ride gRPC error mapper as `Internal`/500; it now maps to `FailedPrecondition`/412.
- Driver location errors are logged with the underlying cause instead of returning a bare 503.

### Verification

- `go build`, `go vet` and `go test ./...` pass in `dispatch-service` and `gateway`; `buf lint` passes; `gofmt` is clean on the touched Go files.
- `apps/dispatch` `tsc --noEmit` is clean. `apps/ehub` still has its pre-existing baseline of unrelated errors; the touched service/type/tracking files typecheck.
- Migration `009_ride_hardening.sql` was applied twice (all `NOTICE ... skipping`) and `contracts.sql` + `verify_contracts.sql` pass with the 58-row matrix.
- Through the gateway against live data, **all checks passed**:
  - an economy ride priced at **125.84 KES** for 2.18 km / 4.68 min with no client amount; the sedan saw it in `requests`, declined it (204), and the feed went empty;
  - an offer whose `expires_at` was moved into the past refused accept with **409**, then the reaper cancelled it as `system` / `offer expired`;
  - accept → start → complete succeeded; an XL ride refused the sedan's accept with **412**;
  - a delivery priced at **176.34 KES** for 5.00 km; merchant ready → courier accept → pickup photo → cashier release → dropoff photo → customer confirm, with `orders.status` mirrored `in_transit` → `delivered` and the merchant's confirm attempt returning **404** (anti-probing);
  - a `file:///` proof URL was refused with **400**; a `delivered` order 25 hours old was auto-confirmed (`customer_confirmed_at` set, `completed`);
  - contract verification printed the full privilege matrix.

### Notes

- The hand-over proof remains a **URL**, not bytes. Firebase Storage rules must allow an authenticated driver to write its own `dispatch-proofs/` prefix; a server-issued signed upload and retention policy are planned.
- Order totals keep the order row's stored currency (legacy fixtures use `Ksh`), while fare quotes are KES. Unifying the order currency is a monolith-owned data change, not a dispatch one.
- The merchant cashier screen is still not built; the backend queue and release/reject endpoints are ready for it.
- `services/eHubGo` was not modified.

---


Neither party could say how a finished job went. A rider had no way to report a
dangerous driver; a courier had no way to flag an abusive customer; a customer
had no way to complain about a late or damaged delivery. Matching also had no
real reputation signal: `drivers.rating` was a default zero that nothing wrote.

Every completed ride and every delivered delivery can now be rated by both
parties. A rating is a 1-5 score, a `complaint`/`neutral`/`compliment`
classification and a free-text note. Either party rates the other: a rider rates
the driver, a driver rates the rider, a customer rates the courier, and a courier
rates the customer. Re-rating replaces the author's previous score.

### Added

- `services/dispatch-service/migrations/008_service_ratings.sql` —
  `service_ratings` (one row per author per job, the 1-5 score, the sentiment and
  the note), a unique index on `(service, reference_id, author_user_id)`, and
  subject/reference indexes. Mounted as `11_service_ratings.sql`; contracts move
  to `13_contracts.sql` (the ride-hardening migration later takes mount `12`).
- `services.RatingService` — `RateTrip`, `RateDelivery`, `ListTripRatings`,
  `ListDeliveryRatings` and `RatingSummary`. Eligibility is checked against the
  job's own state under `FOR UPDATE`: a ride must be `completed`; a delivery must
  be `delivered` (soft-completed) or `completed`. Only a party to the job may
  rate it; anyone else gets not-found.
- `Rating` and `RatingSummary` proto messages, and the RPCs `RateTrip`,
  `ListTripRatings`, `RateDelivery`, `ListDeliveryRatings` and
  `GetRatingSummary`.
- `POST /dispatch/trips/:id/rating`, `GET /dispatch/trips/:id/ratings`,
  `POST /dispatch/deliveries/:id/rating`, `GET /dispatch/deliveries/:id/ratings`
  and `GET /dispatch/ratings/summary?subject_user_id=&subject_role=`.
- A kind is optional: 1-2 derives `complaint`, 3 `neutral`, 4-5 `compliment`.
  An explicit kind is kept even when it disagrees with the score (a five-star
  note can still be a complaint). Comments are trimmed and truncated to 1000
  characters.
- Recording a driver rating refreshes the monolith's denormalized
  `drivers.rating` to the average of their received ratings, so the existing
  match UI shows a real number. `service_ratings` remains the source of truth.
- Merchant staff may read a delivery's ratings (for the dashboard); only the
  customer and the courier may write one.
- Handler and service tests: transport forwarding, HTTP/gRPC error mapping,
  score/kind derivation, comment truncation and role validation.

### Changed

- The dispatch ownership contract gains `SELECT, INSERT, UPDATE` on
  `public.service_ratings`; `verify_contracts.sql` asserts it.

### Notes

- Ratings are **upserted**, not append-only: the unique index enforces one rating
  per author per job, and `updated_at` records a revision. The original
  `created_at` is kept.
- The API is live, but no app prompts for a rating yet: the rider/driver/customer
  cards are listed in the Planned section above.
- This is the record, not the moderation workflow. Complaint escalation, rating
  reasons/tags, abuse controls and the app prompts are listed in the Planned
  section above.
- A courier's summary spans rides and deliveries, because the person is the same
  `subject_user_id` with `subject_role='driver'`.

---

## [Unreleased] — 2026-10-19 — Proof of hand-over: cashier release and customer confirmation

A courier could mark a delivery collected and delivered with one button each, and
nothing recorded what left the merchant or what reached the customer. The
merchant had no step between "the courier took it" and "it is on the way", and
the customer had no way to say they actually received it.

The delivery lifecycle now requires **evidence** at both hand-overs. The courier
photographs the goods at collection; the merchant's cashier checks that photo
and marks the order out for delivery. At the destination the courier photographs
the goods again and soft-completes the order; the customer confirms receipt,
which makes the completion explicit. All of it is backend-only: the app capture
and upload, the cashier screen and the customer prompt are follow-ups.

### Added

- `services/dispatch-service/migrations/007_delivery_proofs.sql` —
  `delivery_proofs` (append-only pickup/dropoff photos, the courier who took
  them, and the merchant member who verified or rejected a pickup photo),
  `deliveries.out_for_delivery_at`, `completed_at`, `customer_confirmed_at` and
  `released_by`, the two new lifecycle states (`out_for_delivery`, `completed`),
  and the widened courier-reservation index (a courier out for delivery is still
  busy). Mounted as `10_delivery_proofs.sql`; contracts moved to `11_contracts.sql`
  (now `13_contracts.sql` after the ratings and ride-hardening migrations).
- `DeliveryService.PickUp` now requires a `photo_url` and records a pickup proof.
  Calling it again while `picked_up` supersedes the pending photo and inserts a
  retake, so a rejection does not need a new status.
- `DeliveryService.Release` — the cashier verifies the latest pending pickup
  photo and moves the delivery to `out_for_delivery`. The proof is stamped
  `verified` with the caller's user id, so the release is attributable.
- `DeliveryService.RejectPickupProof` — the cashier declines a pickup photo with
  a reason; the delivery stays `picked_up` and the courier can retake it.
- `DeliveryService.Confirm` — the customer confirms receipt, moving `delivered`
  to `completed` and stamping `customer_confirmed_at`. Only the delivery's
  customer may call it.
- `DeliveryService.Deliver` now requires a dropoff `photo_url`, and only runs
  from `out_for_delivery` (the merchant must release first).
- `DeliveryService.ListMerchantDeliveries` / `GET /dispatch/merchant/deliveries` —
  the cashier/owner queue across every business the caller manages, with an
  optional `statuses` filter (`picked_up` is the "waiting for release" queue).
- `POST /dispatch/deliveries/:id/release`, `POST /dispatch/deliveries/:id/reject-pickup`,
  `POST /dispatch/deliveries/:id/confirm`, and the corresponding gRPC RPCs
  (`ReleaseDelivery`, `RejectDeliveryProof`, `ConfirmDelivery`,
  `ListMerchantDeliveries`). `PickUpDelivery` and `CompleteDelivery` carry
  `photo_url` and `notes`; `Delivery` carries the two proofs and the new
  timestamps over both transports.
- `ErrProofRequired` (400 / `InvalidArgument`) when a step needs a photo that the
  request did not carry, or a release/rejection has no pending photo.
- Merchant authorization for readiness, release and rejection now accepts the
  business owner **or an active `manager`/`cashier` in `business_staff`**, read
  only by dispatch (`GRANT SELECT` in the contract). The cashier who served the
  order can therefore act without an owner token.

### Changed

- The courier lifecycle is now
  `requested → assigned → picked_up → out_for_delivery → delivered → completed`
  (or `cancelled` from `requested`/`assigned`). `picked_up` means "courier has
  the goods, photo pending cashier review"; `out_for_delivery` is the true
  in-transit state; `delivered` is a soft completion and `completed` is the
  customer's confirmation.
- `orders.status` stays in the monolith's vocabulary: both `picked_up` and
  `out_for_delivery` mirror `in_transit`, and `delivered`/`completed` both mirror
  `delivered`. The confirmation is recorded on the delivery, so the order views
  keep treating `delivered` as terminal.
- The courier busy check, the active-driver unique index, the driver task feed
  and delivery history all include the new states.
- `MarkDeliveryReady` is no longer owner-only: active manager/cashier staff can
  stamp it too.

### Notes

- Proofs are **URLs**, not bytes. The client uploads the image to object storage
  and sends the URL; dispatch stores the reference. Until the driver app adds
  capture + upload, its pickup and complete calls will get a `400` asking for
  `photo_url` — that is the enforcement working, not a regression.
- The customer confirmation is a prompt, not a blocker: a `delivered` delivery is
  already soft-complete. A timeout-based auto-confirmation sweep is not
  implemented yet; the distinction is kept in `completed_at` vs
  `customer_confirmed_at` so it can be added without a migration.
- The cashier queue and release/reject endpoints are backend-only for now. The
  merchant app will poll `GET /dispatch/merchant/deliveries?statuses=picked_up`
  (SSE later) and show the pickup proof image from `photo_url`.
- Dispatch now reads `business_staff` (select-only) to authorize merchant staff.
  This is the monolith's table and will be superseded by the merchant-service;
  the predicate is isolated in `services/proofs.go` (`merchantCanManageTx`).

---

## [Unreleased] — 2026-10-19 — Courier offers expire, and a pass is recorded

An unclaimed delivery request had no deadline. It sat in every eligible courier's
feed forever, held the sub-order with the partial unique index, and could not be
re-published until the customer cancelled it. An accepted request could also sit
in `assigned` forever if the courier went quiet: nothing released them, and the
merchant's basket stayed stranded. Declining was client-side only, so dispatch
could not tell a courier who had passed from one who had never been offered the
job.

Every delivery now carries an offer clock, a pass is recorded, and a reaper
withdraws expired requests and releases stale accepts. Because the reaper runs
from every replica, the database arbitrates with `FOR UPDATE SKIP LOCKED` and
both sweeps are idempotent.

### Added

- `services/dispatch-service/migrations/006_delivery_offers.sql` —
  `deliveries.expires_at` (backfilled, then `NOT NULL` with a 15-minute default),
  a partial index on open requests, and `delivery_declines`
  (`delivery_id`, `driver_user_id`, `reason`, `created_at`, one row per pair).
  Mounted as `09_delivery_offers.sql`, before `10_contracts.sql`.
- `POST /api/v1/dispatch/deliveries/:id/decline` and
  `rpc DeclineDelivery` — the courier's pass. The offer stays open to everyone
  else and disappears from the decliner's feed. The body is optional; a
  `{"reason":"too far"}` is kept for telemetry (capped at 200 chars).
- `DeliveryService.ExpireStale` and a one-minute reaper in `main`:
  - a `requested` delivery past `expires_at` becomes `cancelled`, its sub-order
    goes back to `pending`, and the quoted `delivery_fee` is cleared, so the
    customer can publish again;
  - an `assigned` delivery whose `accepted_at` is older than ten minutes goes
    back to `requested` with the courier released and a fresh offer clock, and
    the sub-order mirrors `requested` / `driver_id = NULL`.
- `ErrDeliveryExpired` (409 / `Aborted`), surfaced when a courier accepts after
  the offer window has closed.
- `Delivery.expires_at` over proto and JSON, and expiry/decline handling in the
  courier feed (`expires_at > now()` and `NOT EXISTS (declined by me)`).
- Config: `DELIVERY_REQUEST_TTL_SECONDS` (900),
  `DELIVERY_ACCEPT_TTL_SECONDS` (600), `DELIVERY_REAPER_INTERVAL_SECONDS` (60).
- Tests: decline forwards the session courier and reason (204), an expired
  accept maps to 409 with the expiry message, and the existing fare/readiness
  suites still pass. Live smoke: 21 assertions for the offer clock, decline
  telemetry, expiry, re-publication and stale-accept recovery.

### Changed

- The dispatch app's Decline button now calls the server for deliveries (the
  local optimistic hide stays so the card disappears immediately). Declining a
  ride is still client-side; rides have no decline endpoint.
- The reaper is intentionally replica-safe rather than single-runner: each sweep
  locks its batch and does nothing when another replica won.

### Notes

- Expiry is a **cancellation**, not a separate status: the partial unique index
  treats any non-cancelled delivery as live, so cancelling is what frees the
  sub-order. The customer sees the usual `pending` order and can retry.
- A stale accept is re-offered, not cancelled: the merchant may already be
  packing, and the basket is still worth delivering. The customer keeps the
  quoted fee.
- The reaper writes `orders.status` on the same driverless path as the
  lifecycle. It inherits the known RLS gap documented in
  `infra/db/OWNERSHIP.md`: it works because service roles keep `BYPASSRLS`, and
  that must be closed before the bypass is dropped.

---

## [Unreleased] — 2026-10-19 — Delivery fares and the vendor hand-over

A delivery was priced by whoever called it: `POST /dispatch/delivery-requests`
took no distance and stored no fee, so the customer's `delivery_fee` and the
courier's pay had no single source. Collections also had no gate: a courier
could mark `picked_up` while the merchant was still packing, and nothing recorded
that the basket was ready.

Delivery now reuses the ride fare algorithm with **per-tier rate cards** and a
per-tier minimum, computed server-side when the delivery is published, and a
courier cannot collect until the merchant stamps the basket ready.

### Added

- `services/dispatch-service/migrations/005_delivery_fare_and_readiness.sql` —
  `deliveries.fare_amount`, `fare_currency`, the priced `distance_km` and
  `duration_minutes`, and `vendor_ready_at`. Idempotent; mounted as
  `08_delivery_fare_and_readiness.sql`, before `09_contracts.sql`.
- `services/pricing.go` — `DeliveryTier` rate cards for `ev_bike`, `motorbike`,
  `pickup` and `lorry`, keyed by the same tier keys `003_vehicle_tiers.sql`
  seeds. Each card carries `base_fare`, `per_km`, `per_minute`, fuel efficiency
  and a **minimum fare** (KES 70 / 85 / 300 / 600), so a short bike hop is still
  worth running while a lorry is priced for the fuel it burns. `DeliveryTier`
  renders as the ride `VehicleTier`, so both verticals share `CalculateFare`.
- A time-of-day peak multiplier (morning/evening commute, late night) in the
  shared fare engine, exposed as `PeakSurcharge`; a provider can override it
  later without changing the algorithm.
- `EstimateDeliveryFare` prices every tier cheapest-first (or one named tier).
  It backs both `POST /api/v1/dispatch/estimate-delivery-fare` and
  `rpc EstimateDeliveryFare`, and it is the **same function** that prices a
  published delivery, so the quote and the charge cannot drift.
- `DeliveryService.MarkReady` and `rpc MarkDeliveryReady` /
  `POST /dispatch/deliveries/:id/ready` — the merchant's packed-basket stamp.
  Only the business owner may set it; anyone else gets the same `NotFound` as a
  stranger. Readiness is a timestamp, not a status, because a merchant prepares
  while a courier is still being found.
- `PickUp` now refuses with `ErrVendorNotReady` (409 / `Aborted`) until
  `vendor_ready_at` is set.
- `Delivery` carries `fare_amount`, `fare_currency`, `distance_km`,
  `duration_minutes` and `vendor_ready_at` over proto, JSON and the driver task
  feed.
- Tests: per-tier ordering, the minimum fare, the peak surcharge, an unknown
  tier, straight-line leg pricing, the no-dropoff minimum, the estimate
  endpoint's ordering and single-tier mode, and the 409 readiness mapping.

### Changed

- Publishing a delivery computes the fare from a straight-line pickup →
  dropoff leg and an urban average speed (24 km/h) until OSRM is wired in, then
  stores it on the row and mirrors it to the sub-order's `delivery_fee`. The
  courier fee and the basket total are now separate amounts.
- `GET /dispatch/driver/tasks` also returns `merchant_name`, `fare_amount`,
  `fare_currency` and `vendor_ready`, so the driver app can show the fee and
  hold the pickup button while the merchant is still packing.
- The driver console shows the courier fee and the leg for each delivery and
  replaces "Mark picked up" with a waiting notice until the merchant confirms;
  the customer's tracking timeline gains a "Vendor packed your order" step and
  the fee.

### Notes

- The merchant order app is still deferred. Until it ships, readiness is stamped
  through the same endpoint it will call. A delivery whose merchant never stamps
  stays `assigned` and can only be cancelled — the stale-accept/expiry recovery
  work will cover that case.
- Weather and traffic still default to clear/free-flow at publish; only the
  deterministic time-of-day peak is live. The straight-line leg and its derived
  duration are placeholders for OSRM routing.

---

## [Unreleased] — 2026-10-19 — Dispatch-owned delivery lifecycle

Delivery had no life of its own. `POST /dispatch/delivery-requests` published a
request and stamped the order, and then the thread stopped: the monolith's
`GET /delivery/requests`, `POST /delivery/driver/accept` and
`POST /delivery/driver/decline` were the only driver endpoints, **no pickup or
delivered transition existed anywhere**, and nothing recorded which vendor's
parcel a courier was carrying. A delivery could not be tracked, completed, or
recovered if a courier went quiet.

Dispatch now owns a `deliveries` table and the whole courier lifecycle:
`requested → assigned → picked_up → delivered`, or `cancelled` before pickup.
One row is created per **vendor sub-order**, because in the food/grocery/liquor
verticals each merchant ships its own basket and a multi-vendor checkout is
several independent courier jobs. Every transition mirrors the sub-order's
`orders.status` in the same transaction so the customer's order view keeps
working without reading the new table, and the state moves are enforced by the
database rather than by convention: a partial unique index allows one live
delivery per sub-order, another allows one live delivery per courier, and losing
the claim race reports a conflict instead of double-booking.

### Added

- `services/dispatch-service/migrations/004_delivery_lifecycle.sql` — the
  `deliveries` table (one row per vendor sub-order) with its geography pickup and
  dropoff, mirrored amount and currency, per-transition timestamps, and the
  invariants that make the lifecycle safe: `deliveries_order_idx` (unique while
  `status <> 'cancelled'`, so a cancelled delivery can be re-published),
  `deliveries_active_driver_idx` (unique over `assigned | picked_up`, so one
  courier cannot hold two live jobs), a feed index, a courier index and a
  parent-order index. Idempotent, and mounted as `07_delivery_lifecycle.sql`
  before the contracts so a fresh database grants on a table that exists.
- `services/delivery_lifecycle` (in `services/deliveries.go`) — `Publish`,
  `ListRequests`, `Accept`, `PickUp`, `Deliver`, `Cancel`, `Get`,
  `ListHistory` and `ListOrderDeliveries`, sharing one `transition` helper that
  locks the delivery row, attributes a lost race to the unique index, and writes
  the `orders` mirror inside the same transaction.
- New errors, each mapped to a distinct gRPC status and HTTP status:
  `ErrInvalidDelivery` (400), `ErrDeliveryNotFound` (404), `ErrDeliveryState`
  (409 via `Aborted`), `ErrCourierBusy` (409), and `ErrNoEligibleCouriers`
  (409 — publishing to nobody would strand the sub-order with no way to
  withdraw it).
- Proto messages and RPCs: `Delivery` plus `ListDeliveryRequests`,
  `AcceptDelivery`, `PickUpDelivery`, `CompleteDelivery`, `CancelDelivery`,
  `GetDelivery`, `ListDeliveryHistory` and `ListOrderDeliveries`.
  `RequestDeliveryResponse` now carries the created `deliveries`.
- Gateway routes: `GET /dispatch/delivery-requests` and
  `GET /dispatch/delivery-history` (driver-only),
  `POST /dispatch/deliveries/:id/{accept,pickup,complete}` (driver-only),
  and `POST /dispatch/deliveries/:id/cancel`, `GET /dispatch/deliveries/:id`,
  `GET /dispatch/orders/:id/deliveries` (any session, service decides).
- Tests: `handlers/deliveries_test.go` (the courier feed requires the driver
  role, transitions forward the session identity, a state conflict is 409) and
  the reworked `handlers/delivery_requests_test.go` (publish forwards the load
  and tier, and the 400/404/409 mapping).

### Changed

- **`GET /dispatch/driver/tasks` now reads `deliveries`** instead of guessing
  the pickup from `order_items`/`businesses` and guessing the dropoff from
  `orders.shipping_address_id`. `in_transit` becomes `picked_up`, and the task id
  for a delivery is the `deliveries` row id, so the driver app can advance it.
- `RequestDelivery` returns `PublishResult` (the parent order, the chosen tier,
  and one entry per created delivery) instead of a single summary.
- The customer can track a checkout group **or** a single sub-order id:
  `ListOrderDeliveries` matches `parent_order_id` or `order_id`.

### Removed

- `services/dispatch-service/services/delivery_requests.go` — the thin publisher
  whose only job was to stamp the order is subsumed by `Publish`.
- The app's last monolith dependency for delivery work:
  `apps/dispatch/src/services/driver-api.ts` no longer calls
  `/delivery/requests`, `/delivery/driver/accept` or `/delivery/driver/decline`.

### Verified

- `go build`, `go vet` and `go test ./...` pass in `dispatch-service` and
  `gateway`; `gofmt` is clean on every touched file (the pre-existing CRLF drift
  in `handlers/drivers.go`, `handlers/pricing.go`, `services/pricing.go` and the
  older `_test.go` files is untouched).
- `004_delivery_lifecycle.sql` + `contracts.sql` applied to the live database and
  `verify_contracts.sql` passes: `ehub_dispatch` holds `SELECT, INSERT, UPDATE`
  on `public.deliveries`.
- Through the gateway against live data, **41 assertions**: publish by load
  (5 kg → `ev_bike`, 500 kg → `pickup`, 3000 kg → `lorry`, 6000 kg → 400, a car
  tier → 400, an unknown tier → 400), the bike courier sees the offer and a
  pickup-only courier does not, accept → pickup → complete with
  `orders.status` mirrored to `assigned` → `in_transit` → `delivered`, the
  courier's history and the customer's tracking both list it, an unrelated
  courier gets 404, and cancelling re-opens the sub-order so it can be
  published again.

### Next

- The `orders.status` mirror is written under a driver-scoped transaction
  context, and the `orders` RLS policy does not cover drivers. It works today
  only because every service role still has `BYPASSRLS`; see the new note in
  `infra/db/OWNERSHIP.md`. Fixing it is a prerequisite for dropping
  `BYPASSRLS` on `ehub_dispatch`.
- Product-level `weight_kg` and a cart-summed load, rather than the caller
  passing `cargo_weight_kg`.

---

## [Unreleased] — 2026-10-19 — Delivery classes and load-based courier selection

The previous delivery change made the courier a real constraint but pinned it to
one class: `RequestDelivery` asked for `delivery` + `motorbike` no matter what
was being sent. A lorry-load of goods or a bulky pickup-only parcel was offered
to a motorbike, and the caller had no way to say how heavy the parcel was — the
order model has no weight on `order_items`, so nothing in dispatch could have
worked it out anyway.

Dispatch now chooses the courier by the load: the caller gives a
`cargo_weight_kg` (or names a tier), and the cheapest delivery tier that can
carry it is used — motorbike, then pickup, then lorry. A load nothing can carry
is refused before the order is touched, and the response says which vehicle will
collect.

### Added

- `VehicleCatalogue.CheapestForCargo(ctx, service, kg)` — the cheapest tier whose
  `cargo_capacity_kg` covers the load, ordered by `display_order` (ties by name,
  so the result is deterministic). Nothing big enough is `ErrNoVehicleForCargo`
  (400), not an empty result: an order no vehicle can serve must not be
  published.
- `ErrVehicleNotForDelivery` (400) — the caller named a tier that exists but
  cannot take deliveries (a car). Distinct from `ErrUnknownVehicleType`, so the
  log says *why* the tier was rejected.
- `RequestDeliveryRequest.cargo_weight_kg` and `.vehicle_type`, and
  `RequestDeliveryResponse.vehicle_type` / `.vehicle_class` /
  `.cargo_capacity_kg`, so the client can say what is coming. HTTP
  `POST /dispatch/delivery-requests` accepts the same two fields.
- `services/deliveries_test.go` — the load-to-vehicle rule tested without
  a database: 20 kg stays on a bike, 21 kg steps up to a pickup, 1200 kg to a
  lorry, an explicit tier overrides the weight, and a car or an unknown key is
  refused. Error mapping to 400/404/409 is covered in
  `handlers/delivery_requests_test.go`.

### Changed

- **`RequestDelivery` chooses the courier by the load.** The signature takes a
  `DeliveryRequest` (`VehicleTypeKey`, `WeightKg`) instead of nothing. Neither
  field set means the cheapest delivery tier — the old bike behaviour — so
  existing callers are unaffected; a `vehicle_type` wins over a weight, and the
  tier must be one that serves deliveries.
- The "no couriers" error names the class that was searched for (a pickup request
  that finds no pickup, rather than a blanket "no motorbike couriers").
- `checkout` in the app now reports the chosen tier alongside the courier count,
  and `apps/ehub/app/miniservices/edelivery.tsx` renders the delivery fleet —
  each class with the load it can carry and how many are online — instead of the
  single hardcoded motorbike count.

### Verified

- `go build`, `go vet` and `go test ./...` pass in `dispatch-service` and
  `gateway`. New coverage: the load-to-vehicle rule without a database
  (`deliveries_test.go`), and the 400/404/409 mapping for a load nothing
  can carry, a tier that does not take deliveries, an unknown tier, no couriers
  and an undispatched order (`handlers/delivery_requests_test.go`).
- Through the gateway against live data, with one courier of each delivery class
  online at the same point and a dispatchable food order — 24 assertions, all
  passing:
  - `cargo_weight_kg=5` → `ev_bike` (the cheapest tier; it matches both bike
    tiers, since a bike request fits any bike), one courier.
  - `cargo_weight_kg=500` → `pickup`; `=3000` → `lorry`; each matches only the
    courier of that class, which is the point — the count is one because the
    class filter excludes the other two couriers.
  - `cargo_weight_kg=6000` → `400`; `vehicle_type=economy` → `400`;
    `vehicle_type=submarine` → `400`.
  - `vehicle_type=lorry` with a light weight → `lorry`, the explicit tier
    overriding the load; a second request for an order already published → `404`.
  - `lorry` also comes back with `vehicle_class=LORRY` and
    `cargo_capacity_kg=5000`, so the client can name the vehicle it is sending.
- Regression: the ride path still works end to end (a tier-`xl` trip is created,
  appears in the 7-seater's `GET /dispatch/ride-requests`, and cancels), and
  `GET /dispatch/vehicle-types?service_type=delivery` returns exactly the four
  delivery tiers cheapest-first.
- `apps/ehub` `tsc --noEmit` still reports its 129 pre-existing errors — none in
  the touched client files.

### Known limits

- The weight is **client-supplied**. `order_items` has no weight, so dispatch
  cannot derive it from the cart; the app sends none today and therefore gets the
  cheapest tier. Adding a `weight_kg` to the catalogue (products) and summing the
  order items is the follow-up.
- Courier selection only affects which class is *searched* — the assignment
  itself is still the monolith's. Giving dispatch the driver-facing delivery feed
  and accept/start/complete RPCs is a separate step.
- Both bike tiers carry 20 kg, so a light parcel resolves to `ev_bike`; that is
  the cheapest tier and it matches motorbike couriers too. A consumer that wants
  to show "Motorbike" specifically should read the class, not the tier key.

---

## [Unreleased] — 2026-10-17 — Vehicle tiers and real matching

The rider's vehicle choice did not reach the service. `nearby-drivers` and
`find-driver` took one overloaded `service_type` string, and the app sent `taxi`
or `motorbike` — names that matched no row in `vehicle_types`. The eligibility
query had a deliberate fall-through for exactly that case (`NOT EXISTS (...)`),
so an unrecognised value matched **every** online driver. Choosing a motorbike
and choosing a taxi returned the same list, and a ride request carried no
vehicle requirement at all (`CreateTripRequest.vehicle_type` was documented as
"accepted for client compatibility"). The same fall-through applied to
deliveries: `RequestDelivery` asked for `"motorbike"` and got every online
driver, cars included.

`vehicle_types` now carries a taxonomy, a trip records the tier the rider asked
for, and the request feed and driver search both filter on it.

### Added

- `services/dispatch-service/migrations/003_vehicle_tiers.sql` — adds
  `tier_key`, `vehicle_class` (`BIKE | CAR | PICKUP | LORRY`),
  `cargo_capacity_kg`, `services` (`ride | delivery`) and `display_order` to
  `vehicle_types`, seeds the eight tiers (EV Bike, Motorbike, Economy, Normal,
  XL, Luxury, Pickup, Lorry), and adds `taxi_trips.vehicle_type_id`. Purely
  additive, `ADD COLUMN IF NOT EXISTS` throughout, so it can be re-applied.
  Applied as `06_vehicle_tiers.sql` in the compose bootstrap and by
  `push_to_neon.ps1`; `contracts.sql` moved to `07_`.
- `services/dispatch-service/services/vehicles.go` — `VehicleCatalogue` (the
  tier reference data) and the `VehicleLookup` interface the finder and the ride
  service share, so both transports resolve a tier the same way. An unrecognised
  key is `ErrUnknownVehicleType`, mapped to 400 / `InvalidArgument` — a booking
  no longer silently degrades to "match anything".
- `GET /dispatch/vehicle-types` (gateway, dispatch HTTP) and the
  `ListVehicleTypes` RPC: the catalogue is served to clients instead of being
  hardcoded in the app.
- `handlers/vehicles.go` — the tier list handler.

### Changed

- **Matching is class + capacity, not a string compare.**
  `db.DriverRepository.EligibleDriverIDs` now takes a service and a tier key and
  requires the same `vehicle_class` with at least the requested
  `passenger_capacity` and `cargo_capacity_kg`. A rider asking for XL (6 seats)
  matches a 7-seater and not a 4-seat car; a bike never answers a car request.
  The `NOT EXISTS` fall-through is gone. Candidates remain nearest-first; the
  *tier list* is cheapest-first (`display_order`).
- **`service_type` means the service; `vehicle_type` means the tier.** They were
  conflated. `service_type` is now `ride | delivery` and `vehicle_type` is the
  tier key; both are optional, and an empty pair still means "any online
  driver". This is a breaking change for callers that sent `taxi` / `motorbike`
  as `service_type` — the app was the only one, and it is updated here.
- **The ride request feed filters by what the driver can serve.**
  `ListRideRequests` joins the driver's tier and only returns requests whose tier
  the vehicle matches; a lorry or pickup driver never sees ride requests.
  Requests created before this migration carry no tier and stay visible to
  everyone.
- **`TaxiTrip` exposes the requested tier.** `Trip.vehicle_type` (the tier key)
  is returned by every trip response, and `taxi_trips.vehicle_type_id` stores it.
- **Deliveries are bikes, by tier rather than by accident.** `RequestDelivery`
  passes `delivery` + `motorbike`. This is *stricter* than before: cars can no
  longer be offered a food delivery. Choosing pickup/lorry by load size is the
  delivery-size work.
- `contracts.sql` and `verify_contracts.sql` are **unchanged**: the migration
  runs as the table owner, so `dispatch` keeps `SELECT` on `vehicle_types` and
  its existing `taxi_trips` grants. `OWNERSHIP.md` records `vehicle_types` as
  bootstrap-seeded reference data.

### Fixed

- The vehicle picker in `apps/ehub/app/miniservices/taxi.tsx` is now built from
  `GET /dispatch/vehicle-types` (with seat counts) instead of a hardcoded
  `['taxi', 'motorbike']` toggle, and the selected tier is sent both to the
  driver search and to trip creation. Switching tiers re-searches.
- `apps/ehub/app/miniservices/edelivery.tsx` passes `delivery` + `motorbike`,
  matching the new parameters.

### Verification

- `go build`, `go vet` and `go test ./...` pass in both `dispatch-service` and
  `gateway`. New coverage: an unknown tier is rejected before the location index
  is queried (`find_driver_test.go`), `nearby-drivers` forwards both
  `service_type` and `vehicle_type` to the finder (`drivers_test.go`), an
  unknown tier maps to 400 on trip creation (`rides_test.go`), and the
  vehicle-types route is public and renders proto field names
  (`gateway_test.go`).
- `003_vehicle_tiers.sql` was applied to the live database **twice** to confirm
  it is idempotent, and `verify_contracts.sql` still reports a complete matrix —
  the grants did not change.
- Exercised through the gateway against live data, with a 4-seat car driver and
  a 7-seat car driver online at the same point:
  - `GET /dispatch/vehicle-types?service_type=ride` → `ev_bike, motorbike,
    economy, normal, xl, luxury` (cheapest-first; the delivery-only pickup and
    lorry are absent).
  - `vehicle_type=xl` → only the **7-seater**; `vehicle_type=economy` → both
    cars; `vehicle_type=motorbike` → nobody; `service_type=taxi` (the old app
    value) → nobody, where it used to match everyone.
  - `vehicle_type=submarine` → `400`, before any location lookup.
  - Booking with `vehicle_type=xl` returns a trip whose `vehicle_type` is `xl`;
    the 7-seater's `GET /dispatch/ride-requests` contains it and the 4-seat
    car's feed does not. Accept → start → complete succeeds and keeps the tier.
  - A booking with no tier stores none, and is visible to the 4-seat car.
  - Through the gateway the whole run was 23 assertions, all passing.
- `apps/ehub` `tsc --noEmit` reports the same 129 errors as before this change —
  all from unresolved third-party types (`lucide-react-native`, `expo-router`,
  `@tensorflow/tfjs`) — and none in the touched client files.

### Known limits

- `passenger_capacity` is a property of the **tier**, not of the driver's actual
  car: `drivers.vehicle_type_id` points at a tier row, so a 7-seater can only be
  matched as though it were the 6-seat XL tier — unless a separate 7-seat tier
  exists, which is how the verification above set up its XL driver. Storing the
  real seat count on `drivers` is the fix, and it is the same gap for
  `cargo_capacity_kg`.
- The tiers are English-named here; a localized label belongs in the client.
- Fares are still client-supplied (the app sends `500.0`) and the fare
  algorithm still has no time-of-day factor. Both are the next step, together
  with wiring weather/traffic to providers.

---

## [Unreleased] — 2026-10-17 — Taxi ride lifecycle

The taxi vertical was discovery-only: `find-driver` returned a candidate and
nothing persisted the ride, so the flow could not complete without the monolith
and two riders could be offered the same driver. `dispatch-service` now owns the
whole lifecycle — `requested → accepted → in_transit → completed`, or
`cancelled` — and the two rules that make it safe are database invariants rather
than checks the application hopes to win.

### Added

- `services/dispatch-service/migrations/002_ride_lifecycle.sql` — two partial
  unique indexes that make "one active ride per driver" and "one open ride per
  rider" impossible to violate, plus an index for the request feed. Also
  normalises legacy `taxi_trips.driver_id` values (see *Changed*). Applied as
  `05_ride_lifecycle.sql` in the compose bootstrap and by `push_to_neon.ps1`.
- `services/dispatch-service/services/rides.go` — `RideService`: creates trips,
  lists the requests near a driver, and applies the accept/start/complete/cancel
  transitions under a row lock. Every write sets the RLS session context.
- `services/dispatch-service/handlers/rides.go` — the same surface over the
  service's own HTTP API, for parity with its other routes.
- `services/gateway/handlers/rides.go` and eight gateway routes: rider-side
  `POST /dispatch/trips`, `GET /dispatch/trips/:id`,
  `POST /dispatch/trips/:id/cancel`, `GET /dispatch/ride-history`, and
  driver-side `GET /dispatch/ride-requests`,
  `POST /dispatch/trips/:id/{accept,start,complete}`.
- Eight RPCs on `proto/dispatch/dispatch.proto` (`CreateTrip`,
  `ListRideRequests`, `AcceptTrip`, `StartTrip`, `CompleteTrip`, `CancelTrip`,
  `GetTrip`, `ListRideHistory`) and a `Trip` message.
- Handler tests (`handlers/rides_test.go`) and location-updater tests
  (`services/driver_location_test.go`).

### Changed

- **`dispatch-service` is now a writer of `taxi_trips`.** `contracts.sql` grants
  it `SELECT, INSERT, UPDATE` (was `SELECT`), the expected matrix in
  `verify_contracts.sql` grew to 44 rows, and `OWNERSHIP.md` records the ride
  lifecycle as contracted. This is the first contract change made because a
  service gained a write, and the verifier caught both copies as designed.
- **`taxi_trips.driver_id` means `drivers.id`.** The monolith's accept path
  stored the driver's *user* id in that column even though it references
  `drivers(id)`; readers work around it with `d.id = t.driver_id OR d.user_id = t.driver_id`.
  The migration rewrites the existing rows, and readers stay tolerant, because a
  user id and a row id cannot be told apart afterwards. The API exposes the
  driver's **user id** as `Trip.driver_id`, which is what the location index and
  the mobile client use, so the tracking view resolves the assigned driver.
- **Driver location is persisted, not only indexed.** `UpdateDriverLocation`
  wrote Redis only, leaving `drivers.last_location` empty for anyone who used the
  extracted service; because the request feed and the monolith's nearby queries
  read that column, such a driver would never see a ride request. A new
  `services.TrackedLocationUpdater` writes both stores.
- **The app's taxi flow goes through the gateway.**
  `createTaxiTrip`, `getTaxiTripStatus`, `getRideRequests` and
  `acceptRideRequest` now reach `dispatch-service` instead of the monolith;
  `startRide`, `completeRide`, `cancelTaxiTrip` and `getTaxiRideHistory` were
  added. `declineRideRequest` was removed: offers are a pull feed, so declining
  has no server representation.

### Fixed

- `infra/db/push_to_neon.ps1` aborted on the first `psql` `NOTICE`. Windows
  PowerShell promotes a native command's stderr to a terminating error while
  `$ErrorActionPreference` is `'Stop'`, so an idempotent "does not exist,
  skipping" — which every `DROP ... IF EXISTS` in the bootstrap emits on a fresh
  database — killed the deploy after the third file. Native calls now run with
  the preference relaxed and the exit code decides. Found while re-testing the
  script against a scratch database.

### Verification

- The whole lifecycle was driven through the gateway with real session tokens:
  201 on request, the trip appearing in the driver's feed, then accept → start →
  complete, with `409` for a rider who already has an open ride, a driver who is
  already riding, a lost race and an invalid transition; `403` for a non-driver
  on driver routes, `401` anonymously, and `404` when a stranger reads someone
  else's trip (the endpoint does not confirm that a trip exists).
- The database invariant was probed directly, bypassing the service: a second
  active trip for one driver is rejected with `duplicate key value violates
  unique constraint "taxi_trips_active_driver_idx"`.
- Two drivers raced for the same trip: the first accept won, the second received
  `409`.
- App compatibility: `trip.driver_id` equals the `driver_id` that
  `/dispatch/nearby-drivers` returns, which the app normalises to `id`, so
  `nearbyDrivers.find(d => d.id === trip.driver_id)` matches.
- `go test ./...` passes in `dispatch-service` (handlers + services) and the
  gateway.

### Known limits

- **No fare estimate.** `total_amount` is still supplied by the client (the app
  hardcodes 500); pricing exists as an estimate endpoint but the ride does not
  call it.
- **No route.** `CreateTripResponse.polyline` is reserved and always empty, so
  the map shows markers rather than a line. Dispatch has no routing provider.
- **Offers are a pull feed, not a pushed offer with a timeout.** Any nearby
  driver can accept; there is no reservation while a rider waits, and no
  decline.
- The monolith's own taxi code is unchanged and outside the contract; it could
  still write the legacy `driver_id` form for rides it creates.

### Note

`services/eHubGo` was not modified.

---

## [Unreleased] — 2026-10-17 — Database ownership contracts

Each extracted service now connects to Postgres with its own least-privilege
role, and the set of tables it may read or write is declared in one place and
enforced by the database. Previously every service connected as the
superuser/table owner, so any service could read or write any table: the service
boundaries were real in the process layout but fictional in the data.

### Added

- `infra/db/contracts.sql` — the contract: per-service login roles
  (`ehub_auth`, `ehub_wallet`, `ehub_dispatch`, `ehub_fulfillment`, `ehub_b2b`,
  `ehub_c2c`), their `search_path`, and the table grants. Idempotent; applied
  automatically on a fresh database and re-appliable to an existing one.
- `infra/db/verify_contracts.sql` — compares the live grants (`pg_class.relacl`)
  against the declared contract in both directions and raises on any mismatch
  (a promised privilege that is absent, or a held privilege that is not
  declared).
- `infra/db/OWNERSHIP.md` — the table-level matrix (who writes what, who may
  read it), the rationale, how to change a contract, and the known deviations.
- `infra/db/push_to_neon.ps1` — deploys the same schema + contract SQL to a Neon
  database in the compose bootstrap order, then runs `verify_contracts.sql` as a
  deploy gate. It uses a `psql` from `PATH` or a throwaway PostGIS container (no
  client install), refuses a target that looks local unless `-AllowLocal`, and
  can copy reference tables from the local database with `-LoadReferenceData`.

### Changed

- **wallet-service owns the `wallet` schema.** `wallet_*` was the only table set
  with no writer outside the service (the monolith never references it), so it
  moved out of `public`. `migrations/001_wallet.sql` now creates the schema and
  its tables, and `main.go` checks `wallet.wallet_accounts`.
- All services now connect with their own role through `DATABASE_URL` (compose
  and README updated). The roles are `NOSUPERUSER` / `NOCREATEROLE` /
  `NOCREATEDB` and receive only the privileges their code uses: for example
  `dispatch-service` may `UPDATE` `drivers` and `orders` but may not read the
  catalogue. Dev passwords are `ehub-dev-<service>` and are not secrets.
- Each role carries `BYPASSRLS`, deliberately: the monolith's row policies
  filter on session context (`get_app_user_id()` / `has_role()`) that the
  extracted services do not set, so enforcing RLS today would silently hide
  rows from them. Table-level ownership is enforced regardless; dropping
  `BYPASSRLS` (starting with marketplace-c2c, which already propagates
  `app.current_user_id`) is the next step.

### Verification

- `verify_contracts.sql` passes against the live database and prints the
  effective matrix.
- Probed each role directly: `ehub_b2b` can read `products` but is denied
  `UPDATE products` and `SELECT users`; `ehub_wallet` can read
  `wallet.wallet_accounts` but is denied `SELECT orders`; `ehub_dispatch` can
  `UPDATE orders` but is denied `SELECT products`.
- Restarted the five Go services with their roles: healthchecks pass, and
  `pg_stat_activity` shows one connection each for `ehub_auth`, `ehub_wallet`,
  `ehub_dispatch` and `ehub_fulfillment`.
- The deploy path was exercised end to end against a scratch database: the five
  SQL files applied in bootstrap order, `verify_contracts.sql` passed, and
  `-LoadReferenceData` copied the reference tables from the local database.

### Known limits

- The monolith (`services/eHubGo`) connects as the table owner and is outside
  the contract; it is why several tables still have two writers.
- Cross-domain reads are still SQL joins (`dispatch-service` reads orders,
  addresses, businesses and users in single statements); removing them is a
  separate workstream.
- One database, one instance. Only `wallet` has a schema of its own.

### Note

`services/eHubGo` was not modified.

---

## [Unreleased] — 2026-10-17 — Dispatch hardening, KES fares, opt-in monolith

Follow-up to the runtime smoke test: the defects it surfaced that live outside
`services/eHubGo`.

### Changed

- **Fares are priced in KES.** `CalculateFare` hardcoded `currency: "USD"` and
  `DefaultVehicleTiers` used USD-scale figures (2.50 base / 1.20 per km), while
  the product is Kenyan — the monolith prices at 50 KES base / 20 KES per km in
  `Ksh`. The tier table is converted at 20 KES per unit, which reproduces the
  monolith's 50 KES base fare and preserves every tier ratio. An Economy trip of
  5 km / 15 min now quotes 245 KES instead of a 5,373 "USD" number.
- **Fuel surcharge defaults are unit-safe.** Both transports defaulted
  `baseline_fuel_price` to `1.50` — a USD/gallon figure — so a Kenyan
  `current_fuel_price=180` produced a 5,355 surcharge. The new
  `services.ResolveFuelPrices` treats the current price as the baseline when no
  baseline is supplied, so no surcharge is invented. The default minimum fare
  moved from 5 to 50 (the Economy base fare).
- `infra/docker-compose.yml`: `ehubgo` now sits behind a `monolith` profile.
  `docker compose up` starts the extracted stack; `docker compose --profile
  monolith up` opts in to the monolith. Nothing depends on it.

### Fixed

- `EligibleDriverIDs` compared `service_type` against `drivers.vehicle_type_id`
  / `vehicle_types.name`, but the apps send service categories (`taxi`,
  `motorbike`), so nearby-driver searches returned nothing. The filter now
  applies only when `service_type` actually names a known vehicle type.
- Driver `status` for a user without a `drivers` row returned 503 from both
  transports; both now map `db.ErrDriverNotFound` to 404.
- The legacy geo fallback aborted the entire nearby-driver search when Redis did
  not implement `GEORADIUS_RO` (Redis < 3.2, or a compatible server). It is now
  skipped on "unknown command"; the H3 index remains the primary path.
- The legacy geo query asked for distances but not coordinates, so drivers taken
  from the legacy `drivers:active` index were reported at 0,0. It now requests
  coordinates (`WithCoord`).

### Verification

- `go build ./...` and `go test ./...` pass in `dispatch-service`; the pricing
  tests were updated to the KES tariff and a `ResolveFuelPrices` test added.
- Through the gateway: `service_type=taxi` / `motorbike` / `Sedan` all return the
  seeded driver; a user without a driver profile gets 404; fare estimates return
  KES 245 / 369 / 562 with a zero fuel surcharge.
- `docker compose config --services` excludes `ehubgo`; with
  `--profile monolith` it is included.

### Note

`services/eHubGo` was not modified.

---

## [Unreleased] — 2026-10-17 — Runtime smoke test of the gateway stack

First end-to-end run of the extracted stack. Everything before this had been
verified statically (`tsc`, `go test`, `docker compose config`); this brought the
services up and drove real HTTP requests through the gateway using a locally
minted session JWT (so protected routes could be exercised without Firebase).

### Added

- `infra/docker-compose.yml`: Postgres now runs `postgis/postgis:15-3.4-alpine`
  and mounts an ordered `/docker-entrypoint-initdb.d` set — `eHubGo/schema.sql`,
  the eHubGo attributes migration, the wallet migration and the c2c listing
  policies — so a fresh `docker compose up` creates the full 63-table schema.
  Previously the stack used plain `postgres:15-alpine`, which cannot create the
  four `GEOGRAPHY` columns, and loaded no schema at all: the database came up
  empty and no service could work.
  - `services/eHubGo/migrations/002_add_is_featured_to_products.sql` is
    deliberately **not** applied — it drops `products.is_flash_sale`, a column
    `schema.sql` defines and `SearchProducts` queries.

### Fixed

- `services/dispatch-service/db/tasks.go`: the driver task feed could never
  return a row. The `taxi_trips` branch selected
  `u.first_name || ' ' || u.last_name` without joining `users`, so every
  `GET /dispatch/driver/tasks` failed with
  `pq: missing FROM-clause entry for table "u"` and surfaced as HTTP 503. It now
  uses a `LEFT JOIN` with a `'Customer'` fallback (the same placeholder the
  delivery branch already used), which also keeps the task visible when `users`
  RLS hides the passenger row from the driver.

### Verified against the running stack

- `GET /health`, `POST /auth/validate` (valid and invalid tokens), and
  `POST /auth/sync` (401 on a bogus Firebase token, with the service account
  configured).
- Marketplace: product list, `featured` filter, product detail, 404 for a
  missing product, categories, listings. Wire shapes match the typed client:
  `{"products": [...]}` / `{"product": {...}}`, protojson `snake_case`, numerics
  as strings.
- Dispatch: `estimate-fare`, `nearby-businesses`, `nearby-drivers` and
  `find-driver` (a seeded driver with an indexed location was returned), driver
  `status`/`location`/`tasks` after seeding a `drivers` row.
- Wallet and BFF: `POST /wallet/accounts`, `GET /wallet/balance`,
  `GET /wallet/transactions`, `GET /me/home`.
- Auth boundary: 401 without a token, 403 without the `driver` role.
- Empty results are served as `{}` (protojson omits empty lists / zero values);
  the typed clients already treat those as empty collections, so no client fix
  was needed.

### Known issues found (not fixed)

- **`services/eHubGo` does not compile**: import cycle
  `handlers` → `handlers/ehost` → `handlers` (the `ehost` package imports
  `handlers.WithRLS`). This blocks the monolith container and every
  un-extracted route; out of scope while `eHubGo` stays read-only.
- **Port collision on 6379**: a native Redis 3.0 for Windows shadows Docker's
  published port, so host-run services silently talk to the old server. It lacks
  `GEORADIUS_RO`, and the dispatch service's "legacy geo" fallback treats that as
  fatal, turning `nearby-drivers` into a 500. The H3 index is the primary path;
  the optional fallback arguably should degrade instead of failing the request.
- **Fare pricing is US-denominated**: breakdowns report `currency: "USD"` and
  default `baseline_fuel_price` to `1.50`, so a Kenyan `current_fuel_price=180`
  produces a nonsense surcharge (5355 on a 5 km trip). Needs a pricing decision
  rather than a code tweak.
- **`service_type` semantics**: `EligibleDriverIDs` compares the caller's
  `service_type` against `drivers.vehicle_type_id` / `vehicle_types.name`, while
  the app sends `taxi` / `motorbike`. It only matches when vehicle types are
  literally named that way.
- Driver `status` for a user with no `drivers` row returns 503 (not 404/409),
  while `driver/location` succeeds without one because it only writes to Redis.
- The dev stack connects as the Postgres superuser, so RLS policies are bypassed
  (they only take effect under a non-superuser role).

---

## [Unreleased] — 2026-10-17 — Frontend API calls: correct backends, real data, typed layer

The mobile apps now call the right backend for each domain and render real
responses instead of mocked data.

### Added

- `apps/ehub/packages/services/types.ts`: shared entity/payload types (session, catalog, marketplace listings, wallet, dispatch, and the monolith verticals), replacing `any` in the API layer.
- `api` facade methods: `searchProducts`, `getBrands`, `getModels`, `getMe`, `getMyMall`, `getWalletTransactions`, `walletTopUp`, `walletPayout`, `createWallet`, `getC2CListing`.
- `gatewayClient`: `home` (BFF aggregate), `getWalletTransactions`, `walletTopUp`, `walletPayout`, `createWallet`, `validateToken`.

### Fixed

- `api.searchProducts` did not exist, so the search screen could never load results; it now calls `GET /products/search?q=&limit=`.
- Wrong-branch calls: `getServicesByType` → `GET /services/{type}` (was `/services?type=`, unregistered) and `calculateDeliveryQuote` → `POST /delivery/pricing/estimate` (was `/delivery/quote`, unregistered).
- `getDriverTasks` now reads the extracted dispatch service through the gateway instead of the monolith.
- Removed methods that pointed at non-existent monolith routes: `register`, `getHubs`, `createHub`, `getTasks`, `createTask`, `createDriver`, `createCinemaHall`, `listCinemaHalls`; `createMovie` retargeted to `POST /movies`.
- Session identity: the `/auth/sync` user blob is normalised so `id`/`uid`/`name`/`displayName` all resolve, fixing `session.user.uid` / `session.user.id` reads in the store, orders and social screens.
- `shop/product/[id].tsx` read a non-existent `data.images` field; images now come from `image_urls`.
- `miniservices/ecinema.tsx` read `res.data` on a response that is a bare array.
- `TaxiMap`: tracked-driver and nearby-driver responses are now normalised from the real `Driver` row (including the raw `last_location` value); tracking previously read fields the backend never sent.
- `apps/dispatch`: `updateDriverLocation` treats the gateway (dispatch-service) write as authoritative and mirrors to the monolith best-effort, so a monolith hiccup can no longer take the driver offline.
- `food-items/[id].tsx` guards a null `business_id`; `taxi.tsx` guards a missing place lookup.

### Changed

- Mocked screen data replaced with real calls: `miniservices/epay.tsx` → wallet transactions, `miniservices/ehealth.tsx` → `getMe`, `c2c-listings/new.tsx` → live categories/brands, `listings/stay/[id].tsx` → `getProperty`, `(tabs)/store.tsx` shop tab → `getMyMall` + `getBusinessProducts`, `miniservices/ehost.tsx` maps `Property` onto its view model.
- `(tabs)/index.tsx`: added a live flash-sale countdown, removed the hard-coded loyalty-points badge (no points endpoint exists), and wired the wallet quick actions to the wallet screen.

### Notes / known debt

- These monolith routes are still unregistered, so their client methods will 404 until the backend catches up: `getAttributes` (`/attributes`), `getAttributeValues` (`/attributes/:id/values`), `getAddresses` (`/addresses`), `bookBusTicket` (`/bus/tickets`). Call sites already degrade gracefully.
- `c2c-listings/new.tsx` still uses a placeholder image URL: `expo-image-picker` is not installed, so no real device upload exists yet.
- `(tabs)/store.tsx` listing "sold"/"delete" actions remain local-only; no backend endpoint exists for either.

### Verification

- `npx tsc --noEmit` in `apps/ehub`: 130 → 129 errors with no new errors. The remainder are pre-existing module-resolution failures (`lucide-react-native`, `@tensorflow/tfjs`) and one `@react-native-picker/picker` typing issue.
- `apps/dispatch` typecheck unchanged by this work (its dependencies are not installed locally, so only module-resolution errors appear).

---

## [Unreleased] — 2026-10-17 — Backend ports moved off the frontend range

Backends now use a dedicated local port block so they never clash with the dev
servers: **9000s for HTTP**, **9100s for gRPC**. The frontends keep their
framework defaults (Vite/SvelteKit `5173+`, Expo/Metro `8081+`).

| Service | HTTP | gRPC |
|---|---|---|
| eHubGo | 9000 | — |
| auth-service | 9001 | 9101 |
| wallet-service | 9002 | 9102 |
| dispatch-service | 9003 | 9103 |
| fulfillment-service | 9004 | 9104 |
| marketplace-b2b | 9005 | 9105 |
| marketplace-c2c | 9006 | 9106 |
| gateway | 9010 | — |

### Changed

- Service config defaults: auth-service, wallet-service, dispatch-service and fulfillment-service (`PORT`/`GRPC_PORT`), `services/gateway` (`PORT`), `marketplace-b2b`/`marketplace-c2c` (`src/main.ts`), and the `packages/clients` gRPC target defaults (`localhost:9101`–`9106`).
- `infra/docker-compose.yml`: every published port, the services' `PORT`/`GRPC_PORT` values, the gateway's `*_GRPC_ADDR` targets, and each healthcheck URL.
- Mobile apps: the `EXPO_PUBLIC_GATEWAY_URL` fallbacks in `gatewayClient.ts` / `driver-api.ts` and both `.env.example` files now use gateway `9010`.
- Docs updated for the new ports: `README.md` (service table, dual-serving table, port-convention note, local-infrastructure section), `services/DISPATCH_LOCAL_SETUP.md`, `services/MARKETPLACE_LOCAL_SETUP.md`, and the wallet/dispatch/marketplace READMEs.

### Notes

- `services/eHubGo` is untouched; compose sets its `PORT` (and `APP_URL`) to `9000`.
- Historical entries below keep their original `8080s`/`9080s` references.

### Verification

- Go: `packages/clients` and `services/gateway` tests pass; auth/wallet/dispatch/fulfillment `go test ./...` pass; `gofmt` clean.
- NestJS: both marketplace services `nest build` pass.
- `docker compose -f infra/docker-compose.yml config` validates and publishes `9000`–`9010` / `9101`–`9106`.

---

## [Unreleased] — 2026-10-17 — Local stack runs the whole system

The local compose stack now brings up every extracted service, so the gateway's
marketplace routes work out of the box.

### Added

- `marketplace-b2b` (8085 / gRPC 9085) and `marketplace-c2c` (8086 / gRPC 9086) in `infra/docker-compose.yml`, each with its own healthcheck. Their `node_modules` live in named volumes so the container's Linux install never clobbers the host's, and the repo `proto/` directory is mounted at `/proto` for runtime proto loading.
- Healthchecks for `ehubgo`, `auth-service`, `wallet-service`, `dispatch-service`, `fulfillment-service` and `gateway`, so dependents wait on `service_healthy` before starting.

### Changed

- `services/MARKETPLACE_LOCAL_SETUP.md` rewritten for the gateway-first setup: public read routes, `EXPO_PUBLIC_GATEWAY_URL`, and compose-based startup.
- `README.md` local-infrastructure section now lists the marketplace services and the healthcheck behaviour.

### Verification

- `docker compose -f infra/docker-compose.yml config` validates.

---

## [Unreleased] — 2026-10-17 — Mobile apps routed through the gateway

The mobile apps now reach the extracted services exclusively through
`services/gateway`. The gateway and the underlying contracts were extended so it
can serve the full surface the apps call, instead of the apps talking to each
service directly.

### Added

- `apps/ehub/packages/services/gatewayClient.ts`: one HTTP/JSON client for the gateway that attaches the stored session token. `marketplaceClient` and `dispatchClient` are now thin adapters over it, so their app-facing surface is unchanged.
- Gateway dispatch endpoints, with handler tests: `GET /dispatch/nearby-businesses`, `GET /dispatch/places/autocomplete`, `GET /dispatch/place/:id` (public), `POST /dispatch/delivery-requests` (authenticated) and `GET /dispatch/driver/tasks` (driver role).
- `dispatch.DispatchService` RPCs `AutocompletePlaces`, `GetPlace`, `NearbyBusinesses`, `RequestDelivery` and `ListDriverTasks`, implemented in `dispatch-service` on top of its existing service/repository layer.
- `marketplace` contract fields needed to mirror the read API: `Product.brand_id/model_id/brand_name/model_name/created_at/updated_at`, `Category.created_at/updated_at`, and `Listing.seller_avatar_url/created_at`.

### Changed

- `apps/ehub`: `api.syncUser` and `api.getWalletBalance` now go through the gateway; marketplace and dispatch calls are gateway-backed; the wallet available balance is mapped to the `balance` string the app expects.
- `apps/dispatch`: driver sign-in, availability, location and active tasks use `EXPO_PUBLIC_GATEWAY_URL`; the ride/delivery request feeds still come from `eHubGo`.
- `apps/ehub/.env.example` / `apps/dispatch/.env.example` now document `EXPO_PUBLIC_GATEWAY_URL`; the per-service `EXPO_PUBLIC_DISPATCH_URL` and marketplace URLs are no longer used.
- `dispatch-service`'s gRPC server now takes a `grpcserver.Config` of collaborators, and its public-method allowlist includes the places/business discovery RPCs (`RequestDelivery`/`ListDriverTasks` stay authenticated).
- `marketplace-b2b`/`marketplace-c2c` gRPC mappers emit the new contract fields.
- `README.md` route, environment and status sections refreshed.

### Fixed

- `apps/ehub` sign-in/sign-up read `response.data` from an already-unwrapped response body, which left `persistBackendSession` with `undefined`; the flows now accept either shape.

### Verification

- Go: `dispatch-service` and `gateway` (`go build`/`vet`/`test`, including the new handler tests) pass; `packages/proto` builds; `packages/clients` tests pass.
- NestJS: `marketplace-b2b` and `marketplace-c2c` `nest build` pass.
- `proto`: `buf lint` clean; `buf generate` regenerated `packages/proto`.
- Apps: `tsc --noEmit` reports only the pre-existing module-resolution errors (unresolved `@/…` path aliases and missing ambient types); no errors in the changed files.

---

## [Unreleased] — 2026-10-17

Delivery of the gRPC foundation, unified service authentication, and the shared
client layer plus HTTP gateway that ties the services together.

### Added

#### Protobuf contracts and code generation
- `proto/` restructured to one package per service (`proto/<service>/<service>.proto`) with `buf.yaml` (`lint: STANDARD`, `PACKAGE_VERSION_SUFFIX` disabled) and `buf.gen.yaml`.
- Contracts rewritten to mirror each service's **real** API for `auth`, `dispatch`, `fulfillment`, `wallet`, `marketplace` (split into `MarketplaceB2BService` and `MarketplaceC2CService`) and `spatial` (moved out of `services/eHubGo`).
- `generate_protos.ts` now drives `buf generate` into the shared Go module.
- New Go module `packages/proto` (module path `ehub/proto`) containing the generated `*.pb.go` / `*_grpc.pb.go` bindings for every contract.

#### gRPC servers (dual-serving alongside HTTP)
- New `grpcserver` packages for `auth-service` (gRPC `9081`), `wallet-service` (`9082`), `dispatch-service` (`9083`) and `fulfillment-service` (`9084`), delegating to the existing service/handler logic.
- New gRPC front-ends for the NestJS services `marketplace-b2b` (`src/grpc.ts`, port `9085`) and `marketplace-c2c` (`src/grpc.ts`, port `9086`) using `@grpc/grpc-js` + `@grpc/proto-loader` (loading the same `.proto` at runtime).
- `GRPC_PORT` added to each Go service's config; each `main.go` now starts both an HTTP and a gRPC front-end.
- Shared logic extracted so HTTP and gRPC share one path: `AuthHandler.SyncUserCore`, `WalletHandler.InitTopUpCore` / `RequestPayoutCore`.

#### Shared auth module — `packages/platform` (`ehub/platform`)
- `auth/jwt.go`: session-JWT `Claims` and a `Validator` (`NewValidator`/`Generate`/`Parse`) that pins **HS256** and rejects unsigned/`alg=none`/foreign-signed tokens and tokens without a `user_id`.
- `auth/gin.go`: `GinMiddleware`/`GinSession` (reads `Authorization: Bearer` and the `session_token` cookie; opt-in `?token=`), `DenyAll`, `GinRequireRole`, and token-extraction helpers.
- `auth/grpc.go`: `UnaryInterceptor(publicMethods...)` with a per-method public allowlist, plus `UserIDFromContext` / `HasRole` helpers.
- Unit tests covering token round-trips, wrong secret, `alg=none`, missing subject, Gin header/cookie/query carriers, the role guard, and the gRPC interceptor.

#### Shared gRPC clients — `packages/clients` (`ehub/clients`)
- `config.go`: `Config` + `DefaultConfig()` reading `AUTH_GRPC_ADDR`, `WALLET_GRPC_ADDR`, `DISPATCH_GRPC_ADDR`, `FULFILLMENT_GRPC_ADDR`, `MARKETPLACE_B2B_GRPC_ADDR`, `MARKETPLACE_C2C_GRPC_ADDR` (local ports by default).
- `interceptor.go`: `WithSessionToken` / `SessionToken` and `UnaryClientInterceptor`, which forwards the caller's token as outgoing `authorization` + `session_token` metadata and applies a default deadline without overriding a caller-set one.
- `clients.go`: `Clients` aggregates typed clients for all six services over lazily-connected channels; unconfigured services are skipped; `Close()` releases connections.
- Tests for token/deadline propagation, preserved caller deadlines, and defaults.

#### Gateway service — `services/gateway` (HTTP `8090`)
- New HTTP/JSON entry point that validates the session JWT, then calls the owning service over gRPC via `packages/clients`, forwarding the caller's token so each service re-validates it (defence in depth).
- Routes (all under `/api/v1/gateway`):
  - **Public:** `POST /auth/sync`, `POST /auth/validate`, `POST /dispatch/find-driver`, `GET /dispatch/estimate-fare`, `GET /dispatch/nearby-drivers`, and marketplace reads (`/marketplace/products`, `/marketplace/products/:id`, `/marketplace/categories`, `/marketplace/listings`, `/marketplace/listings/:id`).
  - **Authenticated:** wallet (`/wallet/balance`, `POST /wallet/accounts`, `POST /wallet/topup`, `POST /wallet/payout`, `/wallet/transactions`), fulfillment (`/fulfillment/orders/:id/status`, `POST /fulfillment/orders/:id/assign-driver`, `POST /fulfillment/orders/:id/status`), and `POST /marketplace/listings`.
  - **Driver-only:** `PUT /dispatch/driver/location`, `PUT /dispatch/driver/status`.
- BFF aggregate `GET /me/home`: concurrently fans out to the wallet (and dispatch when `lat`/`lng` are supplied) and returns a combined payload, reporting any failed section without failing the whole request.
- JSON bodies/responses use `protojson` with proto field names, mirroring the contract; gRPC statuses map onto HTTP codes.
- Handler tests with fake clients, plus an end-to-end test that runs a real gRPC server with the shared interceptor and drives it through `clients.New` over a real socket.

### Changed

- The four Go services now serve **both** HTTP/JSON and gRPC from one process and import `ehub/proto`, `ehub/platform` and (for the gateway) `ehub/clients` via `replace` directives.
- `fulfillment-service` HTTP routes now go through the shared session middleware (see Security).
- `infra/docker-compose.yml`: Go image bumped to `golang:1.26-alpine`; `packages/proto`, `packages/platform` and `packages/clients` mounted into the relevant services; gRPC ports published; new `gateway` service (`8090`) wired to the four Go services.
- `README.md`: contracts table, generated-module description, HTTP/gRPC port table, a “Service authentication” section, a “Service-to-service gRPC and the gateway” section, and refreshed status/gaps.

### Security

- **gRPC surface is now authenticated.** The gRPC servers previously accepted unauthenticated calls, and several RPCs take a `user_id`/`driver_id` in the payload (e.g. wallet top-up/payout). Every Go gRPC server now runs the shared interceptor; only intentional public RPCs are exempt. Handlers derive the caller's identity from the token and reject a mismatched payload id.
- **`fulfillment-service` HTTP auth was a no-op stub** that accepted every request as `system-user`. It now validates the session token like the other services (a valid JWT is required).
- **Algorithm-confusion hardening:** all JWT validation now pins `HS256` and rejects `alg=none`/unsigned tokens.
- `marketplace-c2c` now shares one HS256 verification implementation (`src/security/session.ts`) between its HTTP guard and its gRPC `CreateListing` handler, which requires a valid session token and derives the seller from it.

### Notes

- `@grpc/grpc-js` / `@grpc/proto-loader` resolve from the root `node_modules`; `grpc-js` 1.9.x requires `server.start()` after `bindAsync`.
- The marketplace (NestJS) services are not part of `infra/docker-compose.yml`; start them separately (see `services/MARKETPLACE_LOCAL_SETUP.md`) and point the gateway at their gRPC ports.

### Verification

- `packages/proto`: `go build ./...`.
- `packages/platform`: `go vet`, `go test ./...` pass.
- `packages/clients`: `go vet`, `go test ./...` pass.
- `services/gateway`: `go build`, `go vet`, `go test ./...` pass (including the end-to-end client↔server test).
- `services/auth-service`, `services/wallet-service`, `services/dispatch-service`, `services/fulfillment-service`: `go build`, `go vet`, `go test ./...` pass.
- `services/marketplace-b2b`, `services/marketplace-c2c`: `npm run build` pass.
- `proto`: `buf lint` clean.
- `infra/docker-compose.yml`: `docker compose config` valid.
