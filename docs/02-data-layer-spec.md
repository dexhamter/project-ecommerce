# 2.2 — Ecommerce Data Layer Specification

## What This Does & Why

The data layer spec is the contract between Shopify's theme code and GTM. It defines exactly what data is pushed, when, from which source, and in what format — for every ecommerce event in the funnel. Writing this spec before creating a single tag or snippet ensures that the implementation, QA, and any future debugging all reference the same ground truth.

This spec covers two distinct tracking environments:

- **Storefront** (`fashion-sandbox.myshopify.com`) — GTM via `theme.liquid`, data pushed via `dataLayer.push()` from Liquid-rendered snippets
- **Checkout** (`checkout.shopify.com`) — Shopify Custom Pixel, data pushed via `analytics.subscribe()` and forwarded to sGTM via `fetch()` POST

---

## Prerequisites

- [ ] GTM container created and `theme.liquid` snippet installed (Subproject 2.3)
- [ ] Stape.io sGTM container provisioned and endpoint URL recorded (Subproject 2.16)
- [ ] Custom Pixel created in Shopify Admin → Settings → Customer Events
- [ ] `read_customer_email` access scope granted to the Custom Pixel (required for Enhanced Conversions)
- [ ] `read_customer_phone` access scope granted (optional — for Enhanced Conversions phone field)
- [ ] Google & YouTube app NOT installed on the store

---

## Architecture Overview

```
STOREFRONT (fashion-sandbox.myshopify.com)
  Liquid snippets render product data into JS variables on page load
  JS event listeners capture user interactions
  dataLayer.push() → GTM Web Container → sGTM → GA4 + Google Ads

  Events: view_item_list · select_item · view_item
          add_to_cart · view_cart · remove_from_cart

CHECKOUT (checkout.shopify.com — sandboxed)
  Custom Pixel subscribes to Shopify native checkout events
  event.data.checkout object provides all product + order data
  fetch() POST → sGTM endpoint directly (bypasses client GTM)

  Events: begin_checkout · add_shipping_info
          add_payment_info · purchase
```

---

## Snippet File Architecture

All tracking code lives in dedicated Liquid snippet files. The snippet is injected into the relevant page via a **Custom Liquid block** in the Shopify Theme Editor — not by editing section files directly.

| Snippet File | Injected Into | Events Handled |
|---|---|---|
| `snippets/tracking-collection.liquid` | Default Collection template | `view_item_list` · `select_item` |
| `snippets/tracking-product.liquid` | Default Product template | `view_item` · `add_to_cart` |
| `snippets/tracking-cart.liquid` | Default Cart template + Header section | `view_cart` · `remove_from_cart` |

**Why Custom Liquid blocks:** blocks are stored in the template's JSON configuration, not the core section `.liquid` files. Theme updates do not overwrite them. The only post-update task is adding back the single `{% render 'tracking-X' %}` line if the block is lost.

**What to avoid:**
- ❌ Embedding tracking scripts directly in `sections/main-product.liquid`, `sections/main-collection-product-grid.liquid`, etc. — these get overwritten on theme updates
- ❌ Adding `data-` attributes to `snippets/card-product.liquid` for `select_item` — same theme update risk, and unnecessary given `ShopifyAnalytics.meta.products`

---

## Universal Rules

### Rule 1 — `ecommerce: null` before every push

Every ecommerce `dataLayer.push()` must be preceded by a null push:

```javascript
window.dataLayer = window.dataLayer || [];
dataLayer.push({ ecommerce: null }); // REQUIRED — clears previous ecommerce object
dataLayer.push({
  event: 'view_item',
  ecommerce: { ... }
});
```

Failure to include the null push causes GA4 to inherit parameters from the previous event, silently corrupting your data.

### Rule 2 — Price format

All price values must be numeric floats, never strings. Use Liquid's `| divided_by: 100.0` on Shopify's cent-based price fields, or read directly from `ShopifyAnalytics.meta` which already returns decimal format.

