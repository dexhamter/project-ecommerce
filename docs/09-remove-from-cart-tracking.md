# 2.9 — Remove from Cart Tracking (`remove_from_cart`) + Cart-Surface `add_to_cart`

## What This Does & Why

`remove_from_cart` closes the last gap in the pre-checkout ecommerce funnel. Without it, GA4 shows items entering the cart but has no signal for when they leave — making cart abandonment analysis unreliable and hiding the difference between customers who genuinely abandon at checkout and those who removed items before ever reaching it. For Google Ads, the same `GAds - Remarketing` tag that fires on `add_to_cart` also fires on `remove_from_cart` — so removed items repopulate dynamic remarketing audiences correctly, ensuring a customer who reconsidered a product remains addressable with it.

This subproject also completes `add_to_cart` coverage. Subproject 2.8 only captured adds via the product form (`/cart/add`). Dawn's cart page and cart drawer use a different endpoint — `/cart/change` — for all quantity adjustments, including quantity increases. The `+` button on the cart drawer fires `/cart/change` with a higher quantity, not `/cart/add`. This means every "add from cart" was missing until this subproject. The same `/cart/change` intercept that fires `remove_from_cart` on decreases also fires `add_to_cart` on increases, with the event direction determined purely by the sign of the quantity delta.

## Prerequisites

- [ ] `v2.5.0 - Google Ads Foundation` published in GTM container `GTM-NMPNZ4TV` — the `GA4 - Ecommerce Events` (All Custom Events) tag and `GAds - Remarketing` (regex trigger) that fire on `remove_from_cart` are already in this version; no new GTM tags required
- [ ] `snippets/tracking-product.liquid` exists and handles `/cart/add` intercept for product-form adds (2.8 complete)
- [ ] GA4 property `G-JPWF3JGT5P` active with measurement ID in GTM constant variable
- [ ] Shopify theme: Dawn. The intercept reads Dawn's JSON request body format for `/cart/change` — other themes may use FormData or a different API endpoint
- [ ] Access to `layout/theme.liquid` to render the snippet globally

## Business Requirement

Fire `remove_from_cart` when a customer decreases or removes a line item quantity from the cart page or cart drawer, and fire `add_to_cart` when a customer increases a line item quantity from the cart page or cart drawer — both confirmed by a 2xx HTTP response from Shopify's `/cart/change` endpoint — so that cart abandonment analysis is complete and dynamic remarketing audiences reflect item removal.

## Data Layer Specification

### Event Name

`remove_from_cart` (for quantity decreases and full removals)
`add_to_cart` (for quantity increases from the cart surface — not from the product form)

### Event Parameters

| Parameter | Type | Example Value | Notes |
|-----------|------|---------------|-------|
| `currency` | string | `"GBP"` | Hardcoded — single-currency store |
| `value` | number | `29.98` | `price × abs(delta)` — units affected only, not total cart value |
| `items[0].item_id` | string | `"49455622783199"` | Shopify variant ID — from `_kovaCartState.items[line-1].variant_id`, cast to String |
| `items[0].item_group_id` | number | `9389168066783` | Shopify parent product ID — from `_kovaCartState.items[line-1].product_id` |
| `items[0].item_name` | string | `"Essential Cotton Tee"` | From `_kovaCartState.items[line-1].product_title` |
| `items[0].item_brand` | string | `"Kova Studio"` | Hardcoded |
| `items[0].item_category` | string | `"Tops"` | From `_kovaCartState.items[line-1].product_type` |
| `items[0].item_variant` | string | `"M / White"` | From `_kovaCartState.items[line-1].variant_title` |
| `items[0].price` | number | `14.99` | `_kovaCartState.items[line-1].price / 100` — Shopify stores prices in cents |
| `items[0].quantity` | integer | `2` | `Math.abs(requestQty - oldQty)` — units removed/added, never negative |

**Important:** `ecommerce: null` push must immediately precede every ecommerce event push.

**Delta quantity rule:**
- `delta = requestQty - oldQty`
- `delta > 0` → `add_to_cart`, `quantity: delta`
- `delta < 0` → `remove_from_cart`, `quantity: Math.abs(delta)`
- `delta === 0` → no event (no-op change, e.g. re-submitting same quantity)

### Full dataLayer Code

[Screenshot: `snippets/tracking-cart.liquid` open in editor — show full file]

