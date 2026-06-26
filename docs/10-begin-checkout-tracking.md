# 2.10 — View Cart & Begin Checkout Tracking (`view_cart`, `begin_checkout`)

## What This Does & Why

This subproject instruments two adjacent funnel events: `view_cart` (user views their shopping cart) and `begin_checkout` (user clicks the checkout button and is about to leave the storefront). Together they complete the mid-funnel picture between `add_to_cart` (2.8/2.9) and `purchase` (2.11), enabling checkout funnel analysis in GA4 and Abandoned Cart audience targeting in Google Ads.

`begin_checkout` is the last event that fires on the storefront domain (`fashion-sandbox.myshopify.com`) before the browser navigates to `checkout.shopify.com`, where GTM cannot run. This makes it the critical handoff point: if it isn't captured here, the checkout abandonment signal is permanently lost. `view_cart` is implemented alongside it because both events share the same data source (`window._kovaCartState`) and the same file (`snippets/tracking-cart.liquid`).

## Prerequisites

- [x] GTM container `GTM-NMPNZ4TV` at `v2.5.0` minimum (GA4 Config, GA4 - Ecommerce Events, GAds - Remarketing regex trigger all present)
- [x] `snippets/tracking-cart.liquid` exists and is rendered globally via `layout/theme.liquid` (completed in 2.9)
- [x] `window._kovaCartState` cache seeding implemented in `tracking-cart.liquid` (completed in 2.9)
- [x] GA4 property `G-JPWF3JGT5P` — no additional configuration required
- [x] Google Ads account `AW-18244478477` — no new conversion action required for these events
- [x] Subprojects 2.1–2.9 complete

## Business Requirement

Capture the cart view and checkout initiation steps of the Kova Studio purchase funnel to enable GA4 funnel analysis and Google Ads Abandoned Cart / Checkout Initiator remarketing audiences.

---

## Data Layer Specification

### Event 1: `view_cart`

#### Event Name
`view_cart`

#### Event Parameters
| Parameter | Type | Example Value | Notes |
|-----------|------|---------------|-------|
| `currency` | string | `"GBP"` | Mandatory — GA4 silently drops revenue if missing |
| `value` | number | `334.93` | `cart.total_price / 100` — Shopify prices in pence |
| `items` | array | see below | All items currently in cart |

#### Items Array Schema
| Parameter | Type | Example Value | Source |
|-----------|------|---------------|--------|
| `item_id` | string | `"49500686057695"` | `String(item.variant_id)` from `/cart.js` |
| `item_group_id` | string | `"9389168066783"` | `String(item.product_id)` |
| `item_name` | string | `"Athletic Performance Tee"` | `item.product_title` |
| `item_brand` | string | `"House of Thread"` | `item.vendor` |
| `item_category` | string | `"Activewear"` | `item.product_type` |
| `item_variant` | string | `"S / Black"` | `item.variant_title` |
| `price` | number | `34.99` | `item.price / 100` |
| `quantity` | number | `1` | `item.quantity` |

#### Full dataLayer Code — `view_cart`

Two separate trigger paths in `tracking-cart.liquid`:

**Cart drawer** (fires when the drawer is opened):
```javascript
// Triggered by cart:view DOM event bubbling from cart-drawer-items.
// Dawn's CartDrawer.open() calls dispatchViewEvent() unconditionally.
document.addEventListener('cart:view', function() {
  if (window._kovaCartState && window._kovaCartState.items) {
    dataLayer.push({ ecommerce: null });
    dataLayer.push({
      event: 'view_cart',
      ecommerce: {
        currency: 'GBP',
        value:    window._kovaCartState.total_price / 100,
        items:    buildItems(window._kovaCartState)
      }
    });
  }
});
```

**Cart page** (fires on `/cart` page load):
```javascript
// CartItems.connectedCallback() auto-dispatch of cart:view requires
// view-event-payload attribute in Liquid — not reliably set. Use pathname
// check + direct fetch instead.
if (window.location.pathname === '/cart') {
  fetch('/cart.js')
    .then(function(r) { return r.json(); })
    .then(function(cart) {
      window._kovaCartState = cart;
      dataLayer.push({ ecommerce: null });
      dataLayer.push({
        event: 'view_cart',
        ecommerce: {
          currency: 'GBP',
          value:    cart.total_price / 100,
          items:    buildItems(cart)
        }
      });
    })
    .catch(function() {});
}
```

---

### Event 2: `begin_checkout`

#### Event Name
`begin_checkout`

#### Event Parameters
Same schema as `view_cart` — `currency`, `value`, `items[]` — all items in cart at moment of checkout click.