```liquid
{%- assign item_price = variant.price | divided_by: 100.0 -%}
```

```javascript
// In Custom Pixel: MoneyV2 objects — always target .amount
item.variant.price.amount  // ✅ returns 89.00
item.variant.price         // ❌ returns { amount: 89.00, currencyCode: "GBP" }
```

### Rule 3 — `| json` filter on all Liquid string outputs

Apply `| json` to any Liquid string value to escape quotes, apostrophes, and line breaks that would break the JavaScript payload.

```liquid
title: {{ product.title | json }},  // outputs: "Oversized Linen Jacket"
```

---

## Items Array Schema

All ecommerce events include an `items` array. The schema is identical across storefront and checkout events — the only difference is the data source.

| Parameter | Type | Storefront Source | Checkout Source |
|---|---|---|---|
| `item_id` | string | `variant.id` (Liquid) | `item.variant.id` |
| `item_name` | string | `product.title \| json` (Liquid) | `item.title` |
| `item_brand` | string | `"Kova Studio"` (hardcoded) | `"Kova Studio"` (hardcoded) |
| `item_category` | string | `product.type \| json` (Liquid) | `item.variant.product.type` |
| `item_variant` | string | `variant.title \| json` (Liquid) | `item.variant.title` |
| `price` | number | `variant.price \| divided_by: 100.0` | `item.variant.price.amount` |
| `quantity` | integer | `1` (default on storefront) or cart qty | `item.quantity` |

**Note on `item_category`:** Uses Shopify's Product Type field (set in Shopify Admin on each product), not the collection handle. This is the only category field accessible in both storefront Liquid and the checkout sandbox payload. Set meaningful product types in Shopify Admin (e.g., `"Jackets"`, `"Tops"`, `"Trousers"`) before implementing.

**Note on `item_list_id` / `item_list_name`:** These parameters are added only on `view_item_list` events. They do not belong on individual product or checkout events.

---

## Storefront Events

### Event 1 — `view_item_list`

**Trigger:** Collection page load  
**Snippet:** `snippets/tracking-collection.liquid`  
**Data source:** Liquid-rendered JSON — `{{ collection.products }}` loop

```liquid
{% comment %} snippets/tracking-collection.liquid {% endcomment %}
<script>
  window.dataLayer = window.dataLayer || [];

  var collectionItems = [];
  {% for product in collection.products %}
    {% for variant in product.variants limit: 1 %}
    collectionItems.push({
      item_id:       {{ variant.id | json }},
      item_name:     {{ product.title | json }},
      item_brand:    "Kova Studio",
      item_category: {{ product.type | json }},
      item_variant:  {{ variant.title | json }},
      price:         {{ variant.price | divided_by: 100.0 }},
      quantity:      1,
      index:         {{ forloop.index0 }},
      item_list_id:  {{ collection.handle | json }},
      item_list_name: {{ collection.title | json }}
    });
    {% endfor %}
  {% endfor %}

  dataLayer.push({ ecommerce: null });
  dataLayer.push({
    event: 'view_item_list',
    ecommerce: {
      currency:       "GBP",
      item_list_id:   {{ collection.handle | json }},
      item_list_name: {{ collection.title | json }},
      items:          collectionItems
    }
  });
</script>
```

**Note:** Only the first variant is pushed per product (`limit: 1`). The `index` field (0-based position in the list) is included for GA4 funnel position reporting.

---

### Event 2 — `select_item`

**Trigger:** User clicks a product card link on a collection page  
**Snippet:** `snippets/tracking-collection.liquid` (same file as `view_item_list`)  
**Data source:** `ShopifyAnalytics.meta.products` (Shopify auto-renders this array on all collection pages)

