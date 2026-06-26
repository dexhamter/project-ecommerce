# ADR-0013: `/cart/change` Intercept Fires Both `add_to_cart` and `remove_from_cart`

**Date:** 2026-06-26
**Status:** Accepted
**Subproject:** 2.9 — Remove from Cart Tracking

---

## Context

Dawn uses two distinct Shopify Cart API endpoints for cart operations:

| Endpoint | Trigger | Body format |
|----------|---------|-------------|
| `/cart/add` | Add new item from product form | FormData |
| `/cart/change` | Change quantity of existing item (from cart page or cart drawer) | JSON |

The `+` and `−` quantity buttons on the cart page and cart drawer both call `/cart/change` with the NEW quantity. This means:
- Clicking `+` on the cart page → `/cart/change` with quantity INCREASE
- Clicking `−` on the cart page → `/cart/change` with quantity DECREASE
- Clicking the trash icon → `/cart/change` with `quantity: 0`

GA4's `add_to_cart` spec covers **any** item addition to the cart, regardless of surface. A user increasing quantity from 1 to 3 from the cart page has added 2 units — that is an `add_to_cart` event with `quantity: 2`. Similarly, a decrease is a `remove_from_cart` event.

## Decision

The `/cart/change` intercept in `tracking-cart.liquid` fires `add_to_cart` OR `remove_from_cart` based on the direction of the quantity delta:

```
delta = requestQty - oldQty   (requestQty from request body, oldQty from _kovaCartState cache)

delta > 0  → add_to_cart     (quantity: delta,      value: price × delta)
delta < 0  → remove_from_cart (quantity: abs(delta), value: price × abs(delta))
delta = 0  → no-op, no event
```

`add_to_cart` from the product form (Dawn's "Add to cart" button → `/cart/add`) continues to be handled by `tracking-product.liquid`. This means:
- Adding from product page: handled by tracking-product.liquid via `/cart/add`
- Increasing quantity from cart page/drawer: handled by tracking-cart.liquid via `/cart/change`

Both correctly produce `add_to_cart` events per GA4 spec.

## Alternatives Considered

| Option | Reason Rejected |
|--------|----------------|
| Only fire `remove_from_cart` for `/cart/change` | Leaves a tracking gap: quantity increases from cart page/drawer are not measured as `add_to_cart`, understating add events |
| Use a GTM Click trigger instead of fetch intercept | Click triggers fire before the server confirms the change. A network error (e.g., out of stock) after a click would fire a false event. |
| Fire `add_to_cart` only from `/cart/add` | Correct for the product form, but misses all adds from the cart surface |

## Consequences

- Cart page / drawer adds and removes are fully measured.
- A user adding from the product form AND then increasing from the cart will fire TWO `add_to_cart` events — both correct.
- No duplicate events between tracking-product.liquid and tracking-cart.liquid: they intercept different endpoints (`/cart/add` vs `/cart/change`).
- `value` in both events reflects only the units affected (delta), not the total cart value.