#### Full dataLayer Code — `begin_checkout`

```javascript
// Shared buildItems() helper used by both view_cart and begin_checkout.
function buildItems(cartData) {
  return cartData.items.map(function(item) {
    return {
      item_id:       String(item.variant_id),
      item_group_id: String(item.product_id),
      item_name:     item.product_title,
      item_brand:    item.vendor,
      item_category: item.product_type,
      item_variant:  item.variant_title,
      price:         item.price / 100,
      quantity:      item.quantity
    };
  });
}

// Inject name="checkout" hidden input before programmatic submit.
// form.submit() does not include the submitter button's name/value in the
// POST body. Shopify requires name="checkout" to route to checkout
// rather than returning /cart.
function submitToCheckout(form) {
  var input = document.createElement('input');
  input.type  = 'hidden';
  input.name  = 'checkout';
  input.value = '';
  form.appendChild(input);
  form.submit();
}

function pushBeginCheckout(cartData, form) {
  dataLayer.push({ ecommerce: null });
  dataLayer.push({
    event:         'begin_checkout',
    eventCallback: function() { submitToCheckout(form); },
    eventTimeout:  1000,
    ecommerce: {
      currency: 'GBP',
      value:    cartData.total_price / 100,
      items:    buildItems(cartData)
    }
  });
}

var checkoutFiring = false;

function handleCheckoutSubmit(e) {
  // Submitter guard: ignore other form buttons (future-proofing against
  // apps adding buttons to CartDrawer-Form or cart).
  if (e.submitter && e.submitter.name !== 'checkout') return;

  // Double-submit guard: e.preventDefault() keeps the button active
  // during the 1000ms GTM callback window — impatient users will click again.
  if (checkoutFiring) return;
  checkoutFiring = true;

  e.preventDefault();
  var form = e.target;

  if (window._kovaCartState && window._kovaCartState.items) {
    pushBeginCheckout(window._kovaCartState, form);
  } else {
    // Fallback fetch: user clicked checkout before page-load /cart.js resolved.
    // Race fetch against 1000ms timeout — if timeout wins, proceed without tracking.
    var raceTimeout = new Promise(function(resolve) {
      setTimeout(function() { resolve(null); }, 1000);
    });
    Promise.race([
      fetch('/cart.js').then(function(r) { return r.json(); }),
      raceTimeout
    ])
      .then(function(cartData) {
        if (cartData && cartData.items) {
          pushBeginCheckout(cartData, form);
        } else {
          submitToCheckout(form);
        }
      })
      .catch(function() { submitToCheckout(form); });
  }
}

// Attach to both checkout button surfaces.
var drawerForm = document.getElementById('CartDrawer-Form');
if (drawerForm) drawerForm.addEventListener('submit', handleCheckoutSubmit);

var cartForm = document.getElementById('cart');
if (cartForm) cartForm.addEventListener('submit', handleCheckoutSubmit);
```

---

## GTM Setup

No new tags or triggers were created in this subproject. The existing `GA4 - Ecommerce Events` (All Custom Events) tag covers `view_cart` and `begin_checkout` automatically. The only GTM change was updating the remarketing regex trigger to include `begin_checkout`.

[Screenshot: GTM workspace showing CE - Ecommerce Events (Remarketing) trigger with updated regex]

### Step-by-Step Instructions

1. Go to **Triggers** → find `CE - Ecommerce Events (Remarketing)`
2. Edit the trigger → find the **Event Name** regex field
3. Update the regex from:
   ```
   view_item_list$|select_item$|view_item$|add_to_cart$|view_cart$|remove_from_cart$|purchase$
   ```
   to:
   ```
   view_item_list$|select_item$|view_item$|add_to_cart$|view_cart$|remove_from_cart$|begin_checkout$|purchase$
   ```
4. Save → **Submit** → version name: `v2.6.0 - Begin Checkout & View Cart`

### Tag Configuration

No new tags. Both events fire through:

**Tag name:** `GA4 - Ecommerce Events`
**Tag type:** Google Analytics: GA4 Event
**Key settings:**
- Event Name: `{{Event}}`
- Send Ecommerce data: true
- Data source: dataLayer
- Trigger: All Custom Events (built-in)

**Tag name:** `GAds - Remarketing`
**Key settings:**
- Trigger: `CE - Ecommerce Events (Remarketing)` (updated regex)
- `begin_checkout` now included

### Trigger Configuration

**Trigger name:** `CE - Ecommerce Events (Remarketing)`
**Trigger type:** Custom Event
**Event name:** regex match
**Updated regex:**
```
view_item_list$|select_item$|view_item$|add_to_cart$|view_cart$|remove_from_cart$|begin_checkout$|purchase$
```