```javascript
{% comment %} snippets/tracking-cart.liquid {% endcomment %}
{% comment %} 2.9 — add_to_cart and remove_from_cart via /cart/change fetch intercept.
  Rendered globally in layout/theme.liquid (cart drawer appears on all pages).
  Handles quantity increases (add_to_cart) AND decreases / full removals (remove_from_cart).
  add_to_cart from the product form is handled separately in tracking-product.liquid via /cart/add.
  See ADR-0009, ADR-0011, ADR-0012, ADR-0013. {% endcomment %}
<script>
  window.dataLayer = window.dataLayer || [];

  // Cart state cache — populated on load and refreshed after every successful cart operation.
  // Provides pre-change item data (quantity, price, title, variant) for delta computation.
  // See ADR-0011.
  window._kovaCartState = window._kovaCartState || null;

  (function() {
    // Refresh helper — re-fetches /cart.js and replaces the full cache.
    // Called on load, after /cart/add responses, and as fallback if response clone fails.
    function refreshCartCache() {
      fetch('/cart.js')
        .then(function(r) { return r.json(); })
        .then(function(cart) { window._kovaCartState = cart; })
        .catch(function() {});
    }

    refreshCartCache();

    var _origFetch = window.fetch;

    window.fetch = function(input, init) {
      var promise = _origFetch.apply(this, arguments);

      var url = typeof input === 'string' ? input : (input && input.url) || '';

      // After any successful /cart/add, refresh cache so items added during the
      // session appear in _kovaCartState before the user changes them from the cart.
      // /cart/add returns only the added line item, not the full cart, so we re-fetch
      // /cart.js rather than parsing the response body. See ADR-0011.
      if (url.indexOf('/cart/add') !== -1) {
        promise.then(function(response) {
          if (response.ok) refreshCartCache();
        });
        return promise;
      }

      if (url.indexOf('/cart/change') === -1) return promise;

      // Parse JSON request body — Dawn sends {line, quantity, sections, sections_url}.
      // 'line' is 1-indexed. 'quantity' is the NEW (destination) quantity, not the delta.
      // Guard against missing or non-numeric values. See ADR-0012.
      // Dawn serialises 'line' as a string ("2"), not a number, even though
      // the Cart API accepts both. Use parseInt for both fields. See ADR-0012.
      var requestLine = NaN;
      var requestQty  = NaN;
      try {
        if (init && typeof init.body === 'string') {
          var bodyObj = JSON.parse(init.body);
          requestLine = parseInt(bodyObj.line,     10);
          requestQty  = parseInt(bodyObj.quantity, 10);
        }
      } catch(e) {}

      // requestQty may legitimately be 0 (full removal) — guard on NaN only.
      if (isNaN(requestLine) || requestLine < 1 || isNaN(requestQty)) return promise;

      // IMPORTANT: snapshot item data and oldQty SYNCHRONOUSLY here — before the async
      // .then() fires. preItem is a reference; any in-place cache update inside .then()
      // would mutate preItem.quantity before we read it, making the computed delta wrong.
      // Snapshotting oldQty now avoids that mutation bug entirely. See ADR-0011.
      var preItem = null;
      var oldQty  = null;
      if (window._kovaCartState && Array.isArray(window._kovaCartState.items)) {
        var candidate = window._kovaCartState.items[requestLine - 1];
        if (candidate) {
          preItem = candidate;
          oldQty  = candidate.quantity; // integer, captured before any async mutation
        }
      }

      promise.then(function(response) {
        if (!response.ok) return;

        // Refresh cache from the cloned response body (full cart JSON, includes items[]).
        // Our .then() fires before Dawn's .then(response => response.text()) because we
        // registered first, so response.clone() is safe here — body not yet consumed.
        // Falls back to refreshCartCache() if clone/parse fails. See ADR-0011.
        try {
          response.clone().json()
            .then(function(updatedCart) {
              if (updatedCart && Array.isArray(updatedCart.items)) {
                window._kovaCartState = updatedCart;
              }
            })
            .catch(function() { refreshCartCache(); });
        } catch(e) {
          refreshCartCache();
        }

        // No pre-change data → can't compute delta or build items[]. Silent fail.
        if (oldQty === null) return;

        // Delta: positive = quantity increased (add), negative = quantity decreased (remove).
        // Dawn uses /cart/change for both quantity increases and decreases from the cart
        // page and cart drawer. We fire the appropriate GA4 event based on direction.
        // add_to_cart from the product form is handled separately via /cart/add
        // in tracking-product.liquid. See ADR-0013.
        var delta = requestQty - oldQty;
        if (delta === 0) return; // no-op change — don't fire

        var price      = preItem.price / 100; // Shopify prices in cents
        var qty        = Math.abs(delta);
        var value      = Math.round(price * qty * 100) / 100;
        var eventName  = delta > 0 ? 'add_to_cart' : 'remove_from_cart';

        dataLayer.push({ ecommerce: null });
        dataLayer.push({
          event: eventName,
          ecommerce: {
            currency: 'GBP',
            value:    value,
            items: [{
              item_id:       String(preItem.variant_id),
              item_group_id: preItem.product_id,
              item_name:     preItem.product_title,
              item_brand:    'Kova Studio',
              item_category: preItem.product_type,
              item_variant:  preItem.variant_title,
              price:         price,
              quantity:      qty
            }]
          }
        });
      });

      return promise;
    };
  })();
</script>
```

