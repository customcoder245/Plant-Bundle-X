# Plant-Bundle-X — Knowledge Transfer (KT)

Version: 2.0
Date: 2026-06-30
Prepared for: customcoder245

What this repo does (quick)
--------------------------
Plant-Bundle-X is a Shopify app that sells plants as bundled products with pots, keeping separate inventory for plant variants and a global pool for pots. It integrates with Shopify via app endpoints and webhooks, provides an admin dashboard, and includes a theme extension for product pages (pot color selector and composite images).

Stack
-----
- Language(s): JavaScript (Node.js backend + React frontend), Liquid templates for theme extension
- Runtime / Frameworks: Node.js + Express (backend), React + Vite (frontend), PostgreSQL for persistence
- Notable files that anchor the stack:
  - README: https://github.com/customcoder245/Plant-Bundle-X/blob/main/README.md
  - server entry: https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/index.js
  - client app: https://github.com/customcoder245/Plant-Bundle-X/blob/main/client/src/App.jsx
  - templates: https://github.com/customcoder245/Plant-Bundle-X/blob/main/templates/product.json

Repository layout (top-level)
-----------------------------
```
.env.example                 # example env vars
README.md
package.json
railway.json                 # deployment config
shopify.app.toml             # Shopify app manifest
KT/                          # Knowledge Transfer (this document lives here)
client/                      # Frontend React app (Vite)
  package.json
  src/
    App.jsx
    main.jsx
    pages/                    # React pages for admin dashboard
server/                      # Express backend
  index.js                   # app bootstrap and routing
  get-token.js
  register-webhooks.js
  routes/                    # HTTP route handlers (one file per feature)
    activity.js
    analytics.js
    appSettings.js
    auth.js
    collections.js
    images.js
    inventory.js
    noPotDiscounts.js
    plantInventory.js
    potPrices.js
    pots.js
    productConfig.js
    products.js
    webhookAdmin.js
    webhooks.js
  services/                  # Business logic and helpers
    activityService.js
    bundleSetup.js
    collectionService.js
    inventoryService.js
    legacyConvert.js
    potPricingService.js
    settingsService.js
    sizeRules.js
    webhookService.js
templates/
  product.json               # Shopify product template used for bulk product creation
scripts/
  various helper scripts (db / maintenance)
other top-level: several FIX_NOTES and UPDATE_NOTES markdowns
```