[Screenshot: Updated CE - Ecommerce Events (Remarketing) trigger showing new regex with begin_checkout$]

### Variable Configuration

N/A — no new variables created.

### GTM Version

**Version name:** `v2.6.0 - Begin Checkout & View Cart`
**Export:** `gtm/GTM-NMPNZ4TV_v2.6.0.json`

---

## Shopify Theme Changes

Both events are implemented in `snippets/tracking-cart.liquid`, which is already rendered globally via `layout/theme.liquid` (added in 2.9). No new snippet files or `theme.liquid` changes required.

[Screenshot: tracking-cart.liquid in Shopify theme editor or VS Code showing the 2.10 section]

### Dawn Checkout Button Architecture

Dawn's checkout buttons are native HTML form submit buttons — not JavaScript-driven:

- **Cart drawer:** `<button type="submit" name="checkout" form="CartDrawer-Form">` inside `<form action="{{ routes.cart_url }}" id="CartDrawer-Form" method="post">`
- **Cart page:** `<button type="submit" name="checkout" form="cart">` referencing the main cart form

When submitted with `name="checkout"` in the POST body, Shopify routes to checkout. When `form.submit()` is called programmatically, the button's name/value is omitted — Shopify returns the cart page instead. Fix: inject `<input type="hidden" name="checkout" value="">` before calling `form.submit()`.

### Why `e.preventDefault()` is Required

GTM processes tags **asynchronously** after `dataLayer.push()` returns. If the form submits immediately, the browser may begin tearing down the JS execution context before GTM has finished building the GA4 payload and calling `navigator.sendBeacon()`. In GTM Preview, this manifests as tags stuck in "Still Running".

The solution: prevent the default form submission, push `begin_checkout` with GTM's `eventCallback` + `eventTimeout: 1000`, then call `submitToCheckout(form)` from the callback. If GTM never calls back (ad-blocker, GTM failure), `eventTimeout` submits the form after 1000ms — the user is never permanently blocked. See ADR-0014.

---

## GA4 Configuration

No new GA4 configuration required. Both events use the GA4 native ecommerce schema and are automatically parsed into Monetization reports.

[Screenshot: GA4 DebugView showing view_cart event with correct parameters]
[Screenshot: GA4 DebugView showing begin_checkout event with correct parameters]

- **`view_cart` event name:** `view_cart` (GA4 recommended ecommerce event)
- **`begin_checkout` event name:** `begin_checkout` (GA4 recommended ecommerce event)
- **Marked as conversion:** No — these are funnel steps, not conversion actions
- **Custom dimensions registered:** None — all parameters are GA4 native ecommerce schema fields

---

## Google Ads Configuration

No new conversion actions created. `begin_checkout` is captured by the `GAds - Remarketing` tag for use in audience building (Abandoned Cart / Checkout Initiator segments). No direct conversion value is attributed to these events.

[Screenshot: Google Ads Audiences section showing remarketing tag active]

- **Conversion action:** N/A
- **Remarketing signal:** `begin_checkout` now included in `CE - Ecommerce Events (Remarketing)` regex trigger → feeds `GAds - Remarketing` tag

---

## Validation Steps

[Screenshot: GTM Preview event stream showing view_cart event (Event 17) on the cart page]

### view_cart — Cart Page
1. Open GTM Preview → paste `fashion-sandbox.myshopify.com` URL
2. Navigate to `/cart` (ensure at least one item is in cart)
3. Confirm `view_cart` appears in the event stream
4. Check Data Layer tab: `currency: "GBP"`, `value` matches cart total, `items[]` contains all cart items with correct fields
5. Confirm `GA4 - Ecommerce Events` tag shows Fired → Succeeded

[Screenshot: GTM Preview Data Layer tab showing view_cart payload with items array]

### view_cart — Cart Drawer
1. From any storefront page, click the cart icon to open the drawer
2. Confirm `view_cart` fires in the event stream
3. Check parameters as above

[Screenshot: GTM Preview showing view_cart firing on cart drawer open]

### begin_checkout — Cart Drawer
1. With GTM Preview active, open the cart drawer
2. Click the **Check out** button
3. Confirm `begin_checkout` fires (event appears before page navigates)
4. Check Data Layer tab: `eventCallback: "Function"`, `eventTimeout: 1000`, `currency`, `value`, `items[]`
5. Confirm both `GA4 - Ecommerce Events` and `GAds - Remarketing` show Fired → Succeeded
6. Confirm browser navigates to `checkout.shopify.com` (not back to `/cart`)