## GTM Setup

[Screenshot: GTM workspace showing existing tags — confirm no new tags were created for 2.9]

### No new GTM configuration required

The existing `GA4 - Ecommerce Events` tag (fires on All Custom Events) and `GAds - Remarketing` tag (fires on regex trigger matching all GA4 ecommerce events) both cover `remove_from_cart` automatically. The regex trigger established in 2.5 was built to match all ecommerce event names, so `remove_from_cart` fires through it without any GTM changes.

**Container version:** `v2.5.0 - Google Ads Foundation` — unchanged. No new version was published for 2.9.

### Theme file changes

All changes for 2.9 are in the Shopify theme, not in GTM.

#### Step 1 — Create `snippets/tracking-cart.liquid`

Create the file at `snippets/tracking-cart.liquid` in the theme. Paste the full code block from the Data Layer Specification section above.

#### Step 2 — Render globally in `layout/theme.liquid`

Open `layout/theme.liquid`. Locate the closing `</body>` tag. Add the render call just before it (after the deferred cart-drawer JS block):

```liquid
{% comment %} 2.9 — remove_from_cart tracking (global — cart drawer appears on all pages) {% endcomment %}
{% render 'tracking-cart' %}
```

[Screenshot: `layout/theme.liquid` showing the render tag just before `</body>`]

**Why global, not PDP-only:** The cart drawer is accessible from every page of the Shopify storefront. If `tracking-cart.liquid` were only rendered on the product template, removes and quantity changes from the cart drawer on the homepage or collection pages would be untracked. `layout/theme.liquid` ensures the snippet is present regardless of which page the customer is on when they interact with the cart.

**Why not a Custom Liquid Block:** Unlike `tracking-product.liquid` (which is PDP-only and placed in the product template via a Custom Liquid Block), the cart drawer scope requires theme.liquid. A Custom Liquid Block exists only within a specific template and does not run on other page types.

### Tag Configuration

No new tags created.

**Existing tags that fire on `remove_from_cart`:**

| Tag Name | Type | Fires On |
|----------|------|----------|
| `GA4 - Ecommerce Events` | GA4 Event | All Custom Events trigger |
| `GAds - Remarketing` | Google Ads Remarketing | Regex trigger matching ecommerce event names |

### Trigger Configuration

No new triggers created. The All Custom Events trigger already covers `remove_from_cart`.

### Variable Configuration

No new variables created.

### GTM Version

**Version:** `v2.5.0 - Google Ads Foundation` (unchanged — no GTM publish required for 2.9)

## GA4 Configuration

[Screenshot: GA4 DebugView showing `remove_from_cart` event with ecommerce parameters expanded]

- **Event name:** `remove_from_cart`
- **Marked as conversion:** No
- **Custom dimensions registered:** None (item parameters flow via standard ecommerce schema)

The event appears in GA4 Reports → Ecommerce → Cart under the standard GA4 ecommerce funnel once it has accumulated data.

## Google Ads Configuration

[Screenshot: Google Ads → Tools → Conversions — confirm no new conversion action created for 2.9]

- **Conversion action:** None created for `remove_from_cart` (removal is not a conversion)
- **Dynamic remarketing:** `GAds - Remarketing` tag fires on `remove_from_cart` via the existing regex trigger — items removed from cart are eligible for dynamic remarketing audiences at SKU level

No Google Ads configuration changes were required for 2.9.

## Validation Steps

[Screenshot: GTM Preview — Summary panel showing `remove_from_cart` event in the event stream]

