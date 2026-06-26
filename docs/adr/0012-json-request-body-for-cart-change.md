# ADR-0012: JSON Request Body for `/cart/change` (vs. FormData for `/cart/add`)

**Date:** 2026-06-26
**Status:** Accepted
**Subproject:** 2.9 — Remove from Cart Tracking

---

## Context

In 2.8 (`add_to_cart`), Dawn submits `/cart/add` as **FormData**, so quantity is read via:
```javascript
init.body instanceof FormData ? parseInt(init.body.get('quantity'), 10) : NaN
```

For 2.9 (`remove_from_cart`), Dawn submits `/cart/change` as **JSON** (verified in `assets/cart.js`, `updateQuantity()` method):
```javascript
const body = JSON.stringify({
  line,
  quantity,
  sections: sectionsToRender.map(s => s.section),
  sections_url: window.location.pathname,
});
fetch(`${routes.cart_change_url}`, { ...fetchConfig(), ...{ body } })
```

The two endpoints use different serialization formats. Using the wrong parser (e.g., `FormData.get()` on a JSON string body) returns null/NaN and breaks the intercept silently.

## Decision

Parse the `/cart/change` request body as **JSON**:
```javascript
var bodyObj   = JSON.parse(init.body);  // init.body is a JSON string
var line      = bodyObj.line;           // 1-indexed line item position
var quantity  = bodyObj.quantity;       // new quantity (not delta)
```

Guard against missing or malformed body with a try/catch and null checks on both fields.

## Endpoint Summary

| Endpoint | Method | Body Format | Quantity Field | Line Identifier |
|----------|--------|-------------|----------------|-----------------|
| `/cart/add` | POST | FormData | `quantity` (added qty) | N/A |
| `/cart/change` | POST | JSON string | `quantity` (NEW qty, number) | `line` (1-indexed, **string**) |

## Critical: `line` is a string, not a number

Observed in the wild (confirmed via console diagnostics 2026-06-26):

```json
{"line":"2","quantity":2,"sections":[...],"sections_url":"/cart"}
```

`line` is serialised as a **string** (`"2"`), not a number (`2`), even though the Shopify Cart API accepts both. A `typeof bodyObj.line === 'number'` guard silently fails for every request. Use `parseInt(bodyObj.line, 10)` and `parseInt(bodyObj.quantity, 10)` for both fields, guarding on `isNaN` rather than type.

## Consequences

- The `line` field (1-indexed) is used to look up the pre-change item in `window._kovaCartState.items[line - 1]` (0-indexed).
- `quantity` from the request is the **destination** quantity — not the delta. Delta is computed as `requestQty - oldQty` (positive = add, negative = remove).
- `requestQty` may legitimately be `0` (full removal) — guard on `isNaN` only, never on falsy.