```javascript
// Inside snippets/tracking-collection.liquid — add after the view_item_list push

document.addEventListener('click', function(e) {
  var link = e.target.closest('a[href*="/products/"]');
  if (!link) return;

  // Extract clean handle — strips ?variant=, trailing slashes
  var hrefParts = link.getAttribute('href').split('/products/');
  if (hrefParts.length < 2) return;
  var handle = hrefParts[1].split('?')[0].split('/')[0].split('#')[0];

  // Look up in ShopifyAnalytics.meta.products
  var products = window.ShopifyAnalytics && window.ShopifyAnalytics.meta
                 ? window.ShopifyAnalytics.meta.products
                 : [];
  var product = products.find(function(p) { return p.handle === handle; });
  if (!product) return;

  var variant = product.variants[0]; // default variant

  dataLayer.push({ ecommerce: null });
  dataLayer.push({
    event: 'select_item',
    ecommerce: {
      currency:       "GBP",
      item_list_id:   {{ collection.handle | json }},
      item_list_name: {{ collection.title | json }},
      items: [{
        item_id:       String(variant.id),
        item_name:     product.title,
        item_brand:    "Kova Studio",
        item_category: product.type,
        item_variant:  variant.title,
        price:         variant.price / 100,
        quantity:      1
      }]
    }
  });
});
```

**Note:** `ShopifyAnalytics.meta.products` is auto-rendered by Shopify on collection pages — no custom Liquid required to populate it. Product price in this object is in cents; divide by 100.

---

### Event 3 — `view_item`

**Trigger:** Product detail page load  
**Snippet:** `snippets/tracking-product.liquid`  
**Data source:** Liquid-rendered JSON — `product` and `product.selected_or_first_available_variant` objects

```liquid
{% comment %} snippets/tracking-product.liquid {% endcomment %}
<script>
  window.dataLayer = window.dataLayer || [];

  {% assign current_variant = product.selected_or_first_available_variant %}

  dataLayer.push({ ecommerce: null });
  dataLayer.push({
    event: 'view_item',
    ecommerce: {
      currency: "GBP",
      value:    {{ current_variant.price | divided_by: 100.0 }},
      items: [{
        item_id:       {{ current_variant.id | json }},
        item_name:     {{ product.title | json }},
        item_brand:    "Kova Studio",
        item_category: {{ product.type | json }},
        item_variant:  {{ current_variant.title | json }},
        price:         {{ current_variant.price | divided_by: 100.0 }},
        quantity:      1
      }]
    }
  });
</script>
```

**Note:** `product.selected_or_first_available_variant` resolves to the variant in the URL (`?variant=123`) if present, otherwise defaults to the first available variant. This ensures the pushed item always matches what the customer is actually viewing.

---

### Event 4 — `add_to_cart`

**Trigger:** Successful response from Shopify's `/cart/add.js` AJAX endpoint (Dawn AJAX cart pattern)  
**Snippet:** `snippets/tracking-product.liquid` (same file as `view_item`)  
**Data source:** Intercept `/cart/add.js` fetch response; item data read from pre-rendered Liquid variables

```javascript
// Inside snippets/tracking-product.liquid — add after the view_item push

// Store Liquid-rendered product data for use in the fetch intercept
var _kovaProductData = {
  {% assign current_variant = product.selected_or_first_available_variant %}
  item_id:       {{ current_variant.id | json }},
  item_name:     {{ product.title | json }},
  item_brand:    "Kova Studio",
  item_category: {{ product.type | json }},
  item_variant:  {{ current_variant.title | json }},
  price:         {{ current_variant.price | divided_by: 100.0 }}
};

// Intercept /cart/add.js
var _originalFetch = window.fetch;
window.fetch = function() {
  var url = arguments[0];
  var args = arguments;

  if (typeof url === 'string' && url.includes('/cart/add')) {
    return _originalFetch.apply(this, args).then(function(response) {
      var cloned = response.clone();
      cloned.json().then(function(data) {
        var qty = data.quantity || 1;
        dataLayer.push({ ecommerce: null });
        dataLayer.push({
          event: 'add_to_cart',
          ecommerce: {
            currency: "GBP",
            value:    _kovaProductData.price * qty,
            items: [{
              item_id:       _kovaProductData.item_id,
              item_name:     _kovaProductData.item_name,
              item_brand:    _kovaProductData.item_brand,
              item_category: _kovaProductData.item_category,
              item_variant:  _kovaProductData.item_variant,
              price:         _kovaProductData.price,
              quantity:      qty
            }]
          }
        });
      });
      return response;
    });
  }
  return _originalFetch.apply(this, args);
};
```

