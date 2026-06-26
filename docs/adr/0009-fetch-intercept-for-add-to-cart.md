# Fetch intercept over GTM click trigger for add_to_cart

Dawn's "Add to Cart" button fires a click event regardless of whether the cart add succeeds. An out-of-stock item, a network error, or a 422 from Shopify still triggers the button click — a GTM click trigger would fire a false `add_to_cart` event in all of these cases.

We intercept `window.fetch` instead, filtering for `/cart/add` calls and gating on `response.ok`. Dawn POSTs to `/cart/add` (not `/cart/add.js`) — the filter uses `indexOf('/cart/add')` to match both this endpoint and any `.js` variant. The event only fires when Shopify confirms the add with a 2xx response, which is precisely when the customer's intent has been fulfilled. The intercept is wrapped in an IIFE to scope `_origFetch` and avoid polluting the global namespace. Dawn's own fetch promise is returned untouched — the tracking code attaches a side-effect `.then()` on the same promise and does not interfere with Dawn's cart drawer update.
