# Request FormData quantity over response body quantity for add_to_cart

Shopify's `/cart/add.js` response returns the line item's quantity as it now exists in the cart — a running total, not the quantity just added. If a customer already has 2 units and adds 1 more, the response returns `quantity: 3`. Passing this to GA4 would overstate both quantity and value on the `add_to_cart` event and corrupt funnel and revenue attribution.

We read `quantity` from the request `FormData` (`init.body.get('quantity')`) instead, which is definitively what was submitted in this interaction. Dawn submits `/cart/add.js` as FormData (not JSON) with a top-level `quantity` key. Any value that is absent, zero, negative, or unparsable is treated as `1` — GA4 requires a positive integer and a `quantity: 0` event would be silently degraded out of Monetization reports. Reading `response.ok` (a top-level boolean) is sufficient to gate on success; we never need to consume the response body stream.