1. Open GTM Preview mode — paste `https://fashion-sandbox.myshopify.com` as the URL
2. Navigate to the cart page or open the cart drawer
3. Add an item to the cart if the cart is empty (to ensure `_kovaCartState` is populated before the removal)
4. Decrease the quantity of a line item using the `−` button, or click the trash/remove icon to set quantity to 0
5. Confirm `remove_from_cart` appears in the GTM Preview event stream
6. Click the event — verify `event` = `remove_from_cart` and check the `ecommerce` object:
   - `currency` = `"GBP"`
   - `value` = price × units removed (not total cart value)
   - `items[0].item_id` = correct Shopify variant ID (as string)
   - `items[0].item_name` = correct product title
   - `items[0].quantity` = number of units removed (absolute delta, not the new remaining qty)
   - `items[0].price` = unit price in GBP (not pence)
7. Confirm `ecommerce: null` push appears immediately before the `remove_from_cart` push in the dataLayer
8. Confirm `GA4 - Ecommerce Events` tag shows status **Succeeded**
9. Confirm `GAds - Remarketing` tag shows status **Succeeded**

[Screenshot: GTM Preview — Tags Fired panel for the `remove_from_cart` event showing GA4 and GAds tags succeeded]

10. Test `add_to_cart` from cart surface: increase a line item quantity using the `+` button in the cart drawer
11. Confirm `add_to_cart` appears in the event stream with the delta quantity (e.g., 1→3 = `quantity: 2`)

[Screenshot: GTM Preview — `add_to_cart` event from cart drawer, showing correct delta quantity in items[]]

12. In GA4 DebugView, confirm `remove_from_cart` event appears with correct parameters

**Validated result (2026-06-26):**
- `remove_from_cart` event 43 — Essential Cotton Tee, qty 2, value 29.98 GBP ✓
- `add_to_cart` event 41 — fired from cart surface ✓
- `GA4 - Ecommerce Events` Fired/Succeeded ✓
- `GAds - Remarketing` Fired/Succeeded ✓

## QA Checklist

- [ ] `remove_from_cart` fires when quantity is decreased via the `−` button on the cart page
- [ ] `remove_from_cart` fires when an item is removed (quantity set to 0) via the trash icon
- [ ] `remove_from_cart` fires when quantity is decreased via the `−` button in the cart drawer (any page)
- [ ] `add_to_cart` fires when quantity is increased via the `+` button in the cart drawer
- [ ] `add_to_cart` does NOT fire from the product form (that remains in `tracking-product.liquid`)
- [ ] No event fires on a no-op quantity change (submitting the same qty)
- [ ] `quantity` in the event equals `Math.abs(delta)`, not the new remaining quantity
- [ ] `value` equals `price × quantity` (units affected only), not the total cart value
- [ ] `item_id` is a string (not a number)
- [ ] `price` is in GBP, not in pence
- [ ] `ecommerce: null` push precedes every ecommerce event push
- [ ] `GA4 - Ecommerce Events` tag fires and succeeds
- [ ] `GAds - Remarketing` tag fires and succeeds
- [ ] No events fire on failed `/cart/change` calls (server returns 4xx/5xx)
- [ ] `window._kovaCartState` is populated before the first cart interaction (check in console after page load)
- [ ] Items added during the session (via `/cart/add`) appear correctly in `_kovaCartState` before being removed

## Common Errors & Fixes

