# 2.8 — Add to Cart Tracking (`add_to_cart`)

## What This Does & Why

The `add_to_cart` event is the first hard purchase-intent signal in the ecommerce funnel. It fires when a customer successfully adds a product to their cart — not when they click the button, but when Shopify confirms the add with a 2xx HTTP response. This distinction matters enormously: Dawn's "Add to Cart" button fires a click event even when the add fails (out of stock, network error, 422 Unprocessable Entity). A GTM click trigger would count those failures as intent signals and inflate Smart Bidding's ROAS model with phantom conversions.

For Google Ads, `add_to_cart` populates dynamic remarketing audiences at SKU level — the `GAds - Remarketing` tag already fires on this event via the regex trigger established in 2.5. Every customer who adds a specific variant to cart without purchasing becomes addressable with that exact product in a dynamic ad. The event also anchors GA4's ecommerce funnel analysis between `view_item` and `begin_checkout`, making drop-off between those stages measurable for conversion rate optimisation.

## Prerequisites

- [ ] `v2.5.0 - Google Ads Foundation` published in GTM container `GTM-NMPNZ4TV`
- [ ] `snippets/tracking-product.liquid` exists with `view_item` block and `window._kovaProductData` populated (2.6 complete)
- [ ] `window._kovaProductData` ownership established — defined in the `view_item` block, never redefined by other intercepts (ADR-0002)
- [ ] `GA4 - Ecommerce Events` tag (All Custom Events) and `GAds - Remarketing` (regex trigger) already cover `add_to_cart` — no new GTM tags required
- [ ] GA4 property `G-JPWF3JGT5P` active, `G-JPWF3JGT5P` measurement ID in GTM constant variable

## Business Requirement

Fire a GA4-compliant `add_to_cart` event — including full item parameters and the exact quantity added — only when Shopify confirms a cart add succeeds, so that dynamic remarketing audiences are populated at SKU level and the GA4 ecommerce funnel between `view_item` and `begin_checkout` is accurately measurable.

## Data Layer Specification

### Event Name
`add_to_cart`

### Event Parameters

| Parameter | Type | Example Value | Notes |
|-----------|------|---------------|-------|
| `currency` | string | `"GBP"` | Hardcoded — single-currency GBP store |
| `value` | number | `69.98` | `price × quantity` — computed locally |
| `items[0].item_id` | string | `"49500686057695"` | Shopify variant ID — from `window._kovaProductData.item_id` |
| `items[0].item_group_id` | number | `9389168066783` | Shopify parent product ID — from `window._kovaProductData.item_group_id` |
| `items[0].item_name` | string | `"Athletic Performance Tee"` | From `window._kovaProductData.item_name` |
| `items[0].item_brand` | string | `"Kova Studio"` | From `window._kovaProductData.item_brand` |
| `items[0].item_category` | string | `"Activewear"` | Shopify Product Type — from `window._kovaProductData.item_category` |
| `items[0].item_variant` | string | `"S / Black"` | From `window._kovaProductData.item_variant` — reflects current variant after any switch |
| `items[0].price` | number | `34.99` | From `window._kovaProductData.price` |
| `items[0].quantity` | integer | `2` | From request FormData (`.get('quantity')`), defaults to `1` if absent/invalid |

**Important:** `ecommerce: null` push must immediately precede the `add_to_cart` push to clear the previous event's ecommerce object from the dataLayer.

### Full dataLayer Code

[Screenshot: `snippets/tracking-product.liquid` open in editor — show the add_to_cart intercept block at the bottom of the file]

