# Ecommerce Tracking — Kova Studio (Shopify)

Ubiquitous language for the GA4 / Google Ads measurement implementation on the Kova Studio Shopify store. Terms here are canonical across all snippets, GTM tags, and documentation in this project.

---

## Environments

**Storefront**:
The customer-facing Shopify theme running at `fashion-sandbox.myshopify.com`. GTM runs here via `theme.liquid`. All storefront tracking events are pushed via `dataLayer.push()` from Tracking Snippets.
_Avoid_: frontend, theme, client-side

**Checkout**:
The sandboxed Shopify checkout UI at `fashion-sandbox.myshopify.com/checkouts/...`. GTM cannot run here. Tracking is handled exclusively via the Custom Pixel. Same domain as the Storefront, but an isolated execution environment.
_Avoid_: checkout page, order flow

---

## Tracking Infrastructure

**Tracking Snippet**:
A dedicated Liquid file (`snippets/tracking-*.liquid`) that owns all dataLayer pushes for a given page type. Injected via a Custom Liquid Block — never by editing section files directly.
_Avoid_: tracking script, snippet file

**Custom Liquid Block**:
A Theme Editor block that renders a Tracking Snippet. Stored in the template's JSON configuration, not in section `.liquid` files, so it survives theme updates.
_Avoid_: block, liquid block

**Custom Pixel**:
Shopify's sandboxed JavaScript environment for checkout event tracking. Subscribes to native Shopify checkout events and POSTs payloads directly to sGTM. Cannot read parent DOM, cookies, or `localStorage`.
_Avoid_: web pixel, pixel

**dataLayer**:
The GTM data layer array (`window.dataLayer`). Ecommerce events are pushed here from Tracking Snippets on the Storefront. Not accessible from the Custom Pixel.

**sGTM**:
The server-side GTM container hosted on Stape.io. Receives Storefront events relayed from client GTM, and Checkout events POSTed directly from the Custom Pixel. Forwards to GA4 and Google Ads.
_Avoid_: server container, server-side tag manager

---

## Data Structures

**Items Array**:
The GA4 ecommerce `items` array. Identical schema across Storefront and Checkout environments. Each entry represents one product variant. Fields: `item_id`, `item_group_id`, `item_name`, `item_brand`, `item_category`, `item_variant`, `price`, `quantity`.

**Ecommerce Null Push**:
The `dataLayer.push({ ecommerce: null })` that must precede every ecommerce event push. Clears the previous ecommerce object from the dataLayer to prevent GA4 from inheriting parameters across events.
_Avoid_: null push, clear push

**MoneyV2**:
Shopify's checkout price format — an object with `amount` (decimal number) and `currencyCode` (string). Always target `.amount`. Never pass the object itself as a price value.

**`window._kovaProductData`**:
The page-level product data store on product detail pages. Populated by the `view_item` Tracking Snippet at page load time using Liquid-rendered values. Consumed by the `add_to_cart` intercept (2.8) and the `variant:changed` listener. Contains the currently-selected variant's `item_id` and `price`, plus static parent fields (`item_group_id`, `item_name`, `item_brand`, `item_category`, `item_variant`).

---

## Identifiers

**`item_id`**:
The Shopify variant ID. The leaf-level identifier in the Items Array. Used to construct the GMC Composite ID.
_Avoid_: product ID, SKU

**`item_group_id`**:
The Shopify parent product ID. Distinguishes the base product from its variants. Used alongside `item_id` to construct the GMC Composite ID.
_Avoid_: product group ID

**GMC Composite ID**:
The Google Merchant Center product identifier in the format `shopify_GB_{product_id}_{variant_id}`. Constructed in GTM by the `CJS - GAds Items Array` Custom JavaScript variable. Used by the `GAds - Remarketing` tag for dynamic remarketing audience matching.
_Avoid_: composite ID, remarketing ID, feed ID

---

## List Tracking

**`window._kovaLists`**:
The page-level staging array for product list data on pages with multiple product sections (currently: homepage). Each section with a product grid pushes a GA4-formatted list object here via an inline `<script>` rendered by Liquid. Consumed by the Tracking Snippet at `DOMContentLoaded`. Parallel to `window._kovaProductData` on product detail pages.
_Avoid_: list array, product lists array

**`item_list_id`**:
The stable identifier for a product list. Equals `collection.handle` on Collection Pages, a hardcoded `section.id`-scoped string on homepage sections, and `"search_results"` on the Search Results page. Must match exactly on any subsequent `select_item` event for list attribution to hold.
_Avoid_: list ID, list identifier

**`item_list_name`**:
The human-readable label for a product list. Equals `collection.title` on Collection Pages, a hardcoded string on homepage sections, and `"Search Results"` on the Search Results page. Decoupled from display headings to prevent Theme Editor changes from fracturing GA4 list reports.
_Avoid_: list name, list label

---

## Page Types

**Collection Page**:
A Shopify page type at `/collections/{handle}` that renders a paginated product grid for a single collection. Fires `view_item_list` via the Collection Tracking Snippet. The `collection.handle` and `collection.title` Liquid objects are the canonical sources for `item_list_id` and `item_list_name` respectively.
_Avoid_: category page, product listing page, PLP

**Search Results Page**:
The Shopify search page at `/search?q=...`. Renders products from `search.results` filtered by type. Fires `view_item_list` with fixed identifiers (`"search_results"` / `"Search Results"`) regardless of the search query. Query intent is captured separately via the `search_term` GA4 parameter, not baked into the list identity.
_Avoid_: search page, search results

---

## Cart Add Intercept

**Cart Add Intercept**:
The mechanism by which the `add_to_cart` Tracking Snippet detects a confirmed cart add without relying on a button click. Monkey-patches `window.fetch`, filters for `/cart/add.js` calls, and gates on `response.ok`. Fires only on HTTP 2xx — out-of-stock 422s and network errors are excluded. Does not read the response body; all required event parameters are sourced from the request FormData and `window._kovaProductData`.
_Avoid_: click intercept, cart listener, fetch hook

---

## Events and Mechanisms

**`variant:changed`**:
A custom DOM event dispatched by the Dawn theme when the user selects a different product variant. Used to re-fire the `view_item` event with the newly selected variant's data.

**Variant ID Comparison Guard**:
The mechanism that prevents duplicate `view_item` pushes when `variant:changed` fires. Compares the incoming variant ID (cast to string) against `window._kovaProductData.item_id` (cast to string). Skips the push if IDs match; fires and updates `window._kovaProductData` if they differ. Handles both init-time fires and no-op re-selections.
_Avoid_: dedup guard, double-fire guard

**GCLID Bridge**:
The Cart Attributes mechanism that carries the `_gcl_aw` cookie value from Storefront cookies into the Checkout sandbox. A Custom HTML tag writes the GCLID to Shopify cart attributes via `/cart/update.js` on Window Loaded. The Custom Pixel reads it from `checkout.attributes` and includes it in every sGTM payload.
_Avoid_: GCLID cart stamp, attribution bridge
