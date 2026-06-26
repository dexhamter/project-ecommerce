# ADR-0011: Cart State Cache (`window._kovaCartState`) for Delta Quantity in `remove_from_cart`

**Date:** 2026-06-26
**Status:** Accepted
**Subproject:** 2.9 — Remove from Cart Tracking

---

## Context

The `remove_from_cart` GA4 event requires a `quantity` parameter that represents how many units were **removed** — not the new remaining quantity. Dawn's `/cart/change` API endpoint receives the **new** quantity as input (e.g., going from 3 to 2 sends `quantity: 2`), not the delta. The delta must be computed as `old_quantity - new_quantity`.

The response from `/cart/change.js` does not include the previous quantity. It returns the updated cart state only. Additionally, we cannot read the response body in our fetch intercept because Dawn itself consumes the response as `.text()` (then `JSON.parse`), and a Response body can only be consumed once — reading it a second time would throw or return empty.

We need the pre-change quantity AND the item's metadata (title, price, variant, etc.) to populate the `items[]` array. All of this must be available synchronously in the `.then()` handler before we push to the dataLayer.

## Decision

Maintain `window._kovaCartState` as a local copy of the Shopify cart JSON, seeded by a single `fetch('/cart.js')` call on page load and updated manually after each successful `/cart/change` operation.

**Initialization:**
```javascript
fetch('/cart.js')
  .then(r => r.json())
  .then(cart => { window._kovaCartState = cart; });
```

**Usage in intercept:**
- Before the fetch request goes out: snapshot `window._kovaCartState.items[line - 1]` (the pre-change item at the requested line index).
- On successful response: update the cache in place — splice out the item if new `quantity === 0`, otherwise set `items[line - 1].quantity = newQty`.
- Compute delta: `preItem.quantity - newQty`.
- Fire `remove_from_cart` only if delta > 0.

## Alternatives Considered

| Option | Reason Rejected |
|--------|----------------|
| Read response body (`.clone().json()`) | Dawn consumes response as `.text()` on the SAME promise. Response body can only be consumed once. Cloning after consumption throws. |
| Call `/cart.js` before each change | Extra network round-trip per user interaction; introduces race conditions if user changes quantities rapidly. |
| Read quantity from DOM inputs | Fragile — DOM is updated by Dawn asynchronously, and matching a DOM input to a variant ID is error-prone. |
| Fire with `requestQty` as the removed quantity | Semantically wrong — `requestQty` is the **new** quantity, not the removed delta. |

## Consequences

**Benefits:**
- No extra network calls per cart interaction.
- All item data (title, price, variant) available without response parsing.
- Cache stays in sync because every cart change goes through the monkey-patched fetch.

**Known gaps:**
- If `/cart.js` fetch races with the first cart change (e.g., user removes an item immediately on page load), `preItem` will be null and the event is silently dropped. Acceptable tradeoff — a missed event is preferable to a corrupt one.
- Manual cache update (splice + quantity set) does not account for Shopify line-item merging or server-side reordering. In practice, merging only happens on `/cart/add`, not `/cart/change`, so the cache remains accurate for remove_from_cart scenarios.
- If a different `window.fetch` patch (e.g., from a third-party app) also patches the fetch chain, our `_origFetch` may point to that patch's version rather than the native fetch. This is an accepted risk of the monkey-patch pattern (documented in ADR-0009).