```javascript
// Add to cart intercept — fires add_to_cart on confirmed AJAX success.
// Dawn uses fetch() for cart operations, so we monkey-patch window.fetch.
// We do NOT read the response body — response.ok is sufficient to gate on success,
// and all required event parameters are available from the request and window._kovaProductData.
// See ADR-0009 (fetch intercept over GTM click trigger) and ADR-0010 (request quantity).
(function() {
  var _origFetch = window.fetch;

  window.fetch = function(input, init) {
    var promise = _origFetch.apply(this, arguments);

    // Only intercept /cart/add calls.
    // Dawn POSTs to /cart/add (not /cart/add.js) — indexOf matches both variants.
    var url = typeof input === 'string' ? input : (input && input.url) || '';
    if (url.indexOf('/cart/add') === -1) return promise;

    promise.then(function(response) {
      // Gate on HTTP success — response.ok is false for 422 (out of stock), 5xx, etc.
      // Reading response.ok does not consume the body stream. Dawn's own .json() is unaffected.
      if (!response.ok) return;

      // Guard: bail silently if the product data store is missing or incomplete.
      // A missing event is a known gap; a corrupt event (undefined fields, NaN value)
      // is invisible noise that cannot be cleaned up retroactively.
      var p = window._kovaProductData || {};
      if (!p.item_id) return;

      // Parse quantity from the request FormData — this is what was added,
      // not the running cart total returned in the response body. See ADR-0010.
      // Dawn submits /cart/add as FormData with top-level 'quantity' key.
      // Guard: treat 0, negative, unparsable, and missing values as 1 (GA4 requires positive integer).
      var rawQty = (init && init.body instanceof FormData)
        ? parseInt(init.body.get('quantity'), 10)
        : NaN;
      var quantity = rawQty > 0 ? rawQty : 1;

      var price = p.price || 0;
      var value = price * quantity;

      dataLayer.push({ ecommerce: null });
      dataLayer.push({
        event: 'add_to_cart',
        ecommerce: {
          currency: 'GBP',
          value:    value,
          items: [{
            item_id:       p.item_id,
            item_group_id: p.item_group_id,
            item_name:     p.item_name,
            item_brand:    p.item_brand,
            item_category: p.item_category,
            item_variant:  p.item_variant,
            price:         price,
            quantity:      quantity
          }]
        }
      });
    });

    return promise;
  };
})();
```

**Placement:** This block is appended inside the same `<script>` tag in `snippets/tracking-product.liquid`, immediately after the closing `})();` of the `replaceState` IIFE (the variant switch listener). Do not add a new `<script>` tag — keep the entire file as one script block.

**Critical architecture notes:**

- The intercept reads from `window._kovaProductData` — it must NEVER redefine it. The object is owned by the `view_item` block (ADR-0002). If the user switched variants before adding to cart, the `replaceState` listener will have already updated `window._kovaProductData` with the correct variant's data. The intercept always reflects the current variant.
- `window.fetch` is patched before Dawn's own JavaScript modules execute (inline script in `<head>` via Custom Liquid Block). Dawn's fetch calls pick up the patch because they reference `fetch` through the global scope at call time, not at module definition time.
- The original `promise` is returned to Dawn unchanged. Our side-effect `.then()` runs independently — Dawn's cart drawer update is never interrupted.

## GTM Setup

**No new GTM work required for this subproject.**

The existing `GA4 - Ecommerce Events` tag (trigger: All Custom Events) and `GAds - Remarketing` tag (trigger: regex `view_item_list$|select_item$|view_item$|add_to_cart$|...`) were configured in 2.5 to handle all ecommerce events including `add_to_cart`. The dataLayer push from the intercept is picked up automatically.

[Screenshot: GTM Preview event list showing `add_to_cart` in the left panel — confirms All Custom Events trigger fired]

[Screenshot: `GA4 - Ecommerce Events` tag detail for the `add_to_cart` event — show Firing Status: Succeeded and Send Ecommerce data: true]

[Screenshot: `GAds - Remarketing` tag detail for the `add_to_cart` event — show regex trigger match and Firing Status: Succeeded]

### Tag Configuration

**No new tags created.**

Reference (existing tags that fire):

| Tag | Type | Trigger |
|-----|------|---------|
| `GA4 - Ecommerce Events` | Google Analytics: GA4 Event | All Custom Events (built-in) |
| `GAds - Remarketing` | Google Ads Remarketing | CE regex — `add_to_cart$` |

### Trigger Configuration

**No new triggers created.**

### Variable Configuration

N/A — no new variables.

### GTM Version

