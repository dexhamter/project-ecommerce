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

## Events and Mechanisms

**`variant:changed`**:
A custom DOM event dispatched by the Dawn theme when the user selects a different product variant. Used to re-fire the `view_item` event with the newly selected variant's data.

**Variant ID Comparison Guard**:
The mechanism that prevents duplicate `view_item` pushes when `variant:changed` fires. Compares the incoming variant ID (cast to string) against `window._kovaProductData.item_id` (cast to string). Skips the push if IDs match; fires and updates `window._kovaProductData` if they differ. Handles both init-time fires and no-op re-selections.
_Avoid_: dedup guard, double-fire guard

**GCLID Bridge**:
The Cart Attributes mechanism that carries the `_gcl_aw` cookie value from Storefront cookies into the Checkout sandbox. A Custom HTML tag writes the GCLID to Shopify cart attributes via `/cart/update.js` on Window Loaded. The Custom Pixel reads it from `checkout.attributes` and includes it in every sGTM payload.
_Avoid_: GCLID cart stamp, attribution bridge