**What to avoid:**
- ❌ Firing `add_to_cart` on button click before the AJAX response — the request may fail (out of stock race condition), resulting in a false event
- ❌ Relying on Shopify's native `product_added_to_cart` Web Pixel event — unreliable for AJAX cart drawers, Quick Add buttons, and Buy Now buttons which bypass Shopify's native form hooks

---

### Event 5 — `view_cart`

**Trigger:** Cart drawer open (`cart:open` Dawn event) OR `/cart` page load — whichever fires first; deduplicated via `sessionStorage`  
**Snippet:** `snippets/tracking-cart.liquid`  
**Data source:** Shopify Ajax Cart API response (`/cart.js`) or cart Liquid object on `/cart` page

```javascript
// snippets/tracking-cart.liquid

function pushViewCart(cartData) {
  var items = cartData.items.map(function(item) {
    return {
      item_id:       String(item.variant_id),
      item_name:     item.product_title,
      item_brand:    "Kova Studio",
      item_category: item.product_type,
      item_variant:  item.variant_title,
      price:         item.price / 100,
      quantity:      item.quantity
    };
  });

  dataLayer.push({ ecommerce: null });
  dataLayer.push({
    event: 'view_cart',
    ecommerce: {
      currency: "GBP",
      value:    cartData.total_price / 100,
      items:    items
    }
  });
}

// On cart drawer open (Dawn custom event)
document.addEventListener('cart:open', function() {
  if (sessionStorage.getItem('kova_view_cart_fired')) return;
  sessionStorage.setItem('kova_view_cart_fired', '1');

  fetch('/cart.js')
    .then(function(r) { return r.json(); })
    .then(pushViewCart);
});

// On /cart page load
if (window.location.pathname === '/cart') {
  if (!sessionStorage.getItem('kova_view_cart_fired')) {
    fetch('/cart.js')
      .then(function(r) { return r.json(); })
      .then(pushViewCart);
  }
  // Clear flag on full page load so next drawer open can fire again
  sessionStorage.removeItem('kova_view_cart_fired');
}
```

**Deduplication logic:** The flag `kova_view_cart_fired` is set when the drawer opens, preventing a double-fire if the user subsequently navigates to `/cart`. The flag is cleared on the cart page load itself so that future drawer opens in the same tab session fire normally.

---

### Event 6 — `remove_from_cart`

**Trigger:** Successful response from `/cart/change.js` where item quantity decreased (full removal or quantity decrease)  
**Snippet:** `snippets/tracking-cart.liquid`  
**Data source:** Diff of cart state before and after the `/cart/change.js` response

**Schema rule:** Push the **delta** quantity (how many were removed), not the remaining quantity. Push the **delta value** (delta qty × item price), not the remaining cart value.