**No new GTM publish for this subproject.** Container stays at `v2.5.0 - Google Ads Foundation`. The regex trigger and All Custom Events trigger established in 2.5 handle `add_to_cart` without modification.

## GA4 Configuration

[Screenshot: GA4 DebugView showing `add_to_cart` event with ecommerce parameters expanded]

- **Event name:** `add_to_cart`
- **Marked as key event:** No — `add_to_cart` is a funnel signal, not a conversion. Purchase (`add_to_cart` → `begin_checkout` → `purchase`) is where conversion actions are defined. Marking `add_to_cart` as a key event would inflate conversion counts and distort Smart Bidding.
- **Custom dimensions registered:** None — all fields (`item_id`, `item_variant`, `item_category`, etc.) are GA4 native ecommerce item parameters, parsed automatically from the `items[]` array. Do not attempt to register them as custom dimensions (GA4 blocks registration of reserved parameter names).
- **Where it appears:** GA4 Admin → Reports → Monetization → Ecommerce purchases (funnel). Also visible in Explore → Funnel Exploration between `view_item` and `begin_checkout`.

## Google Ads Configuration

- **Conversion action:** None created for `add_to_cart`. Cart adds are micro-conversions used for audience building, not Smart Bidding signals.
- **Remarketing:** `GAds - Remarketing` tag fires on `add_to_cart` via the existing regex trigger. Sends `item_id` (via `CJS - GAds Items Array` — constructs `shopify_GB_{product_id}_{variant_id}`), `value`, and `currency` to Google Ads. Populates dynamic remarketing audiences for "added to cart but didn't purchase" segments.

[Screenshot: Google Ads Audience Manager — confirm `add_to_cart` is visible as a signal in remarketing audience rules (once sufficient data accumulates)]

## Validation Steps

[Screenshot: GTM Preview — left panel showing `add_to_cart` event in the event sequence after clicking Add to Cart]

1. Open GTM Preview → connect to the dev theme URL (`shopify theme dev` preview URL)
2. Navigate to any product detail page
3. **Scenario 1 — Basic add (qty 1):** Click Add to Cart. Confirm `add_to_cart` appears in the GTM Preview left panel. Click it → Data Layer tab → verify `currency: "GBP"`, `value` matches the product price, `quantity: 1`, all item fields populated.
4. **Scenario 2 — Qty > 1:** Change the quantity picker to 2, click Add to Cart. Confirm `quantity: 2` and `value = price × 2` in the dataLayer. This validates ADR-0010 — if quantity were incorrectly read from the response body, this would only pass if the cart was previously empty.
5. **Scenario 3 — Variant switch then add:** Select a different variant (confirm `view_item` re-fires with the new `item_id`). Then click Add to Cart. Confirm `add_to_cart` carries the switched variant's `item_id`, `item_variant`, and `price` — not the page-load defaults.
6. **Scenario 4 — Out of stock (if available):** Temporarily set a variant's inventory to 0 in Shopify Admin. Attempt to add it. Confirm `add_to_cart` does NOT appear in GTM Preview. This validates the `response.ok` gate (422 Unprocessable Entity).

[Screenshot: Data Layer tab for the `add_to_cart` event — show the full ecommerce object with all fields]

[Screenshot: Data Layer tab for the qty > 1 scenario — show `quantity: 2` and `value: [price × 2]`]