[Screenshot: GTM Preview event 21 showing begin_checkout with full data layer payload]
[Screenshot: GTM Preview Tags tab showing GA4 - Ecommerce Events and GAds - Remarketing both Succeeded]

### begin_checkout — Cart Page
1. Navigate to `/cart`
2. Click the **Check out** button
3. Same validation as cart drawer above

---

## QA Checklist

### view_cart
- [ ] Fires on `/cart` page load with correct items and value
- [ ] Fires when cart drawer is opened from any storefront page
- [ ] Does NOT fire when cart drawer is closed and re-opened on the same page (each open fires once — intended)
- [ ] Does NOT fire on pages without the cart drawer being opened
- [ ] `currency: "GBP"` present at top level
- [ ] `value` equals cart total (pence divided by 100)
- [ ] All items in cart appear in `items[]` with correct fields
- [ ] `GA4 - Ecommerce Events` tag Fired → Succeeded
- [ ] `GAds - Remarketing` tag Fired → Succeeded (via updated regex)

### begin_checkout
- [ ] Fires when clicking Check out from cart drawer
- [ ] Fires when clicking Check out from cart page (`/cart`)
- [ ] Browser navigates to `checkout.shopify.com` after the event (not back to `/cart`)
- [ ] Does NOT fire twice on double-click (checkoutFiring guard active)
- [ ] `currency: "GBP"` present at top level
- [ ] `value` equals cart total at moment of click
- [ ] All cart items in `items[]`
- [ ] `eventCallback: "Function"` and `eventTimeout: 1000` visible in dataLayer
- [ ] `GA4 - Ecommerce Events` tag Fired → Succeeded
- [ ] `GAds - Remarketing` tag Fired → Succeeded
- [ ] GTM version `v2.6.0` published and named correctly
- [ ] GTM container JSON exported to repo

---

## Common Errors & Fixes

| Error / Symptom | Root Cause | Fix |
|----------------|------------|-----|
| Clicking Check out returns to `/cart` instead of going to checkout | `form.submit()` doesn't include `name="checkout"` in POST body — Shopify treats it as a plain cart request | `submitToCheckout()` injects `<input type="hidden" name="checkout" value="">` before calling `form.submit()` |
| `begin_checkout` fires multiple times per checkout click | No double-submit guard; `e.preventDefault()` leaves button active during GTM callback delay | `checkoutFiring` boolean set on first fire, never reset (page navigates away) |
| `begin_checkout` event missing in GA4 (intermittent) | GTM async tag processing races with page navigation — beacon never queued before context destroyed | `e.preventDefault()` + `eventCallback` + `eventTimeout: 1000` ensures GTM completes before navigation |
| `view_cart` not firing on `/cart` page (only drawer) | Dawn's `CartItems.connectedCallback()` auto-dispatch of `cart:view` requires `view-event-payload` attribute in Liquid — not set by default | Pathname check + direct `fetch('/cart.js')` bypasses the event dependency entirely |
| `view_cart` firing with empty `items[]` | `_kovaCartState` not yet seeded when `cart:view` listener fires (async race) | Cart page path does its own `fetch('/cart.js')` independently; drawer path guards on `_kovaCartState && _kovaCartState.items` |
| `begin_checkout` missing for users who click checkout very quickly on page load | `_kovaCartState` null (page-load `/cart.js` fetch not resolved yet) | `Promise.race([fetch('/cart.js'), timeout(1000)])` fallback — fetches fresh cart data or times out gracefully |
| `GAds - Remarketing` not firing on `begin_checkout` | `begin_checkout` not in the `CE - Ecommerce Events (Remarketing)` regex trigger | Add `begin_checkout$` to regex, publish new GTM version |

---

## Reusable Assets

- **GTM Container Export:** `project-ecommerce/gtm/GTM-NMPNZ4TV_v2.6.0.json`
- **Snippet file:** `project-ecommerce/theme_export__fashion-sandbox-myshopify-com/snippets/tracking-cart.liquid`
- **ADR:** `project-ecommerce/docs/adr/0014-prevent-default-for-begin-checkout-timing.md`

---

## Related Guides

- [2.9 — Remove from Cart Tracking](./09-remove-from-cart-tracking.md) — implements `tracking-cart.liquid` and `window._kovaCartState`
- [2.8 — Add to Cart Tracking](./08-add-to-cart-tracking.md) — establishes `window._kovaProductData` and fetch intercept pattern
- 2.11 — Purchase Tracking (upcoming) — `purchase` event fires from Custom Pixel on `checkout.shopify.com`, not from Storefront