```javascript
// Inside snippets/tracking-cart.liquid

// Snapshot current cart state before any change request
var _kovaCartSnapshot = null;

fetch('/cart.js')
  .then(function(r) { return r.json(); })
  .then(function(cart) { _kovaCartSnapshot = cart; });

// Intercept /cart/change.js
var _originalFetchCart = window.fetch;
window.fetch = function() {
  var url = arguments[0];
  var args = arguments;

  if (typeof url === 'string' && url.includes('/cart/change')) {
    var prevCart = _kovaCartSnapshot
                   ? JSON.parse(JSON.stringify(_kovaCartSnapshot))
                   : null;

    return _originalFetchCart.apply(this, args).then(function(response) {
      var cloned = response.clone();
      cloned.json().then(function(newCart) {
        _kovaCartSnapshot = newCart; // update snapshot

        if (!prevCart) return;

        prevCart.items.forEach(function(prevItem) {
          var newItem = newCart.items.find(function(i) {
            return i.variant_id === prevItem.variant_id;
          });
          var newQty = newItem ? newItem.quantity : 0;
          var delta  = prevItem.quantity - newQty;

          if (delta > 0) {
            var deltaValue = (prevItem.price / 100) * delta;
            dataLayer.push({ ecommerce: null });
            dataLayer.push({
              event: 'remove_from_cart',
              ecommerce: {
                currency: "GBP",
                value:    deltaValue,
                items: [{
                  item_id:       String(prevItem.variant_id),
                  item_name:     prevItem.product_title,
                  item_brand:    "Kova Studio",
                  item_category: prevItem.product_type,
                  item_variant:  prevItem.variant_title,
                  price:         prevItem.price / 100,
                  quantity:      delta  // DELTA — not new total
                }]
              }
            });
          }
        });
      });
      return response;
    });
  }
  return _originalFetchCart.apply(this, args);
};
```

**Two removal scenarios handled by this single listener:**
1. User clicks "Remove" → item quantity goes to 0 → delta = full previous quantity
2. User clicks "–" to decrease quantity → delta = 1 (or more if decremented by more than 1)

---

## Checkout Events (Custom Pixel)

All four checkout events are handled inside a single Custom Pixel subscribed to Shopify's native checkout events. The pixel formats the data as a GA4 Measurement Protocol payload and POSTs it directly to sGTM.

### Custom Pixel — Event Subscriptions

| GA4 Event | Shopify Event | `shippingLine` available? | `order.id` available? |
|---|---|---|---|
| `begin_checkout` | `checkout_started` | ❌ null | ❌ null |
| `add_shipping_info` | `checkout_shipping_info_submitted` | ✅ | ❌ null |
| `add_payment_info` | `payment_info_submitted` | ✅ | ❌ null |
| `purchase` | `checkout_completed` | ✅ | ✅ |

### Payload Data Paths

| Field | Checkout Payload Path | Notes |
|---|---|---|
| Order ID | `event.data.checkout.order.id` | Only available on `checkout_completed`; null on all others |
| Customer email | `event.data.checkout.email` | Requires `read_customer_email` scope |
| Customer phone | `event.data.checkout.phone` | Requires `read_customer_phone` scope |
| Line items array | `event.data.checkout.lineItems` | Available on all four events |
| Item variant ID | `item.variant.id` | Inside each line item |
| Item name | `item.title` | Inside each line item |
| Item variant title | `item.variant.title` | Inside each line item |
| Item product type | `item.variant.product.type` | Inside each line item — used for `item_category` |
| Item price | `item.variant.price.amount` | MoneyV2 — must target `.amount` |
| Item quantity | `item.quantity` | Inside each line item |
| Tax | `event.data.checkout.totalTax.amount` | MoneyV2 |
| Shipping cost | `event.data.checkout.shippingLine.price.amount` | MoneyV2; null until shipping selected |
| Shipping tier | `event.data.checkout.delivery.selectedDeliveryOptions[0].title` | e.g., `"Standard Shipping"` |
| Payment type | `event.data.checkout.transactions[0].paymentMethod.type` | e.g., `"creditCard"`, `"wallet"`, `"offsite"` |
| Subtotal value | `event.data.checkout.subtotalPrice.amount` | MoneyV2 |

### Items Array Helper Function