[Screenshot: Data Layer tab for the variant-switch scenario — show the switched variant's `item_id` and `item_variant`]

7. Open GA4 DebugView → confirm `add_to_cart` appears with correct parameters within ~30 seconds

[Screenshot: GA4 DebugView — `add_to_cart` event expanded showing ecommerce item parameters]

## QA Checklist

- [ ] `add_to_cart` fires on confirmed cart add (Shopify responds 200)
- [ ] `add_to_cart` does NOT fire on button click alone (no HTTP request yet)
- [ ] `add_to_cart` does NOT fire on a failed add (out of stock / 422)
- [ ] `quantity` reflects what was added — not hardcoded 1, not cart total (test with qty picker = 2)
- [ ] `value` = `price × quantity` (not just price)
- [ ] After variant switch, `add_to_cart` carries the switched variant's `item_id`, `item_variant`, and `price`
- [ ] `ecommerce: null` push precedes `add_to_cart` push in the dataLayer sequence
- [ ] `GA4 - Ecommerce Events` tag fires with `Firing Status: Succeeded` and `Send Ecommerce data: true`
- [ ] `GAds - Remarketing` tag fires on `add_to_cart` event
- [ ] GA4 DebugView shows `add_to_cart` with correct parameters
- [ ] No JS errors in browser console during cart add flow
- [ ] `window.fetch.toString()` includes `_origFetch` (confirms patch is active and hasn't been overwritten)
- [ ] No GTM publish required — container stays at `v2.5.0`

## Common Errors & Fixes

| Error / Symptom | Root Cause | Fix |
|----------------|------------|-----|
| `add_to_cart` never fires despite successful cart add | URL filter mismatch — Dawn calls `/cart/add`, not `/cart/add.js`. `indexOf('/cart/add.js')` misses it. | Change filter to `url.indexOf('/cart/add')` — matches both variants. Confirmed: Dawn POSTs to `/cart/add`. |
| `add_to_cart` fires with `item_id: undefined` and `value: 0` | `window._kovaProductData` was not populated before the intercept ran — tracking-product.liquid not included on this page | Verify the Custom Liquid Block is present in the product template JSON. Check `window._kovaProductData` in console before adding to cart. |
| `add_to_cart` fires on every page load | URL filter too broad — matching unrelated fetch calls | Narrow the filter — `url.indexOf('/cart/add')` is already specific; check for other fetch patches on the page that might be double-firing |
| `quantity` always 1 even when picker set to 2 | `init.body` is not a `FormData` instance (theme override using JSON body) | Log `typeof init.body` in the intercept. If JSON, switch to `JSON.parse(init.body).quantity`. |
| `add_to_cart` fires on out-of-stock add | `response.ok` check missing or positioned after the push | Ensure `if (!response.ok) return;` is the first check inside the `.then()` callback, before any data reads. |
| Cart drawer fails to update after add | `response` body stream consumed by our intercept | Confirm intercept never calls `response.json()` or `response.text()` — only `response.ok` is read. We do not call `.clone()`. |
| `window.fetch` reverts to native after page interaction | Dawn or a third-party script is re-assigning `window.fetch` after our patch | Run `window.fetch.toString()` in console after the add fails. If it returns `[native code]`, a later script is overwriting it. Move the patch to fire later or re-patch after the conflict. |
| `add_to_cart` fires twice on one click | Dawn fires two `/cart/add` requests (can happen with some bundle apps) | Add a timestamp-based debounce or a `Set` tracking recent pushes to deduplicate within 500ms. |

## Reusable Assets

- **Snippet:** `project-ecommerce/theme_export__fashion-sandbox-myshopify-com/snippets/tracking-product.liquid` (the `add_to_cart` intercept block is at the bottom, after the `replaceState` IIFE)
- **ADRs:** `project-ecommerce/docs/adr/0009-fetch-intercept-for-add-to-cart.md` · `project-ecommerce/docs/adr/0010-request-formdata-quantity-over-response-body.md`
- **Domain glossary:** `project-ecommerce/CONTEXT.md` — "Cart Add Intercept" term added in 2.8

## Related Guides

- [2.6 — Product View Tracking (`view_item`)](./06-product-view-tracking.md) — defines `window._kovaProductData` that this intercept reads
- [2.5 — Google Ads Foundation (Shopify)](./05-google-ads-foundation.md) — establishes the regex trigger and `GA4 - Ecommerce Events` tag that fire on `add_to_cart`
- [2.9 — Remove from Cart Tracking (`remove_from_cart`)](./09-remove-from-cart-tracking.md) — next event in the funnel; uses a similar `/cart/change.js` intercept pattern
- [GTM Ecommerce Events Architecture](../../google-ads-measurement-library/docs/guides/02-data-collection/gtm/ecommerce-events.md) — reusable guide for the single-tag/one-regex-trigger pattern