How it fits together (runtime shape)
-----------------------------------
- server/index.js boots the Express app, registers middleware and mounts the routes under appropriate paths (auth, webhooks, admin APIs, public images).
- Each routes/*.js file exposes a set of REST endpoints that perform input validation, call into the corresponding service in server/services/*, and return JSON responses.
- server/services/* contains business logic (inventory calculations, pot pricing, product configuration, composite image handling, webhook side-effects).
- client/src contains the admin UI used by store admins (collection selection, product config, inventory pages). The client talks to server APIs for data and actions.
- templates/product.json is used to provision Shopify product structures when onboarding or syncing selected collections.

Database and primary schema
---------------------------
The README lists the core tables and fields; the important entities and their roles are:

- pot_colors
  - columns: id, name, hex_code, display_order, is_active
  - purpose: manages available pot colors for selection and composite image generation

- pot_inventory
  - columns: id, pot_color_id, size, quantity, low_stock_threshold
  - purpose: global pool of pots by color & size (shared across plants)

- plant_inventory
  - columns: id, product_config_id, shopify_variant_id, size, sku, barcode, quantity, low_stock_threshold
  - purpose: per-plant-variant inventory; each plant/size is tracked independently

- product_pot_config (aka product_config)
  - columns: id, shopify_product_id, product_title, is_enabled, no_pot_discount
  - purpose: per-Shopify-product configuration for bundle behavior

- size_mappings
  - columns: id, product_config_id, shopify_variant_id, variant_title, pot_size
  - purpose: maps Shopify variant titles/IDs to pot sizes used for pricing and inventory

- synced_collections
  - columns: id, shopify_collection_id, title, handle
  - purpose: which collections to import / sync from store

- composite_images
  - columns: id, product_config_id, pot_color_id, size, image_url
  - purpose: links plant+pot composite images for product pages

- activity_log
  - columns: id, event_type, description, metadata
  - purpose: audit trail and admin-facing activity feed

Primary server APIs (detailed reference)
---------------------------------------
Below I map the route files to the API surface and describe expected endpoints, parameters, behavior, and related services. Use these as the authoritative API reference while developing.

Base note: All endpoints serve JSON and expect standard REST verbs. Authentication for admin APIs is the app auth flow (see server/auth.js and get-token.js).

1) Authentication & App bootstrap
- server/routes/auth.js (file: server/routes/auth.js)
  - Expected endpoints:
    - GET /auth/start or GET /auth -> Kick off OAuth flow with Shopify
    - GET /auth/callback -> OAuth callback to exchange code for tokens
  - Purpose: manage Shopify OAuth and session tokens for stores.
  - Related files: server/get-token.js

2) Webhooks
- server/routes/webhooks.js (server/routes/webhooks.js)
  - Endpoints:
    - POST /webhooks/orders/create
    - POST /webhooks/orders/cancelled
    - POST /webhooks/orders/refunded
    - Possibly other Shopify webhook endpoints (app/uninstalled)
  - Behavior:
    - Validate Shopify HMAC signature
    - Parse order payload
    - For orders/create: deduct plant inventory and pot inventory according to the bundle configuration and whether the customer selected "no pot" (apply noPot discount)
    - For orders/refunded/cancelled: optionally restore inventory depending on business rules
  - Calls into: server/services/webhookService.js and server/services/inventoryService.js
  - Dev note: register webhooks using admin UI or server/register-webhooks.js

3) Webhook admin & registration
- server/routes/webhookAdmin.js
  - Endpoints:
    - POST /admin/webhooks/register -> register app webhooks for a given store
    - GET /admin/webhooks -> list registered webhooks
  - Purpose: administrative webhook management

4) Product and product configuration APIs
- server/routes/productConfig.js
  - Endpoints:
    - GET /product-config/:productId -> fetch config for product
    - POST /product-config -> create/update product pot config (is_enabled, no_pot_discount, size mappings)
    - DELETE /product-config/:productId -> disable product configuration
  - Behavior:
    - Validate that product is in a synced collection (or allow enabling individually)
    - Maintain size mappings and composite image links via product_pot_config, size_mappings and composite_images tables
  - Calls into: bundleSetup.js, settingsService.js, sizeRules.js

5) Products listing & sync
- server/routes/products.js (this file is large — see file)
  - Endpoints:
    - GET /products?collection= -> list products or synced products
    - POST /products/sync -> sync a collection (import product & variant metadata)
    - GET /products/:id -> product details including config, variants, images
  - Behavior:
    - Sync imports product metadata into product_config-like records
    - Create default mappings when possible
  - Calls into: collectionService.js, legacyConvert.js

6) Collections
- server/routes/collections.js
  - Endpoints:
    - GET /collections -> list available Shopify collections
    - POST /collections/sync -> mark collection as synced and import products
    - DELETE /collections/:id -> un-sync a collection
  - Calls into: collectionService.js

7) Inventory & plant inventory endpoints
- server/routes/inventory.js and server/routes/plantInventory.js
  - Endpoints:
    - GET /inventory/pots -> return pot inventory summary
    - GET /inventory/plants -> return plant inventory summary
    - POST /inventory/plants/:variantId/adjust -> manual adjust plant quantity
    - POST /inventory/pots/:color/:size/adjust -> manual adjust pot quantity
  - Behavior:
    - Expose low stock thresholds and quantities for admin dashboard
    - Support CSV uploads or bulk operations (common pattern)
  - Calls into: inventoryService.js (core logic for deducting, rolling back, validating quantities)

8) Pots & pot pricing
- server/routes/pots.js
  - Endpoints:
    - GET /pots -> list available pots and colors
    - POST /pots -> add pot color/size
    - DELETE /pots/:id -> remove pot color
  - server/routes/potPrices.js
    - GET /pot-prices -> price lookup for pot size & color
    - POST /pot-prices -> adjust pricing rules
  - Calls into: potPricingService.js and settingsService.js

9) Images and composite image management
- server/routes/images.js
  - Endpoints:
    - GET /images/:productId -> list composite images
    - POST /images/compose -> create a composite plant+pot image (probably asynchronous)
    - GET /images/serve/:imageId -> serve composite image URL or redirect to S3/CDN
  - Behavior:
    - Composite image generation may use server-side image processing and store URLs in composite_images table
    - Admin UI to regenerate composites for new pot colors or sizes
  - Calls into: image-related utilities in services (some logic may be in bundleSetup.js)

10) Analytics & activity
- server/routes/analytics.js and server/routes/activity.js
  - Endpoints:
    - GET /analytics/sales -> sales and conversion analytics around pots & bundles
    - GET /activity -> admin activity logs
  - Calls into: activityService.js

11) No-pot-discounts
- server/routes/noPotDiscounts.js
  - Endpoints:
    - GET /discounts/no-pot -> shows discount rules
    - POST /discounts/no-pot -> create/update discount threshold/rules
  - Behavior:
    - Add discount or override logic when a customer opts out of a pot
    - Discounts applied either via Shopify discount API or cart-level price adjustments through theme JS

12) Webhook & background job patterns
- register-webhooks.js helps bootstrap webhook subscriptions.
- Many operations (image composition, heavy syncs) may be implemented asynchronously (queued or background scripts under scripts/).

Core business logic (services) — what they do
---------------------------------------------
Files under server/services implement the rules used by the routes. Key services and responsibilities:

- inventoryService.js (server/services/inventoryService.js)
  - Centralized inventory operations: reserve/deduct, restore, atomic adjustments across plant_inventory and pot_inventory.
  - Handles low-stock detection and emits activity logs.
  - Exposes methods like deductForOrder(order), restoreForCancellation(order), adjustPlantQuantity(variantId, delta)

- potPricingService.js
  - Pricing rules for pots by size/color
  - Computes final bundle price when plant+pot combined, or discount applied when “no pot” is chosen.

- bundleSetup.js
  - Setup utilities to bootstrap product configurations for new stores or when syncing a collection.
  - Creates product_config rows, default size mappings, generates composite image placeholders.

- collectionService.js
  - Handles syncing of Shopify collections and their products into product_config.
  - Deals with pagination of Shopify APIs and mapping product variants to size rules.

- sizeRules.js
  - Contains mapping logic: given a variant title or size label, map to canonical pot size.
  - Contains heuristics used on import/legacy data transformation (legacyConvert.js references)

- webhookService.js
  - Lightweight wrapper around webhook processing: signature validation, parsing, and orchestration to call inventoryService logic, activity logging, and any follow-up tasks.

- activityService.js
  - Writes entries into activity_log and supports retrieval for admin pages and audit.

- legacyConvert.js
  - Scripts and helpers for migrating legacy product data into the new product_config/size_mappings format.

Client (admin UI) — pages and UX
-------------------------------
Key files:
- Client entry: client/src/main.jsx
- Root app: client/src/App.jsx
- client/src/pages/ (directory contains the admin pages; open to inspect exact filenames)

High-level pages you will find / implement:
- Dashboard — KPI overview (total bundles sold, low-stock alerts, recent activity)
- Products / Product Config page — list synced products and per-product bundle settings (enable/disable, map variants to pot sizes, set no-pot discount)
- Collections page — pick which Shopify collections to sync
- Pot Inventory page — manage pot color rows, sizes, quantities
- Plant Inventory page — manage plant variant quantities and SKUs
- Composite Images page — view/regenerate plant+pot images
- Settings / App Settings — global settings and webhook registration

Client-server interactions:
- Client calls the REST endpoints listed above to fetch product lists, inventory, and to trigger actions (sync collections, regenerate images).
- App uses Polaris components (mentioned in README) to build admin UIs that match Shopify look-and-feel.

Theme Extension & product page UX
---------------------------------
- A theme extension (Liquid assets and JS) provides a pot color selector widget on Shopify product pages.
- customer choices:
  - choose pot color
  - choose pot size
  - opt out of a pot (apply noPot discount)
- The extension communicates with storefront (cart) or with Shopify cart scripts; it uses composite images (plant+pot) to preview bundles. Template and extension manifest: shopify.app.toml and templates/product.json.

Inventory flow for an order (detailed)
-------------------------------------
Typical flow when an order is placed:
1. Shopify posts order data to /webhooks/orders/create
2. server/routes/webhooks.js validates and forwards payload to webhookService
3. webhookService:
   - Parses line items and detects products configured for bundles
   - For each bundle line item:
     - Determine which plant variant(s) and pot size/color were selected (using variant IDs and variant metadata)
     - If "no pot" option selected: apply configured no-pot discount and only deduct plant inventory
     - Otherwise: call inventoryService.deductForOrder which:
       - Deducts plant_inventory for each plant variant
       - Deducts pot_inventory for corresponding pot_color_id & size (global pool)
       - Ensures atomicity: if pot inventory short, handle according to policy (hold, partial fulfill, notify admin)
4. webhookService logs activity entries to activity_log via activityService
5. If composites need to be updated (no), image generation is not triggered by order; images are admin-managed

Edge cases and policies to consider (the repo contains logic for some)
- Race conditions on pot inventory: global pool requires atomic DB updates or advisory locks to avoid oversell.
- Partial fulfillment: what to do if plant available but pot out of stock; code should detect and either:
  - Fulfill plant-only and notify, or
  - Block fulfillment and notify admin
- Refunds & cancellations: whether to restore pot inventory is configurable (see README and settingsService)
- Shopify cart vs order: theme extension handles cart UX — server uses webhooks on order creation for inventory operations (ensures finality)

Files and code locations (quick links)
-------------------------------------
- README: https://github.com/customcoder245/Plant-Bundle-X/blob/main/README.md
- Server entry: https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/index.js
- All routes (see these files in repo):
  - server/routes/products.js (largest): https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/routes/products.js
  - server/routes/productConfig.js: https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/routes/productConfig.js
  - server/routes/inventory.js: https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/routes/inventory.js
  - server/routes/plantInventory.js: https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/routes/plantInventory.js
  - server/routes/pots.js: https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/routes/pots.js
  - server/routes/potPrices.js: https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/routes/potPrices.js
  - server/routes/images.js: https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/routes/images.js
  - server/routes/webhooks.js: https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/routes/webhooks.js
  - other routes: auth.js, collections.js, analytics.js, activity.js, webhookAdmin.js, noPotDiscounts.js
- Services:
  - server/services/inventoryService.js: https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/services/inventoryService.js
  - server/services/potPricingService.js: https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/services/potPricingService.js
  - server/services/bundleSetup.js: https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/services/bundleSetup.js
  - server/services/collectionService.js: https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/services/collectionService.js
  - other services: sizeRules.js, webhookService.js, activityService.js, legacyConvert.js, settingsService.js

Developer setup & running (actual commands)
------------------------------------------
From README (shortest path):

```bash
# clone
git clone https://github.com/customcoder245/Plant-Bundle-X.git
cd Plant-Bundle-X

# install root dependencies
npm install

# install client deps
cd client
npm install
cd ..

# configure env
cp .env.example .env
# edit .env with Shopify API key, secret, DB URL, etc.

# setup DB (migrations/seed)
npm run db:migrate
npm run db:seed

# run dev environment (concurrently runs server + client or as configured)
npm run dev
```

Key env variables (placeholders)
- SHOPIFY_API_KEY
- SHOPIFY_API_SECRET
- SHOPIFY_APP_URL (public URL)
- DATABASE_URL (Postgres)
- NODE_ENV
- PORT

Deployment
----------
- railway.json hints at Railway deployment (see root)
- Use the shopify.app.toml manifest to create/update the app in Shopify partner dashboard
- Ensure webhook endpoints are reachable externally and registered with the store

Testing and maintenance scripts
-------------------------------
- There are scripts for DB checks and maintenance at project root: check_db.js, test_db.js, fix_db.js, clear_old_data.js, sync_pots_inventory.js
- There are many FIX_NOTES and UPDATE_NOTES files documenting recent fixes and migration notes — read them for context before making large changes.

Design decisions & rationale (explicit)
--------------------------------------
- Separate inventories: plants are per-variant to track SKU/barcode data; pots are a shared pool to simplify stock and cost management — this is why inventoryService coordinates across two tables.
- Templates and theme extension separate frontend customer experience from the admin app; admin config writes DB and composite images consumed by theme.
- Heavy syncs (collection/product import) are separated into collectionService and legacyConvert to allow incremental adoption and data transformation.
- Composite images are pre-generated and stored (not generated at read-time) to keep storefront load fast.

Common maintenance tasks & checklists
------------------------------------
- Adding support for a new pot color:
  1. Add row in pot_colors (admin UI or DB)
  2. Add initial pot_inventory record for sizes
  3. Generate composite images for products using the new color
  4. Verify price rules in potPricingService

- Addressing oversell race:
  1. Inspect inventoryService for atomic operations
  2. Add DB transaction / SELECT FOR UPDATE or use advisory locks around pot inventory updates
  3. Add tests simulating concurrent webhook processing

- Regenerating product composites:
  1. Admin action triggers images.compose
  2. Composite is written and composite_images table updated
  3. Templates or product metafields updated to reflect new image URL

Known TODOs & risks (derive from repo artifacts)
------------------------------------------------
- Ensure inventoryService handles concurrent webhooks safely — potential for oversell if not transactional.
- Improve error handling and retries for webhook processing (idempotency).
- Add unit tests for potPricingService and sizeRules heuristics to prevent pricing regressions.
- Move large sync operations to background jobs to avoid request timeouts.
- Add more explicit documentation for env variables and DB migration steps (README references migrations but exact commands depend on tooling used).

Onboarding checklist (practical)
-------------------------------
- [ ] Clone and run dev environment locally
- [ ] Run DB migrations & seeds
- [ ] Register webhooks locally (use an ngrok-like tunnel) and test webhook processing with a sample order payload
- [ ] Review server/services/inventoryService.js and run unit tests around deductions
- [ ] Run the client, open admin pages and confirm product listing and inventory pages render
- [ ] Create a small PR fixing a README typo or add a missing doc entry (practice flow)

Appendix: sample API call examples (patterns)
--------------------------------------------
GET products (list)
- Request: GET /products?collection=12345
- Response: { products: [{ id, title, variants: [{id, title, sku}], config: { is_enabled, no_pot_discount } }, ...] }

POST sync collection
- Request: POST /products/sync
  - body: { collectionId: 12345 }
- Response: { importedCount: N, errors: [] }

POST order webhook (Shopify)
- Shopify sends the raw order JSON to POST /webhooks/orders/create
- server validates hmac using SHOPIFY_API_SECRET
- server returns 200 OK after processing

Where to look next (concrete files to inspect first)
----------------------------------------------------
1. server/index.js — app entry, middleware, route mounting
   - https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/index.js
2. server/services/inventoryService.js — core inventory logic
   - https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/services/inventoryService.js
3. server/routes/webhooks.js — webhook entry points
   - https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/routes/webhooks.js
4. server/routes/products.js — how products are synced and represented
   - https://github.com/customcoder245/Plant-Bundle-X/blob/main/server/routes/products.js
5. client/src/App.jsx and pages/ — admin UI entry and pages
   - https://github.com/customcoder245/Plant-Bundle-X/blob/main/client/src/App.jsx

End notes
---------
This document is intended to be a comprehensive KT you can use at onboarding, in design reviews, or as a base for written developer docs. If you want, I can:
- expand any API endpoint into exact parameter lists and example responses extracted from the code,
- produce a PDF version of this KT,
- or create a condensed one-page quickstart for non-technical stakeholders.

If you want the KT edited (add owners, security notes, or fill in exact client page filenames), tell me what to include and I will update the document accordingly.