| Error / Symptom | Root Cause | Fix |
|----------------|------------|-----|
| Nothing happens — no event in GTM Preview when removing from cart | `typeof bodyObj.line === 'number'` guard silently fails. Dawn serialises `line` as a **string** (`"2"`), not a number. Every `/cart/change` call bails at the guard. Confirmed via console: `{"line":"2","quantity":2,...}` | Use `parseInt(bodyObj.line, 10)` and guard with `isNaN`, never `typeof`. Do the same for `quantity`. |
| `remove_from_cart` fires with `quantity: 0` or wrong delta | Reference mutation bug: `preItem` is a reference to `_kovaCartState.items[line-1]`. If the cache is updated in-place inside the `.then()`, `preItem.quantity` changes before the delta is computed — `delta = requestQty - preItem.quantity = 0`. | Snapshot `oldQty = candidate.quantity` **synchronously** before the `.then()` block. Use `oldQty` in the delta calculation, never `preItem.quantity`. |
| `remove_from_cart` never fires for items added in the same session | `/cart.js` is fetched only once at page load. Items added via the product form during the session appear in Shopify's cart but not in the stale `_kovaCartState`. `preItem = null` → silent bail. | Intercept `/cart/add` responses: on `response.ok`, call `refreshCartCache()` to repopulate `_kovaCartState`. |
| `add_to_cart` not firing from cart page / drawer | The `+` button on the cart surface calls `/cart/change` (not `/cart/add`). The 2.8 intercept in `tracking-product.liquid` only handles `/cart/add`. | Handle both directions in `tracking-cart.liquid`: positive delta → `add_to_cart`, negative delta → `remove_from_cart`. |
| `remove_from_cart` event has quantity equal to new cart quantity, not units removed | Using `requestQty` (the destination quantity) as the event `quantity`, rather than the delta. E.g., going from 5 to 2 fires `quantity: 2` instead of `quantity: 3`. | `quantity` must be `Math.abs(requestQty - oldQty)` — the absolute delta. |
| `tracking-cart.liquid` render tag has no effect | Added in wrong template. `tracking-cart.liquid` must be in `layout/theme.liquid` so it runs on every page type. If placed in a product template or section file, it will not be active on the cart page or cart drawer on other page types. | Confirm the `{% render 'tracking-cart' %}` tag is in `layout/theme.liquid` just before `</body>`. |
| Console error: `Cannot read properties of null (reading 'items')` | `_kovaCartState` is null because the initial `/cart.js` fetch hasn't resolved yet when the first `/cart/change` call fires. | The code already guards with `if (window._kovaCartState && Array.isArray(...))`. The event is silently dropped. No action needed — this is acceptable behaviour per ADR-0011. |

## Architecture Notes

### Why `window._kovaCartState` instead of reading the response body

The `/cart/change` response contains the updated cart state, which could theoretically provide new item quantities. However, Dawn reads the response as `response.text()` (then `JSON.parse`) in its own `.then()` handler. A Response body can only be consumed once. We use `response.clone().json()` to get the post-change cart state for cache refresh. For the PRE-change state (needed to compute the delta), cloning only gives the post-change cart, so we maintain `_kovaCartState` as a pre-populated cache seeded at page load. See ADR-0011.

### Dual intercept chain on PDP pages

On product detail pages, both `tracking-product.liquid` (from the Custom Liquid Block) and `tracking-cart.liquid` (from theme.liquid) run simultaneously, patching `window.fetch` twice. The chain works correctly: each patch filters to its own URL pattern (`/cart/add` vs `/cart/change`) and passes through everything else. The outer patch calls the inner patch as `_origFetch`, which calls the real `fetch`. No events are doubled or lost.

### Endpoint and body format summary

| Endpoint | Fired by | Body format | Handled in |
|----------|----------|-------------|------------|
| `/cart/add` | Product form "Add to cart" button | FormData | `tracking-product.liquid` |
| `/cart/change` | Cart drawer / cart page `+` / `−` / remove | JSON string | `tracking-cart.liquid` |
| `/cart/add` (cache only) | Also intercepted in `tracking-cart.liquid` to keep `_kovaCartState` current | — | Cache refresh only, no event fired |

### ADR references

- **ADR-0009** (`docs/adr/0009-fetch-intercept-over-gtm-click-trigger.md`) — why fetch intercept is used instead of a GTM click trigger
- **ADR-0011** (`docs/adr/0011-cart-state-cache-for-delta-quantity.md`) — `window._kovaCartState` design, synchronous oldQty snapshot, `response.clone()` cache update
- **ADR-0012** (`docs/adr/0012-json-request-body-for-cart-change.md`) — JSON vs FormData body; `line` is a string, `parseInt` required
- **ADR-0013** (`docs/adr/0013-cart-change-fires-both-add-and-remove.md`) — `/cart/change` fires for both adds and removes; direction determined by delta

## Reusable Assets

- **Theme snippet:** `project-ecommerce/theme_export__fashion-sandbox-myshopify-com/snippets/tracking-cart.liquid`
- **Layout change:** `project-ecommerce/theme_export__fashion-sandbox-myshopify-com/layout/theme.liquid` (render tag before `</body>`)
- **ADR files:** `project-ecommerce/docs/adr/0011-*.md`, `0012-*.md`, `0013-*.md`

## Related Guides

- [2.8 — Add to Cart Tracking (`add_to_cart`)](./08-add-to-cart-tracking.md) — the fetch intercept pattern and `tracking-product.liquid` this builds on
- [2.6 — View Item Tracking (`view_item`)](./06-view-item-tracking.md) — `window._kovaProductData` pattern that parallels `window._kovaCartState`