```javascript
function buildItems(lineItems) {
  return lineItems.map(function(item) {
    return {
      item_id:       item.variant.id,
      item_name:     item.title,
      item_brand:    "Kova Studio",
      item_category: item.variant.product.type,
      item_variant:  item.variant.title,
      price:         parseFloat(item.variant.price.amount),
      quantity:      item.quantity
    };
  });
}
```

---

### Checkout Event 1 — `begin_checkout`

**Shopify event:** `checkout_started`

```javascript
analytics.subscribe("checkout_started", function(event) {
  var checkout = event.data.checkout;
  var items    = buildItems(checkout.lineItems);
  var value    = parseFloat(checkout.subtotalPrice.amount);

  fetch('https://[YOUR-SGTM-ENDPOINT]/mp/collect', {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify({
      event_name: 'begin_checkout',
      currency:   'GBP',
      value:      value,
      items:      items
    })
  });
});
```

**Note:** `shippingLine` is null at this stage — do not include shipping cost. Use `subtotalPrice` (pre-shipping) for `value`.

---

### Checkout Event 2 — `add_shipping_info`

**Shopify event:** `checkout_shipping_info_submitted`

```javascript
analytics.subscribe("checkout_shipping_info_submitted", function(event) {
  var checkout      = event.data.checkout;
  var items         = buildItems(checkout.lineItems);
  var value         = parseFloat(checkout.subtotalPrice.amount);
  var deliveryOptions = checkout.delivery && checkout.delivery.selectedDeliveryOptions;
  var shippingTier  = deliveryOptions && deliveryOptions[0]
                      ? deliveryOptions[0].title
                      : undefined;

  fetch('https://[YOUR-SGTM-ENDPOINT]/mp/collect', {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify({
      event_name:    'add_shipping_info',
      currency:      'GBP',
      value:         value,
      shipping_tier: shippingTier,
      items:         items
    })
  });
});
```

---

### Checkout Event 3 — `add_payment_info`

**Shopify event:** `payment_info_submitted`

```javascript
analytics.subscribe("payment_info_submitted", function(event) {
  var checkout     = event.data.checkout;
  var items        = buildItems(checkout.lineItems);
  var value        = parseFloat(checkout.subtotalPrice.amount);
  var transactions = checkout.transactions || [];
  var paymentType  = transactions[0] && transactions[0].paymentMethod
                     ? transactions[0].paymentMethod.type
                     : undefined;

  fetch('https://[YOUR-SGTM-ENDPOINT]/mp/collect', {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify({
      event_name:   'add_payment_info',
      currency:     'GBP',
      value:        value,
      payment_type: paymentType,
      items:        items
    })
  });
});
```

---

### Checkout Event 4 — `purchase`

**Shopify event:** `checkout_completed`

```javascript
analytics.subscribe("checkout_completed", function(event) {
  var checkout       = event.data.checkout;
  var items          = buildItems(checkout.lineItems);
  var transactionId  = checkout.order ? String(checkout.order.id) : undefined;

  // Purchase deduplication — prevent double-fire on thank-you page refresh
  if (transactionId && sessionStorage.getItem('kova_purchase_' + transactionId)) return;
  if (transactionId) sessionStorage.setItem('kova_purchase_' + transactionId, '1');

  var value    = parseFloat(checkout.subtotalPrice.amount);
  var tax      = checkout.totalTax ? parseFloat(checkout.totalTax.amount) : 0;
  var shipping = checkout.shippingLine && checkout.shippingLine.price
                 ? parseFloat(checkout.shippingLine.price.amount)
                 : 0;

  fetch('https://[YOUR-SGTM-ENDPOINT]/mp/collect', {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify({
      event_name:     'purchase',
      currency:       'GBP',
      value:          value,
      transaction_id: transactionId,
      tax:            tax,
      shipping:       shipping,
      // Enhanced Conversions fields — requires read_customer_email scope
      user_data: {
        email:        checkout.email,
        phone_number: checkout.phone || undefined
      },
      items: items
    })
  });
});
```

**Deduplication note:** `sessionStorage` is scoped to the browser tab. The key `kova_purchase_[order_id]` is set on the first `checkout_completed` fire; any subsequent fires (page refresh, re-navigation) check for the key and exit early. This mirrors the same pattern used for `view_cart`.

**`transaction_id` format:** Shopify's long numeric order ID (e.g., `"5901234567304"`). The human-readable order name (`#1001`) is not exposed in the Web Pixels API sandbox — `order.id` is the only available identifier and is globally unique.

---

## sGTM Endpoint Placeholder

Replace `[YOUR-SGTM-ENDPOINT]` throughout the Custom Pixel with your Stape.io container URL once provisioned (Subproject 2.16). Format: `https://[container-id].stape.io`

---

## What to Avoid

| Mistake | Consequence | Correct Approach |
|---|---|---|
| Missing `ecommerce: null` before each push | Parameters bleed between events; GA4 silently corrupts data | Always push `{ ecommerce: null }` immediately before every ecommerce push |
| Firing `add_to_cart` on button click (before AJAX response) | False events when request fails (out of stock, network error) | Intercept `/cart/add.js` response; fire only on success |
| Using `product_added_to_cart` / `product_removed_from_cart` Web Pixel events | Massively underreported for AJAX carts — Quick Add, cart drawer, Buy Now all bypass native hooks | Intercept `/cart/add.js` and `/cart/change.js` directly |
| Loading GTM or `gtag.js` inside the Custom Pixel | Silent data loss; Enhanced Conversions, consent mode passthrough, and Preview Mode all fail | Use `fetch()` POST to sGTM from inside the Custom Pixel only |
| Storing data in `localStorage` in the Custom Pixel to retrieve storefront data | Custom Pixel runs in an isolated sandbox iframe — `localStorage` of the parent page is inaccessible | Use `item.variant.product.type` from the checkout payload for `item_category` |
| Using `data-` attributes on `snippets/card-product.liquid` for `select_item` | Attributes wiped on theme update | Use Option A: extract handle from `<a href>`, look up in `ShopifyAnalytics.meta.products` |
| Editing `sections/main-product.liquid` (or other section files) directly | Overwritten on theme update | Use dedicated snippet files + Custom Liquid blocks in Theme Editor |
| Passing `order.name` (`#1001`) as `transaction_id` | `order.name` is not exposed in the Web Pixels API sandbox | Use `event.data.checkout.order.id` (long numeric ID) |
| Passing MoneyV2 objects directly as price/value | Sends `{ amount: 89.00, currencyCode: "GBP" }` instead of `89.00` | Always target `.amount` on every MoneyV2 field |

---

## Event Summary Table

| Event | Environment | Trigger | Snippet / Pixel |
|---|---|---|---|
| `view_item_list` | Storefront | Collection page load | `tracking-collection.liquid` |
| `select_item` | Storefront | Product card click | `tracking-collection.liquid` |
| `view_item` | Storefront | Product page load | `tracking-product.liquid` |
| `add_to_cart` | Storefront | `/cart/add.js` success | `tracking-product.liquid` |
| `view_cart` | Storefront | Drawer open OR `/cart` load | `tracking-cart.liquid` |
| `remove_from_cart` | Storefront | `/cart/change.js` → qty decrease | `tracking-cart.liquid` |
| `begin_checkout` | Checkout (Pixel) | `checkout_started` | Custom Pixel |
| `add_shipping_info` | Checkout (Pixel) | `checkout_shipping_info_submitted` | Custom Pixel |
| `add_payment_info` | Checkout (Pixel) | `payment_info_submitted` | Custom Pixel |
| `purchase` | Checkout (Pixel) | `checkout_completed` | Custom Pixel |

---

## QA Checklist

### Snippet Files
- [ ] `snippets/tracking-collection.liquid` created
- [ ] `snippets/tracking-product.liquid` created
- [ ] `snippets/tracking-cart.liquid` created
- [ ] Each snippet injected via Custom Liquid block in Theme Editor (not via section file edits)
- [ ] Custom Liquid block positioned at top of each template to fire before user interaction

### Storefront Events (use GTM Preview Mode + GA4 DebugView)
- [ ] `view_item_list` fires on collection page load with correct `items` array
- [ ] `select_item` fires on product card click; `item_id` matches clicked product's variant ID
- [ ] `view_item` fires on product page load; variant-specific data updates on variant switch
- [ ] `add_to_cart` fires after successful cart AJAX response (not on click)
- [ ] `add_to_cart` does NOT fire when product is out of stock and AJAX fails
- [ ] `view_cart` fires on drawer open
- [ ] `view_cart` does NOT double-fire when drawer opens then user navigates to `/cart`
- [ ] `view_cart` fires on direct `/cart` page load (no drawer interaction)
- [ ] `remove_from_cart` fires on "Remove" link click; `quantity` = full removed quantity
- [ ] `remove_from_cart` fires on quantity decrease; `quantity` = delta (e.g., 1 when going from 3 → 2)
- [ ] `remove_from_cart` `value` = delta qty × price (not remaining cart value)
- [ ] No `ecommerce: null` missing — check GTM Preview Mode event log for parameter bleed

### Checkout Events (use Shopify Custom Pixel Preview + GA4 DebugView)
- [ ] `begin_checkout` fires on `checkout_started`; no `shipping` field in payload
- [ ] `add_shipping_info` fires on `checkout_shipping_info_submitted`; `shipping_tier` populated
- [ ] `add_payment_info` fires on `payment_info_submitted`; `payment_type` populated
- [ ] `purchase` fires on `checkout_completed`; `transaction_id` is the long numeric order ID
- [ ] `purchase` does NOT double-fire on thank-you page refresh (sessionStorage dedup)
- [ ] `user_data.email` present in `purchase` payload (verify `read_customer_email` scope granted)
- [ ] All MoneyV2 fields resolve to numbers, not objects
- [ ] `item_category` populated with product type (not empty/undefined)
- [ ] `shippingLine` gracefully handled as null/undefined on `begin_checkout`

---

## Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `view_item` pushes `undefined` for price | Liquid price is in cents but `divided_by: 100.0` not applied | Add `\| divided_by: 100.0` to all price Liquid filters |
| `select_item` fires but `item_id` is undefined | Handle not found in `ShopifyAnalytics.meta.products` | Check product is published and collection is not password-protected |
| `add_to_cart` fires twice | Fetch interceptor duplicated by snippet loading twice | Ensure `tracking-product.liquid` is only rendered once per page |
| `remove_from_cart` quantity is 0 | Delta calculated as negative (new qty > old qty) | Check diff direction: `delta = prevItem.quantity - newQty` |
| `purchase` not firing | `checkout_completed` not subscribed or `read_customer_email` scope missing | Verify Custom Pixel event subscriptions; check scope in Shopify Admin → Settings → Customer Events |
| `item_category` is empty in checkout events | Product Type not set in Shopify Admin | Set product type on each product in Shopify Admin → Products → [Product] → Product type |
| `payment_type` is undefined | `transactions` array empty at `payment_info_submitted` | Log `event.data.checkout.transactions` in pixel preview to confirm payload structure |
| sGTM not receiving checkout events | Incorrect sGTM endpoint URL in Custom Pixel | Confirm endpoint URL from Stape.io dashboard; check CORS headers |

---

## Related Guides

- `google-ads-measurement-library/docs/guides/02-data-collection/shopify-data-layer.md` *(written after this subproject)*
- `project-ecommerce/docs/01-measurement-plan.md` — items array schema and event taxonomy
- `project-ecommerce/docs/03-gtm-foundation.md` — GTM trigger and tag setup that reads these pushes
